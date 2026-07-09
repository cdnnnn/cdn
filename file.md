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
  WandSparkles,
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
  askAi: WandSparkles,
} as const;

export type IconName = keyof typeof Icon;
