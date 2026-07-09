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













//DatabaseList.tsx
import React, { useState } from 'react';
import styles from './DatabaseList.module.scss';
import { DatabaseItem } from '../types';
import { Icon } from '../icons';
import { getDbTypeMeta } from '../dbTypeMeta';
import SkeletonListItem from './SkeletonListItem';
import DbSchemaSlider from './DbSchemaSlider';

interface Props {
  databases: DatabaseItem[];
  onManage: () => void;
  disabled?: boolean;
  loading?: boolean;
}

const DatabaseList: React.FC<Props> = ({ databases, onManage, disabled, loading }) => {
  const disabledTitle = disabled ? 'Wait for the current response to finish' : undefined;
  const [schemaTarget, setSchemaTarget] = useState<DatabaseItem | null>(null);

  return (
    <div className={styles['db-analytics-db-panel']}>
      <div className={styles['db-analytics-db-panel__header']}>
        <h2 className={styles['db-analytics-db-panel__title']}>Databases</h2>
        <div className={styles['db-analytics-db-panel__header-actions']}>
          <button
            className={styles['db-analytics-db-panel__icon-btn']}
            aria-label="Manage databases"
            onClick={onManage}
            disabled={disabled}
            title={disabled ? disabledTitle : 'Manage databases'}
          >
            <Icon.settings size={15} aria-hidden="true" />
          </button>
          <button
            className={styles['db-analytics-db-panel__icon-btn']}
            aria-label="Add database connection"
            onClick={onManage}
            disabled={disabled}
            title={disabled ? disabledTitle : 'Add database'}
          >
            <Icon.plus size={16} aria-hidden="true" />
          </button>
        </div>
      </div>
      <div className={styles['db-analytics-db-panel__body']}>
        {loading ? (
          <SkeletonListItem variant="db" count={4} />
        ) : databases.length === 0 ? (
          <div className={styles['db-analytics-db-panel__empty']}>
            <span className={styles['db-analytics-db-panel__empty-icon']}>
              <Icon.database size={18} aria-hidden="true" />
            </span>
            <p>No databases connected</p>
            <button
              className={styles['db-analytics-db-panel__link']}
              onClick={onManage}
              disabled={disabled}
              title={disabledTitle}
            >
              Connect a database
            </button>
          </div>
        ) : (
          <ul className={styles['db-analytics-db-list']}>
            {databases.map((db) => {
              const typeMeta = getDbTypeMeta(db.db_type);
              const isCsv = db.db_type === 'csv';
              const schemaEnabled = isCsv || db.connected;
              return (
                <li key={db.id} className={styles['db-analytics-db-item']}>
                  <div
                    className={`${styles['db-analytics-db-item__icon']} ${
                      styles[`db-analytics-db-item__icon--${typeMeta.modifier}`] ?? ''
                    }`}
                  >
                    {isCsv ? (
                      <Icon.csv size={14} aria-hidden="true" />
                    ) : (
                      <Icon.database size={14} aria-hidden="true" />
                    )}
                  </div>
                  <div className={styles['db-analytics-db-item__info']}>
                    <span className={styles['db-analytics-db-item__name']}>{db.name}</span>
                    <span className={styles['db-analytics-db-item__meta']}>
                      <span
                        className={`${styles['db-analytics-db-item__type-badge']} ${
                          styles[`db-analytics-db-item__type-badge--${typeMeta.modifier}`] ?? ''
                        }`}
                      >
                        {typeMeta.label}
                      </span>
                      <span className={styles['db-analytics-db-item__db-name']}>{db.db_name}</span>
                    </span>
                  </div>
                  <div className={styles['db-analytics-db-item__actions']}>
                    <button
                      type="button"
                      className={styles['db-analytics-db-item__schema-btn']}
                      onClick={() => schemaEnabled && setSchemaTarget(db)}
                      disabled={!schemaEnabled}
                      aria-label="View schema and ER diagram"
                      title={schemaEnabled ? 'View schema & ER diagram' : 'Connect this database to view its schema'}
                    >
                      <Icon.schema size={13} aria-hidden="true" />
                    </button>
                    <span
                      className={`${styles['db-analytics-db-status-dot']} ${
                        schemaEnabled
                          ? styles['db-analytics-db-status-dot--connected']
                          : styles['db-analytics-db-status-dot--disconnected']
                      }`}
                      title={isCsv ? 'Ready' : db.connected ? 'Connected' : 'Disconnected'}
                      aria-label={isCsv ? 'Ready' : db.connected ? 'Connected' : 'Disconnected'}
                    />
                  </div>
                </li>
              );
            })}
          </ul>
        )}
      </div>

      <DbSchemaSlider database={schemaTarget} onClose={() => setSchemaTarget(null)} />
    </div>
  );
};

export default DatabaseList;















//DatabaseManagerSlider.module.scss
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
  align-items: center;
  justify-content: space-between;
  gap: 8px;
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

.db-analytics-btn--icon {
  width: 30px;
  height: 30px;
  padding: 0;
  background: t.$surface-0;
  border-color: t.$border-strong;
  color: t.$text-secondary;

  &:hover:not(:disabled) {
    background: t.$surface-2;
    border-color: t.$border-stronger;
    color: t.$text-primary;
  }
}











//DatabaseManagerSlider.tsx
import React, { useRef, useState } from 'react';
import styles from './DatabaseManagerSlider.module.scss';
import { DatabaseItem } from '../types';
import { api, CreateDbPayload } from '../../../services/api';
import ConnectPasswordModal from './ConnectPasswordModal';
import DbSchemaSlider from './DbSchemaSlider';
import { Icon } from '../icons';
import { getDbTypeMeta } from '../dbTypeMeta';

interface Props {
  open: boolean;
  databases: DatabaseItem[];
  onClose: () => void;
  onDatabaseCreated: (db: DatabaseItem) => void;
  onDatabaseConnected: (dbId: string, cacheUntil: string) => void;
  onDatabaseDisconnected: (dbId: string) => void;
}

type Tab = 'existing' | 'new' | 'csv';

const emptyForm: CreateDbPayload = {
  name: '',
  db_name: '',
  db_type: 'mysql',
  host: '',
  port: 3306,
  username: '',
};

const emptyCsvForm = {
  name: '',
  table_name: '',
};

