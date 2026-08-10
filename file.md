@use '../../styles/_variables' as *;

.dash {
  &__header {
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    gap: 1rem;
    padding: 24px 32px 18px;
    margin-bottom: 24px;
    border-bottom: 1px solid $border-light;

    h1 {
      font-family: $font-display;
      font-size: 1.5rem;
      font-weight: 800;
      letter-spacing: -0.02em;
      color: $text-primary;
      line-height: 1.2;
    }
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
    color: $indigo;
    margin-bottom: 6px;

    &::before {
      content: '';
      width: 16px;
      height: 2px;
      border-radius: 2px;
      background: $indigo;
    }
  }

  &__header-sub {
    margin-top: 4px;
    font-size: 0.875rem;
    color: $text-secondary;
  }

  &__header-meta {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-size: 0.75rem;
    font-weight: 600;
    color: $text-secondary;
    background: $surface-alt;
    border: 1px solid $border-light;
    border-radius: 999px;
    padding: 7px 13px;
    white-space: nowrap;
    margin-bottom: 3px;
  }
}

.d-stats { display: grid; grid-template-columns: repeat(3, 1fr); gap: 16px; margin-bottom: 24px; }
.d-stat {
  background: $surface; border: 1px solid $border; border-radius: 16px; padding: 22px 24px; transition: all .2s;
}
.d-stat:hover { box-shadow: $shadow-2; transform: translateY(-2px); }
.d-stat-top { display: flex; justify-content: space-between; align-items: flex-start; }
.d-stat-label { font-size: 12px; color: $text-secondary; font-weight: 600; text-transform: uppercase; letter-spacing: .5px; }
.d-stat-val { font-family: $font-mono; font-size: 34px; font-weight: 700; letter-spacing: -1px; line-height: 1; margin-top: 8px; }
.d-stat-change { font-size: 12px; color: $emerald; font-weight: 600; margin-top: 6px; display: flex; align-items: center; gap: 4px; }

.dash {
  &__grid { display: grid; grid-template-columns: 1.2fr 1fr; gap: 16px; margin-bottom: 24px; }
  &__panel-hdr {
    padding: 20px 24px; border-bottom: 1px solid $border-light; display: flex; justify-content: space-between;
    align-items: center; font-family: $font-display; font-weight: 700; font-size: 15px;
  }
  &__run-row {
    padding: 16px 24px; border-bottom: 1px solid $border-light; display: flex; justify-content: space-between;
    align-items: center; transition: background .15s; cursor: pointer;
  }
  &__run-row:hover { background: $surface-hover; }
  &__spinner {
    width: 44px; height: 44px; border-radius: 50%; background: $indigo-pale;
    display: flex; align-items: center; justify-content: center;
  }
  &__empty { padding: 32px 24px; color: $text-secondary; font-size: 13px; }
  &__legend { display: flex; justify-content: center; gap: 20px; margin-top: 4px; flex-wrap: wrap; }
  &__legend span { display: flex; align-items: center; gap: 6px; font-size: 12px; font-weight: 600; color: $text-secondary; }
  &__dot { width: 8px; height: 8px; border-radius: 50%; display: inline-block; }
  &__actions { display: grid; grid-template-columns: repeat(4, 1fr); gap: 14px; }
}

// Note: .radar-wrap moved to src/styles/global.scss — it's shared with Comparison.

.qa-btn {
  display: flex; flex-direction: column; align-items: center; gap: 12px; padding: 24px 20px;
  border: 1px solid $border; border-radius: 16px; background: $surface; cursor: pointer; transition: all .25s;
  text-align: center;
}
.qa-btn:hover { transform: translateY(-4px); box-shadow: $shadow-3; border-color: transparent; }
.qa-btn__icon { width: 48px; height: 48px; border-radius: 14px; display: flex; align-items: center; justify-content: center; }
.qa-btn--ind .qa-btn__icon { background: $indigo-pale; color: $indigo; }
.qa-btn--em .qa-btn__icon { background: $emerald-pale; color: $emerald; }
.qa-btn--amb .qa-btn__icon { background: $amber-pale; color: $amber; }
.qa-btn--sky .qa-btn__icon { background: $sky-pale; color: $sky; }
.qa-btn__label { font-size: 14px; font-weight: 700; font-family: $font-display; }
.qa-btn__desc { font-size: 12px; color: $text-secondary; margin-top: 2px; }

@media (max-width: 768px) {
  .d-stats { grid-template-columns: repeat(2, 1fr); }
  .dash__grid { grid-template-columns: 1fr; }
  .dash__actions { grid-template-columns: 1fr 1fr; }
}
