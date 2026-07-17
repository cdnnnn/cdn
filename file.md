// ═══════════════════════════════════════════════
// pages/UploadInfer/UploadInfer.tsx
// LectureAI · Upload & Inference page
// ═══════════════════════════════════════════════
import React, { useState, useEffect, useCallback, useRef } from 'react';
import { useTranslation } from 'react-i18next';
import { useAppSelector, useAppDispatch } from '../../store/hooks';
import { clearServerSelection } from '../../store/uploadSlice';
import api from '../../services/api';
import FilePanel from './FilePanel';
import InferencePanel from './InferencePanel';
import WorkspacePanel from './WorkspacePanel';
import FileSidebar from './FileSidebar';
import styles from './UploadInfer.module.scss';

// ── Keyword Insights (from GET /files/{id}) ──────
export interface KGEdge { type: string; source: string; target: string; }
export interface KGNode { id: string; label: string; title?: string; value?: number; }
export interface KnowledgeGraph { edges: KGEdge[]; nodes: KGNode[]; }
export interface WordCloudData { wordcloud: [string, number][]; complexity_map: Record<string, string>; }
export interface TimelineDataset { data: number[]; label: string; }
export interface TimelineData { labels: string[]; datasets: TimelineDataset[]; timestamped: boolean; }
export interface HeatmapData { matrix: number[][]; keywords: string[]; segments: string[]; timestamped: boolean; }
export interface ClusterItem { id: string; label: string; value: number; }
export interface ClustersData { clusters: Record<string, ClusterItem[]>; }
export interface FrequencyItem { count: number; keyword: string; relative_pct: number; first_mention_pct: number; }
export interface FrequencyData { data: FrequencyItem[]; }
export interface PrereqEdge { reason: string; enables: string; prerequisite: string; }
export interface PrereqNode { id: string; label: string; value: number; }
export interface PrerequisitesData { edges: PrereqEdge[]; nodes: PrereqNode[]; }
export interface ImportanceComplexityItem { reason: string; keyword: string; frequency: number; complexity: string; importance: number; }
export interface ImportanceComplexityData { data: ImportanceComplexityItem[]; }
export interface CooccurrenceData { matrix: number[][]; keywords: string[]; }
export interface GlossaryItem { term: string; definition: string; first_mentioned_ms: number; }
export interface GlossaryData { glossary: GlossaryItem[]; }

export interface KeywordInsights {
  enriched_keywords: unknown | null;
  knowledge_graph: KnowledgeGraph | null;
  word_cloud: WordCloudData | null;
  timeline: TimelineData | null;
  heatmap: HeatmapData | null;
  clusters: ClustersData | null;
  frequency: FrequencyData | null;
  prerequisites: PrerequisitesData | null;
  importance_complexity: ImportanceComplexityData | null;
  cooccurrence: CooccurrenceData | null;
  congnitive_Load: unknown | null;
  segments: { segments: unknown[] } | null;
  glossary: GlossaryData | null;
}

export interface FileResult {
  summary: string;
  keywords: string[];
  faq: string;
  shortAnswer: string;
  trueFalse: string;
  timestampedSummary: string;
  keywordInsights: KeywordInsights | null;
  fileName: string;
  fileId: number;
  insertedAt: string;
}

// ── Tabs ──────────────────────────────────────────
// All three are always clickable — there's no gating on a previous tab
// being "complete". Every panel below stays mounted at all times (just
// hidden with CSS) so file lists, batch polling, and scroll position
// survive switching tabs instead of resetting.
type TabId = 'upload' | 'infer' | 'results';