const DatabaseManagerSlider: React.FC<Props> = ({
  open,
  databases,
  onClose,
  onDatabaseCreated,
  onDatabaseConnected,
  onDatabaseDisconnected,
}) => {
  const [tab, setTab] = useState<Tab>('existing');
  const [form, setForm] = useState<CreateDbPayload>(emptyForm);
  const [creating, setCreating] = useState(false);
  const [createError, setCreateError] = useState<string | null>(null);
  const [busyDbId, setBusyDbId] = useState<string | null>(null);
  const [connectTarget, setConnectTarget] = useState<DatabaseItem | null>(null);
  const [schemaTarget, setSchemaTarget] = useState<DatabaseItem | null>(null);

  const [csvForm, setCsvForm] = useState(emptyCsvForm);
  const [csvFile, setCsvFile] = useState<File | null>(null);
  const [csvUploading, setCsvUploading] = useState(false);
  const [csvError, setCsvError] = useState<string | null>(null);
  const fileInputRef = useRef<HTMLInputElement>(null);

  if (!open) return null;

  const updateField = <K extends keyof CreateDbPayload>(key: K, value: CreateDbPayload[K]) => {
    setForm((prev) => ({ ...prev, [key]: value }));
  };

  const handleCreate = async () => {
    setCreateError(null);
    if (!form.name.trim() || !form.db_name.trim() || !form.host.trim() || !form.username.trim()) {
      setCreateError('Please fill in all required fields.');
      return;
    }
    setCreating(true);
    try {
      const created = await api.createDatabase(form);
      onDatabaseCreated(created);
      setForm(emptyForm);
      setTab('existing');
    } catch (err) {
      setCreateError(err instanceof Error ? err.message : 'Failed to create database.');
    } finally {
      setCreating(false);
    }
  };

  const handleDisconnect = async (db: DatabaseItem) => {
    setBusyDbId(db.id);
    try {
      await api.disconnectDatabase(db.id);
      onDatabaseDisconnected(db.id);
    } catch (err) {
      console.error(err);
    } finally {
      setBusyDbId(null);
    }
  };

  const handleFilePick = (fileList: FileList | null) => {
    const file = fileList?.[0];
    if (!file) return;
    if (!file.name.toLowerCase().endsWith('.csv')) {
      setCsvError('Only .csv files are supported.');
      return;
    }
    setCsvError(null);
    setCsvFile(file);
  };

  const handleCsvUpload = async () => {
    setCsvError(null);
    if (!csvFile) {
      setCsvError('Please select a CSV file to upload.');
      return;
    }
    if (!csvForm.name.trim() || !csvForm.table_name.trim()) {
      setCsvError('Please provide both a name and a table name.');
      return;
    }
    setCsvUploading(true);
    try {
      const created = await api.uploadCsv({
        file: csvFile,
        name: csvForm.name.trim(),
        table_name: csvForm.table_name.trim(),
      });
      onDatabaseCreated(created);
      setCsvForm(emptyCsvForm);
      setCsvFile(null);
      if (fileInputRef.current) fileInputRef.current.value = '';
      setTab('existing');
    } catch (err) {
      setCsvError(err instanceof Error ? err.message : 'Failed to upload CSV file.');
    } finally {
      setCsvUploading(false);
    }
  };

  return (
    <div className={styles['db-analytics-slider-overlay']} role="dialog" aria-modal="true">
      <div className={styles['db-analytics-slider-overlay__backdrop']} onClick={onClose} />
      <aside className={styles['db-analytics-slider-panel']}>
        <div className={styles['db-analytics-slider-panel__header']}>
          <h2 className={styles['db-analytics-slider-panel__title']}>Manage databases</h2>
          <button className={styles['db-analytics-slider-panel__close-btn']} aria-label="Close" onClick={onClose}>
            <Icon.close size={16} aria-hidden="true" />
          </button>
        </div>

        <div className={styles['db-analytics-slider-tabs']}>
          <button
            className={`${styles['db-analytics-slider-tabs__item']} ${
              tab === 'existing' ? styles['db-analytics-slider-tabs__item--active'] : ''
            }`}
            onClick={() => setTab('existing')}
          >
            Existing databases
          </button>
          <button
            className={`${styles['db-analytics-slider-tabs__item']} ${
              tab === 'new' ? styles['db-analytics-slider-tabs__item--active'] : ''
            }`}
            onClick={() => setTab('new')}
          >
            <Icon.plus size={14} aria-hidden="true" />
            New connection
          </button>
          <button
            className={`${styles['db-analytics-slider-tabs__item']} ${
              tab === 'csv' ? styles['db-analytics-slider-tabs__item--active'] : ''
            }`}
            onClick={() => setTab('csv')}
          >
            <Icon.upload size={14} aria-hidden="true" />
            Upload CSV
          </button>
        </div>

        <div className={styles['db-analytics-slider-panel__body']}>
          {tab === 'existing' ? (
            <ul className={styles['db-analytics-db-manage-list']}>
              {databases.length === 0 && (
                <li className={styles['db-analytics-db-manage-empty']}>
                  <span className={styles['db-analytics-db-manage-empty__icon']}>
                    <Icon.database size={22} aria-hidden="true" />
                  </span>
                  <p className={styles['db-analytics-db-manage-empty__title']}>No databases yet</p>
                  <p className={styles['db-analytics-db-manage-empty__desc']}>
                    Create a connection or upload a CSV file to start querying data.
                  </p>
                </li>
              )}
              {databases.map((db) => {
                const typeMeta = getDbTypeMeta(db.db_type);
                const isCsv = db.db_type === 'csv';
                return (
                  <li key={db.id} className={styles['db-analytics-db-manage-item']}>
                    <div className={styles['db-analytics-db-manage-item__top']}>
                      <div className={styles['db-analytics-db-manage-item__identity']}>
                        <div
                          className={`${styles['db-analytics-db-manage-item__icon']} ${
                            styles[`db-analytics-db-manage-item__icon--${typeMeta.modifier}`] ?? ''
                          }`}
                        >
                          {isCsv ? (
                            <Icon.csv size={17} aria-hidden="true" />
                          ) : (
                            <Icon.database size={17} aria-hidden="true" />
                          )}
                        </div>
                        <div className={styles['db-analytics-db-manage-item__name-block']}>
                          <span className={styles['db-analytics-db-manage-item__name']}>{db.name}</span>
                          <span
                            className={`${styles['db-analytics-db-manage-item__type-badge']} ${
                              styles[`db-analytics-db-manage-item__type-badge--${typeMeta.modifier}`] ?? ''
                            }`}
                          >
                            {typeMeta.label}
                          </span>
                        </div>
                      </div>
                      {isCsv ? (
                        <span
                          className={`${styles['db-analytics-db-status']} ${styles['db-analytics-db-status--connected']}`}
                        >
                          <span className={styles['db-analytics-db-status__dot']} aria-hidden="true" />
                          Ready
                        </span>
                      ) : (
                        <span
                          className={`${styles['db-analytics-db-status']} ${
                            db.connected
                              ? styles['db-analytics-db-status--connected']
                              : styles['db-analytics-db-status--disconnected']
                          }`}
                        >
                          <span className={styles['db-analytics-db-status__dot']} aria-hidden="true" />
                          {db.connected ? 'Connected' : 'Disconnected'}
                        </span>
                      )}
                    </div>

                    <dl className={styles['db-analytics-db-manage-item__meta-grid']}>
                      <div className={styles['db-analytics-db-manage-item__meta-cell']}>
                        <dt>{isCsv ? 'Table' : 'Database'}</dt>
                        <dd>{db.db_name}</dd>
                      </div>
                      {!isCsv && (
                        <div className={styles['db-analytics-db-manage-item__meta-cell']}>
                          <dt>Port</dt>
                          <dd>{db.port}</dd>
                        </div>
                      )}
                      <div className={styles['db-analytics-db-manage-item__meta-cell']}>
                        <dt>{isCsv ? 'Source' : 'User'}</dt>
                        <dd>{isCsv ? 'CSV upload' : db.username}</dd>
                      </div>
                    </dl>

                    <div className={styles['db-analytics-db-manage-item__footer']}>
                      <button
                        type="button"
                        className={`${styles['db-analytics-btn']} ${styles['db-analytics-btn--icon']}`}
                        onClick={() => setSchemaTarget(db)}
                        disabled={!isCsv && !db.connected}
                        title={
                          isCsv || db.connected
                            ? 'View schema & ER diagram'
                            : 'Connect this database to view its schema'
                        }
                        aria-label="View schema and ER diagram"
                      >
                        <Icon.schema size={14} aria-hidden="true" />
                      </button>
                      {!isCsv &&
                        (db.connected ? (
                          <button
                            className={`${styles['db-analytics-btn']} ${styles['db-analytics-btn--ghost']} ${styles['db-analytics-btn--sm']}`}
                            onClick={() => handleDisconnect(db)}
                            disabled={busyDbId === db.id}
                          >
                            {busyDbId === db.id ? 'Disconnecting…' : 'Disconnect'}
                          </button>
                        ) : (
                          <button
                            className={`${styles['db-analytics-btn']} ${styles['db-analytics-btn--primary']} ${styles['db-analytics-btn--sm']}`}
                            onClick={() => setConnectTarget(db)}
                          >
                            Connect
                          </button>
                        ))}
                    </div>
                  </li>
                );
              })}
            </ul>
          ) : tab === 'new' ? (
            <div className={styles['db-analytics-form-card']}>
              <div className={styles['db-analytics-form-card__header']}>
                <span className={styles['db-analytics-form-card__icon']}>
                  <Icon.database size={16} aria-hidden="true" />
                </span>
                <div>
                  <h3 className={styles['db-analytics-form-card__title']}>New database connection</h3>
                  <p className={styles['db-analytics-form-card__subtitle']}>
                    Connect a MySQL or PostgreSQL database to query from this workspace.
                  </p>
                </div>
              </div>

              <div className={styles['db-analytics-form']}>
                {createError && <div className={styles['db-analytics-form__error']}>{createError}</div>}

                <label className={styles['db-analytics-form__field']}>
                  <span className={styles['db-analytics-form__label']}>Connection name</span>
                  <input
                    className={styles['db-analytics-form__input']}
                    value={form.name}
                    onChange={(e) => updateField('name', e.target.value)}
                    placeholder="e.g. Production Warehouse"
                  />
                </label>

                <label className={styles['db-analytics-form__field']}>
                  <span className={styles['db-analytics-form__label']}>Database type</span>
                  <select
                    className={styles['db-analytics-form__input']}
                    value={form.db_type}
                    onChange={(e) => updateField('db_type', e.target.value as CreateDbPayload['db_type'])}
                  >
                    <option value="mysql">MySQL</option>
                    <option value="postgres">PostgreSQL</option>
                  </select>
                </label>

                <div className={styles['db-analytics-form__row']}>
                  <label className={styles['db-analytics-form__field']}>
                    <span className={styles['db-analytics-form__label']}>Host</span>
                    <input
                      className={styles['db-analytics-form__input']}
                      value={form.host}
                      onChange={(e) => updateField('host', e.target.value)}
                      placeholder="e.g. 10.0.0.4"
                    />
                  </label>
                  <label
                    className={`${styles['db-analytics-form__field']} ${styles['db-analytics-form__field--narrow']}`}
                  >
                    <span className={styles['db-analytics-form__label']}>Port</span>
                    <input
                      className={styles['db-analytics-form__input']}
                      type="number"
                      value={form.port}
                      onChange={(e) => updateField('port', Number(e.target.value))}
                    />
                  </label>
                </div>

                <label className={styles['db-analytics-form__field']}>
                  <span className={styles['db-analytics-form__label']}>Database name</span>
                  <input
                    className={styles['db-analytics-form__input']}
                    value={form.db_name}
                    onChange={(e) => updateField('db_name', e.target.value)}
                    placeholder="e.g. analytics_prod"
                  />
                </label>

                <label className={styles['db-analytics-form__field']}>
                  <span className={styles['db-analytics-form__label']}>Username</span>
                  <input
                    className={styles['db-analytics-form__input']}
                    value={form.username}
                    onChange={(e) => updateField('username', e.target.value)}
                    placeholder="e.g. readonly_user"
                  />
                </label>

                <p className={styles['db-analytics-form__hint']}>
                  <Icon.infoCircle size={14} aria-hidden="true" /> You'll be asked for the password
                  separately when connecting.
                </p>

                <div className={styles['db-analytics-form__actions']}>
                  <button
                    className={`${styles['db-analytics-btn']} ${styles['db-analytics-btn--ghost']}`}
                    onClick={() => setForm(emptyForm)}
                    disabled={creating}
                  >
                    Reset
                  </button>
                  <button
                    className={`${styles['db-analytics-btn']} ${styles['db-analytics-btn--primary']}`}
                    onClick={handleCreate}
                    disabled={creating}
                  >
                    {creating ? 'Creating…' : 'Create connection'}
                  </button>
                </div>
              </div>
            </div>
          ) : (
            <div className={styles['db-analytics-form-card']}>
              <div className={styles['db-analytics-form-card__header']}>
                <span
                  className={`${styles['db-analytics-form-card__icon']} ${styles['db-analytics-form-card__icon--csv']}`}
                >
                  <Icon.csv size={16} aria-hidden="true" />
                </span>
                <div>
                  <h3 className={styles['db-analytics-form-card__title']}>Upload a CSV file</h3>
                  <p className={styles['db-analytics-form-card__subtitle']}>
                    Turn a spreadsheet into a queryable table — no database connection required.
                  </p>
                </div>
              </div>

              <div className={styles['db-analytics-form']}>
                {csvError && <div className={styles['db-analytics-form__error']}>{csvError}</div>}

                <label className={styles['db-analytics-form__field']}>
                  <span className={styles['db-analytics-form__label']}>CSV file</span>
                  <button
                    type="button"
                    className={`${styles['db-analytics-file-drop']} ${
                      csvFile ? styles['db-analytics-file-drop--filled'] : ''
                    }`}
                    onClick={() => fileInputRef.current?.click()}
                  >
                    <span className={styles['db-analytics-file-drop__icon']}>
                      {csvFile ? (
                        <Icon.csv size={18} aria-hidden="true" />
                      ) : (
                        <Icon.upload size={18} aria-hidden="true" />
                      )}
                    </span>
                    <span className={styles['db-analytics-file-drop__text']}>
                      <span className={styles['db-analytics-file-drop__title']}>
                        {csvFile ? csvFile.name : 'Choose a .csv file'}
                      </span>
                      <span className={styles['db-analytics-file-drop__subtitle']}>
                        {csvFile
                          ? `${(csvFile.size / 1024).toFixed(1)} KB — click to change`
                          : 'Only .csv files are accepted'}
                      </span>
                    </span>
                  </button>
                  <input
                    ref={fileInputRef}
                    type="file"
                    accept=".csv,text/csv"
                    className={styles['db-analytics-file-drop__input']}
                    onChange={(e) => handleFilePick(e.target.files)}
                  />
                </label>

                <label className={styles['db-analytics-form__field']}>
                  <span className={styles['db-analytics-form__label']}>Name</span>
                  <input
                    className={styles['db-analytics-form__input']}
                    value={csvForm.name}
                    onChange={(e) => setCsvForm((prev) => ({ ...prev, name: e.target.value }))}
                    placeholder="e.g. Sample"
                  />
                </label>

                <label className={styles['db-analytics-form__field']}>
                  <span className={styles['db-analytics-form__label']}>Table name</span>
                  <input
                    className={styles['db-analytics-form__input']}
                    value={csvForm.table_name}
                    onChange={(e) => setCsvForm((prev) => ({ ...prev, table_name: e.target.value }))}
                    placeholder="e.g. Sample Table"
                  />
                </label>

                <p className={styles['db-analytics-form__hint']}>
                  <Icon.infoCircle size={14} aria-hidden="true" /> Uploaded CSVs are ready to query
                  immediately — no password or connection step needed.
                </p>

                <div className={styles['db-analytics-form__actions']}>
                  <button
                    className={`${styles['db-analytics-btn']} ${styles['db-analytics-btn--ghost']}`}
                    onClick={() => {
                      setCsvForm(emptyCsvForm);
                      setCsvFile(null);
                      setCsvError(null);
                      if (fileInputRef.current) fileInputRef.current.value = '';
                    }}
                    disabled={csvUploading}
                  >
                    Reset
                  </button>
                  <button
                    className={`${styles['db-analytics-btn']} ${styles['db-analytics-btn--primary']}`}
                    onClick={handleCsvUpload}
                    disabled={csvUploading}
                  >
                    {csvUploading ? 'Uploading…' : 'Upload CSV'}
                  </button>
                </div>
              </div>
            </div>
          )}
        </div>
      </aside>

      {connectTarget && (
        <ConnectPasswordModal
          database={connectTarget}
          onClose={() => setConnectTarget(null)}
          onConnected={(cacheUntil) => {
            onDatabaseConnected(connectTarget.id, cacheUntil);
            setConnectTarget(null);
          }}
        />
      )}

      <DbSchemaSlider database={schemaTarget} onClose={() => setSchemaTarget(null)} />
    </div>
  );
};

