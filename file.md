//DatabaseList.module.scss
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
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 14px;
  font-weight: 500;
  margin: 0;
  color: t.$text-primary;
}

.db-analytics-db-panel__refresh-spin {
  color: t.$text-muted;
  animation: db-analytics-db-panel-refresh-spin 0.8s linear infinite;
}

@keyframes db-analytics-db-panel-refresh-spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
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
  flex-direction: column;
  gap: 4px;
  padding: 7px 8px;
  border-radius: t.$radius;
  transition: background 0.15s ease;

  &:hover {
    background: t.$surface-0;
  }
}

.db-analytics-db-item__row {
  display: flex;
  align-items: center;
  gap: 10px;
}

.db-analytics-db-item__connect-row {
  display: flex;
  justify-content: flex-end;
  padding-left: 40px;
}

.db-analytics-db-item__connect-btn {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  font-size: 10.5px;
  font-weight: 600;
  padding: 3px 9px;
  border-radius: 999px;
  border: 0.5px solid t.$border-strong;
  background: t.$surface-2;
  color: t.$text-secondary;
  cursor: pointer;
  font-family: inherit;
  transition: background 0.12s ease, border-color 0.12s ease, color 0.12s ease;

  &:hover:not(:disabled) {
    background: t.$accent-bg;
    border-color: t.$accent;
    color: t.$text-primary;
  }

  &:disabled {
    opacity: 0.55;
    cursor: not-allowed;
  }
}

.db-analytics-db-item__connect-btn--primary {
  background: t.$gradient-primary;
  color: #fff;
  border-color: transparent;

  &:hover:not(:disabled) {
    background: t.$gradient-primary;
    filter: brightness(1.08);
    color: #fff;
    border-color: transparent;
  }
}

.db-analytics-db-item__spin {
  animation: db-analytics-db-item-spin 0.8s linear infinite;
}

@keyframes db-analytics-db-item-spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
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

.db-analytics-db-item__icon--mssql {
  background: t.$amber-bg;
  color: t.$amber-fg;
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

.db-analytics-db-item__type-badge--mssql {
  color: t.$amber-fg;
  background: t.$amber-bg;
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

.db-analytics-db-item__actions {
  display: flex;
  align-items: center;
  gap: 8px;
  flex-shrink: 0;
}

.db-analytics-db-item__schema-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 22px;
  height: 22px;
  border-radius: 6px;
  border: 0.5px solid t.$border-strong;
  background: transparent;
  color: t.$text-muted;
  cursor: pointer;
  flex-shrink: 0;
  transition: background 0.15s ease, border-color 0.15s ease, color 0.15s ease;

  &:hover:not(:disabled) {
    background: t.$surface-2;
    border-color: t.$border-stronger;
    color: t.$accent;
  }

  &:disabled {
    opacity: 0.35;
    cursor: not-allowed;
  }
}
