//SchemaErDiagram.module.scss
@use '../../../styles/tokens' as t;

.db-analytics-er-diagram {
  display: flex;
  flex-direction: column;
  gap: 10px;
  position: relative;
}

.db-analytics-er-diagram__scroll {
  overflow: hidden;
  border: 1px solid t.$border-strong;
  border-radius: t.$radius-lg;
  background: t.$surface-1;
  width: 100%;
  height: 560px;
  cursor: grab;

  &:active {
    cursor: grabbing;
  }
}

.db-analytics-er-diagram__content {
  padding: 24px;
}

.db-analytics-er-diagram__svg {
  display: block;
}

.db-analytics-er-diagram__zoom-controls {
  position: absolute;
  top: 12px;
  right: 12px;
  z-index: 10;
  display: flex;
  align-items: center;
  gap: 2px;
  padding: 4px;
  border-radius: t.$radius;
  background: t.$surface-2;
  border: 1px solid t.$border-strong;
  box-shadow: 0 3px 10px rgba(11, 11, 11, 0.1);
}

.db-analytics-er-diagram__zoom-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 26px;
  height: 26px;
  border-radius: calc(t.$radius - 3px);
  border: none;
  background: transparent;
  color: t.$text-secondary;
  cursor: pointer;
  transition: background 0.12s ease, color 0.12s ease;

  &:hover {
    background: t.$surface-0;
    color: t.$text-primary;
  }
}

.db-analytics-er-diagram__zoom-level {
  min-width: 38px;
  text-align: center;
  font-size: 11px;
  font-weight: 600;
  color: t.$text-secondary;
  font-variant-numeric: tabular-nums;
}