export default DatabaseManagerSlider;













//DbSchemaSlider.module.scss
@use '../../../styles/tokens' as t;

.db-analytics-schema-overlay {
  position: fixed;
  inset: 0;
  z-index: 150;
  display: flex;
  justify-content: flex-end;
}

.db-analytics-schema-overlay__backdrop {
  position: absolute;
  inset: 0;
  background: rgba(11, 11, 11, 0.32);
  animation: db-analytics-schema-fade-in 0.15s ease;
}

@keyframes db-analytics-schema-fade-in {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

.db-analytics-schema-panel {
  position: relative;
  width: 620px;
  max-width: 100vw;
  height: 100%;
  background: t.$surface-2;
  display: flex;
  flex-direction: column;
  box-shadow: -8px 0 24px rgba(11, 11, 11, 0.14);
  animation: db-analytics-schema-slide-in 0.22s cubic-bezier(0.16, 1, 0.3, 1);
}

@keyframes db-analytics-schema-slide-in {
  from {
    transform: translateX(100%);
  }
  to {
    transform: translateX(0);
  }
}

.db-analytics-schema-panel__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 64px;
  padding: 0 20px;
  border-bottom: 1px solid t.$border-strong;
  background: t.$surface-1;
  flex-shrink: 0;
}

.db-analytics-schema-panel__heading {
  display: flex;
  align-items: center;
  gap: 12px;
  min-width: 0;
}