const UploadInfer: React.FC = () => {
  const { t } = useTranslation();
  const isBatchRunning = useAppSelector(s => s.upload.isBatchRunning);
  const dispatch = useAppDispatch();

  const [activeTab, setActiveTab] = useState<TabId>('upload');

  // "selectMode" is FilePanel's checkbox-selection UI for building the set
  // of files to run inference on. Turning it on stays on the Upload tab
  // (so people can keep browsing/checking files there) — the Infer tab
  // badge below nudges them over once something's selected.
  const [selectMode, setSelectMode] = useState(false);
  useEffect(() => { if (!isBatchRunning) setSelectMode(false); }, [isBatchRunning]);
  useEffect(() => { if (!selectMode) dispatch(clearServerSelection()); }, [selectMode]); // eslint-disable-line
  useEffect(() => { return () => { dispatch(clearServerSelection()); }; }, []); // eslint-disable-line

  const enterInferSelection = useCallback(() => setSelectMode(true), []);
  const exitInferSelection = useCallback(() => setSelectMode(false), []);

  const [fileResult, setFileResult] = useState<FileResult | null>(null);
  const [fileLoading, setFileLoading] = useState(false);
  const [activeFileId, setActiveFileId] = useState<number | null>(null);

  const fetchFileData = useCallback(async (fileId: number) => {
    setFileLoading(true);
    try {
      const res = await api.get(`/files/${fileId}`);
      const d = (res.data as any)?.data ?? {};
      setFileResult(prev => ({
        fileId,
        fileName: d.original_name ?? prev?.fileName ?? String(fileId),
        insertedAt: d.inserted_at ?? prev?.insertedAt ?? '',
        summary: d.summary ?? '',
        keywords: d.keywords ?? [],
        faq: d.faq ?? '[]',
        shortAnswer: d.short_answer ?? '[]',
        trueFalse: d.true_false ?? '[]',
        // NOTE: backend key is genuinely "timstamped_summary" (missing an "e") — match it exactly.
        timestampedSummary: d.timstamped_summary ?? '[]',
        keywordInsights: d.keyword_insights ?? null,
      }));
    } catch { setFileResult(null); } finally { setFileLoading(false); }
  }, []);

  // Clicking a file (from the Upload tab's list, or the Workspace tab's
  // own picker) loads its results and jumps straight to the Workspace tab.
  const handleFileClick = useCallback(async (fileId: number) => {
    setActiveTab('results');
    if (fileId === activeFileId) return;
    setActiveFileId(fileId);
    setFileResult(null);
    await fetchFileData(fileId);
  }, [activeFileId, fetchFileData]);

  const handleDeleteComplete = useCallback((deletedIds: number[], all?: boolean) => {
    if (all || (activeFileId !== null && deletedIds.includes(activeFileId))) {
      setActiveFileId(null);
      setFileResult(null);
    }
  }, [activeFileId]);

  const prevBatchRunning = useRef(false);
  useEffect(() => {
    const justFinished = prevBatchRunning.current && !isBatchRunning;
    prevBatchRunning.current = isBatchRunning;
    if (justFinished && activeFileId !== null) fetchFileData(activeFileId);
  }, [isBatchRunning]); // eslint-disable-line

  const tabs: { id: TabId; label: string; desc: string }[] = [
    { id: 'upload', label: t('uploadInfer.tabs.upload'), desc: 'Upload lecture files and browse everything you\u2019ve added.' },
    { id: 'infer', label: t('uploadInfer.tabs.infer'), desc: 'Pick files, configure what to generate, and run analysis.' },
    { id: 'results', label: t('uploadInfer.tabs.results'), desc: 'Select a processed file to open its summary, keywords, and quiz.' },
  ];
  const activeTabInfo = tabs.find(tab => tab.id === activeTab) ?? tabs[0];

  return (
    <div className={styles.page}>
      {/* ── Header — tabs stand in for the title, active one reads as the page heading ── */}
      <div className={styles.headerBar}>
        <div>
          <div className={styles.headerTabs} role="tablist">
            {tabs.map(tab => (
              <button
                key={tab.id}
                type="button"
                role="tab"
                aria-selected={activeTab === tab.id}
                className={`${styles.headerTab} ${activeTab === tab.id ? styles.headerTabActive : ''}`}
                onClick={() => setActiveTab(tab.id)}
              >
                {tab.label}
              </button>
            ))}
          </div>
          <div className={styles.headerSub}>{activeTabInfo.desc}</div>
        </div>
        <div className={styles.headerActions} />
      </div>

      <div className={styles.upbody}>
        <div className={styles.tabPane} style={{ display: activeTab === 'upload' ? 'flex' : 'none' }}>
          <FilePanel
            selectMode={selectMode}
            onEnterSelectMode={enterInferSelection}
            onExitSelectMode={exitInferSelection}
            onDeleteComplete={handleDeleteComplete}
            onGoToInfer={() => setActiveTab('infer')}
          />
        </div>

        <div className={styles.tabPane} style={{ display: activeTab === 'infer' ? 'flex' : 'none' }}>
          <FileSidebar mode="select" active={activeTab === 'infer'} />
          <InferencePanel />
        </div>

        <div className={styles.tabPane} style={{ display: activeTab === 'results' ? 'flex' : 'none' }}>
          <FileSidebar mode="view" active={activeTab === 'results'} activeFileId={activeFileId} onFileClick={handleFileClick} />
          <WorkspacePanel
            step2Visible={false}
            fileResult={fileResult}
            fileLoading={fileLoading}
            activeFileId={activeFileId}
            onResultUpdate={patch => setFileResult(prev => prev ? { ...prev, ...patch } : prev)}
          />
        </div>
      </div>
    </div>
  );
};

