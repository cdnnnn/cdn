@use '../../../styles/variables' as *;

.models-page {
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

  &__header-right {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 10px;
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

  &__btn {
    display: inline-flex;
    align-items: center;
    gap: 7px;
    font-family: $font-body;
    font-size: 0.8125rem;
    font-weight: 600;
    padding: 9px 14px;
    border-radius: 8px;
    border: 1px solid transparent;
    cursor: pointer;
    white-space: nowrap;
    transition: background 0.14s ease, border-color 0.14s ease, color 0.14s ease, opacity 0.14s ease;

    &:disabled {
      opacity: 0.6;
      cursor: not-allowed;
    }

    &--outline {
      background: $bg-main;
      border-color: $border-default;
      color: $text-secondary;

      &:hover:not(:disabled) {
        border-color: $text-primary;
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

  &__spin {
    animation: models-page-spin 0.9s linear infinite;
  }

  /* ---------- filters ---------- */
  &__filters {
    flex-shrink: 0;
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 12px;
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

  /* ---------- capability dropdown (structurally mirrors History's <Select />) ---------- */
  &__select {
    position: relative;
    flex-shrink: 0;
    width: 200px;
  }

  &__select-trigger {
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

  &__select-chevron {
    flex-shrink: 0;
    color: $text-tertiary;
    transition: transform 0.16s ease;
  }

  &__select-trigger--open &__select-chevron {
    transform: rotate(180deg);
  }

  &__select-menu {
    position: absolute;
    top: calc(100% + 6px);
    left: 0;
    right: 0;
    z-index: 20;
    max-height: 16rem;
    overflow-y: auto;
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 10px;
    box-shadow: $shadow-lg;
    padding: 5px;
    display: flex;
    flex-direction: column;
    gap: 1px;
  }

  &__select-option {
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

  /* ---------- tags ---------- */
  &__tag {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 4px;
    font-size: 0.6875rem;
    font-weight: 700;
    border-radius: 999px;
    padding: 3px 10px;

    &--blue {
      color: $primary;
      background: $primary-light;
    }

    &--jade {
      color: $success;
      background: $success-subtle;
    }

    &--gray {
      color: $text-tertiary;
      background: $bg-inset;
    }

    &--outline {
      color: $text-secondary;
      background: transparent;
      border: 1px solid $border-default;
    }
  }

  /* ---------- capability pills ---------- */
  &__caps {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
  }

  &__cap-pill {
    font-size: 0.71875rem;
    font-weight: 600;
    border-radius: 999px;
    padding: 3px 10px;

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
    border-radius: 0.75rem;
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
    align-items: flex-start;
    justify-content: space-between;
    gap: 10px;
    margin-bottom: 6px;
  }

  &__card-meta {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    margin-bottom: 10px;
  }

  &__card-name {
    font-size: 0.9375rem;
    font-weight: 700;
    letter-spacing: -0.01em;
    color: $text-primary;
  }

  &__card-desc {
    font-size: 0.78125rem;
    color: $text-secondary;
    line-height: 1.5;
    margin-bottom: 10px;
  }

  &__card-stats {
    display: flex;
    flex-direction: column;
    gap: 8px;
    margin-bottom: 10px;
  }

  &__card-stats-row {
    display: flex;
    flex-wrap: wrap;
    gap: 16px 20px;
  }

  &__card-stat-label {
    font-size: 0.625rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.04em;
    color: $text-tertiary;
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

  &__card-foot {
    margin-top: auto;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 10px;
  }

  &__card-foot-source {
    font-family: $font-mono;
    font-size: 0.6875rem;
    color: $text-tertiary;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  /* ---------- empty state ---------- */
  /* ---------- loading — plain, no border, just centers the spinner ---------- */
  &__loading {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 64px 20px;
  }

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

    &--error {
      border-style: solid;
      border-color: $danger-subtle;
      background: $danger-subtle;
      color: $danger;

      svg {
        color: $danger;
      }
    }
  }

  /* ---------- add-model modal ---------- */
  &__overlay {
    position: fixed;
    inset: 0;
    z-index: 200;
    background: rgba(0, 0, 0, 0.5);
  }

  &__modal {
    position: fixed;
    top: 0;
    right: 0;
    bottom: 0;
    width: 500px;
    max-width: 90vw;
    background: $bg-main;
    border-left: 1px solid $border-subtle;
    box-shadow: $shadow-lg;
    display: flex;
    flex-direction: column;
    overflow: hidden;
    animation: models-page-slide-in 0.24s cubic-bezier(0.16, 1, 0.3, 1);
  }

  &__modal-head {
    flex-shrink: 0;
    display: flex;
    align-items: flex-start;
    gap: 12px;
    padding: 20px 22px 16px;
    border-bottom: 1px solid $border-subtle;
  }

  &__modal-icon {
    flex-shrink: 0;
    width: 36px;
    height: 36px;
    border-radius: 10px;
    background: $primary-light;
    color: $primary;
    display: grid;
    place-items: center;
  }

  &__modal-head-text {
    flex: 1;
    min-width: 0;
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
    transition: border-color 0.14s ease, color 0.14s ease, opacity 0.14s ease;

    &:disabled {
      opacity: 0.5;
      cursor: not-allowed;
    }

    &:hover:not(:disabled) {
      border-color: $text-primary;
      color: $text-primary;
    }
  }

  /* ---------- form ---------- */
  &__form {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding: 22px 24px 24px;
    display: flex;
    flex-direction: column;
    gap: 12px;
  }

  &__form-section-label {
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 0.6875rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    color: $primary;
    margin-top: 6px;

    &:first-child {
      margin-top: 0;
    }

    svg {
      opacity: 0.85;
    }
  }

  &__form-row {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 12px;
  }

  &__field {
    display: flex;
    flex-direction: column;
    gap: 6px;
    min-width: 0;
  }

  &__field-label {
    display: flex;
    align-items: center;
    gap: 5px;
    font-size: 0.71875rem;
    font-weight: 600;
    color: $text-secondary;
  }

  &__input,
  &__textarea {
    width: 100%;
    border: 1px solid $border-default;
    border-radius: 8px;
    padding: 8px 11px;
    font-size: 0.8125rem;
    font-family: $font-body;
    color: $text-primary;
    background: $bg-main;
    transition: border-color 0.14s ease;

    &:focus {
      outline: none;
      border-color: $primary;
    }

    &:disabled {
      opacity: 0.6;
      cursor: not-allowed;
    }

    &::placeholder {
      color: $text-tertiary;
    }
  }

  &__textarea {
    resize: vertical;
    min-height: 4.5rem;
    line-height: 1.5;
    background: $bg-subtle;
    border-color: transparent;

    &:focus {
      background: $bg-main;
    }
  }

  &__input-wrap {
    display: flex;
    align-items: center;
    gap: 9px;
    border: 1px solid transparent;
    border-radius: 8px;
    padding: 0 11px;
    background: $bg-subtle;
    color: $text-tertiary;
    transition: background 0.14s ease, border-color 0.14s ease, color 0.14s ease;

    &:focus-within {
      background: $bg-main;
      border-color: $primary;
      color: $primary;
    }
  }

  &__input--inset {
    border: none;
    padding: 9px 0;
    background: transparent;
  }

  &__form-error {
    display: flex;
    align-items: flex-start;
    gap: 6px;
    font-size: 0.75rem;
    color: $danger;
    background: $danger-subtle;
    border-radius: 8px;
    padding: 9px 11px;
    line-height: 1.45;

    svg {
      flex-shrink: 0;
      margin-top: 1px;
    }
  }

  &__form-actions {
    position: sticky;
    bottom: 0;
    display: flex;
    justify-content: flex-end;
    gap: 8px;
    margin: 8px -24px -24px;
    padding: 16px 24px;
    background: $bg-main;
    border-top: 1px solid $border-subtle;

    .models-page__btn {
      min-width: 6.5rem;
    }
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

  @media (max-width: 640px) {
    &__card-stats-row {
      flex-wrap: wrap;
      gap: 14px;
    }

    &__form-row {
      grid-template-columns: 1fr;
    }
  }

  /* ---------- ultra-wide: nudge key text sizes up a touch ---------- */
  @media (min-width: 1800px) {
    &__title {
      font-size: 22px;
    }

    &__subtitle {
      font-size: 0.875rem;
    }

    &__card-name {
      font-size: 0.96875rem;
    }

    &__card-desc {
      font-size: 0.8125rem;
    }

    &__card-stat-value {
      font-size: 0.96875rem;
    }

    &__card-stat-value--sm {
      font-size: 0.8125rem;
    }

    &__cap-pill {
      font-size: 0.75rem;
    }

    &__card-stat-label {
      font-size: 0.65625rem;
    }
  }
}

@keyframes models-page-spin {
  to {
    transform: rotate(360deg);
  }
}

@keyframes models-page-slide-in {
  from {
    transform: translateX(100%);
  }
  to {
    transform: translateX(0);
  }
}