.db-analytics-schema-panel__icon {
  width: 34px;
  height: 34px;
  border-radius: 9px;
  background: t.$gradient-primary;
  color: #fff;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  box-shadow: 0 2px 6px rgba(79, 70, 229, 0.25);
}

.db-analytics-schema-panel__title {
  font-size: 15px;
  font-weight: 600;
  color: t.$text-primary;
  margin: 0 0 2px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 320px;
}

.db-analytics-schema-panel__subtitle {
  font-size: 11.5px;
  color: t.$text-muted;
}

.db-analytics-schema-panel__close-btn {
  width: 32px;
  height: 32px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border-radius: t.$radius;
  border: 0.5px solid t.$border-strong;
  background: transparent;
  color: t.$text-secondary;
  cursor: pointer;
  flex-shrink: 0;

  &:hover {
    background: t.$surface-0;
  }
}

.db-analytics-schema-tabs {
  display: flex;
  gap: 4px;
  padding: 12px 20px 0;
  border-bottom: 1px solid t.$border-strong;
  background: t.$surface-1;
  flex-shrink: 0;
}

.db-analytics-schema-tabs__item {
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

.db-analytics-schema-tabs__item--active {
  color: t.$text-primary;
  border-bottom-color: t.$text-primary;
}

.db-analytics-schema-tabs__count {
  font-size: 10.5px;
  font-weight: 600;
  color: t.$text-secondary;
  background: t.$surface-0;
  border: 0.5px solid t.$border;
  padding: 1px 6px;
  border-radius: 999px;
}

.db-analytics-schema-panel__body {
  flex: 1;
  overflow-y: auto;
  padding: 18px 20px;
}

.db-analytics-schema-empty {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 10px;
  padding: 60px 20px;
  color: t.$text-muted;
  text-align: center;

  p {
    font-size: 13px;
    margin: 0;
    max-width: 260px;
  }
}

.db-analytics-schema-spin {
  animation: db-analytics-schema-spin 0.8s linear infinite;
}

@keyframes db-analytics-schema-spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

.db-analytics-schema-summary {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 12.5px;
  color: t.$text-secondary;
  margin-bottom: 14px;

  strong {
    color: t.$text-primary;
    font-weight: 700;
  }
}

.db-analytics-schema-summary__dot {
  color: t.$text-muted;
}

.db-analytics-schema-table-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.db-analytics-schema-table {
  border: 1px solid t.$border-strong;
  border-radius: t.$radius-lg;
  overflow: hidden;
}

.db-analytics-schema-table__header {
  display: flex;
  align-items: center;
  gap: 10px;
  width: 100%;
  padding: 11px 14px;
  background: t.$surface-1;
  border: none;
  cursor: pointer;
  font-family: inherit;
  text-align: left;

  &:hover {
    background: t.$surface-0;
  }
}

.db-analytics-schema-table__icon {
  width: 24px;
  height: 24px;
  border-radius: 6px;
  background: t.$accent-bg;
  color: #0c447c;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.db-analytics-schema-table__name {
  font-size: 13px;
  font-weight: 600;
  color: t.$text-primary;
  font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
}

.db-analytics-schema-table__meta {
  flex: 1;
  text-align: right;
  font-size: 11px;
  color: t.$text-muted;
}

.db-analytics-schema-table__chevron {
  color: t.$text-muted;
  transform: rotate(90deg);
  transition: transform 0.15s ease;
  flex-shrink: 0;
}

.db-analytics-schema-table__chevron--open {
  transform: rotate(-90deg);
}

.db-analytics-schema-table__body {
  padding: 4px 0;
  border-top: 1px solid t.$border;
}

.db-analytics-schema-columns {
  width: 100%;
  border-collapse: collapse;
  font-size: 12px;

  th {
    text-align: left;
    font-weight: 600;
    font-size: 10.5px;
    text-transform: uppercase;
    letter-spacing: 0.02em;
    color: t.$text-muted;
    padding: 8px 14px 6px;
  }

  td {
    padding: 6px 14px;
    color: t.$text-primary;
    border-top: 0.5px solid t.$border;
  }

  tr:first-child td {
    border-top: none;
  }
}

.db-analytics-schema-columns__name {
  display: flex;
  align-items: center;
  gap: 6px;
  font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
  font-weight: 500;
}

.db-analytics-schema-columns__pk {
  font-size: 9px;
  font-weight: 700;
  color: t.$accent;
  background: t.$accent-bg;
  padding: 1px 5px;
  border-radius: 4px;
  flex-shrink: 0;
}

.db-analytics-schema-columns__type {
  font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
  color: t.$text-secondary;
  font-size: 11px;
}


















//DbSchemaSlider.tsx
import React, { useEffect, useMemo, useState } from 'react';
import styles from './DbSchemaSlider.module.scss';
import { DatabaseItem, DbSchema } from '../types';
import { Icon } from '../icons';
import { api } from '../../../services/api';
import SchemaErDiagram from './SchemaErDiagram';

interface Props {
  database: DatabaseItem | null;
  onClose: () => void;
}

type Tab = 'tables' | 'diagram';

const DbSchemaSlider: React.FC<Props> = ({ database, onClose }) => {
  const [tab, setTab] = useState<Tab>('tables');
  const [schema, setSchema] = useState<DbSchema | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [expandedTable, setExpandedTable] = useState<string | null>(null);

  useEffect(() => {
    if (!database) return;
    let cancelled = false;
    setSchema(null);
    setError(null);
    setTab('tables');
    setExpandedTable(null);
    setLoading(true);

    api
      .getSchema(database.id)
      .then((res) => {
        if (!cancelled) {
          setSchema(res);
          setExpandedTable(res.tables[0]?.name ?? null);
        }
      })
      .catch((err) => {
        if (!cancelled) setError(err instanceof Error ? err.message : 'Failed to load schema.');
      })
      .finally(() => {
        if (!cancelled) setLoading(false);
      });

    return () => {
      cancelled = true;
    };
  }, [database?.id]);

  const totalRows = useMemo(
    () => schema?.tables.reduce((sum, t) => sum + (t.row_count ?? 0), 0) ?? 0,
    [schema]
  );

  if (!database) return null;

  return (
    <div className={styles['db-analytics-schema-overlay']} role="dialog" aria-modal="true">
      <div className={styles['db-analytics-schema-overlay__backdrop']} onClick={onClose} />
      <aside className={styles['db-analytics-schema-panel']}>
        <div className={styles['db-analytics-schema-panel__header']}>
          <div className={styles['db-analytics-schema-panel__heading']}>
            <span className={styles['db-analytics-schema-panel__icon']}>
              <Icon.schema size={16} aria-hidden="true" />
            </span>
            <div>
              <h2 className={styles['db-analytics-schema-panel__title']}>{database.name}</h2>
              <span className={styles['db-analytics-schema-panel__subtitle']}>
                Schema &amp; relationships
              </span>
            </div>
          </div>
          <button className={styles['db-analytics-schema-panel__close-btn']} aria-label="Close" onClick={onClose}>
            <Icon.close size={18} aria-hidden="true" />
          </button>
        </div>

        <div className={styles['db-analytics-schema-tabs']}>
          <button
            className={`${styles['db-analytics-schema-tabs__item']} ${
              tab === 'tables' ? styles['db-analytics-schema-tabs__item--active'] : ''
            }`}
            onClick={() => setTab('tables')}
          >
            <Icon.table size={14} aria-hidden="true" />
            Tables
            {schema && <span className={styles['db-analytics-schema-tabs__count']}>{schema.tables.length}</span>}
          </button>
          <button
            className={`${styles['db-analytics-schema-tabs__item']} ${
              tab === 'diagram' ? styles['db-analytics-schema-tabs__item--active'] : ''
            }`}
            onClick={() => setTab('diagram')}
          >
            <Icon.schema size={14} aria-hidden="true" />
            ER Diagram
          </button>
        </div>

        <div className={styles['db-analytics-schema-panel__body']}>
          {loading && (
            <div className={styles['db-analytics-schema-empty']}>
              <Icon.loader size={22} aria-hidden="true" className={styles['db-analytics-schema-spin']} />
              <p>Loading schema…</p>
            </div>
          )}

          {!loading && error && (
            <div className={styles['db-analytics-schema-empty']}>
              <Icon.infoCircle size={22} aria-hidden="true" />
              <p>{error}</p>
            </div>
          )}

          {!loading && !error && schema && schema.tables.length === 0 && (
            <div className={styles['db-analytics-schema-empty']}>
              <Icon.table size={22} aria-hidden="true" />
              <p>No tables found for this database.</p>
            </div>
          )}

          {!loading && !error && schema && schema.tables.length > 0 && tab === 'tables' && (
            <>
              <div className={styles['db-analytics-schema-summary']}>
                <span>
                  <strong>{schema.tables.length}</strong> table{schema.tables.length !== 1 ? 's' : ''}
                </span>
                <span className={styles['db-analytics-schema-summary__dot']}>&middot;</span>
                <span>{totalRows.toLocaleString()} total rows</span>
              </div>

              <div className={styles['db-analytics-schema-table-list']}>
                {schema.tables.map((table) => {
                  const isOpen = expandedTable === table.name;
                  return (
                    <div key={table.name} className={styles['db-analytics-schema-table']}>
                      <button
                        className={styles['db-analytics-schema-table__header']}
                        onClick={() => setExpandedTable(isOpen ? null : table.name)}
                        aria-expanded={isOpen}
                      >
                        <span className={styles['db-analytics-schema-table__icon']}>
                          <Icon.table size={13} aria-hidden="true" />
                        </span>
                        <span className={styles['db-analytics-schema-table__name']}>{table.name}</span>
                        <span className={styles['db-analytics-schema-table__meta']}>
                          {table.columns.length} col{table.columns.length !== 1 ? 's' : ''} &middot;{' '}
                          {table.row_count.toLocaleString()} rows
                        </span>
                        <Icon.arrowRight
                          size={13}
                          aria-hidden="true"
                          className={`${styles['db-analytics-schema-table__chevron']} ${
                            isOpen ? styles['db-analytics-schema-table__chevron--open'] : ''
                          }`}
                        />
                      </button>

                      {isOpen && (
                        <div className={styles['db-analytics-schema-table__body']}>
                          <table className={styles['db-analytics-schema-columns']}>
                            <thead>
                              <tr>
                                <th>Column</th>
                                <th>Type</th>
                                <th>Nullable</th>
                                <th>Default</th>
                              </tr>
                            </thead>
                            <tbody>
                              {table.columns.map((col) => (
                                <tr key={col.name}>
                                  <td>
                                    <span className={styles['db-analytics-schema-columns__name']}>
                                      {col.primary_key && (
                                        <span
                                          className={styles['db-analytics-schema-columns__pk']}
                                          title="Primary key"
                                        >
                                          PK
                                        </span>
                                      )}
                                      {col.name}
                                    </span>
                                  </td>
                                  <td>
                                    <span className={styles['db-analytics-schema-columns__type']}>{col.type}</span>
                                  </td>
                                  <td>{col.nullable ? 'Yes' : 'No'}</td>
                                  <td>{col.default ?? '—'}</td>
                                </tr>
                              ))}
                            </tbody>
                          </table>
                        </div>
                      )}
                    </div>
                  );
                })}
              </div>
            </>
          )}

          {!loading && !error && schema && schema.tables.length > 0 && tab === 'diagram' && (
            <SchemaErDiagram tables={schema.tables} />
          )}
        </div>
      </aside>
    </div>
  );
};

export default DbSchemaSlider;


















//SchemaErDiagram.module.scss
@use '../../../styles/tokens' as t;

.db-analytics-er-diagram {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.db-analytics-er-diagram__scroll {
  overflow: auto;
  border: 1px solid t.$border-strong;
  border-radius: t.$radius-lg;
  background: t.$surface-1;
  padding: 12px;
}

.db-analytics-er-diagram__svg {
  display: block;
}

.db-analytics-er-diagram__card {
  fill: t.$surface-2;
  stroke: t.$border-strong;
  stroke-width: 1;
}

.db-analytics-er-diagram__card-header {
  fill: t.$accent-bg;
}

.db-analytics-er-diagram__card-title {
  font-size: 12px;
  font-weight: 700;
  fill: t.$text-primary;
  font-family: inherit;
}

.db-analytics-er-diagram__card-count {
  font-size: 10px;
  font-weight: 500;
  fill: t.$text-muted;
  font-family: inherit;
}

.db-analytics-er-diagram__row-alt {
  fill: t.$surface-1;
}

.db-analytics-er-diagram__pk {
  font-size: 9px;
  font-weight: 700;
  fill: t.$accent;
  font-family: inherit;
}

.db-analytics-er-diagram__col-name {
  font-size: 11px;
  fill: t.$text-secondary;
  font-family: inherit;
}

.db-analytics-er-diagram__col-type {
  font-size: 10px;
  fill: t.$text-muted;
  font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
}

.db-analytics-er-diagram__more {
  font-size: 10px;
  font-style: italic;
  fill: t.$text-muted;
  font-family: inherit;
}

.db-analytics-er-diagram__link {
  fill: none;
  stroke: t.$accent;
  stroke-width: 1.5;
  opacity: 0.55;
  marker-end: url(#er-arrow);
}

.db-analytics-er-diagram__arrow {
  fill: t.$accent;
  opacity: 0.7;
}

.db-analytics-er-diagram__note {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 11.5px;
  color: t.$text-muted;
  padding: 8px 2px;
}

















//SchemaErDiagram.tsx
import React, { useMemo } from 'react';
import styles from './SchemaErDiagram.module.scss';
import { DbTable } from '../types';
import { Icon } from '../icons';

interface Props {
  tables: DbTable[];
}

interface Relationship {
  fromTable: string;
  fromColumn: string;
  toTable: string;
  toColumn: string;
}

// Infers foreign-key-style relationships purely from naming conventions,
// since the schema endpoint doesn't return explicit FK metadata. A column
// like `customer_id` on `orders` is treated as pointing at the `id` (or
// `customer_id`) primary key column on a table named `customer`/`customers`.
function inferRelationships(tables: DbTable[]): Relationship[] {
  const tableByName = new Map(tables.map((t) => [t.name.toLowerCase(), t]));
  const relationships: Relationship[] = [];

  for (const table of tables) {
    for (const column of table.columns) {
      if (column.primary_key) continue;
      const match = column.name.match(/^(.+?)_id$/i);
      if (!match) continue;

      const base = match[1].toLowerCase();
      const candidates = [base, `${base}s`, base.replace(/s$/, '')];
      const targetName = candidates.find((c) => tableByName.has(c) && c !== table.name.toLowerCase());
      if (!targetName) continue;

      const targetTable = tableByName.get(targetName);
      if (!targetTable) continue;

      const targetPk = targetTable.columns.find((c) => c.primary_key) ?? targetTable.columns[0];
      if (!targetPk) continue;

      relationships.push({
        fromTable: table.name,
        fromColumn: column.name,
        toTable: targetTable.name,
        toColumn: targetPk.name,
      });
    }
  }

  return relationships;
}

const CARD_WIDTH = 220;
const CARD_HEADER_HEIGHT = 34;
const ROW_HEIGHT = 22;
const COL_GAP_X = 80;
const ROW_GAP_Y = 40;
const MAX_VISIBLE_COLUMNS = 6;

const SchemaErDiagram: React.FC<Props> = ({ tables }) => {
  const relationships = useMemo(() => inferRelationships(tables), [tables]);

  // Simple grid layout: wrap tables into rows of up to 3, sized to fit each
  // table's own column count so nothing overlaps.
  const layout = useMemo(() => {
    const perRow = tables.length <= 4 ? 2 : 3;
    const positions = new Map<string, { x: number; y: number; height: number }>();
    const rowHeights: number[] = [];
    let currentRowMax = 0;

    tables.forEach((table, i) => {
      const visibleCols = Math.min(table.columns.length, MAX_VISIBLE_COLUMNS);
      const extra = table.columns.length > MAX_VISIBLE_COLUMNS ? ROW_HEIGHT : 0;
      const height = CARD_HEADER_HEIGHT + visibleCols * ROW_HEIGHT + extra + 10;
      currentRowMax = Math.max(currentRowMax, height);

      if ((i + 1) % perRow === 0 || i === tables.length - 1) {
        rowHeights.push(currentRowMax);
        currentRowMax = 0;
      }
    });

    let rowIndex = 0;
    let colIndex = 0;
    let yOffset = 20;

    tables.forEach((table, i) => {
      const visibleCols = Math.min(table.columns.length, MAX_VISIBLE_COLUMNS);
      const extra = table.columns.length > MAX_VISIBLE_COLUMNS ? ROW_HEIGHT : 0;
      const height = CARD_HEADER_HEIGHT + visibleCols * ROW_HEIGHT + extra + 10;

      const x = 24 + colIndex * (CARD_WIDTH + COL_GAP_X);
      const y = yOffset;
      positions.set(table.name, { x, y, height });

      colIndex++;
      if ((i + 1) % perRow === 0 || i === tables.length - 1) {
        yOffset += rowHeights[rowIndex] + ROW_GAP_Y;
        rowIndex++;
        colIndex = 0;
      }
    });

    const totalWidth = 24 * 2 + perRow * CARD_WIDTH + (perRow - 1) * COL_GAP_X;
    const totalHeight = yOffset + 20;

    return { positions, totalWidth, totalHeight };
  }, [tables]);

  const getColumnY = (tableName: string, columnName: string): number | null => {
    const pos = layout.positions.get(tableName);
    const table = tables.find((t) => t.name === tableName);
    if (!pos || !table) return null;
    const idx = table.columns.findIndex((c) => c.name === columnName);
    if (idx === -1 || idx >= MAX_VISIBLE_COLUMNS) return pos.y + CARD_HEADER_HEIGHT / 2;
    return pos.y + CARD_HEADER_HEIGHT + idx * ROW_HEIGHT + ROW_HEIGHT / 2;
  };

  if (tables.length === 0) return null;

  return (
    <div className={styles['db-analytics-er-diagram']}>
      <div className={styles['db-analytics-er-diagram__scroll']}>
        <svg
          width={layout.totalWidth}
          height={layout.totalHeight}
          viewBox={`0 0 ${layout.totalWidth} ${layout.totalHeight}`}
          className={styles['db-analytics-er-diagram__svg']}
        >
          <defs>
            <marker id="er-arrow" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
              <path d="M0,0 L6,3 L0,6 Z" className={styles['db-analytics-er-diagram__arrow']} />
            </marker>
          </defs>

          {relationships.map((rel, i) => {
            const fromPos = layout.positions.get(rel.fromTable);
            const toPos = layout.positions.get(rel.toTable);
            const fromY = getColumnY(rel.fromTable, rel.fromColumn);
            const toY = getColumnY(rel.toTable, rel.toColumn);
            if (!fromPos || !toPos || fromY === null || toY === null) return null;

            const fromOnLeft = fromPos.x < toPos.x;
            const startX = fromOnLeft ? fromPos.x + CARD_WIDTH : fromPos.x;
            const endX = fromOnLeft ? toPos.x : toPos.x + CARD_WIDTH;
            const midX = (startX + endX) / 2;

            const path = `M ${startX} ${fromY} C ${midX} ${fromY}, ${midX} ${toY}, ${endX} ${toY}`;

            return (
              <path
                key={i}
                d={path}
                className={styles['db-analytics-er-diagram__link']}
                markerEnd="url(#er-arrow)"
              />
            );
          })}

          {tables.map((table) => {
            const pos = layout.positions.get(table.name);
            if (!pos) return null;
            const visibleColumns = table.columns.slice(0, MAX_VISIBLE_COLUMNS);
            const hiddenCount = table.columns.length - visibleColumns.length;

            return (
              <g key={table.name} transform={`translate(${pos.x}, ${pos.y})`}>
                <rect
                  width={CARD_WIDTH}
                  height={pos.height}
                  rx={10}
                  className={styles['db-analytics-er-diagram__card']}
                />
                <rect width={CARD_WIDTH} height={CARD_HEADER_HEIGHT} rx={10} className={styles['db-analytics-er-diagram__card-header']} />
                <rect y={CARD_HEADER_HEIGHT - 10} width={CARD_WIDTH} height={10} className={styles['db-analytics-er-diagram__card-header']} />
                <text x={12} y={CARD_HEADER_HEIGHT / 2 + 4} className={styles['db-analytics-er-diagram__card-title']}>
                  {table.name}
                </text>
                <text
                  x={CARD_WIDTH - 12}
                  y={CARD_HEADER_HEIGHT / 2 + 4}
                  textAnchor="end"
                  className={styles['db-analytics-er-diagram__card-count']}
                >
                  {table.row_count.toLocaleString()}
                </text>

                {visibleColumns.map((col, i) => (
                  <g key={col.name} transform={`translate(0, ${CARD_HEADER_HEIGHT + i * ROW_HEIGHT})`}>
                    {i % 2 === 1 && (
                      <rect width={CARD_WIDTH} height={ROW_HEIGHT} className={styles['db-analytics-er-diagram__row-alt']} />
                    )}
                    {col.primary_key && (
                      <text x={12} y={ROW_HEIGHT / 2 + 4} className={styles['db-analytics-er-diagram__pk']}>
                        PK
                      </text>
                    )}
                    <text
                      x={col.primary_key ? 34 : 12}
                      y={ROW_HEIGHT / 2 + 4}
                      className={styles['db-analytics-er-diagram__col-name']}
                    >
                      {col.name}
                    </text>
                    <text
                      x={CARD_WIDTH - 12}
                      y={ROW_HEIGHT / 2 + 4}
                      textAnchor="end"
                      className={styles['db-analytics-er-diagram__col-type']}
                    >
                      {col.type}
                    </text>
                  </g>
                ))}

                {hiddenCount > 0 && (
                  <text
                    x={12}
                    y={CARD_HEADER_HEIGHT + visibleColumns.length * ROW_HEIGHT + ROW_HEIGHT / 2 + 4}
                    className={styles['db-analytics-er-diagram__more']}
                  >
                    +{hiddenCount} more column{hiddenCount !== 1 ? 's' : ''}
                  </text>
                )}
              </g>
            );
          })}
        </svg>
      </div>

      {relationships.length === 0 && (
        <div className={styles['db-analytics-er-diagram__note']}>
          <Icon.infoCircle size={13} aria-hidden="true" />
          No relationships could be inferred from column names — tables are shown without connecting lines.
        </div>
      )}
    </div>
  );
};

export default SchemaErDiagram;












//icons.tsx
// Central place for every SVG icon used across the DBAnalytics feature.
// Swaps the old Tabler icon-font (`<i className="ti ti-x" />`) for real
// inline SVG components from lucide-react.
import React from 'react';
import {
  Plus,
  X,
  Check,
  Database,
  DatabaseZap,
  Settings,
  Info,
  Loader2,
  MessageCircle,
  Sparkles,
  BarChart3,
  Send,
  AreaChart,
  Gauge,
  LineChart,
  PieChart,
  Table,
  ScatterChart,
  Lightbulb,
  ArrowRight,
  Upload,
  FileSpreadsheet,
  Maximize2,
  Network,
  type LucideProps,
} from 'lucide-react';

export type IconComponent = React.FC<LucideProps>;

export const Icon = {
  plus: Plus,
  close: X,
  check: Check,
  database: Database,
  databaseInsights: DatabaseZap,
  settings: Settings,
  infoCircle: Info,
  loader: Loader2,
  messageCircle: MessageCircle,
  sparkles: Sparkles,
  chartBar: BarChart3,
  send: Send,
  chartAreaLine: AreaChart,
  gauge: Gauge,
  chartLine: LineChart,
  chartPie: PieChart,
  chartArea: AreaChart,
  chartDots: ScatterChart,
  table: Table,
  lightbulb: Lightbulb,
  arrowRight: ArrowRight,
  upload: Upload,
  csv: FileSpreadsheet,
  expand: Maximize2,
  askAi: Sparkles,
  schema: Network,
} as const;

export type IconName = keyof typeof Icon;






//api.ts

// TODO: point this at your project's actual configured axios instance
// (the one with the auth-token interceptor already set up).
import { http } from '../lib/http';

const API_PREFIX = '/db-analytics';

export interface ApiDatabase {
  id: string;
  name: string;
  db_type: string;
  host?: string;
  port: number;
  db_name: string;
  username: string;
  created_at: string;
  connected: boolean;
  cache_until: string | null;
}

export interface ApiSession {
  id: string;
  name: string;
  db_ids: string[];
  created_at: string;
}

export interface ApiColumn {
  name: string;
  type: string;
  nullable: boolean;
  primary_key: boolean;
  default: string | null;
}

export interface ApiTable {
  name: string;
  columns: ApiColumn[];
  row_count: number;
}

export interface ApiSchema {
  db_id: string;
  db_name: string;
  tables: ApiTable[];
}

export interface CreateDbPayload {
  db_name: string;
  db_type: 'mysql' | 'postgres';
  host: string;
  name: string;
  port: number;
  username: string;
}

export interface CreateSessionPayload {
  name: string;
  db_ids: string[];
}

// Thrown when POST /sessions/:id/query responds 424 because one or more of
// the session's databases isn't currently connected. Carries the specific
// db ids so the UI can point at exactly which connections need attention.
export class MissingDbConnectionError extends Error {
  missingDbIds: string[];

  constructor(message: string, missingDbIds: string[]) {
    super(message);
    this.name = 'MissingDbConnectionError';
    this.missingDbIds = missingDbIds;
  }
}

export interface UploadCsvPayload {
  file: File;
  name: string;
  table_name: string;
}

export interface ApiGraphQuery {
  db_id: string;
  db_name: string;
  sql: string;
  row_count: number;
}

export type ApiChartType = 'kpi' | 'line' | 'bar' | 'pie' | 'area' | 'scatter' | 'table';

export interface ApiGraph {
  id: string;
  chart_type: ApiChartType;
  title: string;
  x_axis: string | null;
  y_axis: string | null;
  series: string[] | null;
  stacked: boolean;
  data: Record<string, string | number>[];
  queries: ApiGraphQuery[];
}

export interface ApiMessage {
  id: string;
  role: 'user' | 'assistant';
  content: string;
  graphs: ApiGraph[];
  created_at: string;
}

export interface ApiSuggestedPrompt {
  text: string;
  label: string;
  db_name: string;
}

export interface ApiSuggestedPromptsResponse {
  prompts: ApiSuggestedPrompt[];
}

export interface ApiQueryStepEvent {
  type: 'step';
  step: string;
}

export interface ApiQueryMessageEvent {
  type: 'message';
  step: string;
}

export interface ApiQueryDoneEvent {
  type: 'done';
}

export interface ApiQueryErrorEvent {
  type: 'error';
  message: string;
}

export type ApiQueryEvent =
  | ApiQueryStepEvent
  | ApiQueryMessageEvent
  | ApiQueryDoneEvent
  | ApiQueryErrorEvent;

export interface ApiFollowupsResponse {
  followups: string[];
}

export interface ChartInsightPayload {
  chart: ApiGraph;
  question: string;
}

export interface ChartInsightResponse {
  insight: string;
}

// Some list endpoints inconsistently respond either as a bare array
// (`[...]`) or wrapped in an envelope (`{ status: "Success", result: [...] }`).
// This normalizes either shape into a plain array so callers never have to
// guess which one they got — and never crash calling array methods on an
// object if the envelope shape shows up unexpectedly.
function unwrapList<T>(payload: unknown): T[] {
  if (Array.isArray(payload)) return payload;
  if (payload && typeof payload === 'object' && Array.isArray((payload as { result?: unknown }).result)) {
    return (payload as { result: T[] }).result;
  }
  return [];
}

async function getList<T>(path: string): Promise<T[]> {
  const res = await http.get(`${API_PREFIX}${path}`);
  return unwrapList<T>(res.data);
}

// Guarantees db_ids is always a real array. The backend has been observed to
// omit it or send null for sessions with no databases attached yet (e.g.
// right after creation), and `.includes()`/`.filter()` on undefined throws.
function normalizeSession(raw: Partial<ApiSession> & { id: string }): ApiSession {
  return {
    id: raw.id,
    name: raw.name ?? '',
    db_ids: Array.isArray(raw.db_ids) ? raw.db_ids : [],
    created_at: raw.created_at ?? new Date().toISOString(),
  };
}

// Extracts a human-readable message from an axios error response, whatever
// shape the backend used (`detail` as a string, `message`, or nothing at all).
function extractErrorMessage(err: unknown, fallback: string): string {
  const anyErr = err as { response?: { data?: { message?: string; detail?: unknown } }; message?: string };
  const data = anyErr?.response?.data;
  if (typeof data?.detail === 'string') return data.detail;
  if (typeof data?.message === 'string') return data.message;
  if (typeof anyErr?.message === 'string') return anyErr.message;
  return fallback;
}

// Parses a streaming response body into individual JSON payloads as they
// arrive. Two framing styles are supported, detected automatically per line:
//  1) Newline-delimited JSON — one `{...}` object per line (no `data:` prefix,
//     no blank-line separators). This is what a lot of simple SSE-flavored
//     backends actually send even when the content-type says event-stream.
//  2) Standard SSE framing — `data: {...}` lines, events separated by a
//     blank line, potentially multi-line data blocks.
// Critically, this parses and yields as soon as a complete line/frame is
// available rather than waiting for the whole response to buffer, so the
// consumer can render each event the moment it arrives instead of getting
// everything dumped at once when the stream closes.
async function* parseEventStream(res: Response): AsyncGenerator<ApiQueryEvent> {
  if (!res.body) return;
  const reader = res.body.getReader();
  const decoder = new TextDecoder();
  let buffer = '';
  let sseDataBuffer: string[] = [];

  const tryParse = (raw: string): ApiQueryEvent | null => {
    const trimmed = raw.trim();
    if (!trimmed) return null;
    try {
      return JSON.parse(trimmed) as ApiQueryEvent;
    } catch {
      return null;
    }
  };

  const flushSseBuffer = function* (): Generator<ApiQueryEvent> {
    if (sseDataBuffer.length === 0) return;
    const joined = sseDataBuffer.join('\n');
    sseDataBuffer = [];
    const parsed = tryParse(joined);
    if (parsed) yield parsed;
  };

  const processLine = function* (line: string): Generator<ApiQueryEvent> {
    if (line.startsWith('data:')) {
      // Standard SSE frame: accumulate until a blank line flushes it.
      sseDataBuffer.push(line.slice(5).trim());
      return;
    }
    if (line.trim() === '') {
      // Blank line = SSE event boundary.
      yield* flushSseBuffer();
      return;
    }
    // Not SSE-prefixed — try treating the line as a standalone JSON event
    // (newline-delimited JSON framing).
    const parsed = tryParse(line);
    if (parsed) {
      yield parsed;
    }
  };

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    buffer += decoder.decode(value, { stream: true });

    const lines = buffer.split('\n');
    // Keep the last (possibly incomplete) line in the buffer for next read.
    buffer = lines.pop() ?? '';

    for (const line of lines) {
      yield* processLine(line);
    }
  }

  // Flush anything left after the stream closes.
  if (buffer.trim()) {
    yield* processLine(buffer);
  }
  yield* flushSseBuffer();
}

// NOTE: this one deliberately stays on raw `fetch` instead of the axios
// instance. Axios buffers the full response body before resolving (or needs
// extra adapter config to expose a readable stream), which defeats the
// purpose here — we need to read and yield each SSE/NDJSON chunk as it
// arrives so the UI can render step-by-step progress in real time. `fetch`'s
// `res.body` ReadableStream gives us that directly. The auth header is
// applied manually below since this bypasses the axios interceptor.
async function* streamQuery(sessionId: string, query: string): AsyncGenerator<ApiQueryEvent> {
  const authHeader = http.defaults.headers.common?.['Authorization'];

  const res = await fetch(`${http.defaults.baseURL ?? ''}${API_PREFIX}/sessions/${sessionId}/query`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      ...(authHeader ? { Authorization: String(authHeader) } : {}),
    },
    body: JSON.stringify({ query }),
  });

  if (!res.ok) {
    let detail = '';
    try {
      const body = await res.json();
      // 424 shape: { detail: { message: string, missing_db_ids: string[] } }
      if (res.status === 424 && body?.detail && typeof body.detail === 'object') {
        const message = body.detail.message || 'One or more databases are not connected.';
        const missingDbIds = Array.isArray(body.detail.missing_db_ids) ? body.detail.missing_db_ids : [];
        throw new MissingDbConnectionError(message, missingDbIds);
      }
      detail = typeof body?.detail === 'string' ? body.detail : body?.message || '';
    } catch (err) {
      if (err instanceof MissingDbConnectionError) throw err;
      // ignore parse errors from a non-JSON error body
    }
    throw new Error(detail || `Request failed with status ${res.status}`);
  }

  yield* parseEventStream(res);
}

