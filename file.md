// DatabaseList.module.scss


@use '../../../styles/tokens' as t;

.db-analytics-db-panel {
  background: t.$surface-2;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  flex: 1;
  min-height: 0;
}

.db-analytics-db-panel__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 53px;
  padding: 0 16px;
  border-bottom: 1px solid t.$border-strong;
  background: t.$surface-1;
  flex-shrink: 0;
}

.db-analytics-db-panel__title {
  font-size: 14px;
  font-weight: 500;
  margin: 0;
  color: t.$text-primary;
}

.db-analytics-db-panel__header-actions {
  display: flex;
  align-items: center;
  gap: 6px;
}

.db-analytics-db-panel__icon-btn {
  width: 28px;
  height: 28px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border-radius: t.$radius;
  border: 0.5px solid t.$border-strong;
  background: transparent;
  color: t.$text-secondary;
  cursor: pointer;
  font-size: 15px;
  flex-shrink: 0;

  &:hover:not(:disabled) {
    background: t.$surface-0;
    border-color: t.$border-strong;
  }

  &:disabled {
    opacity: 0.45;
    cursor: not-allowed;
  }
}

.db-analytics-db-panel__body {
  padding: 8px;
  flex: 1;
  min-height: 0;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
}

.db-analytics-db-panel__empty {
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
  padding: 32px 16px;
  gap: 3px;

  p {
    font-size: 13px;
    font-weight: 600;
    color: t.$text-primary;
    margin: 4px 0 2px;
  }
}

.db-analytics-db-panel__empty-icon {
  width: 34px;
  height: 34px;
  border-radius: 50%;
  background: t.$surface-0;
  color: t.$text-muted;
  display: flex;
  align-items: center;
  justify-content: center;
}

.db-analytics-db-panel__link {
  background: none;
  border: none;
  padding: 0;
  color: t.$accent;
  font-size: 12px;
  font-weight: 600;
  font-family: inherit;
  cursor: pointer;

  &:hover:not(:disabled) {
    color: #185fa5;
    text-decoration: underline;
  }

  &:disabled {
    opacity: 0.55;
    cursor: not-allowed;
    text-decoration: none;
  }
}

.db-analytics-db-list {
  list-style: none;
  margin: 0;
  padding: 0;
  display: flex;
  flex-direction: column;
  gap: 3px;
}

.db-analytics-db-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 7px 8px;
  border-radius: t.$radius;
  transition: background 0.15s ease;

  &:hover {
    background: t.$surface-0;
  }
}

