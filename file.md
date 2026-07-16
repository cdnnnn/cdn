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
  const serverFilesData = useAppSelector(s => s.upload.serverFilesData);
  const filesTotal = useAppSelector(s => s.upload.filesTotal);
  const selectedServerIds = useAppSelector(s => s.upload.selectedServerIds);
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

  // ── Tab bar badges ──
  const inferBusyCount = serverFilesData.queued.length + serverFilesData.running.length;

  const tabs: { id: TabId; label: string }[] = [
    { id: 'upload', label: t('uploadInfer.tabs.upload') },
    { id: 'infer', label: t('uploadInfer.tabs.infer') },
    { id: 'results', label: t('uploadInfer.tabs.results') },
  ];

  return (
    <div className={styles.page}>
      <div className={styles.ph}>
        <div className={styles.phRow}>
          <div>
            <div className={styles.phTitle}>{t('uploadInfer.pageTitle')}</div>
            <div className={styles.phSub}>{t('uploadInfer.pageSub')}</div>
          </div>
        </div>
      </div>

      {/* ── Tab bar — three connected step buttons, every one always clickable ── */}
      <div className={styles.tabbarWrap}>
        <div className={styles.tabbar} role="tablist">
          <div className={styles.tabbarLine} />
          {tabs.map(tab => (
            <button
              key={tab.id}
              type="button"
              role="tab"
              aria-selected={activeTab === tab.id}
              className={`${styles.tabBtn} ${activeTab === tab.id ? styles.tabBtnActive : ''}`}
              onClick={() => setActiveTab(tab.id)}
            >
              <span className={styles.tabLabel}>{tab.label}</span>

              {tab.id === 'upload' && filesTotal > 0 && (
                <span className={styles.tabBadge}>
                  {t('uploadInfer.tabs.fileCount', { count: filesTotal })}
                </span>
              )}

              {tab.id === 'infer' && isBatchRunning && (
                <span className={styles.tabBadge}>
                  <span className={styles.tabLiveDot} />
                  {inferBusyCount > 0
                    ? t('uploadInfer.tabs.runningCount', { count: inferBusyCount })
                    : t('uploadInfer.tabs.running')}
                </span>
              )}
              {tab.id === 'infer' && !isBatchRunning && selectMode && selectedServerIds.length > 0 && (
                <span className={styles.tabBadge}>
                  {t('uploadInfer.tabs.selectedCount', { count: selectedServerIds.length })}
                </span>
              )}

              {tab.id === 'results' && activeFileId === null && (
                <span className={styles.tabHint}>{t('uploadInfer.tabs.resultsHint')}</span>
              )}
            </button>
          ))}
        </div>
      </div>

      <div className={styles.upbody}>
        <div className={styles.tabPane} style={{ display: activeTab === 'upload' ? 'flex' : 'none' }}>
          <FilePanel
            selectMode={selectMode}
            onEnterSelectMode={enterInferSelection}
            onExitSelectMode={exitInferSelection}
            onFileClick={handleFileClick}
            onDeleteComplete={handleDeleteComplete}
            activeFileId={activeFileId}
            onGoToInfer={() => setActiveTab('infer')}
          />
        </div>

        <div className={styles.tabPane} style={{ display: activeTab === 'infer' ? 'flex' : 'none' }}>
          <FileSidebar mode="select" />
          <InferencePanel />
        </div>

        <div className={styles.tabPane} style={{ display: activeTab === 'results' ? 'flex' : 'none' }}>
          <FileSidebar mode="view" activeFileId={activeFileId} onFileClick={handleFileClick} />
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

// ── Page header ──────────────────────────────────
.ph {
  flex-shrink: 0;
  padding: 18px 24px 0;
}

.phRow {
  @include m.flex-between;
}

.phTitle {
  font-size: 18px;
  font-weight: 600;
  color: var(--t0);
}

.phSub {
  margin-top: 2px;
  font-size: 13px;
  color: var(--t2);
}

// ── Tab bar — three connected step buttons, every tab always clickable ──
.tabbarWrap {
  flex-shrink: 0;
  padding: 20px 24px;
}

.tabbar {
  position: relative;
  display: flex;
  justify-content: space-between;
  max-width: 620px;
}

// Connecting line — sits behind the buttons; only visible in the gaps
// between them since each button has an opaque background.
.tabbarLine {
  position: absolute;
  top: 50%;
  left: 0;
  right: 0;
  height: 2px;
  background: var(--bdr2);
  transform: translateY(-50%);
  z-index: 0;
}

.tabBtn {
  position: relative;
  z-index: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 5px;
  width: 180px;
  height: 58px;
  padding: 0 14px;
  border-radius: 12px;
  border: 1px solid var(--bdr2);
  background: var(--bg1);
  cursor: pointer;
  @include m.theme-transition;

  &:hover {
    border-color: var(--bdr3);
    background: var(--bg2);
  }
}

.tabBtnActive {
  border-color: var(--blue);
  background: var(--blue-dim);
  box-shadow: 0 0 0 3px var(--blue-dim);

  &:hover {
    border-color: var(--blue);
    background: var(--blue-dim);
  }
}

.tabLabel {
  font-family: var(--font-ui);
  font-size: 13.5px;
  font-weight: 500;
  color: var(--t1);
  letter-spacing: 0.01em;
}

.tabBtnActive .tabLabel {
  font-weight: 600;
  color: var(--t0);
}

.tabBadge {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  font-size: 10px;
  font-weight: 600;
  color: var(--t2);
  background: var(--bg2);
  border-radius: 99px;
  padding: 1px 8px;
  line-height: 1.5;
}

.tabBtnActive .tabBadge {
  color: var(--blue);
  background: rgba(0, 0, 0, 0.12);
}

.tabHint {
  font-size: 10px;
  color: var(--t2);
}

.tabLiveDot {
  width: 4px;
  height: 4px;
  border-radius: 50%;
  background: currentColor;
  animation: tabLivePulse 1.4s ease-in-out infinite;
}

@keyframes tabLivePulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.35; }
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
  .ph { padding: 14px 16px 0; }
  .tabbarWrap { padding: 14px 16px; }
  .tabbar { flex-wrap: wrap; gap: 8px; max-width: none; }
  .tabbarLine { display: none; }
  .tabBtn { width: calc(50% - 4px); }

  .tabPane { flex-direction: column; }
}
