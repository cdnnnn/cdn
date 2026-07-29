@use '../../styles/variables' as *;

.app-header {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  z-index: $z-header;
  height: 60px;
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.75rem;
  padding: 0 1.125rem;
  background: rgba(255, 255, 255, 0.88);
  backdrop-filter: blur(0.75rem);
  border-bottom: 1px solid $border-subtle;

  &__brand {
    display: flex;
    align-items: center;
    gap: 0.625rem;
    text-decoration: none;
    color: $text-primary;
    flex-shrink: 0;
  }

  &__brand-mark {
    width: 32px;
    height: 32px;
    border-radius: 0.5rem;
    background: linear-gradient(155deg, $primary 0%, $primary-hover 100%);
    color: #fff;
    display: grid;
    place-items: center;
    box-shadow: 0 0.0625rem 0.125rem rgba(14, 21, 38, 0.06), inset 0 0 0 0.0625rem rgba(255, 255, 255, 0.14);
  }

  &__brand-name {
    font-family: $font-display;
    font-weight: 700;
    font-size: 1.0925rem;
    letter-spacing: -0.02em;
    white-space: nowrap;
  }

  &__user {
    position: relative;
    flex-shrink: 0;
  }

  &__avatar {
    width: 34px;
    height: 34px;
    border-radius: 50%;
    border: 1px solid $border-default;
    background: $primary-light;
    color: $primary;
    font-family: $font-display;
    font-size: 0.8125rem;
    font-weight: 700;
    letter-spacing: 0.01em;
    display: grid;
    place-items: center;
    cursor: pointer;
    transition: border-color 0.14s ease, box-shadow 0.14s ease;

    &:hover {
      border-color: $primary;
    }

    &--open,
    &:focus-visible {
      outline: none;
      border-color: $primary;
      box-shadow: 0 0 0 0.1875rem $primary-subtle;
    }
  }

  &__dropdown {
    position: absolute;
    top: calc(100% + 0.625rem);
    right: 0;
    width: 232px;
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 0.75rem;
    box-shadow: $shadow-lg;
    padding: 0.625rem;
    z-index: $z-header;
  }

  &__drop-user {
    display: flex;
    align-items: center;
    gap: 0.625rem;
    padding: 0.375rem 0.375rem 0.625rem;
  }

  &__drop-avatar {
    width: 36px;
    height: 36px;
    flex-shrink: 0;
    border-radius: 50%;
    background: $primary-light;
    color: $primary;
    font-family: $font-display;
    font-size: 0.84375rem;
    font-weight: 700;
    display: grid;
    place-items: center;
  }

  &__drop-info {
    min-width: 0;
  }

  &__drop-name {
    font-size: 0.90625rem;
    font-weight: 600;
    color: $text-primary;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  &__drop-role {
    font-size: 0.78125rem;
    color: $text-tertiary;
    margin-top: 0.125rem;
  }

  &__drop-divider {
    height: 1px;
    background: $border-subtle;
    margin: 0.125rem 0.375rem 0.5rem;
  }

  &__drop-item {
    display: flex;
    align-items: center;
    gap: 0.5625rem;
    width: 100%;
    padding: 0.5rem 0.625rem;
    border: none;
    border-radius: 0.5rem;
    background: transparent;
    color: $danger;
    font-size: 0.875rem;
    font-weight: 500;
    cursor: pointer;
    transition: background 0.14s ease;

    &:hover {
      background: $danger-subtle;
    }
  }

  @media (max-width: 620px) {
    &__brand-name {
      font-size: 1.0125rem;
    }
  }
}
