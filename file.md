@use '../../../styles/variables' as *;

.datasets-page {
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

      &:hover {
        background: $primary-hover;
        border-color: $primary-hover;
      }
    }
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

  &__seg {
    flex-shrink: 0;
    display: inline-flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 2px;
    padding: 3px;
    border: 1px solid $border-subtle;
    border-radius: 11px;
    background: $bg-subtle;
  }

  &__seg-item {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-family: $font-body;
    font-size: 0.75rem;
    font-weight: 600;
    color: $text-tertiary;
    background: transparent;
    border: none;
    border-radius: 8px;
    padding: 7px 12px;
    cursor: pointer;
    transition: background 0.14s ease, color 0.14s ease, box-shadow 0.14s ease;

    svg {
      opacity: 0.8;
    }

    &:hover {
      color: $text-primary;
    }

    &--active {
      background: $bg-main;
      color: $primary;
      box-shadow: $shadow-xs;

      svg {
        opacity: 1;
      }
    }
  }

  /* ---------- tags (matches History's &__type badge exactly) ---------- */
  &__tag {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 4px;
    padding: 2px 8px;
    background: $primary-light;
    color: $primary;
    border-radius: 4px;
    font-size: 0.75rem;
    font-weight: 500;
  }

  /* ---------- capability pills (matches History's &__type-badge scale) ---------- */
  &__caps {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
  }

  &__cap-pill {
    font-size: 0.625rem;
    font-weight: 700;
    border-radius: 999px;
    padding: 2px 8px;

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

  // Grid tracks default to a min-width equal to their content's min-content
  // size — a single long unbroken string (e.g. a huggingface_dataset id)
  // anywhere in one card can force that whole column wider than the others,
  // which is what was making cards look unequal width. min-width: 0 lets the
  // card shrink to the track's actual (equal) size instead of dictating it.
  &__card {
    display: flex;
    flex-direction: column;
    min-width: 0;
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-left: 3px solid $card-accent;
    border-radius: $radius-lg;
    padding: 14px 16px;
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
    gap: 10px;
    margin-bottom: 10px;
  }

  &__card-icon {
    width: 36px;
    height: 36px;
    flex-shrink: 0;
    background: $primary-light;
    border-radius: $radius-md;
    display: flex;
    align-items: center;
    justify-content: center;

    svg {
      color: $primary;
    }
  }

  &__card-heading {
    flex: 1;
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

  &__card-meta {
    display: flex;
    align-items: center;
    gap: 10px;
    font-size: 12px;
    color: $text-tertiary;
  }

  &__card-desc {
    font-size: 12px;
    color: $text-secondary;
    line-height: 1.5;
    margin-bottom: 10px;
    overflow-wrap: break-word;
  }

  /* ---------- stats + capability pills share one bordered box ---------- */
  &__card-stats {
    display: flex;
    flex-direction: column;
    gap: 8px;
    margin-bottom: 10px;
  }

  &__card-stats-row {
    display: flex;
    gap: 20px;
  }

  &__card-stat-label {
    display: block;
    font-size: 10px;
    text-transform: uppercase;
    letter-spacing: 0.4px;
    color: $text-tertiary;
    margin-bottom: 2px;
  }

  &__card-stat-value {
    font-size: 13px;
    font-weight: 500;
    color: $text-primary;
    display: block;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    max-width: 130px;

    &--sm {
      font-size: 12px;
    }
  }

  &__card-task {
    font-size: 12px;
    line-height: 1.5;
    margin-bottom: 4px;
    overflow-wrap: break-word;

    b {
      color: $text-primary;
      font-weight: 600;
    }

    span {
      color: $text-secondary;
    }
  }

  // Compact icon+count pill in the top-right corner of the card, opens the
  // full tasks modal — kept out of the main flow entirely instead of taking
  // its own row, which is what was adding extra height before.
  &__card-tasks-toggle {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 4px;
    font-family: $font-body;
    font-size: 11px;
    font-weight: 700;
    color: $text-secondary;
    background: $bg-subtle;
    border: 1px solid $border-subtle;
    border-radius: 999px;
    padding: 4px 8px;
    cursor: pointer;
    transition: background 0.14s ease, border-color 0.14s ease, color 0.14s ease;

    svg {
      color: $primary;
    }

    &:hover {
      background: $primary-light;
      border-color: $primary-subtle;
      color: $primary;
    }
  }

  &__card-foot {
    margin-top: auto;
  }

  &__card-foot-source {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    font-size: 12px;
    color: $text-tertiary;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;

    svg {
      flex-shrink: 0;
    }
  }

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

  &__spin {
    animation: datasets-page-spin 0.9s linear infinite;
  }

  /* ---------- tasks modal ---------- */
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
  }

  &__modal-head {
    flex-shrink: 0;
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 12px;
    padding: 20px 22px 16px;
    border-bottom: 1px solid $border-subtle;

    .datasets-page__tag {
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
    padding: 18px 22px 22px;
    display: flex;
    flex-direction: column;
    gap: 10px;
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
    &__card-stats {
      flex-wrap: wrap;
      gap: 14px;
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

    &__card-heading h4 {
      font-size: 15px;
    }

    &__card-meta {
      font-size: 13px;
    }

    &__card-desc {
      font-size: 13px;
    }

    &__card-stat-value {
      font-size: 14px;
    }

    &__card-stat-value--sm {
      font-size: 13px;
    }

    &__card-task {
      font-size: 13px;
    }

    &__cap-pill {
      font-size: 0.6875rem;
    }

    &__card-stat-label {
      font-size: 11px;
    }

    &__card-tasks-toggle {
      font-size: 12px;
    }
  }
}

@keyframes datasets-page-spin {
  to {
    transform: rotate(360deg);
  }
}