// Mirrors unwrapList, but for single-object responses. Some create/action
// endpoints have been observed to wrap their result the same way list
// endpoints do (`{ status: "Success", result: {...} }`) instead of
// returning the object directly — this normalizes either shape.
function unwrapObject<T extends object>(payload: unknown): T {
  if (payload && typeof payload === 'object' && !Array.isArray(payload)) {
    const withResult = payload as { result?: unknown; status?: unknown };
    if (withResult.result && typeof withResult.result === 'object' && !Array.isArray(withResult.result)) {
      return withResult.result as T;
    }
  }
  return payload as T;
}

// Guarantees every field the UI relies on is present with a sane default,
// even if the create/upload response is missing some of them.
function normalizeDatabase(raw: unknown): ApiDatabase {
  const obj = unwrapObject<Partial<ApiDatabase>>(raw) ?? {};
  return {
    id: obj.id ?? '',
    name: obj.name ?? '',
    db_type: obj.db_type ?? '',
    host: obj.host,
    port: obj.port ?? 0,
    db_name: obj.db_name ?? '',
    username: obj.username ?? '',
    created_at: obj.created_at ?? new Date().toISOString(),
    connected: obj.connected ?? false,
    cache_until: obj.cache_until ?? null,
  };
}

async function uploadCsv(payload: UploadCsvPayload): Promise<ApiDatabase> {
  const formData = new FormData();
  formData.append('file', payload.file);
  formData.append('name', payload.name);
  formData.append('table_name', payload.table_name);

  try {
    const res = await http.post(`${API_PREFIX}/dbs/upload-csv`, formData, {
      headers: {
        // Let axios/the browser set the multipart boundary automatically —
        // don't hardcode 'multipart/form-data' here or the boundary gets lost.
        'Content-Type': 'multipart/form-data',
      },
    });
    return normalizeDatabase(res.data);
  } catch (err) {
    throw new Error(extractErrorMessage(err, 'Failed to upload CSV file.'));
  }
}