.db-analytics-er-diagram__zoom-divider {
  width: 1px;
  height: 16px;
  background: t.$border-strong;
  margin: 0 2px;
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
import React, { useMemo, useState } from 'react';
import { useTranslation } from 'react-i18next';
import { TransformWrapper, TransformComponent, useControls } from 'react-zoom-pan-pinch';
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

// Must render inside <TransformWrapper> to access useControls(). A compact
// floating toolbar — zoom in, zoom out, reset to fit, plus a live
// percentage readout — anchored to a corner of the diagram viewport.
const ZoomControls: React.FC<{ zoomPercent: number }> = ({ zoomPercent }) => {
  const { t } = useTranslation();
  const { zoomIn, zoomOut, resetTransform } = useControls();

  return (
    <div className={styles['db-analytics-er-diagram__zoom-controls']}>
      <button
        type="button"
        className={styles['db-analytics-er-diagram__zoom-btn']}
        onClick={() => zoomOut()}
        aria-label={t('db_analytics.schemaErDiagram.zoomOut')}
        title={t('db_analytics.schemaErDiagram.zoomOut')}
      >
        <Icon.zoomOut size={14} aria-hidden="true" />
      </button>
      <span className={styles['db-analytics-er-diagram__zoom-level']}>{zoomPercent}%</span>
      <button
        type="button"
        className={styles['db-analytics-er-diagram__zoom-btn']}
        onClick={() => zoomIn()}
        aria-label={t('db_analytics.schemaErDiagram.zoomIn')}
        title={t('db_analytics.schemaErDiagram.zoomIn')}
      >
        <Icon.zoomIn size={14} aria-hidden="true" />
      </button>
      <span className={styles['db-analytics-er-diagram__zoom-divider']} aria-hidden="true" />
      <button
        type="button"
        className={styles['db-analytics-er-diagram__zoom-btn']}
        onClick={() => resetTransform()}
        aria-label={t('db_analytics.schemaErDiagram.resetZoom')}
        title={t('db_analytics.schemaErDiagram.resetZoom')}
      >
        <Icon.zoomReset size={14} aria-hidden="true" />
      </button>
    </div>
  );
};

const SchemaErDiagram: React.FC<Props> = ({ tables }) => {
  const { t } = useTranslation();
  const [zoomPercent, setZoomPercent] = useState(100);
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
      <TransformWrapper
        initialScale={1}
        minScale={0.25}
        maxScale={2.5}
        limitToBounds={false}
        centerOnInit
        wheel={{ step: 0.12 }}
        doubleClick={{ mode: 'zoomIn', step: 0.5 }}
        onTransformed={(_, state) => setZoomPercent(Math.round(state.scale * 100))}
      >
        <ZoomControls zoomPercent={zoomPercent} />
        <TransformComponent
          wrapperClass={styles['db-analytics-er-diagram__scroll']}
          contentClass={styles['db-analytics-er-diagram__content']}
        >
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
        </TransformComponent>
      </TransformWrapper>

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
  ZoomIn,
  ZoomOut,
  RotateCcw,
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
  zoomIn: ZoomIn,
  zoomOut: ZoomOut,
  zoomReset: RotateCcw,
} as const;

export type IconName = keyof typeof Icon;

























{
  "db_analytics": {
    "app": {
      "title": "DB Analytics",
      "subtitle": "Ask questions about your databases in natural language and get instant charts and insights.",
      "databasesConnected_one": "{{count}} database connected",
      "databasesConnected_other": "{{count}} databases connected"
    },
    "sessionHistory": {
      "title": "Chat sessions",
      "newSession": "New chat session",
      "loadingSessions": "Loading sessions…",
      "waitForResponse": "Wait for the current response to finish",
      "emptyTitle": "No sessions yet",
      "emptyDesc": "Start a new conversation to begin.",
      "viewConnectedDatabases": "View connected databases",
      "connectedDatabasesPopoverTitle": "Connected databases",
      "connectedDatabasesEmpty": "No matching databases found."
    },
    "databaseList": {
      "title": "Databases",
      "manageDatabases": "Manage databases",
      "addDatabase": "Add database",
      "addDatabaseConnection": "Add database connection",
      "loadingDatabases": "Loading databases…",
      "waitForResponse": "Wait for the current response to finish",
      "emptyTitle": "No databases connected",
      "connectADatabase": "Connect a database",
      "viewSchema": "View schema & ER diagram",
      "connectToViewSchema": "Connect this database to view its schema",
      "viewSchemaAndErDiagram": "View schema and ER diagram",
      "status": {
        "ready": "Ready",
        "connected": "Connected",
        "disconnected": "Disconnected"
      },
      "refreshingDatabases": "Refreshing databases…"
    },
    "chatPanel": {
      "newSession": "New session",
      "subtitle": "Ask questions about your connected databases",
      "loadingConversation": "Loading conversation…",
      "askToGetStarted": "Ask a question to get started with this session.",
      "selectSessionToStart": "Select a session to get started.",
      "chartRef_one": "1 chart generated in results panel",
      "chartRef_other": "{{count}} charts generated in results panel",
      "dbNotice": {
        "titleSingle": "A database in this session is disconnected",
        "titlePlural_one": "{{count}} database in this session is disconnected",
        "titlePlural_other": "{{count}} databases in this session are disconnected",
        "reconnectPrefix": "Reconnect",
        "reconnectSuffix": "to continue this conversation.",
        "and": "and",
        "connectNow": "Connect now"
      },
      "composer": {
        "placeholderLoading": "Loading…",
        "placeholderBlocked": "Connect the databases below to start asking questions…",
        "placeholderStreaming": "Waiting for the current response to finish…",
        "placeholderDefault": "Ask about revenue, trends, or run a custom query...",
        "hint": "Enter to send · Shift+Enter for new line",
        "sendLabel": "Send message",
        "send": "Send",
        "preparingFollowups": "Preparing follow-up suggestions…"
      }
    },
    "suggestedPrompts": {
      "finding": "Finding good questions to start with…",
      "intro": "Not sure where to start? Try one of these.",
      "askThis": "Ask this",
      "showMore": "Show full prompt",
      "showLess": "Show less"
    },
    "followupChips": {
      "finding": "Finding follow-up questions…",
      "continueExploring": "Continue exploring"
    },
    "queryProgress": {
      "genericError": "Something went wrong."
    },
    "resultsPanel": {
      "title": "Generated insights",
      "subtitle": "Visualizations from your conversation",
      "emptyState": "Charts generated from your chat will appear here.",
      "drawingChart": "Drawing your chart…",
      "noDataForChart": "No data returned for this chart.",
      "noSeriesFound": "No numeric series found to plot for this chart.",
      "prompt": "Prompt",
      "askAiAboutChart": "Ask AI about this chart",
      "askAi": "Ask AI",
      "expandChart": "Expand chart",
      "expand": "Expand"
    },
    "chartTypeMenu": {
      "changeType": "Change chart type",
      "types": {
        "bar": "Bar",
        "line": "Line",
        "area": "Area",
        "pie": "Pie",
        "scatter": "Scatter"
      }
    },
    "expandChartSlider": {
      "close": "Close"
    },
    "chartAskAiSlider": {
      "title": "Ask about this chart",
      "close": "Close",
      "emptyPrompt": "Ask a question about this specific chart — its trend, an outlier, what's driving it.",
      "thinking": "Thinking…",
      "placeholder": "Ask something about this chart…",
      "ask": "Ask",
      "genericError": "Something went wrong."
    },
    "dbSchemaSlider": {
      "close": "Close",
      "subtitle": "Schema & relationships",
      "genericError": "Failed to load schema.",
      "tabs": {
        "tables": "Tables",
        "erDiagram": "ER Diagram"
      },
      "loadingSchema": "Loading schema…",
      "noTablesFound": "No tables found for this database.",
      "tableLabel": "table",
      "tableLabelPlural": "tables",
      "tableCount_one": "{{count}} table",
      "tableCount_other": "{{count}} tables",
      "totalRows": "{{count}} total rows",
      "columnCount_one": "{{count}} col",
      "columnCount_other": "{{count}} cols",
      "rowCount_one": "{{count}} row",
      "rowCount_other": "{{count}} rows",
      "table": {
        "columnHeader": "Column",
        "typeHeader": "Type",
        "nullableHeader": "Nullable",
        "defaultHeader": "Default",
        "nullableYes": "Yes",
        "nullableNo": "No",
        "primaryKey": "Primary key",
        "pkAbbr": "PK"
      },
      "resizeHandle": "Drag to resize"
    },
    "schemaErDiagram": {
      "noRelationships": "No relationships could be inferred from column names — tables are shown without connecting lines.",
      "moreColumns_one": "+{{count}} more column",
      "moreColumns_other": "+{{count}} more columns",
      "zoomIn": "Zoom in",
      "zoomOut": "Zoom out",
      "resetZoom": "Reset zoom"
    },
    "connectPasswordModal": {
      "title": "Connect to {{name}}",
      "close": "Close",
      "description": "Enter the password for <strong>{{username}}</strong> on <strong>{{dbName}}</strong> to establish this connection.",
      "passwordLabel": "Password",
      "passwordPlaceholder": "••••••••",
      "cancel": "Cancel",
      "connect": "Connect",
      "connecting": "Connecting…",
      "errorRequired": "Password is required.",
      "genericError": "Failed to connect."
    },
    "databaseManagerSlider": {
      "title": "Manage databases",
      "close": "Close",
      "tabs": {
        "existing": "Existing databases",
        "new": "New connection",
        "csv": "Upload CSV"
      },
      "emptyTitle": "No databases yet",
      "emptyDesc": "Create a connection or upload a CSV file to start querying data.",
      "table": "Table",
      "database": "Database",
      "port": "Port",
      "source": "Source",
      "user": "User",
      "csvUploadSource": "CSV upload",
      "disconnect": "Disconnect",
      "disconnecting": "Disconnecting…",
      "connect": "Connect",
      "newConnection": {
        "title": "New database connection",
        "subtitle": "Connect a MySQL, PostgreSQL, or MSSQL database to query from this workspace.",
        "connectionName": "Connection name",
        "connectionNamePlaceholder": "e.g. Production Warehouse",
        "databaseType": "Database type",
        "host": "Host",
        "hostPlaceholder": "e.g. 10.0.0.4",
        "port": "Port",
        "databaseName": "Database name",
        "databaseNamePlaceholder": "e.g. analytics_prod",
        "username": "Username",
        "usernamePlaceholder": "e.g. readonly_user",
        "passwordHint": "You'll be asked for the password separately when connecting.",
        "reset": "Reset",
        "createConnection": "Create connection",
        "creating": "Creating…",
        "errorRequiredFields": "Please fill in all required fields.",
        "genericError": "Failed to create database."
      },
      "csvUpload": {
        "title": "Upload a CSV file",
        "subtitle": "Turn a spreadsheet into a queryable table — no database connection required.",
        "fileLabel": "CSV file",
        "chooseFile": "Choose a .csv file",
        "dropHere": "Drop your CSV file here",
        "releaseToSelect": "Release to select this file",
        "dragOrClick": "Drag & drop or click to browse — only .csv files are accepted",
        "nameLabel": "Name",
        "namePlaceholder": "e.g. Sample",
        "hint": "Uploaded CSVs are ready to query immediately — no password or connection step needed.",
        "reset": "Reset",
        "upload": "Upload CSV",
        "uploading": "Uploading…",
        "errorOnlyCsv": "Only .csv files are supported.",
        "errorSelectFile": "Please select at least one CSV file to upload.",
        "errorRequiredFields": "Please provide a name for this connection.",
        "genericError": "Failed to upload CSV file.",
        "addMoreFiles": "Click or drop to add more files",
        "removeFile": "Remove {{name}}",
        "modeLabel": "Upload mode",
        "modeCombined": "Combined",
        "modeSeparate": "Separate",
        "modeCombinedHint": "All selected files are merged into a single table.",
        "modeSeparateHint": "Each file becomes its own table within this connection.",
        "filesSelectedCount": "{{count}} / {{max}} files selected",
        "limitsHint": "Max {{maxSize}}MB per file, up to {{maxCount}} files.",
        "errorFileTooLarge": "These files exceed the {{maxSize}}MB limit: {{names}}",
        "errorTooManyFiles": "You can upload up to {{max}} files at a time. Extra files were not added."
      },
      "resizeHandle": "Drag to resize"
    },
    "newSessionSlider": {
      "title": "New chat session",
      "close": "Close",
      "sessionName": "Session name",
      "sessionNamePlaceholder": "e.g. Q3 marketing review",
      "connectDatabases": "Connect databases",
      "connectDatabasesHint": "Select one or more connected databases to make available in this session. Disconnected databases can't be added until they're connected.",
      "noDatabasesAvailable": "No databases available.",
      "addOne": "Add one",
      "connectFirstToInclude": "Connect this database first to include it in a session",
      "manageDatabases": "Manage databases",
      "createSession": "Create session",
      "creating": "Creating…",
      "errorNameRequired": "Please enter a session name.",
      "errorSelectDatabase": "Select at least one database for this session.",
      "genericError": "Failed to create session.",
      "status": {
        "connected": "Connected",
        "disconnected": "Disconnected"
      }
    },
    "dbTypes": {
      "mysql": "MySQL",
      "postgres": "PostgreSQL",
      "mssql": "MSSQL",
      "csv": "CSV"
    }
  }
}
























{
  "db_analytics": {
    "app": {
      "title": "DB 분석",
      "subtitle": "자연어로 데이터베이스에 질문하고 즉시 차트와 인사이트를 확인하세요.",
      "databasesConnected_other": "데이터베이스 {{count}}개 연결됨"
    },
    "sessionHistory": {
      "title": "채팅 세션",
      "newSession": "새 채팅 세션",
      "loadingSessions": "세션 불러오는 중…",
      "waitForResponse": "현재 응답이 끝날 때까지 기다려 주세요",
      "emptyTitle": "아직 세션이 없습니다",
      "emptyDesc": "새 대화를 시작해 보세요.",
      "viewConnectedDatabases": "연결된 데이터베이스 보기",
      "connectedDatabasesPopoverTitle": "연결된 데이터베이스",
      "connectedDatabasesEmpty": "일치하는 데이터베이스가 없습니다."
    },
    "databaseList": {
      "title": "데이터베이스",
      "manageDatabases": "데이터베이스 관리",
      "addDatabase": "데이터베이스 추가",
      "addDatabaseConnection": "데이터베이스 연결 추가",
      "loadingDatabases": "데이터베이스 불러오는 중…",
      "waitForResponse": "현재 응답이 끝날 때까지 기다려 주세요",
      "emptyTitle": "연결된 데이터베이스가 없습니다",
      "connectADatabase": "데이터베이스 연결",
      "viewSchema": "스키마 및 ER 다이어그램 보기",
      "connectToViewSchema": "스키마를 보려면 먼저 이 데이터베이스를 연결하세요",
      "viewSchemaAndErDiagram": "스키마 및 ER 다이어그램 보기",
      "status": {
        "ready": "준비됨",
        "connected": "연결됨",
        "disconnected": "연결 끊김"
      },
      "refreshingDatabases": "데이터베이스 새로고침 중…"
    },
    "chatPanel": {
      "newSession": "새 세션",
      "subtitle": "연결된 데이터베이스에 대해 질문해 보세요",
      "loadingConversation": "대화 불러오는 중…",
      "askToGetStarted": "이 세션을 시작하려면 질문을 입력하세요.",
      "selectSessionToStart": "시작하려면 세션을 선택하세요.",
      "chartRef_other": "결과 패널에 차트 {{count}}개가 생성되었습니다",
      "dbNotice": {
        "titleSingle": "이 세션의 데이터베이스 연결이 끊어졌습니다",
        "titlePlural_other": "이 세션의 데이터베이스 {{count}}개가 연결 끊김 상태입니다",
        "reconnectPrefix": "대화를 계속하려면",
        "reconnectSuffix": "을(를) 다시 연결하세요.",
        "and": "및",
        "connectNow": "지금 연결하기"
      },
      "composer": {
        "placeholderLoading": "불러오는 중…",
        "placeholderBlocked": "질문을 시작하려면 아래 데이터베이스를 연결하세요…",
        "placeholderStreaming": "현재 응답이 끝나기를 기다리는 중…",
        "placeholderDefault": "매출, 트렌드에 대해 질문하거나 원하는 쿼리를 실행해 보세요...",
        "hint": "Enter로 전송 · Shift+Enter로 줄바꿈",
        "sendLabel": "메시지 보내기",
        "send": "보내기",
        "preparingFollowups": "후속 질문을 준비하는 중…"
      }
    },
    "suggestedPrompts": {
      "finding": "시작하기 좋은 질문을 찾는 중…",
      "intro": "무엇을 물어봐야 할지 모르시겠나요? 아래 중 하나를 선택해 보세요.",
      "askThis": "이 질문하기",
      "showMore": "전체 프롬프트 보기",
      "showLess": "간략히 보기"
    },
    "followupChips": {
      "finding": "후속 질문을 찾는 중…",
      "continueExploring": "계속 살펴보기"
    },
    "queryProgress": {
      "genericError": "문제가 발생했습니다."
    },
    "resultsPanel": {
      "title": "생성된 인사이트",
      "subtitle": "대화에서 생성된 시각화 자료",
      "emptyState": "대화에서 생성된 차트가 여기에 표시됩니다.",
      "drawingChart": "차트를 그리는 중…",
      "noDataForChart": "이 차트에 대한 데이터가 없습니다.",
      "noSeriesFound": "이 차트에 표시할 숫자 데이터를 찾을 수 없습니다.",
      "prompt": "프롬프트",
      "askAiAboutChart": "이 차트에 대해 AI에게 질문하기",
      "askAi": "AI에게 질문",
      "expandChart": "차트 확대",
      "expand": "확대"
    },
    "chartTypeMenu": {
      "changeType": "차트 유형 변경",
      "types": {
        "bar": "막대형",
        "line": "선형",
        "area": "영역형",
        "pie": "원형",
        "scatter": "분산형"
      }
    },
    "expandChartSlider": {
      "close": "닫기"
    },
    "chartAskAiSlider": {
      "title": "이 차트에 대해 질문하기",
      "close": "닫기",
      "emptyPrompt": "이 차트의 추세, 특이점, 원인 등에 대해 질문해 보세요.",
      "thinking": "생각하는 중…",
      "placeholder": "이 차트에 대해 궁금한 점을 입력하세요…",
      "ask": "질문하기",
      "genericError": "문제가 발생했습니다."
    },
    "dbSchemaSlider": {
      "close": "닫기",
      "subtitle": "스키마 및 관계",
      "genericError": "스키마를 불러오지 못했습니다.",
      "tabs": {
        "tables": "테이블",
        "erDiagram": "ER 다이어그램"
      },
      "loadingSchema": "스키마 불러오는 중…",
      "noTablesFound": "이 데이터베이스에서 테이블을 찾을 수 없습니다.",
      "tableLabel": "개 테이블",
      "tableLabelPlural": "개 테이블",
      "tableCount_other": "테이블 {{count}}개",
      "totalRows": "총 {{count}}개 행",
      "columnCount_other": "열 {{count}}개",
      "rowCount_other": "행 {{count}}개",
      "table": {
        "columnHeader": "열",
        "typeHeader": "유형",
        "nullableHeader": "NULL 허용",
        "defaultHeader": "기본값",
        "nullableYes": "예",
        "nullableNo": "아니요",
        "primaryKey": "기본 키",
        "pkAbbr": "PK"
      },
      "resizeHandle": "드래그하여 크기 조절"
    },
    "schemaErDiagram": {
      "noRelationships": "열 이름으로부터 관계를 추론할 수 없습니다 — 테이블이 연결선 없이 표시됩니다.",
      "moreColumns_other": "열 {{count}}개 더 보기",
      "zoomIn": "확대",
      "zoomOut": "축소",
      "resetZoom": "확대/축소 초기화"
    },
    "connectPasswordModal": {
      "title": "{{name}}에 연결",
      "close": "닫기",
      "description": "<strong>{{dbName}}</strong>의 <strong>{{username}}</strong> 비밀번호를 입력하여 연결하세요.",
      "passwordLabel": "비밀번호",
      "passwordPlaceholder": "••••••••",
      "cancel": "취소",
      "connect": "연결",
      "connecting": "연결하는 중…",
      "errorRequired": "비밀번호를 입력해 주세요.",
      "genericError": "연결에 실패했습니다."
    },
    "databaseManagerSlider": {
      "title": "데이터베이스 관리",
      "close": "닫기",
      "tabs": {
        "existing": "기존 데이터베이스",
        "new": "새 연결",
        "csv": "CSV 업로드"
      },
      "emptyTitle": "아직 데이터베이스가 없습니다",
      "emptyDesc": "연결을 생성하거나 CSV 파일을 업로드하여 데이터 조회를 시작하세요.",
      "table": "테이블",
      "database": "데이터베이스",
      "port": "포트",
      "source": "소스",
      "user": "사용자",
      "csvUploadSource": "CSV 업로드",
      "disconnect": "연결 해제",
      "disconnecting": "연결 해제하는 중…",
      "connect": "연결",
      "newConnection": {
        "title": "새 데이터베이스 연결",
        "subtitle": "이 작업 공간에서 조회할 MySQL, PostgreSQL 또는 MSSQL 데이터베이스를 연결하세요.",
        "connectionName": "연결 이름",
        "connectionNamePlaceholder": "예: Production Warehouse",
        "databaseType": "데이터베이스 유형",
        "host": "호스트",
        "hostPlaceholder": "예: 10.0.0.4",
        "port": "포트",
        "databaseName": "데이터베이스 이름",
        "databaseNamePlaceholder": "예: analytics_prod",
        "username": "사용자 이름",
        "usernamePlaceholder": "예: readonly_user",
        "passwordHint": "비밀번호는 연결할 때 별도로 입력하게 됩니다.",
        "reset": "초기화",
        "createConnection": "연결 생성",
        "creating": "생성하는 중…",
        "errorRequiredFields": "모든 필수 항목을 입력해 주세요.",
        "genericError": "데이터베이스 생성에 실패했습니다."
      },
      "csvUpload": {
        "title": "CSV 파일 업로드",
        "subtitle": "스프레드시트를 조회 가능한 테이블로 변환하세요 — 데이터베이스 연결이 필요하지 않습니다.",
        "fileLabel": "CSV 파일",
        "chooseFile": ".csv 파일 선택",
        "dropHere": "여기에 CSV 파일을 놓으세요",
        "releaseToSelect": "놓으면 이 파일이 선택됩니다",
        "dragOrClick": "드래그 앤 드롭하거나 클릭하여 찾아보기 — .csv 파일만 지원됩니다",
        "nameLabel": "이름",
        "namePlaceholder": "예: Sample",
        "hint": "업로드된 CSV는 즉시 조회할 수 있습니다 — 비밀번호나 연결 절차가 필요하지 않습니다.",
        "reset": "초기화",
        "upload": "CSV 업로드",
        "uploading": "업로드하는 중…",
        "errorOnlyCsv": ".csv 파일만 지원됩니다.",
        "errorSelectFile": "업로드할 CSV 파일을 하나 이상 선택해 주세요.",
        "errorRequiredFields": "연결 이름을 입력해 주세요.",
        "genericError": "CSV 파일 업로드에 실패했습니다.",
        "addMoreFiles": "클릭하거나 파일을 놓아 더 추가하세요",
        "removeFile": "{{name}} 제거",
        "modeLabel": "업로드 방식",
        "modeCombined": "통합",
        "modeSeparate": "개별",
        "modeCombinedHint": "선택한 모든 파일이 하나의 테이블로 합쳐집니다.",
        "modeSeparateHint": "각 파일이 이 연결 내에서 개별 테이블이 됩니다.",
        "filesSelectedCount": "{{count}} / {{max}}개 파일 선택됨",
        "limitsHint": "파일당 최대 {{maxSize}}MB, 최대 {{maxCount}}개까지 업로드할 수 있습니다.",
        "errorFileTooLarge": "다음 파일이 {{maxSize}}MB 제한을 초과했습니다: {{names}}",
        "errorTooManyFiles": "한 번에 최대 {{max}}개의 파일을 업로드할 수 있습니다. 초과된 파일은 추가되지 않았습니다."
      },
      "resizeHandle": "드래그하여 크기 조절"
    },
    "newSessionSlider": {
      "title": "새 채팅 세션",
      "close": "닫기",
      "sessionName": "세션 이름",
      "sessionNamePlaceholder": "예: 3분기 마케팅 리뷰",
      "connectDatabases": "데이터베이스 연결",
      "connectDatabasesHint": "이 세션에서 사용할 연결된 데이터베이스를 하나 이상 선택하세요. 연결이 끊긴 데이터베이스는 연결하기 전까지 추가할 수 없습니다.",
      "noDatabasesAvailable": "사용 가능한 데이터베이스가 없습니다.",
      "addOne": "추가하기",
      "connectFirstToInclude": "세션에 포함하려면 먼저 이 데이터베이스를 연결하세요",
      "manageDatabases": "데이터베이스 관리",
      "createSession": "세션 생성",
      "creating": "생성하는 중…",
      "errorNameRequired": "세션 이름을 입력해 주세요.",
      "errorSelectDatabase": "이 세션에 사용할 데이터베이스를 하나 이상 선택하세요.",
      "genericError": "세션 생성에 실패했습니다.",
      "status": {
        "connected": "연결됨",
        "disconnected": "연결 끊김"
      }
    },
    "dbTypes": {
      "mysql": "MySQL",
      "postgres": "PostgreSQL",
      "mssql": "MSSQL",
      "csv": "CSV"
    }
  }
}




"react-zoom-pan-pinch": "^3.6.1"
