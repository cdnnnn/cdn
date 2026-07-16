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

  const tabs: { id: TabId; label: string; icon: React.ReactNode }[] = [
    {
      id: 'upload',
      label: t('uploadInfer.tabs.upload'),
      icon: (
        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
          <path d="M8 11V4M5 7l3-3 3 3" /><path d="M2.5 13.5h11" />
        </svg>
      ),
    },
    {
      id: 'infer',
      label: t('uploadInfer.tabs.infer'),
      icon: (
        <svg viewBox="0 0 16 16" fill="currentColor" stroke="none">
          <path d="M8 1l1.8 4.4L14 6.2l-3.3 2.5 1.2 4.3L8 10.8 4.1 13l1.2-4.3L2 6.2l4.2-.8z" />
        </svg>
      ),
    },
    {
      id: 'results',
      label: t('uploadInfer.tabs.results'),
      icon: (
        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
          <rect x="2.5" y="2" width="11" height="12" rx="1.5" /><path d="M5 5.5h6M5 8h6M5 10.5h3.5" />
        </svg>
      ),
    },
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

      {/* ── Tab bar — every tab is clickable at any time ── */}
      <div className={styles.tabbarWrap}>
        <div className={styles.tabbar} role="tablist">
          {tabs.map(tab => (
            <button
              key={tab.id}
              type="button"
              role="tab"
              aria-selected={activeTab === tab.id}
              className={`${styles.tabBtn} ${activeTab === tab.id ? styles.tabBtnActive : ''}`}
              onClick={() => setActiveTab(tab.id)}
            >
              <span className={styles.tabIcon}>{tab.icon}</span>
              {tab.label}
              {tab.id === 'upload' && filesTotal > 0 && (
                <span className={styles.tabBadge}>{filesTotal}</span>
              )}
              {tab.id === 'infer' && isBatchRunning && (
                <span className={styles.tabBadgeLive}>
                  <span className={styles.tabLiveDot} />
                  {inferBusyCount > 0
                    ? t('uploadInfer.tabs.runningCount', { count: inferBusyCount })
                    : t('uploadInfer.tabs.running')}
                </span>
              )}
              {tab.id === 'infer' && !isBatchRunning && selectMode && selectedServerIds.length > 0 && (
                <span className={styles.tabBadge}>{selectedServerIds.length}</span>
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

// ── Tab bar — segmented control, every tab always clickable ──
.tabbarWrap {
  flex-shrink: 0;
  padding: 14px 24px;
}

.tabbar {
  display: inline-flex;
  align-items: center;
  background: var(--bg1);
  border: 1px solid var(--bdr);
  border-radius: var(--rl);
  padding: 3px;
  gap: 2px;
}

.tabBtn {
  display: inline-flex;
  align-items: center;
  gap: 7px;
  height: 34px;
  padding: 0 16px;
  border: none;
  border-radius: calc(var(--rl) - 3px);
  background: transparent;
  color: var(--t2);
  font-family: var(--font-ui);
  font-size: 13.5px;
  font-weight: 500;
  cursor: pointer;
  @include m.theme-transition;

  &:hover {
    color: var(--t1);
  }
}

.tabBtnActive {
  background: var(--bg3);
  color: var(--t0);

  &:hover {
    color: var(--t0);
  }
}

.tabIcon {
  display: flex;
  width: 15px;
  height: 15px;
  flex-shrink: 0;

  svg { width: 100%; height: 100%; }
}

.tabBadge {
  font-size: 11px;
  font-weight: 600;
  color: var(--t2);
  background: var(--bg2);
  border-radius: 99px;
  padding: 1px 7px;
  line-height: 1.5;
}

.tabBtnActive .tabBadge {
  color: var(--blue);
  background: var(--blue-dim);
}

.tabBadgeLive {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  font-size: 11px;
  font-weight: 600;
  color: var(--blue);
  background: var(--blue-dim);
  border-radius: 99px;
  padding: 1px 8px 1px 6px;
  line-height: 1.5;
}

.tabLiveDot {
  width: 5px;
  height: 5px;
  border-radius: 50%;
  background: var(--blue);
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
  .tabbarWrap { padding: 12px 16px; }
  .tabBtn { padding: 0 12px; font-size: 12.5px; }

  .tabPane { flex-direction: column; }
}
