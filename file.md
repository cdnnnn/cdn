// ═══════════════════════════════════════════════
// pages/UploadInfer/UploadInfer.tsx
// LectureAI · Upload & Inference page
// ═══════════════════════════════════════════════
import React, { useState, useEffect, useCallback, useRef } from 'react';
import { useTranslation } from 'react-i18next';
import { useAppSelector, useAppDispatch } from '../../store/hooks';
import { clearServerSelection } from '../../store/uploadSlice';
import api from '../../services/api';
import FilePanel, { FilePanelHandle } from './FilePanel';
import InferencePanel from './InferencePanel';
import WorkspacePanel, { WorkspacePanelHandle } from './WorkspacePanel';
import FileSidebar from './FileSidebar';
import TourGuide, { TourStep } from './TourGuide';
import styles from './UploadInfer.module.scss';

// ── Keyword Insights (from GET /files/{id}) ──────
export interface KGEdge { type: string; source: string; target: string; }
export interface KGNode { id: string; label: string; title?: string; value?: number; }
export interface KnowledgeGraph { edges: KGEdge[]; nodes: KGNode[]; }
export interface WordCloudData { wordcloud_image: string; }
export interface TimelineDataset { data: number[]; label: string; }
export interface TimelineData { labels: string[]; datasets: TimelineDataset[]; timestamped: boolean; }
export interface PrereqEdge { reason: string; enables: string; prerequisite: string; }
export interface PrereqNode { id: string; label: string; value: number; }
export interface PrerequisitesData { edges: PrereqEdge[]; nodes: PrereqNode[]; }
// importance is out of a fixed max of 10 (see ImportanceComplexityScatter)
export interface ImportanceComplexityItem { reason: string; keyword: string; frequency: number; complexity: string; importance: number; }
export interface ImportanceComplexityData { data: ImportanceComplexityItem[]; }
export interface GlossaryItem { term: string; definition: string; first_mentioned_ms: number; }
export interface GlossaryData { glossary: GlossaryItem[]; }

export interface KeywordInsights {
  enriched_keywords: unknown | null;
  knowledge_graph: KnowledgeGraph | null;
  word_cloud: WordCloudData | null;
  timeline: TimelineData | null;
  prerequisites: PrerequisitesData | null;
  importance_complexity: ImportanceComplexityData | null;
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
  const filePanelRef = useRef<FilePanelHandle>(null);
  const workspacePanelRef = useRef<WorkspacePanelHandle>(null);
  // Whether the Upload tab currently has any files loaded — lets the tour
  // adapt its "your uploaded files" step to either point at a real card
  // or describe what will appear once something's uploaded. Kept in a ref
  // too (always current, no stale-closure risk) so it can be read at the
  // exact moment the tour starts, from callbacks with static deps.
  const hasUploadedFiles = useAppSelector(s => s.upload.serverFiles.length > 0);
  const hasUploadedFilesRef = useRef(hasUploadedFiles);
  useEffect(() => { hasUploadedFilesRef.current = hasUploadedFiles; }, [hasUploadedFiles]);

  // ── Guided tour ──
  const [tourActive, setTourActive] = useState(false);
  // Snapshotted once when the tour opens, then left untouched until it
  // closes. tourSteps' length/content depends on this — if it tracked the
  // live value instead, a file list finishing its load mid-tour would
  // change the step array under TourGuide's feet, desyncing its internal
  // step index from what's actually being shown (e.g. "Next" appearing to
  // do nothing, or landing on the wrong tab).
  const [tourHasFiles, setTourHasFiles] = useState(false);
  const finishTour = useCallback(() => {
    setTourActive(false);
    filePanelRef.current?.closeActionsMenu();
  }, []);
  const startTour = useCallback(() => {
    setActiveTab('upload');
    setTourHasFiles(hasUploadedFilesRef.current);
    setTourActive(true);
  }, []);

