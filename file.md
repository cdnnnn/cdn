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

  /* ---------- master-detail split ---------- */
  &__split {
    flex: 1;
    min-height: 0;
    display: grid;
    grid-template-columns: 340px 1fr;
    border: 1px solid $border-default;
    border-radius: 14px;
    overflow: hidden;
    background: $bg-main;
  }

  /* ---------- left list ---------- */
  &__list {
    border-right: 1px solid $border-default;
    background: $bg-subtle;
    overflow-y: auto;
  }

  &__row {
    width: 100%;
    display: flex;
    flex-direction: column;
    gap: 4px;
    text-align: left;
    padding: 14px 18px;
    border: none;
    border-bottom: 1px solid $border-subtle;
    background: transparent;
    cursor: pointer;
    transition: background 0.14s ease;

    &:hover {
      background: $bg-inset;
    }

    &--active {
      background: $bg-main;
      border-left: 3px solid $primary;
      padding-left: 15px;

      &:hover {
        background: $bg-main;
      }
    }
  }

  &__row-date {
    font-family: $font-mono;
    font-size: 0.6875rem;
    font-weight: 600;
    color: $text-tertiary;
  }

  &__row-title {
    font-size: 0.8125rem;
    font-weight: 700;
    letter-spacing: -0.01em;
    line-height: 1.4;
    color: $text-primary;
    display: -webkit-box;
    -webkit-line-clamp: 2;
    -webkit-box-orient: vertical;
    overflow: hidden;
  }

  &__row-model {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    font-size: 0.71875rem;
    font-weight: 600;
    color: $primary;

    svg {
      color: $warning;
    }
  }

  /* ---------- right detail (reading pane) ---------- */
  &__detail {
    overflow-y: auto;
    padding: 32px 40px;
  }

  &__detail-head {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 16px;
    margin-bottom: 18px;
  }

  &__detail-head-info {
    display: flex;
    flex-direction: column;
    gap: 6px;
    min-width: 0;
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
    flex-shrink: 0;
  }

  &__btn {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-family: $font-body;
    font-size: 0.8125rem;
    font-weight: 600;
    padding: 8px 13px;
    border-radius: 7px;
    border: 1px solid $border-default;
    background: $bg-main;
    color: $text-primary;
    cursor: pointer;
    white-space: nowrap;
    transition: border-color 0.14s ease;

    &:hover {
      border-color: $text-primary;
    }
  }

  &__icon-btn {
    display: grid;
    place-items: center;
    width: 30px;
    height: 30px;
    flex-shrink: 0;
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
    margin: 0;
    font-size: 1.5rem;
    font-weight: 800;
    letter-spacing: -0.02em;
    line-height: 1.3;
    color: $text-primary;
  }

  &__summary {
    font-size: 0.9375rem;
    line-height: 1.7;
    color: $text-secondary;
    margin-bottom: 24px;
    max-width: 760px;
  }

  /* ---------- verdict box ---------- */
  &__verdict {
    display: flex;
    gap: 14px;
    padding: 18px 20px;
    background: $primary-light;
    border-radius: 12px;
    margin-bottom: 28px;
    max-width: 760px;

    strong {
      display: block;
      font-size: 0.8125rem;
      font-weight: 700;
      color: $primary;
      margin-bottom: 5px;
    }

    p {
      font-size: 0.875rem;
      line-height: 1.6;
      color: $text-primary;
      margin: 0;
    }
  }

  &__verdict-icon {
    flex-shrink: 0;
    width: 32px;
    height: 32px;
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
    gap: 12px;
    padding-top: 18px;
    border-top: 1px solid $border-subtle;
  }

  &__top-model {
    display: flex;
    align-items: baseline;
    gap: 6px;
  }

  &__top-label {
    font-size: 0.8125rem;
    color: $text-tertiary;
  }

  &__top-value {
    font-size: 0.9375rem;
    font-weight: 700;
    color: $text-primary;
  }

  &__metrics {
    display: flex;
    flex-wrap: wrap;
    gap: 7px;
  }

  &__metric-tag {
    font-size: 0.75rem;
    font-weight: 500;
    color: $text-secondary;
    background: $bg-subtle;
    border: 1px solid $border-subtle;
    border-radius: 6px;
    padding: 4px 10px;
  }

  &__empty {
    padding: 40px 24px;
    text-align: center;
    color: $text-tertiary;
    font-size: 0.84375rem;
  }

  /* ---------- responsive ---------- */
  @media (max-width: 800px) {
    &__split {
      grid-template-columns: 1fr;
    }

    &__list {
      display: none;
    }
  }
}
