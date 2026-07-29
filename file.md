@use '../../styles/variables' as *;

$sidebar-width-collapsed: 68px;

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
  border-right: 1px solid $border-subtle;
  padding: 0.75rem 0.875rem 1.125rem;
  transition: width 0.18s ease;

  &--collapsed {
    width: $sidebar-width-collapsed;
    padding-left: 0.625rem;
    padding-right: 0.625rem;
  }

  &__top {
    flex: 1;
    display: flex;
    flex-direction: column;
    min-height: 0;
  }

  /* ---------- collapse toggle ---------- */
  &__collapse {
    display: flex;
    align-items: center;
    justify-content: space-between;
    width: 100%;
    padding: 0.375rem 0.5rem;
    margin-bottom: 0.75rem;
    border: none;
    border-radius: 0.5rem;
    background: transparent;
    color: $text-tertiary;
    font-size: 0.8125rem;
    font-weight: 600;
    cursor: pointer;
    transition: background 0.14s ease, color 0.14s ease;

    &:hover {
      background: $bg-subtle;
      color: $text-secondary;
    }
  }

  &--collapsed &__collapse {
    justify-content: center;
  }

  &__collapse-ic {
    transition: transform 0.18s ease;
    flex-shrink: 0;

    &--flipped {
      transform: rotate(180deg);
    }
  }

  /* ---------- nav sections ---------- */
  &__nav {
    display: flex;
    flex-direction: column;
    gap: 1.375rem;
    margin-top: 0.5rem;
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
    font-size: 0.71875rem;
    font-weight: 600;
    letter-spacing: 0.09em;
    text-transform: uppercase;
    color: $text-tertiary;
    padding: 0 0.625rem;
    margin-bottom: 0.375rem;
  }

  /* ---------- nav items ---------- */
  &__icon {
    display: grid;
    place-items: center;
    flex-shrink: 0;
    color: inherit;
  }

  &__item {
    display: flex;
    align-items: center;
    gap: 0.6875rem;
    padding: 0.5rem 0.625rem;
    border-radius: 0.5rem;
    font-size: 0.9375rem;
    font-weight: 500;
    color: $text-secondary;
    text-decoration: none;
    border: none;
    background: transparent;
    width: 100%;
    cursor: pointer;
    transition: background 0.14s ease, color 0.14s ease;

    svg {
      opacity: 0.75;
      color: $text-tertiary;
      transition: color 0.14s ease, opacity 0.14s ease;
    }

    &:hover {
      background: $bg-subtle;
      color: $text-primary;

      svg {
        opacity: 1;
        color: $text-secondary;
      }
    }

    &--active {
      background: $primary-light;
      color: $primary;
      font-weight: 600;

      svg {
        opacity: 1;
        color: $primary;
      }

      &:hover {
        background: $primary-light;
        color: $primary;
      }
    }

    &--button {
      text-align: left;
    }

    &--collapsed {
      justify-content: center;
      padding: 0.5rem;
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
  }
}
