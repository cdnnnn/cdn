@use '../../styles/variables' as *;

.page-placeholder {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 3px;
  padding-bottom: 18px;
  margin-bottom: 2px;

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

  &__box {
    margin-top: 16px;
    width: 100%;
    padding: 52px 20px;
    border: 1px dashed $border-strong;
    border-radius: 14px;
    background: $bg-subtle;
    color: $text-tertiary;
    font-size: 0.84375rem;
    font-weight: 600;
    text-align: center;
  }
}