export default UploadInfer;





















// ═══════════════════════════════════════════════
// UploadInfer.module.scss
// LectureAI · Upload & Inference page shell + tab bar
// ═══════════════════════════════════════════════
@use '../../styles/mixins' as m;

.page {
  display: flex;
  flex-direction: column;
  height: 100%;
  overflow: hidden;
  background: var(--bg0);
}

// ── Header — matches the reference: title/subtitle on the left, a
// reserved action area on the right, and a plain flat divider beneath.
// Our tabs stand in for the static title — the active one reads at
// title weight/size, the others sit alongside as smaller nav links.
.headerBar {
  flex-shrink: 0;
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 20px;
  padding: 20px 24px 16px;
  border-bottom: 1px solid var(--bdr);
}

.headerTabs {
  display: flex;
  align-items: baseline;
  gap: 18px;
}

.headerTab {
  padding: 0;
  border: none;
  background: transparent;
  font-family: var(--font-ui);
  font-size: 14px;
  font-weight: 500;
  color: var(--t2);
  cursor: pointer;
  @include m.theme-transition;

  &:hover {
    color: var(--t1);
  }
}

.headerTabActive {
  font-size: 19px;
  font-weight: 600;
  color: var(--t0);

  &:hover {
    color: var(--t0);
  }
}

.headerSub {
  margin-top: 4px;
  font-size: 13px;
  color: var(--t2);
}

// Reserved for per-tab actions (e.g. an export/run shortcut), so the
// header keeps the same shape as the reference even before any button
// is wired up here.
.headerActions {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  gap: 8px;
}

// ── Tab content ──────────────────────────────────
.upbody {
  flex: 1;
  min-height: 0;
  position: relative;
}

// Every panel is rendered inside a .tabPane at all times; only the active
// one is switched to `display: flex` (see UploadInfer.tsx). This keeps
// file lists, batch polling, and scroll position alive across tab
// switches instead of remounting on every click.
.tabPane {
  position: absolute;
  inset: 0;
  flex-direction: row;
  overflow: hidden;
}

@media (max-width: 860px) {
  .headerBar { flex-direction: column; align-items: stretch; gap: 10px; padding: 14px 16px; }
  .headerTabs { flex-wrap: wrap; gap: 14px; }
  .headerTabActive { font-size: 17px; }

  .tabPane { flex-direction: column; }
}



















// ═══════════════════════════════════════════════
// FileSidebar.module.scss
// Content Analytics · Shared sidebar for the Infer + Workspace tabs
// ═══════════════════════════════════════════════
@use '../../styles/mixins' as m;

.sidebar {
  width: 400px;
  flex-shrink: 0;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  background: var(--bg0);
  border-right: 1px solid var(--bdr);

  :global(html.light) & {
    background: #fff;
  }
}

