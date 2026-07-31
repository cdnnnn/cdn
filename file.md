@import '../../styles/variables';

.page-placeholder {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 8px;
  padding: 8px 0;

  &__title {
    margin: 0;
    font-size: 20px;
    font-weight: 700;
    color: $text-primary;
    letter-spacing: -0.01em;
  }

  &__subtitle {
    margin: 0;
    font-size: 13.5px;
    color: $text-secondary;
  }

  &__box {
    margin-top: 12px;
    width: 100%;
    padding: 40px 24px;
    border: 1px dashed $border-default;
    border-radius: $radius-lg;
    background: $bg-subtle;
    color: $text-tertiary;
    font-size: 13px;
    font-weight: 600;
    text-align: center;
  }
}
