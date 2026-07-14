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
  width: 800px;
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

.db-analytics-er-diagram__link {
  fill: none;
  stroke-width: 2;
  opacity: 0.7;
  transition: opacity 0.15s ease, stroke-width 0.15s ease;
}

.db-analytics-er-diagram__link-group:hover .db-analytics-er-diagram__link {
  opacity: 1;
  stroke-width: 2.75;
}

.db-analytics-er-diagram__link-badge {
  stroke: t.$surface-1;
  stroke-width: 2;
}

.db-analytics-er-diagram__link-badge-text {
  font-size: 7px;
  font-weight: 800;
  fill: #fff;
  font-family: inherit;
  letter-spacing: 0.02em;
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
import { useTranslation } from 'react-i18next';
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

const CARD_WIDTH = 240;
const CARD_HEADER_HEIGHT = 34;
const ROW_HEIGHT = 24;
const COL_GAP_X = 110;
const ROW_GAP_Y = 56;

// A distinct color per relationship line, cycling through a small palette
// so that when several tables connect to the same hub table, each line
// stays visually distinguishable rather than blurring into one mass.
const LINK_COLORS = ['#4f46e5', '#0aa36f', '#e08a00', '#dc4646', '#0c8fce', '#a238c9', '#d1385c'];

const SchemaErDiagram: React.FC<Props> = ({ tables }) => {
  const { t } = useTranslation();
  const relationships = useMemo(() => inferRelationships(tables), [tables]);

  // Grid layout: wrap tables into rows, sized to fully fit each table's own
  // column count (no truncation) so nothing overlaps and every column is
  // visible without a "+n more" fallback.
  const layout = useMemo(() => {
    const perRow = tables.length <= 3 ? tables.length || 1 : tables.length <= 6 ? 3 : 4;
    const positions = new Map<string, { x: number; y: number; height: number }>();
    const rowHeights: number[] = [];
    let currentRowMax = 0;

    tables.forEach((table, i) => {
      const height = CARD_HEADER_HEIGHT + table.columns.length * ROW_HEIGHT + 10;
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
      const height = CARD_HEADER_HEIGHT + table.columns.length * ROW_HEIGHT + 10;
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

    const totalWidth = 24 * 2 + perRow * CARD_WIDTH + Math.max(0, perRow - 1) * COL_GAP_X;
    const totalHeight = yOffset + 20;

    return { positions, totalWidth, totalHeight, perRow };
  }, [tables]);

  const getColumnY = (tableName: string, columnName: string): number | null => {
    const pos = layout.positions.get(tableName);
    const table = tables.find((t) => t.name === tableName);
    if (!pos || !table) return null;
    const idx = table.columns.findIndex((c) => c.name === columnName);
    if (idx === -1) return pos.y + CARD_HEADER_HEIGHT / 2;
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
            {LINK_COLORS.map((color, i) => (
              <marker
                key={color}
                id={`er-arrow-${i}`}
                markerWidth="9"
                markerHeight="9"
                refX="7"
                refY="3.5"
                orient="auto"
              >
                <path d="M0,0 L7,3.5 L0,7 Z" fill={color} />
              </marker>
            ))}
          </defs>

          {relationships.map((rel, i) => {
            const fromPos = layout.positions.get(rel.fromTable);
            const toPos = layout.positions.get(rel.toTable);
            const fromY = getColumnY(rel.fromTable, rel.fromColumn);
            const toY = getColumnY(rel.toTable, rel.toColumn);
            if (!fromPos || !toPos || fromY === null || toY === null) return null;

            const color = LINK_COLORS[i % LINK_COLORS.length];
            const sameRow = Math.abs(fromPos.y - toPos.y) < 4;
            const fromOnLeft = fromPos.x < toPos.x;

            let path: string;
            let labelX: number;
            let labelY: number;

            if (sameRow || fromPos.x !== toPos.x) {
              // Tables sit in different columns — connect the facing
              // vertical edges with a smooth S-curve, attaching to the
              // exact row of the relevant column on each side.
              const startX = fromOnLeft ? fromPos.x + CARD_WIDTH : fromPos.x;
              const endX = fromOnLeft ? toPos.x : toPos.x + CARD_WIDTH;
              const midX = (startX + endX) / 2;
              path = `M ${startX} ${fromY} C ${midX} ${fromY}, ${midX} ${toY}, ${endX} ${toY}`;
              labelX = midX;
              labelY = (fromY + toY) / 2;
            } else {
              // Tables stacked in the same column — route around the side
              // instead of drawing straight through the cards in between.
              const bulgeX = fromPos.x + CARD_WIDTH + 46 + (i % 3) * 14;
              path = `M ${fromPos.x + CARD_WIDTH} ${fromY} C ${bulgeX} ${fromY}, ${bulgeX} ${toY}, ${toPos.x + CARD_WIDTH} ${toY}`;
              labelX = bulgeX;
              labelY = (fromY + toY) / 2;
            }

            return (
              <g key={i} className={styles['db-analytics-er-diagram__link-group']}>
                <path
                  d={path}
                  className={styles['db-analytics-er-diagram__link']}
                  style={{ stroke: color }}
                  markerEnd={`url(#er-arrow-${i % LINK_COLORS.length})`}
                />
                <circle cx={labelX} cy={labelY} r={9} className={styles['db-analytics-er-diagram__link-badge']} style={{ fill: color }} />
                <text
                  x={labelX}
                  y={labelY + 3}
                  textAnchor="middle"
                  className={styles['db-analytics-er-diagram__link-badge-text']}
                >
                  FK
                </text>
              </g>
            );
          })}

          {tables.map((table) => {
            const pos = layout.positions.get(table.name);
            if (!pos) return null;

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

                {table.columns.map((col, i) => (
                  <g key={col.name} transform={`translate(0, ${CARD_HEADER_HEIGHT + i * ROW_HEIGHT})`}>
                    {i % 2 === 1 && (
                      <rect width={CARD_WIDTH} height={ROW_HEIGHT} className={styles['db-analytics-er-diagram__row-alt']} />
                    )}
                    {col.primary_key && (
                      <text x={12} y={ROW_HEIGHT / 2 + 4} className={styles['db-analytics-er-diagram__pk']}>
                        {t('db_analytics.dbSchemaSlider.table.pkAbbr')}
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
              </g>
            );
          })}
        </svg>
      </div>

      {relationships.length === 0 && (
        <div className={styles['db-analytics-er-diagram__note']}>
          <Icon.infoCircle size={13} aria-hidden="true" />
          {t('db_analytics.schemaErDiagram.noRelationships')}
        </div>
      )}
    </div>
  );
};

export default SchemaErDiagram;
