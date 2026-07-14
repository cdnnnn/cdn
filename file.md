//DbSchemaSlider.tsx
import React, { useEffect, useMemo, useState } from 'react';
import { useTranslation } from 'react-i18next';
import styles from './DbSchemaSlider.module.scss';
import { DatabaseItem, DbSchema } from '../types';
import { Icon } from '../icons';
import { api } from '../../../services/api';
import SchemaErDiagram from './SchemaErDiagram';
import { useResizableWidth } from '../hooks/useResizableWidth';

interface Props {
  database: DatabaseItem | null;
  onClose: () => void;
}

type Tab = 'tables' | 'diagram';

const DbSchemaSlider: React.FC<Props> = ({ database, onClose }) => {
  const { t } = useTranslation();
  const [tab, setTab] = useState<Tab>('tables');
  const [schema, setSchema] = useState<DbSchema | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [expandedTable, setExpandedTable] = useState<string | null>(null);
  const { width, isResizing, handlePointerDown } = useResizableWidth({
    defaultWidth: 800,
    minWidth: 480,
    maxWidth: 'viewport',
  });
  // Sticky for the rest of this panel's lifetime once the person starts
  // dragging — not just tied to the live isResizing flag. If it were tied
  // to isResizing alone, the moment a drag ends the class list changes
  // again, which re-triggers the base slide-in @keyframes from its 0%
  // state (translateX(100%)) and produces a visible snap-back-then-forward
  // instead of just settling at the new width.
  const [hasResizedOnce, setHasResizedOnce] = useState(false);
  useEffect(() => {
    if (isResizing) setHasResizedOnce(true);
  }, [isResizing]);

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
        if (!cancelled) setError(err instanceof Error ? err.message : t('db_analytics.dbSchemaSlider.genericError'));
      })
      .finally(() => {
        if (!cancelled) setLoading(false);
      });

    return () => {
      cancelled = true;
    };
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [database?.id]);

  const totalRows = useMemo(
    () => schema?.tables.reduce((sum, t) => sum + (t.row_count ?? 0), 0) ?? 0,
    [schema]
  );

  if (!database) return null;

  return (
    <div className={styles['db-analytics-schema-overlay']} role="dialog" aria-modal="true">
      <div className={styles['db-analytics-schema-overlay__backdrop']} onClick={onClose} />
      <aside
        className={`${styles['db-analytics-schema-panel']} ${
          hasResizedOnce ? styles['db-analytics-schema-panel--resizing'] : ''
        }`}
        style={{ width: `${width}px` }}
      >
        <div
          className={`${styles['db-analytics-schema-panel__resize-handle']} ${
            isResizing ? styles['db-analytics-schema-panel__resize-handle--active'] : ''
          }`}
          onPointerDown={handlePointerDown}
          role="separator"
          aria-orientation="vertical"
          aria-label={t('db_analytics.dbSchemaSlider.resizeHandle')}
          title={t('db_analytics.dbSchemaSlider.resizeHandle')}
        />
        <div className={styles['db-analytics-schema-panel__header']}>
          <div className={styles['db-analytics-schema-panel__heading']}>
            <span className={styles['db-analytics-schema-panel__icon']}>
              <Icon.schema size={16} aria-hidden="true" />
            </span>
            <div>
              <h2 className={styles['db-analytics-schema-panel__title']}>{database.name}</h2>
              <span className={styles['db-analytics-schema-panel__subtitle']}>
                {t('db_analytics.dbSchemaSlider.subtitle')}
              </span>
            </div>
          </div>
          <button
            className={styles['db-analytics-schema-panel__close-btn']}
            aria-label={t('db_analytics.dbSchemaSlider.close')}
            onClick={onClose}
          >
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
            {t('db_analytics.dbSchemaSlider.tabs.tables')}
            {schema && <span className={styles['db-analytics-schema-tabs__count']}>{schema.tables.length}</span>}
          </button>
          <button
            className={`${styles['db-analytics-schema-tabs__item']} ${
              tab === 'diagram' ? styles['db-analytics-schema-tabs__item--active'] : ''
            }`}
            onClick={() => setTab('diagram')}
          >
            <Icon.schema size={14} aria-hidden="true" />
            {t('db_analytics.dbSchemaSlider.tabs.erDiagram')}
          </button>
        </div>

        <div className={styles['db-analytics-schema-panel__body']}>
          {loading && (
            <div className={styles['db-analytics-schema-empty']}>
              <Icon.loader size={22} aria-hidden="true" className={styles['db-analytics-schema-spin']} />
              <p>{t('db_analytics.dbSchemaSlider.loadingSchema')}</p>
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
              <p>{t('db_analytics.dbSchemaSlider.noTablesFound')}</p>
            </div>
          )}

          {!loading && !error && schema && schema.tables.length > 0 && tab === 'tables' && (
            <>
              <div className={styles['db-analytics-schema-summary']}>
                <span>
                  <strong>{schema.tables.length}</strong>{' '}
                  {schema.tables.length === 1 ? t('db_analytics.dbSchemaSlider.tableLabel') : t('db_analytics.dbSchemaSlider.tableLabelPlural')}
                </span>
                <span className={styles['db-analytics-schema-summary__dot']}>&middot;</span>
                <span>{t('db_analytics.dbSchemaSlider.totalRows', { count: totalRows })}</span>
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
                          {t('db_analytics.dbSchemaSlider.columnCount', { count: table.columns.length })} &middot;{' '}
                          {t('db_analytics.dbSchemaSlider.rowCount', { count: table.row_count })}
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
                                <th>{t('db_analytics.dbSchemaSlider.table.columnHeader')}</th>
                                <th>{t('db_analytics.dbSchemaSlider.table.typeHeader')}</th>
                                <th>{t('db_analytics.dbSchemaSlider.table.nullableHeader')}</th>
                                <th>{t('db_analytics.dbSchemaSlider.table.defaultHeader')}</th>
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
                                          title={t('db_analytics.dbSchemaSlider.table.primaryKey')}
                                        >
                                          {t('db_analytics.dbSchemaSlider.table.pkAbbr')}
                                        </span>
                                      )}
                                      {col.name}
                                    </span>
                                  </td>
                                  <td>
                                    <span className={styles['db-analytics-schema-columns__type']}>{col.type}</span>
                                  </td>
                                  <td>
                                    {col.nullable
                                      ? t('db_analytics.dbSchemaSlider.table.nullableYes')
                                      : t('db_analytics.dbSchemaSlider.table.nullableNo')}
                                  </td>
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





















//useResizableWidth.ts
import { useCallback, useEffect, useRef, useState } from 'react';

interface Options {
  defaultWidth: number;
  minWidth: number;
  // A fixed pixel cap, or 'viewport' to allow dragging all the way out to
  // the full browser width (re-evaluated live during the drag, so it stays
  // correct even if the window is resized while the panel is open).
  maxWidth: number | 'viewport';
  // The panel slides in from the right and the handle sits on its left
  // edge — dragging left (negative delta) should grow it, dragging right
  // should shrink it. Pass `direction: 'right'` for a panel anchored on the
  // left instead, where the relationship is the opposite.
  direction?: 'left' | 'right';
}

interface ResizableWidth {
  width: number;
  isResizing: boolean;
  handlePointerDown: (e: React.PointerEvent) => void;
}

// Drag-to-resize for a fixed-width side panel (a slide-over drawer, not a
// grid column). Tracks width in state, clamps to [minWidth, maxWidth], and
// attaches window-level pointer listeners only while actively dragging.
export function useResizableWidth({ defaultWidth, minWidth, maxWidth, direction = 'left' }: Options): ResizableWidth {
  const [width, setWidth] = useState(defaultWidth);
  const [isResizing, setIsResizing] = useState(false);
  const dragStartXRef = useRef(0);
  const dragStartWidthRef = useRef(defaultWidth);

  const handlePointerDown = useCallback(
    (e: React.PointerEvent) => {
      e.preventDefault();
      dragStartXRef.current = e.clientX;
      dragStartWidthRef.current = width;
      setIsResizing(true);
    },
    [width]
  );

  useEffect(() => {
    if (!isResizing) return;

    const handlePointerMove = (e: PointerEvent) => {
      const rawDelta = e.clientX - dragStartXRef.current;
      const delta = direction === 'left' ? -rawDelta : rawDelta;
      const effectiveMax = maxWidth === 'viewport' ? window.innerWidth : maxWidth;
      const next = Math.min(effectiveMax, Math.max(minWidth, dragStartWidthRef.current + delta));
      setWidth(next);
    };

    const handlePointerUp = () => setIsResizing(false);

    window.addEventListener('pointermove', handlePointerMove);
    window.addEventListener('pointerup', handlePointerUp);
    return () => {
      window.removeEventListener('pointermove', handlePointerMove);
      window.removeEventListener('pointerup', handlePointerUp);
    };
  }, [isResizing, minWidth, maxWidth, direction]);

  return { width, isResizing, handlePointerDown };
}