export const api = {
  listDatabases: () => getList<ApiDatabase>('/dbs'),

  listSessions: async () => {
    const raw = await getList<Partial<ApiSession> & { id: string }>('/sessions');
    return raw.map(normalizeSession);
  },

  createDatabase: async (payload: CreateDbPayload): Promise<ApiDatabase> => {
    try {
      const res = await http.post(`${API_PREFIX}/dbs`, payload);
      return normalizeDatabase(res.data);
    } catch (err) {
      throw new Error(extractErrorMessage(err, 'Failed to create database.'));
    }
  },

  uploadCsv,

  connectDatabase: async (
    dbId: string,
    password: string
  ): Promise<{ message: string; db_id: string; cache_until: string }> => {
    try {
      const res = await http.post(`${API_PREFIX}/dbs/${dbId}/connect`, { password });
      return res.data;
    } catch (err) {
      throw new Error(extractErrorMessage(err, 'Failed to connect.'));
    }
  },

  disconnectDatabase: async (dbId: string): Promise<{ message: string; db_id: string }> => {
    try {
      const res = await http.post(`${API_PREFIX}/dbs/${dbId}/disconnect`);
      return res.data;
    } catch (err) {
      throw new Error(extractErrorMessage(err, 'Failed to disconnect.'));
    }
  },

  getSchema: async (dbId: string): Promise<ApiSchema> => {
    const res = await http.get(`${API_PREFIX}/dbs/${dbId}/schema`);
    const body = unwrapObject<Partial<ApiSchema>>(res.data) ?? {};
    return {
      db_id: body.db_id ?? dbId,
      db_name: body.db_name ?? '',
      tables: Array.isArray(body.tables) ? body.tables : [],
    };
  },

  createSession: async (payload: CreateSessionPayload): Promise<ApiSession> => {
    try {
      const res = await http.post(`${API_PREFIX}/sessions`, payload);
      return normalizeSession(res.data);
    } catch (err) {
      throw new Error(extractErrorMessage(err, 'Failed to create session.'));
    }
  },

  getMessages: (sessionId: string) => getList<ApiMessage>(`/sessions/${sessionId}/messages`),

  getSuggestedPrompts: async (sessionId: string): Promise<ApiSuggestedPromptsResponse> => {
    const res = await http.post(`${API_PREFIX}/sessions/${sessionId}/suggested-prompts`);
    const body = unwrapObject<{ prompts?: unknown }>(res.data) ?? {};
    return { prompts: Array.isArray(body.prompts) ? (body.prompts as ApiSuggestedPrompt[]) : [] };
  },

  streamQuery,

  getFollowups: async (sessionId: string): Promise<ApiFollowupsResponse> => {
    const res = await http.post(`${API_PREFIX}/sessions/${sessionId}/followups`);
    const body = unwrapObject<{ followups?: unknown }>(res.data) ?? {};
    return { followups: Array.isArray(body.followups) ? (body.followups as string[]) : [] };
  },

  getChartInsight: async (sessionId: string, payload: ChartInsightPayload): Promise<ChartInsightResponse> => {
    try {
      const res = await http.post(`${API_PREFIX}/sessions/${sessionId}/chart_insight`, payload);
      const body = unwrapObject<{ insight?: unknown }>(res.data) ?? {};
      return { insight: typeof body.insight === 'string' ? body.insight : '' };
    } catch (err) {
      throw new Error(extractErrorMessage(err, 'Failed to get an insight for this question.'));
    }
  },
};