  const allTourSteps: TourStep[] = [
    {
      target: 'tabbar',
      title: t('uploadInfer.tour.tabsTitle', 'Three steps, always available'),
      content: t('uploadInfer.tour.tabsBody', 'Upload & Manage, Inference, and View Results — you can jump between them any time, nothing is locked behind finishing a previous step.'),
      onEnter: () => setActiveTab('upload'),
    },
    {
      target: 'upload-dropzone',
      title: t('uploadInfer.tour.uploadTitle', 'Add your files here'),
      content: t('uploadInfer.tour.uploadBody', 'Drag in a lecture recording\u2019s captions (.vtt or .srt), or browse for one. You can drop in several files at once.'),
      onEnter: () => setActiveTab('upload'),
    },
    {
      target: 'upload-cards',
      title: t('uploadInfer.tour.cardsTitle', 'Your uploaded files'),
      content: tourHasFiles
        ? t('uploadInfer.tour.cardsBody', 'Every file shows up here as a card. The colored badge in the corner shows the format (VTT/SRT); the small icons below the filename open the exact prompt used for each content type (Summary, Keywords, Questions, Answer, True/False); and the bottom badge shows the file\u2019s current status. If a dictionary or prompt template is linked to the file, you\u2019ll see a small book or checklist icon too.')
        : t('uploadInfer.tour.cardsBodyEmpty', 'You haven\u2019t uploaded anything yet, so this area is empty for now. Once you do, each file becomes a card showing: a format badge (VTT/SRT), the filename, small icons for each content type\u2019s prompt (Summary, Keywords, Questions, Answer, True/False), a book/checklist icon if a dictionary or prompt template is linked, and a status badge at the bottom.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeActionsMenu(); },
    },
    // ── Icon-by-icon walkthrough — only shown when a real card exists to point at ──
    {
      target: 'upload-card-icon-summary',
      title: t('uploadInfer.tour.iconSummaryTitle', 'Summary icon'),
      content: t('uploadInfer.tour.iconSummaryBody', 'Opens the prompt used to generate this file\u2019s summary. Blue means it\u2019s been customized for this file; gray means it\u2019s still using the default.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeActionsMenu(); },
    },
    {
      target: 'upload-card-icon-keywords',
      title: t('uploadInfer.tour.iconKeywordsTitle', 'Keywords icon'),
      content: t('uploadInfer.tour.iconKeywordsBody', 'Opens the prompt used to extract this file\u2019s keywords \u2014 the tag-shaped icon.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeActionsMenu(); },
    },
    {
      target: 'upload-card-icon-questions',
      title: t('uploadInfer.tour.iconQuestionsTitle', 'Assessment Questions icon'),
      content: t('uploadInfer.tour.iconQuestionsBody', 'Opens the prompt used to generate quiz-style assessment questions \u2014 the question-mark icon.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeActionsMenu(); },
    },
    {
      target: 'upload-card-icon-answer',
      title: t('uploadInfer.tour.iconAnswerTitle', 'Short Answer icon'),
      content: t('uploadInfer.tour.iconAnswerBody', 'Opens the prompt used to generate short written answers \u2014 the speech-bubble icon.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeActionsMenu(); },
    },
    {
      target: 'upload-card-icon-truefalse',
      title: t('uploadInfer.tour.iconTrueFalseTitle', 'True/False icon'),
      content: t('uploadInfer.tour.iconTrueFalseBody', 'Opens the prompt used to generate True/False questions \u2014 the toggle-switch icon.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeActionsMenu(); },
    },
    {
      target: 'upload-sort',
      title: t('uploadInfer.tour.sortTitle', 'Sort the list'),
      content: t('uploadInfer.tour.sortBody', 'Sort your files by ID, Name, Date, or Status \u2014 click a column again to flip between ascending and descending.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeActionsMenu(); },
    },
    {
      target: 'upload-date-filter',
      title: t('uploadInfer.tour.dateFilterTitle', 'Filter by date range'),
      content: t('uploadInfer.tour.dateFilterBody', 'Narrow the file list down to a date range, then hit Apply. This affects only what\u2019s shown here on the Upload tab.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeActionsMenu(); },
    },
    {
      target: 'upload-status-filter',
      title: t('uploadInfer.tour.statusFilterTitle', 'Filter by status'),
      content: t('uploadInfer.tour.statusFilterBody', 'Show only files in one status: All, Waiting, Queued, Running, Inferenced, Error, or Pending \u2014 handy once you have a lot of files in flight.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeActionsMenu(); },
    },
    {
      target: 'upload-actions-trigger',
      title: t('uploadInfer.tour.actionsTriggerTitle', 'Click here for bulk actions'),
      content: t('uploadInfer.tour.actionsTriggerBody', 'This button opens a menu of everything you can do to many files at once \u2014 let\u2019s take a look inside.'),
      placement: 'left',
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeActionsMenu(); },
    },
    {
      target: 'upload-actions',
      title: t('uploadInfer.tour.actionsTitle', 'Bulk actions, one click away'),
      content: t('uploadInfer.tour.actionsBody', 'Here are all five bulk actions: Dictionary (link a term dictionary to your files), Prompt Template (link a saved prompt set), Search (find files by name), Export (download files), and Delete (remove files) \u2014 each applies to many files at once.'),
      placement: 'left',
      // Actually opens the menu so every action is visible while this step is shown.
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.openActionsMenu(); },
    },
    // ── One step per action button, menu kept open throughout ──
    {
      target: 'upload-action-dictionary',
      title: t('uploadInfer.tour.actionDictionaryTitle', 'Dictionary'),
      content: t('uploadInfer.tour.actionDictionaryBody', 'Link a term dictionary to one or more files, so inference uses the right spellings and terminology for your subject.'),
      placement: 'left',
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.openActionsMenu(); },
    },
    {
      target: 'upload-action-template',
      title: t('uploadInfer.tour.actionTemplateTitle', 'Prompt Template'),
      content: t('uploadInfer.tour.actionTemplateBody', 'Apply a saved set of prompts to one or more files in one go, instead of customizing each file\u2019s prompts by hand.'),
      placement: 'left',
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.openActionsMenu(); },
    },
    {
      target: 'upload-action-search',
      title: t('uploadInfer.tour.actionSearchTitle', 'Search'),
      content: t('uploadInfer.tour.actionSearchBody', 'Find a file by name \u2014 opens a search box for the Upload tab\u2019s file list.'),
      placement: 'left',
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.openActionsMenu(); },
    },
    {
      target: 'upload-action-export',
      title: t('uploadInfer.tour.actionExportTitle', 'Export'),
      content: t('uploadInfer.tour.actionExportBody', 'Select files and download them \u2014 useful for backing up or sharing results outside the app.'),
      placement: 'left',
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.openActionsMenu(); },
    },
    {
      target: 'upload-action-delete',
      title: t('uploadInfer.tour.actionDeleteTitle', 'Delete'),
      content: t('uploadInfer.tour.actionDeleteBody', 'Select files and remove them permanently \u2014 use with care, this can\u2019t be undone.'),
      placement: 'left',
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.openActionsMenu(); },
    },
    {
      target: 'infer-sidebar',
      title: t('uploadInfer.tour.inferSidebarTitle', 'Pick files to analyze'),
      content: t('uploadInfer.tour.inferSidebarBody', 'Select one or more files here \u2014 this list has its own date range, status filter, search, and sort, all independent of the Upload tab.'),
      placement: 'right',
      onEnter: () => { setActiveTab('infer'); filePanelRef.current?.closeActionsMenu(); },
    },
    {
      target: 'infer-date-filter',
      title: t('uploadInfer.tour.sidebarDateFilterTitle', 'Filter by date range'),
      content: t('uploadInfer.tour.sidebarDateFilterBody', 'Narrow this list down to a date range, then hit Apply \u2014 independent of the Upload tab\u2019s own date filter.'),
      placement: 'right',
      onEnter: () => setActiveTab('infer'),
    },
    {
      target: 'infer-status-filter',
      title: t('uploadInfer.tour.sidebarStatusFilterTitle', 'Filter by status'),
      content: t('uploadInfer.tour.sidebarStatusFilterBody', 'Show only files in one status here \u2014 independent of the Upload tab\u2019s own status filter.'),
      placement: 'right',
      onEnter: () => setActiveTab('infer'),
    },
    {
      target: 'infer-search',
      title: t('uploadInfer.tour.sidebarSearchTitle', 'Search'),
      content: t('uploadInfer.tour.sidebarSearchBody', 'Find a file by name within this list.'),
      placement: 'right',
      onEnter: () => setActiveTab('infer'),
    },
    {
      target: 'infer-sort',
      title: t('uploadInfer.tour.sidebarSortTitle', 'Sort'),
      content: t('uploadInfer.tour.sidebarSortBody', 'Sort this list by ID, Name, Date, or Status \u2014 click again to flip ascending/descending.'),
      placement: 'right',
      onEnter: () => setActiveTab('infer'),
    },
    {
      target: 'infer-settings',
      title: t('uploadInfer.tour.inferSettingsTitle', 'Choose what to generate'),
      content: t('uploadInfer.tour.inferSettingsBody', 'Turn on Summary, Keywords (with optional Keyword Insights), Assessment Questions, Short Answer, and True/False \u2014 each has its own customizable prompt. These options stay unchecked and disabled until you\u2019ve selected at least one file.'),
      onEnter: () => setActiveTab('infer'),
    },
    {
      target: 'infer-model',
      title: t('uploadInfer.tour.modelTitle', 'Pick a model'),
      content: t('uploadInfer.tour.modelBody', 'Choose which model runs the inference. Hit Refresh if the list looks out of date \u2014 you need a model selected before you can run anything.'),
      onEnter: () => setActiveTab('infer'),
    },
    // ── One step per checkbox, explaining what each generates ──
    {
      target: 'infer-check-summary',
      title: t('uploadInfer.tour.checkSummaryTitle', 'Summary'),
      content: t('uploadInfer.tour.checkSummaryBody', 'Generates a written summary of the file\u2019s content. Has its own customizable prompt once turned on.'),
      onEnter: () => setActiveTab('infer'),
    },
    {
      target: 'infer-check-keywords',
      title: t('uploadInfer.tour.checkKeywordsTitle', 'Keywords'),
      content: t('uploadInfer.tour.checkKeywordsBody', 'Extracts the key terms from the file. Turning this on reveals a nested Keyword Insights option underneath.'),
      onEnter: () => setActiveTab('infer'),
    },
    {
      target: 'infer-check-questions',
      title: t('uploadInfer.tour.checkQuestionsTitle', 'Assessment Questions'),
      content: t('uploadInfer.tour.checkQuestionsBody', 'Generates quiz-style assessment questions based on the file\u2019s content.'),
      onEnter: () => setActiveTab('infer'),
    },
    {
      target: 'infer-check-shortanswer',
      title: t('uploadInfer.tour.checkShortAnswerTitle', 'Short Answer'),
      content: t('uploadInfer.tour.checkShortAnswerBody', 'Generates short written-answer questions and responses from the file.'),
      onEnter: () => setActiveTab('infer'),
    },
    {
      target: 'infer-check-truefalse',
      title: t('uploadInfer.tour.checkTrueFalseTitle', 'True/False'),
      content: t('uploadInfer.tour.checkTrueFalseBody', 'Generates True/False questions from the file\u2019s content.'),
      onEnter: () => setActiveTab('infer'),
    },
    {
      target: 'infer-check-timestamped',
      title: t('uploadInfer.tour.checkTimestampedTitle', 'Timestamped Summary'),
      content: t('uploadInfer.tour.checkTimestampedBody', 'Generates a summary broken into timestamped segments, so each point links back to a moment in the recording.'),
      onEnter: () => setActiveTab('infer'),
    },
    {
      target: 'infer-run',
      title: t('uploadInfer.tour.inferRunTitle', 'Run it'),
      content: t('uploadInfer.tour.inferRunBody', 'Once you\u2019ve picked files and settings, run the batch here. You can watch progress or switch tabs \u2014 it keeps running either way.'),
      placement: 'left',
      onEnter: () => setActiveTab('infer'),
    },
    {
      target: 'results-sidebar',
      title: t('uploadInfer.tour.resultsSidebarTitle', 'Find a finished file'),
      content: t('uploadInfer.tour.resultsSidebarBody', 'Once a file\u2019s done, click it here to open its results. This list also has its own date range, status filter, search, and sort.'),
      placement: 'right',
      onEnter: () => setActiveTab('results'),
    },
    {
      target: 'results-date-filter',
      title: t('uploadInfer.tour.sidebarDateFilterTitle', 'Filter by date range'),
      content: t('uploadInfer.tour.sidebarDateFilterBody', 'Narrow this list down to a date range, then hit Apply \u2014 independent of the Upload tab\u2019s own date filter.'),
      placement: 'right',
      onEnter: () => setActiveTab('results'),
    },
    {
      target: 'results-status-filter',
      title: t('uploadInfer.tour.sidebarStatusFilterTitle', 'Filter by status'),
      content: t('uploadInfer.tour.sidebarStatusFilterBody', 'Show only files in one status here \u2014 independent of the Upload tab\u2019s own status filter.'),
      placement: 'right',
      onEnter: () => setActiveTab('results'),
    },
    {
      target: 'results-search',
      title: t('uploadInfer.tour.sidebarSearchTitle', 'Search'),
      content: t('uploadInfer.tour.sidebarSearchBody', 'Find a file by name within this list.'),
      placement: 'right',
      onEnter: () => setActiveTab('results'),
    },
    {
      target: 'results-sort',
      title: t('uploadInfer.tour.sidebarSortTitle', 'Sort'),
      content: t('uploadInfer.tour.sidebarSortBody', 'Sort this list by ID, Name, Date, or Status \u2014 click again to flip ascending/descending.'),
      placement: 'right',
      onEnter: () => setActiveTab('results'),
    },
    {
      target: 'results-content',
      title: t('uploadInfer.tour.resultsContentTitle', 'Your notes, ready to use'),
      content: t('uploadInfer.tour.resultsContentBody', 'Summary, keywords, quiz questions, and more \u2014 all generated from the file you selected. You can edit and re-generate individual sections without re-running the whole batch.'),
      placement: 'left',
      onEnter: () => setActiveTab('results'),
    },
    {
      target: 'results-tabbar',
      title: t('uploadInfer.tour.resultsTabbarTitle', 'Seven kinds of results'),
      content: t('uploadInfer.tour.resultsTabbarBody', 'Switch between Summary, Keywords, Assessment Questions, Short Answer, True/False, Timestamped Summary, and Keyword Insights \u2014 whichever ones you generated for this file.'),
      placement: 'bottom',
      onEnter: () => { setActiveTab('results'); workspacePanelRef.current?.setTab('summary'); },
    },
    // ── One step per tab, switching to it as it's highlighted ──
    {
      target: 'results-tab-summary',
      title: t('uploadInfer.tour.tabSummaryTitle', 'Summary'),
      content: t('uploadInfer.tour.tabSummaryBody', 'A written summary of the file\u2019s content.'),
      placement: 'bottom',
      onEnter: () => { setActiveTab('results'); workspacePanelRef.current?.setTab('summary'); },
    },
    {
      target: 'results-tab-keywords',
      title: t('uploadInfer.tour.tabKeywordsTitle', 'Keywords'),
      content: t('uploadInfer.tour.tabKeywordsBody', 'The key terms extracted from the file, shown as chips.'),
      placement: 'bottom',
      onEnter: () => { setActiveTab('results'); workspacePanelRef.current?.setTab('keywords'); },
    },
    {
      target: 'results-tab-assessment',
      title: t('uploadInfer.tour.tabAssessmentTitle', 'Assessment Questions'),
      content: t('uploadInfer.tour.tabAssessmentBody', 'Quiz-style assessment questions, with a multiple-choice quiz mode and a view-all mode.'),
      placement: 'bottom',
      onEnter: () => { setActiveTab('results'); workspacePanelRef.current?.setTab('assessment'); },
    },
    {
      target: 'results-tab-shortAnswer',
      title: t('uploadInfer.tour.tabShortAnswerTitle', 'Short Answer'),
      content: t('uploadInfer.tour.tabShortAnswerBody', 'Short written-answer questions and responses, revealed one at a time or all at once.'),
      placement: 'bottom',
      onEnter: () => { setActiveTab('results'); workspacePanelRef.current?.setTab('shortAnswer'); },
    },
    {
      target: 'results-tab-trueFalse',
      title: t('uploadInfer.tour.tabTrueFalseTitle', 'True/False'),
      content: t('uploadInfer.tour.tabTrueFalseBody', 'True/False questions, with the same quiz and view-all modes as Assessment Questions.'),
      placement: 'bottom',
      onEnter: () => { setActiveTab('results'); workspacePanelRef.current?.setTab('trueFalse'); },
    },
    {
      target: 'results-tab-timestampedSummary',
      title: t('uploadInfer.tour.tabTimestampedTitle', 'Timestamped Summary'),
      content: t('uploadInfer.tour.tabTimestampedBody', 'A summary broken into timestamped segments, each linking back to a moment in the recording.'),
      placement: 'bottom',
      onEnter: () => { setActiveTab('results'); workspacePanelRef.current?.setTab('timestampedSummary'); },
    },
    {
      target: 'results-tab-keywordInsights',
      title: t('uploadInfer.tour.tabKeywordInsightsTitle', 'Keyword Insights'),
      content: t('uploadInfer.tour.tabKeywordInsightsBody', 'Extra visualizations built from the extracted keywords \u2014 knowledge graph, word cloud, timeline, heatmap, clusters, and more.'),
      placement: 'bottom',
      onEnter: () => { setActiveTab('results'); workspacePanelRef.current?.setTab('keywordInsights'); },
    },
  ];

  // The five per-icon steps only make sense once there's a real card to
  // spotlight; skip them when the Upload tab is empty rather than showing
  // five dim, un-anchored tooltips in a row.
  const tourSteps: TourStep[] = tourHasFiles
    ? allTourSteps
    : allTourSteps.filter(s => !s.target.startsWith('upload-card-icon-'));

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

  const fetchFileData = useCallback(async (fileId: number, knownFileName?: string) => {
    setFileLoading(true);
    try {
      const res = await api.get(`/files/${fileId}`);
      const d = (res.data as any)?.data ?? {};
      setFileResult(prev => ({
        fileId,
        fileName: d.original_name ?? knownFileName ?? prev?.fileName ?? String(fileId),
        insertedAt: d.inserted_at ?? prev?.insertedAt ?? '',
        summary: d.summary ?? '',
        keywords: d.keywords ?? [],
        faq: d.faq ?? '[]',
        shortAnswer: d.short_answer ?? '[]',
        trueFalse: d.true_false ?? '[]',
        timestampedSummary: d.timestamped_summary ?? '[]',
        keywordInsights: d.keyword_insights ?? null,
      }));
    } catch { setFileResult(null); } finally { setFileLoading(false); }
  }, []);

  // Clicking a file (from the Upload tab's list, or the Workspace tab's
  // own picker) loads its results and jumps straight to the Workspace tab.
  // `fileName` is already known from the row that was clicked, so the
  // title can show it immediately/reliably rather than depending on the
  // detail endpoint's field naming.
  const handleFileClick = useCallback(async (fileId: number, fileName?: string) => {
    setActiveTab('results');
    if (fileId === activeFileId) return;
    setActiveFileId(fileId);
    setFileResult(fileName ? {
      fileId, fileName, insertedAt: '', summary: '', keywords: [], faq: '[]',
      shortAnswer: '[]', trueFalse: '[]', timestampedSummary: '[]', keywordInsights: null,
    } : null);
    await fetchFileData(fileId, fileName);
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
    { id: 'upload', label: t('uploadInfer.tabs.upload'), desc: 'Add & browse your files' },
    { id: 'infer', label: t('uploadInfer.tabs.infer'), desc: 'Configure & run analysis' },
    { id: 'results', label: t('uploadInfer.tabs.results'), desc: 'View summaries & quizzes' },
  ];

  return (
    <div className={styles.page}>
      {/* ── Header — title and self-explanatory tab cards share one row ── */}
      <div className={styles.headerBar}>
        <div className={styles.phTitleRow}>
          <div className={styles.phTitle}>{t('uploadInfer.pageTitle')}</div>
          <button type="button" className={styles.tourTriggerBtn} onClick={startTour}>
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
              <circle cx="8" cy="8" r="6.25" />
              <path d="M6.1 6.2a1.9 1.9 0 013.6.7c0 1.3-1.7 1.5-1.7 2.7M8 11.4v.1" />
            </svg>
            {t('uploadInfer.tour.takeTour', 'Take a tour')}
          </button>
        </div>

        <div className={styles.tabbar} role="tablist" data-tour="tabbar">
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
              <span className={styles.tabDesc}>{tab.desc}</span>
            </button>
          ))}
        </div>
      </div>

      <div className={styles.upbody}>
        <div className={styles.tabPane} style={{ display: activeTab === 'upload' ? 'flex' : 'none' }}>
          <FilePanel
            ref={filePanelRef}
            selectMode={selectMode}
            onEnterSelectMode={enterInferSelection}
            onExitSelectMode={exitInferSelection}
            onDeleteComplete={handleDeleteComplete}
            active={activeTab === 'upload'}
            onGoToInfer={() => setActiveTab('infer')}
          />
        </div>

        <div className={styles.tabPane} style={{ display: activeTab === 'infer' ? 'flex' : 'none' }}>
          <FileSidebar mode="select" active={activeTab === 'infer'} />
          <InferencePanel />
        </div>

        <div className={styles.tabPane} style={{ display: activeTab === 'results' ? 'flex' : 'none' }}>
          <FileSidebar mode="view" active={activeTab === 'results'} activeFileId={activeFileId} onFileClick={handleFileClick} />
          <div data-tour="results-content" style={{ display: 'contents' }}>
            <WorkspacePanel
              ref={workspacePanelRef}
              step2Visible={false}
              fileResult={fileResult}
              fileLoading={fileLoading}
              activeFileId={activeFileId}
              onResultUpdate={patch => setFileResult(prev => prev ? { ...prev, ...patch } : prev)}
            />
          </div>
        </div>
      </div>

      <TourGuide steps={tourSteps} active={tourActive} onFinish={finishTour} />
    </div>
  );
};

export default UploadInfer;