// ── Date range filter ─────────────────────────
.dateFilter {
  display: flex;
  align-items: flex-end;
  gap: 6px;
  padding: 12px 14px;
  flex-shrink: 0;
  border-bottom: 1px solid var(--bdr);
}

.dateField {
  display: flex;
  flex-direction: column;
  gap: 3px;
  flex: 1;
  min-width: 0;
}

.dateLabel {
  font-size: 10px;
  font-weight: 600;
  color: var(--t2);
  text-transform: uppercase;
  letter-spacing: 0.1em;
  @include m.mono;
}

.dateInput {
  width: 100%;
  padding: 4px 7px;
  background: var(--bg1);
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  color: var(--t0);
  font-family: var(--font-ui);
  font-size: 13px;
  outline: none;
  appearance: none;
  transition: border-color 0.12s;
  cursor: pointer;

  &:focus {
    border-color: var(--blue);
    box-shadow: 0 0 0 2px var(--blue-dim);
  }

  &::-webkit-calendar-picker-indicator {
    opacity: 0.7;
    cursor: pointer;
    filter: var(--date-icon-filter);
  }
}

.dateSep {
  font-size: 13px;
  color: var(--t2);
  padding-bottom: 4px;
  flex-shrink: 0;
}

.applyBtn {
  align-self: flex-end;
  padding: 4px 12px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t1);
  font-family: var(--font-ui);
  font-size: 13px;
  font-weight: 500;
  cursor: pointer;
  flex-shrink: 0;
  transition: all 0.12s;

  &:hover:not(:disabled) {
    background: var(--bg3);
    border-color: var(--bdr3);
  }

  &:disabled {
    opacity: 0.6;
    cursor: default;
  }
}