.db-analytics-db-item__icon {
  width: 30px;
  height: 30px;
  border-radius: 7px;
  background: t.$indigo-bg;
  color: t.$indigo-fg;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.db-analytics-db-item__icon--mysql {
  background: t.$blue-bg;
  color: t.$blue-fg;
}

.db-analytics-db-item__icon--postgres {
  background: t.$violet-bg;
  color: t.$violet-fg;
}

.db-analytics-db-item__icon--csv {
  background: t.$green-bg;
  color: t.$green-fg;
}

.db-analytics-db-item__info {
  display: flex;
  flex-direction: column;
  gap: 3px;
  min-width: 0;
  flex: 1;
}

.db-analytics-db-item__name {
  font-size: 12.5px;
  font-weight: 600;
  color: t.$text-primary;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.db-analytics-db-item__meta {
  display: flex;
  align-items: center;
  gap: 6px;
  min-width: 0;
}

.db-analytics-db-item__type-badge {
  font-size: 9px;
  font-weight: 700;
  letter-spacing: 0.02em;
  text-transform: uppercase;
  padding: 1px 5px;
  border-radius: 4px;
  flex-shrink: 0;
  color: t.$text-secondary;
  background: t.$surface-0;
  border: 0.5px solid t.$border;
}

.db-analytics-db-item__type-badge--mysql {
  color: t.$blue-fg;
  background: t.$blue-bg;
  border-color: transparent;
}

.db-analytics-db-item__type-badge--postgres {
  color: t.$violet-fg;
  background: t.$violet-bg;
  border-color: transparent;
}

.db-analytics-db-item__type-badge--csv {
  color: t.$green-fg;
  background: t.$green-bg;
  border-color: transparent;
}

.db-analytics-db-item__db-name {
  font-size: 10.5px;
  color: t.$text-muted;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.db-analytics-db-status-dot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  flex-shrink: 0;
  box-shadow: 0 0 0 2px t.$surface-2;
}

.db-analytics-db-status-dot--connected {
  background: t.$success;
}

.db-analytics-db-status-dot--disconnected {
  background: t.$danger;
}











// DatabaseManagerSlider.module.scss


@use '../../../styles/tokens' as t;

.db-analytics-slider-overlay {
  position: fixed;
  inset: 0;
  z-index: 100;
  display: flex;
  justify-content: flex-end;
}

.db-analytics-slider-overlay__backdrop {
  position: absolute;
  inset: 0;
  background: rgba(11, 11, 11, 0.32);
  animation: db-analytics-fade-in 0.15s ease;
}

.db-analytics-slider-panel {
  position: relative;
  width: 600px;
  max-width: 100vw;
  height: 100%;
  background: t.$surface-2;
  display: flex;
  flex-direction: column;
  box-shadow: -8px 0 24px rgba(11, 11, 11, 0.12);
  animation: db-analytics-slide-in-right 0.22s cubic-bezier(0.16, 1, 0.3, 1);
}

.db-analytics-slider-panel__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 60px;
  padding: 0 20px;
  border-bottom: 1px solid t.$border-strong;
  background: t.$surface-1;
  flex-shrink: 0;
}

.db-analytics-slider-panel__title {
  font-size: 16px;
  font-weight: 500;
  margin: 0;
}

.db-analytics-slider-panel__close-btn {
  width: 28px;
  height: 28px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border-radius: t.$radius;
  border: 0.5px solid t.$border-strong;
  background: transparent;
  color: t.$text-secondary;
  cursor: pointer;
  font-size: 15px;
  flex-shrink: 0;

  &:hover {
    background: t.$surface-0;
  }
}

.db-analytics-slider-panel__body {
  flex: 1;
  overflow-y: auto;
  padding: 20px;
}

@keyframes db-analytics-slide-in-right {
  from {
    transform: translateX(100%);
  }
  to {
    transform: translateX(0);
  }
}

@keyframes db-analytics-fade-in {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

.db-analytics-slider-tabs {
  display: flex;
  gap: 4px;
  padding: 12px 20px 0;
  border-bottom: 1px solid t.$border-strong;
  background: t.$surface-1;
  flex-shrink: 0;
}

.db-analytics-slider-tabs__item {
  display: flex;
  align-items: center;
  gap: 6px;
  background: none;
  border: none;
  font-family: inherit;
  font-size: 13px;
  font-weight: 500;
  color: t.$text-muted;
  padding: 10px 14px;
  cursor: pointer;
  border-bottom: 2px solid transparent;
  margin-bottom: -1px;

  &:hover {
    color: t.$text-primary;
  }
}

.db-analytics-slider-tabs__item--active {
  color: t.$text-primary;
  border-bottom-color: t.$text-primary;
}

.db-analytics-db-manage-list {
  list-style: none;
  margin: 0;
  padding: 0;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.db-analytics-db-manage-item {
  padding: 16px;
  border: 1px solid t.$border-strong;
  border-radius: t.$radius-lg;
  background: t.$surface-2;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;

  &:hover {
    border-color: t.$border-stronger;
    box-shadow: 0 2px 8px rgba(11, 11, 11, 0.05);
  }
}

.db-analytics-db-manage-item__top {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 12px;
}

.db-analytics-db-manage-item__identity {
  display: flex;
  align-items: center;
  gap: 12px;
  min-width: 0;
}

.db-analytics-db-manage-item__icon {
  width: 38px;
  height: 38px;
  border-radius: t.$radius;
  background: t.$indigo-bg;
  color: t.$indigo-fg;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.db-analytics-db-manage-item__icon--mysql {
  background: t.$blue-bg;
  color: t.$blue-fg;
}

.db-analytics-db-manage-item__icon--postgres {
  background: t.$violet-bg;
  color: t.$violet-fg;
}

.db-analytics-db-manage-item__icon--csv {
  background: t.$green-bg;
  color: t.$green-fg;
}

.db-analytics-db-manage-item__name-block {
  display: flex;
  align-items: center;
  gap: 8px;
  min-width: 0;
}

.db-analytics-db-manage-item__name {
  font-size: 14.5px;
  font-weight: 600;
  color: t.$text-primary;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.db-analytics-db-manage-item__type-badge {
  font-size: 10px;
  font-weight: 600;
  letter-spacing: 0.02em;
  text-transform: uppercase;
  color: t.$text-secondary;
  background: t.$surface-0;
  border: 0.5px solid t.$border;
  padding: 2px 7px;
  border-radius: 999px;
  flex-shrink: 0;
}

.db-analytics-db-manage-item__type-badge--mysql {
  color: t.$blue-fg;
  background: t.$blue-bg;
  border-color: transparent;
}

.db-analytics-db-manage-item__type-badge--postgres {
  color: t.$violet-fg;
  background: t.$violet-bg;
  border-color: transparent;
}

.db-analytics-db-manage-item__type-badge--csv {
  color: t.$green-fg;
  background: t.$green-bg;
  border-color: transparent;
}

.db-analytics-db-manage-item__meta-grid {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 12px;
  margin: 14px 0 0;
  padding: 12px 14px;
  background: t.$surface-0;
  border-radius: t.$radius;
}

.db-analytics-db-manage-item__meta-cell {
  min-width: 0;

  dt {
    font-size: 10.5px;
    font-weight: 600;
    letter-spacing: 0.02em;
    text-transform: uppercase;
    color: t.$text-muted;
    margin: 0 0 3px;
  }

  dd {
    font-size: 12.5px;
    color: t.$text-primary;
    margin: 0;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
  }
}

.db-analytics-db-manage-item__footer {
  display: flex;
  justify-content: flex-end;
  margin-top: 14px;
  padding-top: 12px;
  border-top: 0.5px solid t.$border;
}

.db-analytics-db-manage-empty {
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
  padding: 44px 20px;
  border: 1px dashed t.$border-strong;
  border-radius: t.$radius-lg;
  background: t.$surface-0;
}

.db-analytics-db-manage-empty__icon {
  width: 44px;
  height: 44px;
  border-radius: 50%;
  background: t.$accent-bg;
  color: #0c447c;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: 12px;
}

.db-analytics-db-manage-empty__title {
  font-size: 13.5px;
  font-weight: 600;
  color: t.$text-primary;
  margin: 0 0 4px;
}

.db-analytics-db-manage-empty__desc {
  font-size: 12px;
  color: t.$text-muted;
  margin: 0;
  max-width: 280px;
}

.db-analytics-db-status {
  font-size: 11px;
  font-weight: 500;
  display: flex;
  align-items: center;
  gap: 5px;
  white-space: nowrap;
  flex-shrink: 0;
}

.db-analytics-db-status__dot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
}

.db-analytics-db-status--connected {
  color: t.$status-connected-fg;

  .db-analytics-db-status__dot {
    background: t.$success;
  }
}

.db-analytics-db-status--disconnected {
  color: t.$status-disconnected-fg;

  .db-analytics-db-status__dot {
    background: t.$danger;
  }
}

.db-analytics-form-card {
  border: 1px solid t.$border-strong;
  border-radius: t.$radius-lg;
  background: t.$surface-2;
  padding: 18px;
}

.db-analytics-form-card__header {
  display: flex;
  align-items: flex-start;
  gap: 12px;
  padding-bottom: 16px;
  margin-bottom: 18px;
  border-bottom: 1px solid t.$border;
}

.db-analytics-form-card__icon {
  width: 34px;
  height: 34px;
  border-radius: t.$radius;
  background: t.$accent-bg;
  color: #0c447c;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.db-analytics-form-card__icon--csv {
  background: #e3f5ea;
  color: #0a7a41;
}

.db-analytics-form-card__title {
  font-size: 14px;
  font-weight: 600;
  color: t.$text-primary;
  margin: 0 0 3px;
}

.db-analytics-form-card__subtitle {
  font-size: 12px;
  color: t.$text-muted;
  margin: 0;
  line-height: 1.5;
}

.db-analytics-file-drop {
  display: flex;
  align-items: center;
  gap: 12px;
  width: 100%;
  padding: 14px;
  border: 1.5px dashed t.$border-strong;
  border-radius: t.$radius;
  background: t.$surface-0;
  cursor: pointer;
  font-family: inherit;
  text-align: left;
  transition: border-color 0.15s ease, background 0.15s ease;

  &:hover {
    border-color: t.$accent;
    background: t.$accent-bg;
  }
}

.db-analytics-file-drop--filled {
  border-style: solid;
  border-color: rgba(10, 122, 65, 0.35);
  background: #e3f5ea;

  &:hover {
    border-color: rgba(10, 122, 65, 0.5);
    background: #d7f0e1;
  }
}

.db-analytics-file-drop__icon {
  width: 34px;
  height: 34px;
  border-radius: t.$radius;
  background: t.$surface-2;
  color: t.$text-muted;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.db-analytics-file-drop--filled .db-analytics-file-drop__icon {
  background: #fff;
  color: #0a7a41;
}

.db-analytics-file-drop__text {
  display: flex;
  flex-direction: column;
  gap: 2px;
  min-width: 0;
}

.db-analytics-file-drop__title {
  font-size: 13px;
  font-weight: 600;
  color: t.$text-primary;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.db-analytics-file-drop__subtitle {
  font-size: 11.5px;
  color: t.$text-muted;
}

.db-analytics-file-drop__input {
  // Visually hidden but still accessible/functional — the styled button
  // above triggers it via ref, so the native input itself stays off-screen.
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}

.db-analytics-form {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.db-analytics-form__row {
  display: flex;
  gap: 12px;
}

.db-analytics-form__field {
  display: flex;
  flex-direction: column;
  gap: 6px;
  flex: 1;
}

.db-analytics-form__field--narrow {
  flex: 0 0 120px;
}

.db-analytics-form__label {
  font-size: 12px;
  font-weight: 500;
  color: t.$text-secondary;
}

.db-analytics-form__input {
  font-family: inherit;
  font-size: 13px;
  padding: 9px 11px;
  border: 1px solid t.$border-strong;
  border-radius: t.$radius;
  background: t.$surface-1;
  color: t.$text-primary;

  &:focus {
    outline: none;
    border-color: t.$accent;
    box-shadow: 0 0 0 2px rgba(42, 120, 214, 0.15);
  }
}

.db-analytics-form__hint {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 12px;
  color: t.$text-muted;
  margin: 0;
}

.db-analytics-form__error {
  font-size: 12px;
  color: #791f1f;
  background: #fbe9e9;
  padding: 8px 11px;
  border-radius: t.$radius;
}

.db-analytics-form__actions {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding-top: 8px;
  border-top: 1px solid t.$border;
}

.db-analytics-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
  font-size: 13px;
  font-weight: 500;
  font-family: inherit;
  border-radius: t.$radius;
  padding: 8px 16px;
  cursor: pointer;
  border: 1px solid transparent;
  transition: filter 0.15s ease, background 0.15s ease, border-color 0.15s ease;

  &:disabled {
    opacity: 0.6;
    cursor: not-allowed;
  }
}

.db-analytics-btn--primary {
  background: t.$gradient-primary;
  color: #fff;
  box-shadow: 0 1px 2px rgba(79, 70, 229, 0.25);

  &:hover:not(:disabled) {
    filter: brightness(1.08);
    box-shadow: 0 2px 6px rgba(79, 70, 229, 0.35);
  }

  &:active:not(:disabled) {
    filter: brightness(0.96);
  }
}

.db-analytics-btn--ghost {
  background: transparent;
  border-color: t.$border-strong;
  color: t.$text-primary;

  &:hover:not(:disabled) {
    background: t.$surface-0;
  }
}

.db-analytics-btn--sm {
  padding: 5px 11px;
  font-size: 12px;
}












// NewSessionSlider.module.scss


@use '../../../styles/tokens' as t;

.db-analytics-slider-overlay {
  position: fixed;
  inset: 0;
  z-index: 100;
  display: flex;
  justify-content: flex-end;
}

.db-analytics-slider-overlay__backdrop {
  position: absolute;
  inset: 0;
  background: rgba(11, 11, 11, 0.32);
  animation: db-analytics-session-fade-in 0.15s ease;
}

.db-analytics-slider-panel {
  position: relative;
  width: 600px;
  max-width: 100vw;
  height: 100%;
  background: t.$surface-2;
  display: flex;
  flex-direction: column;
  box-shadow: -8px 0 24px rgba(11, 11, 11, 0.12);
  animation: db-analytics-session-slide-in 0.22s cubic-bezier(0.16, 1, 0.3, 1);
}

.db-analytics-slider-panel__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 60px;
  padding: 0 20px;
  border-bottom: 1px solid t.$border-strong;
  background: t.$surface-1;
  flex-shrink: 0;
}

.db-analytics-slider-panel__title {
  font-size: 16px;
  font-weight: 500;
  margin: 0;
}

.db-analytics-slider-panel__close-btn {
  width: 28px;
  height: 28px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border-radius: t.$radius;
  border: 0.5px solid t.$border-strong;
  background: transparent;
  color: t.$text-secondary;
  cursor: pointer;
  font-size: 15px;
  flex-shrink: 0;

  &:hover {
    background: t.$surface-0;
  }
}

.db-analytics-slider-panel__body {
  flex: 1;
  overflow-y: auto;
  padding: 20px;
}

@keyframes db-analytics-session-slide-in {
  from {
    transform: translateX(100%);
  }
  to {
    transform: translateX(0);
  }
}

@keyframes db-analytics-session-fade-in {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

.db-analytics-form {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.db-analytics-form__field {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.db-analytics-form__label {
  font-size: 12px;
  font-weight: 500;
  color: t.$text-secondary;
}

.db-analytics-form__input {
  font-family: inherit;
  font-size: 13px;
  padding: 9px 11px;
  border: 1px solid t.$border-strong;
  border-radius: t.$radius;
  background: t.$surface-1;
  color: t.$text-primary;

  &:focus {
    outline: none;
    border-color: t.$accent;
    box-shadow: 0 0 0 2px rgba(42, 120, 214, 0.15);
  }
}

.db-analytics-form__hint {
  font-size: 12px;
  color: t.$text-muted;
  margin: -2px 0 8px;
}

.db-analytics-form__error {
  font-size: 12px;
  color: #791f1f;
  background: #fbe9e9;
  padding: 8px 11px;
  border-radius: t.$radius;
}

.db-analytics-form__actions {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding-top: 8px;
  border-top: 1px solid t.$border;
}

.db-analytics-db-select-list {
  list-style: none;
  margin: 0;
  padding: 0;
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 8px;
  max-height: 360px;
  overflow-y: auto;
}

.db-analytics-db-select-item {
  width: 100%;
  min-width: 0;
  display: flex;
  flex-direction: column;
  align-items: stretch;
  gap: 10px;
  padding: 12px;
  border: 1px solid t.$border-strong;
  border-radius: t.$radius-lg;
  background: t.$surface-1;
  cursor: pointer;
  text-align: left;
  font-family: inherit;

  &:hover:not(:disabled) {
    border-color: t.$border-stronger;
  }

  &:disabled {
    cursor: not-allowed;
  }
}

.db-analytics-db-select-item--checked {
  border-color: t.$accent;
  background: t.$accent-bg;
}

.db-analytics-db-select-item--disabled {
  opacity: 0.55;
  background: t.$surface-0;

  .db-analytics-db-select-item__name,
  .db-analytics-db-select-item__meta {
    color: t.$text-muted;
  }
}

.db-analytics-db-select-item__top {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 8px;
}

.db-analytics-db-select-item__checkbox {
  width: 18px;
  height: 18px;
  border-radius: 5px;
  border: 1.5px solid t.$border-strong;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  background: t.$surface-2;
  color: #fff;
  font-size: 12px;
}

.db-analytics-db-select-item__checkbox--checked {
  background: t.$accent;
  border-color: t.$accent;
}

.db-analytics-db-select-item__icon {
  width: 30px;
  height: 30px;
  border-radius: t.$radius;
  background: t.$indigo-bg;
  color: t.$indigo-fg;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 14px;
  flex-shrink: 0;
}

.db-analytics-db-select-item__info {
  display: flex;
  flex-direction: column;
  gap: 2px;
  min-width: 0;
}

.db-analytics-db-select-item__name {
  font-size: 13px;
  font-weight: 600;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.db-analytics-db-select-item__meta {
  font-size: 11px;
  color: t.$text-muted;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.db-analytics-db-select-empty {
  padding: 16px;
  font-size: 13px;
  color: t.$text-muted;
  text-align: center;
}

.db-analytics-link-btn {
  background: none;
  border: none;
  padding: 0;
  color: t.$accent;
  font-size: inherit;
  font-family: inherit;
  cursor: pointer;
  text-decoration: underline;

  &:hover {
    color: #185fa5;
  }
}

.db-analytics-db-status {
  font-size: 11px;
  font-weight: 500;
  display: flex;
  align-items: center;
  gap: 5px;
  white-space: nowrap;
  flex-shrink: 0;
  align-self: flex-start;
}

.db-analytics-db-status__dot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
}

.db-analytics-db-status--connected {
  color: t.$status-connected-fg;

  .db-analytics-db-status__dot {
    background: t.$success;
  }
}

.db-analytics-db-status--disconnected {
  color: t.$status-disconnected-fg;

  .db-analytics-db-status__dot {
    background: t.$danger;
  }
}

.db-analytics-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
  font-size: 13px;
  font-weight: 500;
  font-family: inherit;
  border-radius: t.$radius;
  padding: 8px 16px;
  cursor: pointer;
  border: 1px solid transparent;
  transition: filter 0.15s ease, background 0.15s ease, border-color 0.15s ease;

  &:disabled {
    opacity: 0.6;
    cursor: not-allowed;
  }
}

.db-analytics-btn--primary {
  background: t.$gradient-primary;
  color: #fff;
  box-shadow: 0 1px 2px rgba(79, 70, 229, 0.25);

  &:hover:not(:disabled) {
    filter: brightness(1.08);
    box-shadow: 0 2px 6px rgba(79, 70, 229, 0.35);
  }

  &:active:not(:disabled) {
    filter: brightness(0.96);
  }
}

.db-analytics-btn--ghost {
  background: transparent;
  border-color: t.$border-strong;
  color: t.$text-primary;

  &:hover:not(:disabled) {
    background: t.$surface-0;
  }
}











// SessionHistory.module.scss


@use '../../../styles/tokens' as t;

.db-analytics-session-panel {
  background: t.$surface-2;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  flex: 1;
  min-height: 0;
  border-bottom: 1px solid t.$border-strong;
}

.db-analytics-session-panel__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 53px;
  padding: 0 16px;
  border-bottom: 1px solid t.$border-strong;
  background: t.$surface-1;
  flex-shrink: 0;
}

.db-analytics-session-panel__title {
  font-size: 14px;
  font-weight: 500;
  margin: 0;
  color: t.$text-primary;
}

.db-analytics-session-panel__icon-btn {
  width: 28px;
  height: 28px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border-radius: t.$radius;
  border: 0.5px solid t.$border-strong;
  background: transparent;
  color: t.$text-secondary;
  cursor: pointer;
  font-size: 15px;
  flex-shrink: 0;

  &:hover:not(:disabled) {
    background: t.$surface-0;
    border-color: t.$border-strong;
  }

  &:disabled {
    opacity: 0.45;
    cursor: not-allowed;
  }
}

.db-analytics-session-panel__body {
  padding: 8px;
  flex: 1;
  min-height: 0;
  overflow-y: auto;
}

.db-analytics-session-panel__empty {
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
  padding: 32px 16px;
  gap: 3px;

  p {
    font-size: 13px;
    font-weight: 600;
    color: t.$text-primary;
    margin: 4px 0 0;
  }

  span {
    font-size: 11.5px;
    color: t.$text-muted;
  }
}

.db-analytics-session-panel__empty-icon {
  width: 34px;
  height: 34px;
  border-radius: 50%;
  background: t.$surface-0;
  color: t.$text-muted;
  display: flex;
  align-items: center;
  justify-content: center;
}

.db-analytics-session-list {
  list-style: none;
  margin: 0;
  padding: 0;
  display: flex;
  flex-direction: column;
  gap: 3px;
}

.db-analytics-session-item {
  position: relative;
  width: 100%;
  display: flex;
  align-items: center;
  gap: 10px;
  text-align: left;
  background: none;
  border: 1px solid transparent;
  border-radius: t.$radius;
  padding: 8px 10px;
  cursor: pointer;
  color: t.$text-primary;
  font-family: inherit;
  transition: background 0.15s ease, border-color 0.15s ease;

  &:hover:not(:disabled) {
    background: t.$surface-0;
  }

  &:disabled {
    opacity: 0.55;
    cursor: not-allowed;
  }
}

.db-analytics-session-item--active {
  background: t.$accent-bg;
  border-color: rgba(79, 70, 229, 0.18);

  &::before {
    content: '';
    position: absolute;
    left: 0;
    top: 6px;
    bottom: 6px;
    width: 3px;
    border-radius: 0 3px 3px 0;
    background: t.$gradient-primary;
  }

  &:hover {
    background: t.$accent-bg;
  }

  .db-analytics-session-item__title {
    color: t.$blue-fg;
  }

  .db-analytics-session-item__icon-wrap {
    background: t.$chip-solid;
    color: t.$violet-fg;
    box-shadow: 0 1px 2px rgba(0, 0, 0, 0.2);
  }
}

.db-analytics-session-item__icon-wrap {
  width: 28px;
  height: 28px;
  border-radius: 7px;
  background: t.$surface-0;
  color: t.$text-muted;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  transition: background 0.15s ease, color 0.15s ease;
}

.db-analytics-session-item__text {
  display: flex;
  flex-direction: column;
  gap: 3px;
  min-width: 0;
  flex: 1;
}

.db-analytics-session-item__title {
  font-size: 12.5px;
  font-weight: 600;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  color: t.$text-primary;
}

.db-analytics-session-item__meta {
  display: flex;
  align-items: center;
  gap: 8px;
}

.db-analytics-session-item__db-pill {
  display: inline-flex;
  align-items: center;
  gap: 3px;
  font-size: 10px;
  font-weight: 600;
  color: t.$text-secondary;
  background: t.$surface-2;
  border: 0.5px solid t.$border-strong;
  padding: 1px 6px;
  border-radius: 999px;
  flex-shrink: 0;
}

.db-analytics-session-item__time {
  font-size: 10.5px;
  color: t.$text-muted;
}








// _tokens.scss


// Every color token below resolves through a CSS custom property so theme
// switching is automatic: set `[data-theme='dark']` (or leave it off / set
// to 'light') on any ancestor element — typically <html> or <body> — and
// every component using these tokens repaints without any component-level
// changes. The actual custom property values live in global.scss under
// `:root` (light, default) and `[data-theme='dark']` (dark overrides).
$border: var(--db-analytics-border);
$border-strong: var(--db-analytics-border-strong);
$border-stronger: var(--db-analytics-border-stronger);
$text-primary: var(--db-analytics-text-primary);
$text-secondary: var(--db-analytics-text-secondary);
$text-muted: var(--db-analytics-text-muted);
$surface-0: var(--db-analytics-surface-0);
$surface-1: var(--db-analytics-surface-1);
$surface-2: var(--db-analytics-surface-2);
$accent: var(--db-analytics-accent);
$accent-bg: var(--db-analytics-accent-bg);
$success: var(--db-analytics-success);
$warning: var(--db-analytics-warning);
$danger: var(--db-analytics-danger);
$radius: 8px;
$radius-lg: 12px;
$gradient-primary: var(--db-analytics-gradient-primary);

// Accent-tinted icon/badge pairs (background + foreground), theme-aware.
$indigo-bg: var(--db-analytics-indigo-bg);
$indigo-fg: var(--db-analytics-indigo-fg);
$blue-bg: var(--db-analytics-blue-bg);
$blue-fg: var(--db-analytics-blue-fg);
$violet-bg: var(--db-analytics-violet-bg);
$violet-fg: var(--db-analytics-violet-fg);
$green-bg: var(--db-analytics-green-bg);
$green-fg: var(--db-analytics-green-fg);
$red-bg: var(--db-analytics-red-bg);
$red-fg: var(--db-analytics-red-fg);
$amber-bg: var(--db-analytics-amber-bg);
$amber-fg: var(--db-analytics-amber-fg);

// Status pill text colors (dot backgrounds already use $success/$danger).
$status-connected-fg: var(--db-analytics-status-connected-fg);
$status-disconnected-fg: var(--db-analytics-status-disconnected-fg);

// Solid chip background for "active"/"selected" icon tiles sitting on a
// tinted surface (e.g. the active session's icon wrap).
$chip-solid: var(--db-analytics-chip-solid);













// global.scss


// NOTE: this file is only relevant for running DBAnalytics as a standalone app.
// When merging into an existing project that already sets box-sizing,
// font-family, and root height at the application level, skip importing
// this file — DBAnalytics.module.scss no longer depends on it.
//
// Theme variables: every color used across the DBAnalytics components reads
// from these CSS custom properties (see src/styles/_tokens.scss). Toggling
// dark mode is just a matter of setting `data-theme="dark"` on <html> or
// <body> — no component code needs to know which theme is active.
//
//   document.documentElement.setAttribute('data-theme', 'dark');
//   document.documentElement.setAttribute('data-theme', 'light'); // or remove it

:root {
  --db-analytics-border: rgba(11, 11, 11, 0.1);
  --db-analytics-border-strong: rgba(11, 11, 11, 0.16);
  --db-analytics-border-stronger: rgba(11, 11, 11, 0.24);
  --db-analytics-text-primary: #0b0b0b;
  --db-analytics-text-secondary: #52514e;
  --db-analytics-text-muted: #898781;
  --db-analytics-surface-0: #f5f4f0;
  --db-analytics-surface-1: #fcfcfb;
  --db-analytics-surface-2: #ffffff;
  --db-analytics-accent: #2a78d6;
  --db-analytics-accent-bg: #e6f1fb;
  --db-analytics-success: #0ca30c;
  --db-analytics-warning: #eda100;
  --db-analytics-danger: #e34948;
  --db-analytics-gradient-primary: linear-gradient(135deg, #7c3aed 0%, #4f46e5 50%, #2563eb 100%);

  // Accent-tinted icon/badge chips (session icons, active states, chart
  // icons, etc.) — a soft tint background with a saturated foreground.
  --db-analytics-indigo-bg: #eeedfe;
  --db-analytics-indigo-fg: #3c3489;
  --db-analytics-blue-bg: #e6f1fb;
  --db-analytics-blue-fg: #0c5c96;
  --db-analytics-violet-bg: #ece9fd;
  --db-analytics-violet-fg: #4530a8;
  --db-analytics-green-bg: #e3f5ea;
  --db-analytics-green-fg: #0a7a41;
  --db-analytics-red-bg: #fbe9e9;
  --db-analytics-red-fg: #791f1f;
  --db-analytics-amber-bg: #fef3e2;
  --db-analytics-amber-fg: #8a5a00;

  // Status pill colors (connected / disconnected)
  --db-analytics-status-connected-fg: #085041;
  --db-analytics-status-disconnected-fg: #791f1f;

  // Solid white used for "active"/"selected" icon chips that sit on a
  // tinted background — needs to flip to a dark surface in dark mode.
  --db-analytics-chip-solid: #ffffff;
}

[data-theme='dark'] {
  --db-analytics-border: rgba(255, 255, 255, 0.1);
  --db-analytics-border-strong: rgba(255, 255, 255, 0.16);
  --db-analytics-border-stronger: rgba(255, 255, 255, 0.24);
  --db-analytics-text-primary: #f5f5f5;
  --db-analytics-text-secondary: #b6b4ae;
  --db-analytics-text-muted: #86847e;
  --db-analytics-surface-0: #141414;
  --db-analytics-surface-1: #0c0c0c;
  --db-analytics-surface-2: #000000;
  --db-analytics-accent: #5b9eec;
  --db-analytics-accent-bg: rgba(79, 70, 229, 0.16);
  --db-analytics-success: #3ecf5f;
  --db-analytics-warning: #f0b429;
  --db-analytics-danger: #f16060;
  --db-analytics-gradient-primary: linear-gradient(135deg, #8b5cf6 0%, #6366f1 50%, #3b82f6 100%);

  --db-analytics-indigo-bg: rgba(124, 58, 237, 0.18);
  --db-analytics-indigo-fg: #b7a6ff;
  --db-analytics-blue-bg: rgba(42, 120, 214, 0.18);
  --db-analytics-blue-fg: #7fb4ee;
  --db-analytics-violet-bg: rgba(124, 58, 237, 0.18);
  --db-analytics-violet-fg: #c3aefc;
  --db-analytics-green-bg: rgba(62, 207, 95, 0.16);
  --db-analytics-green-fg: #6ee497;
  --db-analytics-red-bg: rgba(241, 96, 96, 0.16);
  --db-analytics-red-fg: #f79999;
  --db-analytics-amber-bg: rgba(240, 180, 41, 0.16);
  --db-analytics-amber-fg: #f5c968;

  --db-analytics-status-connected-fg: #6ee497;
  --db-analytics-status-disconnected-fg: #f79999;

  --db-analytics-chip-solid: #1a1a1a;
}

* {
  box-sizing: border-box;
}
