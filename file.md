@use '../../../styles/variables' as *;

.providers-page {
  display: flex;
  flex-direction: column;
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

  /* ---------- filters ---------- */
  &__filters {
    flex-shrink: 0;
    display: flex;
  }

  &__search {
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

  /* ---------- avatar ---------- */
  &__avatar {
    flex-shrink: 0;
    width: 34px;
    height: 34px;
    border-radius: 10px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    font-size: 0.75rem;
    font-weight: 700;
    letter-spacing: -0.01em;

    &--blue {
      color: $primary;
      background: $primary-light;
    }

    &--violet {
      color: $violet;
      background: $violet-light;
    }

    &--amber {
      color: $warning;
      background: $warning-subtle;
    }

    &--jade {
      color: $success;
      background: $success-subtle;
    }

    &--rose {
      color: $danger;
      background: $danger-subtle;
    }
  }

  /* ---------- status tag ---------- */
  &__tag {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 4px;
    font-size: 0.6875rem;
    font-weight: 600;
    border-radius: 999px;
    padding: 3px 10px;

    &--jade {
      color: $success;
      background: $success-subtle;
    }

    &--gray {
      color: $text-tertiary;
      background: $bg-inset;
    }
  }

  /* ---------- full-info card grid ---------- */
  &__grid {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 16px;
  }

  &__card {
    display: flex;
    flex-direction: column;
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-left: 3px solid $card-accent;
    padding: 15px 18px;
    box-shadow: $shadow-xs;
    transition: box-shadow 0.15s ease, transform 0.15s ease;

    &:hover {
      box-shadow: $shadow-md;
      transform: translateY(-2px);
    }
  }

  &__card-top {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 10px;
    margin-bottom: 10px;
  }

  &__card-top-left {
    display: flex;
    align-items: center;
    gap: 10px;
    min-width: 0;
  }

  &__card-name {
    font-size: 0.9375rem;
    font-weight: 800;
    letter-spacing: -0.01em;
    color: $text-primary;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__card-desc {
    font-size: 0.78125rem;
    color: $text-secondary;
    line-height: 1.5;
    margin-bottom: 10px;
  }

  &__card-stats {
    display: flex;
    flex-wrap: wrap;
    gap: 16px 20px;
    margin-bottom: 10px;
    padding-bottom: 10px;
    border-bottom: 1px solid $border-subtle;
  }

  &__card-stat-label {
    display: flex;
    align-items: center;
    gap: 4px;
    font-size: 0.625rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.04em;
    color: $text-tertiary;

    svg {
      opacity: 0.8;
    }
  }

  &__card-stat-value {
    font-size: 0.9375rem;
    font-weight: 800;
    color: $text-primary;
    display: block;
    margin-top: 2px;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;

    &--sm {
      font-size: 0.78125rem;
      font-weight: 700;
    }

    &--accent {
      color: $success;
    }
  }

  &__card-section-label {
    font-size: 0.65625rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    color: $text-tertiary;
    margin-bottom: 6px;
    display: block;
  }

  /* ---------- models list (card preview + modal) ---------- */
  &__card-models {
    display: flex;
    flex-direction: column;
    gap: 4px;
    margin-bottom: 6px;
  }

  &__card-model-row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 10px;
    padding: 5px 0;
    font-size: 0.75rem;
  }

  &__card-model-name {
    color: $text-primary;
    font-weight: 600;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__card-model-version {
    font-family: $font-mono;
    font-size: 0.6875rem;
    font-weight: 400;
    color: $text-tertiary;
  }

  &__card-model-accuracy {
    flex-shrink: 0;
    font-weight: 700;
    color: $success;
  }

  &__card-view-all {
    display: inline-flex;
    align-items: center;
    font-family: $font-body;
    font-size: 0.71875rem;
    font-weight: 700;
    color: $primary;
    background: transparent;
    border: none;
    padding: 0;
    margin-bottom: 10px;
    cursor: pointer;

    &:hover {
      text-decoration: underline;
    }
  }

  /* ---------- card footer / actions ---------- */
  &__card-foot {
    margin-top: auto;
    padding-top: 10px;
    border-top: 1px solid $border-subtle;
    display: flex;
    gap: 8px;
  }

  &__btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 6px;
    font-family: $font-body;
    font-size: 0.78125rem;
    font-weight: 600;
    padding: 6px 12px;
    border-radius: 8px;
    border: 1px solid transparent;
    cursor: pointer;
    white-space: nowrap;
    transition: background 0.14s ease, border-color 0.14s ease, color 0.14s ease;

    &--outline {
      background: $bg-main;
      border-color: $border-default;
      color: $text-primary;

      &:hover {
        border-color: $text-primary;
      }
    }

    &--danger-outline {
      background: $bg-main;
      border-color: $border-default;
      color: $text-tertiary;

      &:hover {
        border-color: $danger;
        color: $danger;
        background: $danger-subtle;
      }
    }

    &--primary {
      background: $primary;
      border-color: $primary;
      color: $on-primary;

      &:hover {
        background: $primary-hover;
        border-color: $primary-hover;
      }
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

  /* ---------- modals (view-all-models + connect form) ---------- */
  &__overlay {
    position: fixed;
    inset: 0;
    z-index: 200;
    background: rgba(0, 0, 0, 0.45);
    backdrop-filter: blur(2px);
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 24px;
  }

  &__modal {
    width: 100%;
    max-width: 32rem;
    max-height: min(80vh, 40rem);
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 14px;
    box-shadow: $shadow-lg;
    display: flex;
    flex-direction: column;
    overflow: hidden;

    &--sm {
      max-width: 24rem;
      max-height: none;
    }
  }

  &__modal-head {
    flex-shrink: 0;
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 12px;
    padding: 20px 22px 16px;
    border-bottom: 1px solid $border-subtle;

    .providers-page__avatar {
      margin-bottom: 8px;
    }
  }

  &__modal-title {
    font-size: 1.0625rem;
    font-weight: 800;
    letter-spacing: -0.01em;
    color: $text-primary;
  }

  &__modal-sub {
    margin-top: 3px;
    font-size: 0.75rem;
    color: $text-tertiary;
  }

  &__modal-close {
    flex-shrink: 0;
    width: 30px;
    height: 30px;
    border-radius: 8px;
    border: 1px solid $border-default;
    background: $bg-main;
    color: $text-tertiary;
    display: grid;
    place-items: center;
    cursor: pointer;
    transition: border-color 0.14s ease, color 0.14s ease;

    &:hover {
      border-color: $text-primary;
      color: $text-primary;
    }
  }

  &__modal-body {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding: 8px 22px 22px;
    display: flex;
    flex-direction: column;
  }

  /* ---------- connect form (inside its own modal) ---------- */
  &__connect-form {
    padding: 20px 22px 22px;
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  &__field-label {
    font-size: 0.71875rem;
    font-weight: 600;
    color: $text-secondary;
  }

  &__input-wrap {
    display: flex;
    align-items: center;
    gap: 8px;
    border: 1px solid $border-default;
    border-radius: 8px;
    padding: 0 11px;
    background: $bg-main;
    color: $text-tertiary;
    transition: border-color 0.14s ease, box-shadow 0.14s ease;

    &:focus-within {
      border-color: $primary;
      box-shadow: 0 0 0 3px $primary-light;
    }
  }

  &__input {
    flex: 1;
    width: 100%;
    border: none;
    outline: none;
    padding: 8px 0;
    font-size: 0.8125rem;
    font-family: $font-body;
    color: $text-primary;
    background: transparent;

    &::placeholder {
      color: $text-tertiary;
    }
  }

  &__form-actions {
    display: flex;
    justify-content: flex-end;
    gap: 8px;
    margin-top: 6px;
  }

  /* ---------- responsive ---------- */
  @media (max-width: 1500px) {
    &__grid {
      grid-template-columns: repeat(2, 1fr);
    }
  }

  @media (max-width: 1000px) {
    &__grid {
      grid-template-columns: 1fr;
    }
  }

  /* ---------- ultra-wide: nudge key text sizes up a touch ---------- */
  @media (min-width: 1800px) {
    &__title {
      font-size: 23px;
    }

    &__subtitle {
      font-size: 0.90625rem;
    }

    &__card-name {
      font-size: 1.03125rem;
    }

    &__card-desc {
      font-size: 0.84375rem;
    }

    &__card-stat-value {
      font-size: 1.03125rem;
    }

    &__card-stat-value--sm {
      font-size: 0.84375rem;
    }

    &__card-stat-label {
      font-size: 0.6875rem;
    }

    &__card-section-label {
      font-size: 0.71875rem;
    }

    &__card-model-name {
      font-size: 0.8125rem;
    }
  }
}