.miniSpinner {
  display: inline-block;
  width: 12px;
  height: 12px;
  border: 2px solid var(--bdr3);
  border-top-color: var(--t1);
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

// ── List header ────────────────────────────────
.head {
  @include m.flex-center;
  gap: 8px;
  padding: 12px 14px 8px;
  flex-shrink: 0;
}

.headTitle {
  font-size: 12px;
  font-weight: 600;
  color: var(--t1);
}

.headCount {
  font-size: 11px;
  font-weight: 600;
  color: var(--t2);
  background: var(--bg2);
  border-radius: 99px;
  padding: 1px 7px;
}

.headSelected {
  margin-left: auto;
  font-size: 11px;
  color: var(--blue);
}

// ── Search ─────────────────────────────────────
.searchWrap {
  position: relative;
  margin: 0 14px 10px;
  flex-shrink: 0;
}

.searchIcon {
  position: absolute;
  left: 9px;
  top: 50%;
  transform: translateY(-50%);
  width: 13px;
  height: 13px;
  color: var(--t2);
  pointer-events: none;
}

.searchInput {
  width: 100%;
  height: 30px;
  padding: 0 10px 0 28px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: var(--bg1);
  color: var(--t0);
  font-family: var(--font-ui);
  font-size: 12px;

  &::placeholder { color: var(--t2); }
  &:focus { outline: none; border-color: var(--bdr3); }
}

// ── Rows ───────────────────────────────────────
.rows {
  flex: 1;
  overflow-y: auto;
  padding: 0 8px 12px;
  @include m.scrollbar;
}

.empty {
  padding: 20px 12px;
  text-align: center;
  font-size: 12px;
  color: var(--t2);
}

.row {
  display: flex;
  align-items: center;
  gap: 9px;
  padding: 8px 9px;
  border-radius: var(--r);
  color: var(--t1);
  cursor: pointer;
  @include m.theme-transition;

  &:hover { background: var(--bg2); }
}

.rowDisabled {
  opacity: 0.5;
  cursor: default;
  pointer-events: none;
}

// ── File row — mirrors FilePanel's pre-card .hitm design ──────
.hitm {
  position: relative;
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 11px 12px;
  border-radius: var(--rl);
  border: 1px solid transparent;
  transition: all 0.12s;
  margin-bottom: 4px;
  user-select: none;

  &:hover {
    background: var(--bg2);
    border-color: var(--bdr);
  }

  &.active {
    background: rgba(91, 164, 239, 0.06);
    border-color: var(--blue-bdr);

    &::before {
      content: '';
      position: absolute;
      left: 0;
      top: 6px;
      bottom: 6px;
      width: 3px;
      border-radius: 0 3px 3px 0;
      background: linear-gradient(180deg, var(--blue), #a78bfa);
    }
  }

  &.activeView {
    background: rgba(167, 139, 250, 0.06);
    border-color: rgba(167, 139, 250, 0.3);

    &::before {
      content: '';
      position: absolute;
      left: 0;
      top: 6px;
      bottom: 6px;
      width: 3px;
      border-radius: 0 3px 3px 0;
      background: linear-gradient(180deg, #a78bfa, var(--amber));
    }
  }
}

.hitmSelectable {
  cursor: pointer;
}

.ficon {
  width: 30px;
  height: 30px;
  border-radius: 7px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 10px;
  font-weight: 700;
  @include m.mono;
  flex-shrink: 0;

  &.vtt { background: var(--blue-dim); color: var(--blue); }
  &.srt { background: var(--green-dim); color: var(--green); }
}

.hi {
  flex: 1;
  min-width: 0;
}

.hn {
  font-size: 13px;
  font-weight: 500;
  color: var(--t0);
  @include m.truncate;
}

.hm {
  font-size: 12px;
  color: var(--t2);
  margin-top: 2px;
  @include m.mono;
}

.badge {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  font-size: 11px;
  padding: 2px 7px;
  border-radius: 99px;
  font-weight: 500;
  border: 1px solid transparent;
  white-space: nowrap;
  flex-shrink: 0;
  @include m.mono;
}

.bSelected {
  background: var(--blue-dim);
  color: var(--blue);
  border-color: var(--blue-bdr);
}

.bInferred {
  background: var(--green-dim);
  color: var(--green);
  border-color: var(--green-bdr);
}

.bNotInferred {
  background: var(--bg2);
  color: var(--t2);
}

// ── Checkbox (mirrors FilePanel's) ─────────────
.cb {
  width: 15px;
  height: 15px;
  flex-shrink: 0;
  border-radius: 4px;
  border: 1px solid var(--cb-bdr);
  background: transparent;
  cursor: pointer;
  position: relative;
  transition: all 0.12s;

  &:hover { border-color: var(--blue); }
}

.cbChecked {
  background: var(--blue);
  border-color: var(--blue);

  &::after {
    content: '';
    position: absolute;
    left: 4px;
    top: 1px;
    width: 4px;
    height: 8px;
    border: solid white;
    border-width: 0 2px 2px 0;
    transform: rotate(45deg);
  }
}

.cbIndet {
  background: var(--blue);
  border-color: var(--blue);

  &::after {
    content: '';
    position: absolute;
    left: 3px;
    top: 6px;
    width: 7px;
    height: 2px;
    background: white;
  }
}

.cbDisabled {
  opacity: 0.4;
  pointer-events: none;
}

// ── Pagination footer ──────────────────────────
.paginationBar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
  padding: 8px 12px;
  border-top: 1px solid var(--bdr);
  flex-shrink: 0;
}

.pageSizeSelect {
  padding: 3px 6px;
  background: var(--bg1);
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  color: var(--t1);
  font-family: var(--font-ui);
  font-size: 12px;
  outline: none;
  cursor: pointer;
  transition: border-color 0.12s;

  &:focus { border-color: var(--blue); }
}

.pageNav {
  display: flex;
  align-items: center;
  gap: 6px;
}

.pageNavBtn {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 24px;
  height: 24px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: var(--bg1);
  color: var(--t2);
  cursor: pointer;
  transition: all 0.12s;

  svg { width: 13px; height: 13px; }

  &:hover:not(:disabled) {
    background: var(--bg2);
    color: var(--t0);
  }

  &:disabled {
    opacity: 0.35;
    cursor: default;
  }
}

.pageIndicator {
  font-size: 12px;
  color: var(--t2);
  @include m.mono;
  min-width: 44px;
  text-align: center;
}
