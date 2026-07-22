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
export type TimelineTimeUnit = 'minutes' | 'hours' | 'seconds';
export interface TimelineData { labels: string[]; datasets: TimelineDataset[]; timestamped: boolean; time_unit?: TimelineTimeUnit; }
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
    // ── Step 1: page title ──
    {
      target: 'ph-title',
      title: t('uploadInfer.tour.pageTitleTitle', 'Welcome to LectureAI'),
      content: t('uploadInfer.tour.pageTitleBody', 'Let\u2019s walk through how to upload lecture captions, run inference on them, and browse the results \u2014 step by step.'),
      onEnter: () => setActiveTab('upload'),
    },
    // ── Step 2: Upload & Manage tab ──
    {
      target: 'tab-upload',
      title: t('uploadInfer.tour.tabUploadTitle', 'Upload & Manage'),
      content: t('uploadInfer.tour.tabUploadBody', 'This is where you add new files and manage everything you\u2019ve already uploaded.'),
      onEnter: () => setActiveTab('upload'),
    },
    // ── Step 3: the drop-zone card ──
    {
      target: 'upload-step1-card',
      title: t('uploadInfer.tour.uploadTitle', 'Add your files here'),
      content: t('uploadInfer.tour.uploadBody', 'Drag in a lecture recording\u2019s captions (.vtt or .srt), or browse for one. You can drop in several files at once.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeSearch(); },
    },
    // ── Step 4: the uploaded-files header row ──
    {
      target: 'upload-section2-title-row',
      title: t('uploadInfer.tour.section2TitleRowTitle', 'Your file list'),
      content: t('uploadInfer.tour.section2TitleRowBody', 'Everything you\u2019ve uploaded shows up below, with its count, sorting, and date range controls right here at the top.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeSearch(); },
    },
    // ── Step 5: Sort By ──
    {
      target: 'upload-sort',
      title: t('uploadInfer.tour.sortTitle', 'Sort the list'),
      content: t('uploadInfer.tour.sortBody', 'Sort your files by ID, Name, Date, or Status \u2014 click a column again to flip between ascending and descending.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeSearch(); },
    },
    // ── Step 6: Date Range ──
    {
      target: 'upload-date-filter',
      title: t('uploadInfer.tour.dateFilterTitle', 'Filter by date range'),
      content: t('uploadInfer.tour.dateFilterBody', 'Narrow the file list down to a date range, then hit Apply. This affects only what\u2019s shown here on the Upload tab.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeSearch(); },
    },
    // ── Step 7: the filter/actions row ──
    {
      target: 'upload-filter-sort-row',
      title: t('uploadInfer.tour.filterSortRowTitle', 'Filter and bulk actions'),
      content: t('uploadInfer.tour.filterSortRowBody', 'Status filtering and every bulk action \u2014 Dictionary, Prompt Template, Search, Export, and Delete \u2014 live in this row.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeSearch(); },
    },
    // ── Step 8: Status dropdown ──
    {
      target: 'upload-status-filter',
      title: t('uploadInfer.tour.statusFilterTitle', 'Filter by status'),
      content: t('uploadInfer.tour.statusFilterBody', 'Show only files in one status: All, Waiting, Queued, Running, Inferenced, Error, or Pending \u2014 handy once you have a lot of files in flight.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeSearch(); },
    },
    // ── Step 9: Map Dictionary ──
    {
      target: 'upload-action-dictionary',
      title: t('uploadInfer.tour.actionDictionaryTitle', 'Dictionary'),
      content: t('uploadInfer.tour.actionDictionaryBody', 'Link a term dictionary to one or more files, so inference uses the right spellings and terminology for your subject.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeSearch(); },
    },
    // ── Step 10: Map prompt template ──
    {
      target: 'upload-action-template',
      title: t('uploadInfer.tour.actionTemplateTitle', 'Prompt Template'),
      content: t('uploadInfer.tour.actionTemplateBody', 'Apply a saved set of prompts to one or more files in one go, instead of customizing each file\u2019s prompts by hand.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeSearch(); },
    },
    // ── Step 11: Search button ──
    {
      target: 'upload-action-search',
      title: t('uploadInfer.tour.actionSearchTitle', 'Search'),
      content: t('uploadInfer.tour.actionSearchBody', 'Click here to open a search box for the Upload tab\u2019s file list.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeSearch(); },
    },
    // ── Step 12: the search box itself ──
    {
      target: 'upload-search-bar',
      title: t('uploadInfer.tour.searchBarTitle', 'Find a file by name'),
      content: t('uploadInfer.tour.searchBarBody', 'Type here to filter the list down to files whose name matches what you type.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.openSearch(); },
    },
    // ── Step 13: Export ──
    {
      target: 'upload-action-export',
      title: t('uploadInfer.tour.actionExportTitle', 'Export'),
      content: t('uploadInfer.tour.actionExportBody', 'Select files and download them \u2014 useful for backing up or sharing results outside the app.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeSearch(); },
    },
    // ── Step 14: Delete ──
    {
      target: 'upload-action-delete',
      title: t('uploadInfer.tour.actionDeleteTitle', 'Delete'),
      content: t('uploadInfer.tour.actionDeleteBody', 'Select files and remove them permanently \u2014 use with care, this can\u2019t be undone.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeSearch(); },
    },
    // ── Step 15: the file list body ──
    {
      target: 'upload-uploaded-body',
      title: t('uploadInfer.tour.uploadedBodyTitle', 'Your uploaded files'),
      content: tourHasFiles
        ? t('uploadInfer.tour.cardsBody', 'Every file shows up here as a card. The colored badge in the corner shows the format (VTT/SRT); the small icons below the filename open the exact prompt used for each content type (Summary, Keywords, Questions, Answer, True/False); and the bottom badge shows the file\u2019s current status. If a dictionary or prompt template is linked to the file, you\u2019ll see a small book or checklist icon too.')
        : t('uploadInfer.tour.cardsBodyEmpty', 'You haven\u2019t uploaded anything yet, so this area is empty for now. Once you do, each file becomes a card showing: a format badge (VTT/SRT), the filename, small icons for each content type\u2019s prompt (Summary, Keywords, Questions, Answer, True/False), a book/checklist icon if a dictionary or prompt template is linked, and a status badge at the bottom.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeSearch(); },
    },
    // ── Step 16: only shown when a real card exists to point at ──
    {
      target: 'upload-cards',
      title: t('uploadInfer.tour.cardsTitle', 'A single file card'),
      content: t('uploadInfer.tour.cardsSingleBody', 'Here\u2019s one of your files. Let\u2019s look at the small prompt icons below its name, one by one.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeSearch(); },
    },
    {
      target: 'upload-card-icon-summary',
      title: t('uploadInfer.tour.iconSummaryTitle', 'Summary icon'),
      content: t('uploadInfer.tour.iconSummaryBody', 'Opens the prompt used to generate this file\u2019s summary. Blue means it\u2019s been customized for this file; gray means it\u2019s still using the default.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeSearch(); },
    },
    {
      target: 'upload-card-icon-keywords',
      title: t('uploadInfer.tour.iconKeywordsTitle', 'Keywords icon'),
      content: t('uploadInfer.tour.iconKeywordsBody', 'Opens the prompt used to extract this file\u2019s keywords \u2014 the tag-shaped icon.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeSearch(); },
    },
    {
      target: 'upload-card-icon-questions',
      title: t('uploadInfer.tour.iconQuestionsTitle', 'Assessment Questions icon'),
      content: t('uploadInfer.tour.iconQuestionsBody', 'Opens the prompt used to generate quiz-style assessment questions \u2014 the question-mark icon.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeSearch(); },
    },
    {
      target: 'upload-card-icon-answer',
      title: t('uploadInfer.tour.iconAnswerTitle', 'Short Answer icon'),
      content: t('uploadInfer.tour.iconAnswerBody', 'Opens the prompt used to generate short written answers \u2014 the speech-bubble icon.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeSearch(); },
    },
    {
      target: 'upload-card-icon-truefalse',
      title: t('uploadInfer.tour.iconTrueFalseTitle', 'True/False icon'),
      content: t('uploadInfer.tour.iconTrueFalseBody', 'Opens the prompt used to generate True/False questions \u2014 the toggle-switch icon.'),
      onEnter: () => { setActiveTab('upload'); filePanelRef.current?.closeSearch(); },
    },
    // ── Step 17: Inference tab ──
    {
      target: 'tab-infer',
      title: t('uploadInfer.tour.tabInferTitle', 'Inference'),
      content: t('uploadInfer.tour.tabInferBody', 'Once files are uploaded, come here to choose which of them to analyze and what to generate.'),
      onEnter: () => setActiveTab('infer'),
    },
    // ── Step 18: the left sidebar container ──
    {
      target: 'infer-sidebar',
      title: t('uploadInfer.tour.inferSidebarTitle', 'Pick files to analyze'),
      content: t('uploadInfer.tour.inferSidebarBody', 'Select one or more files here \u2014 this list has its own date range, status filter, search, and sort, all independent of the Upload tab.'),
      placement: 'right',
      onEnter: () => setActiveTab('infer'),
    },
    // ── Step 19: from/to date ──
    {
      target: 'infer-date-filter',
      title: t('uploadInfer.tour.sidebarDateFilterTitle', 'Filter by date range'),
      content: t('uploadInfer.tour.sidebarDateFilterBody', 'Narrow this list down to a date range, then hit Apply \u2014 independent of the Upload tab\u2019s own date filter.'),
      placement: 'right',
      onEnter: () => setActiveTab('infer'),
    },
    // ── Step 20: Status dropdown ──
    {
      target: 'infer-status-filter',
      title: t('uploadInfer.tour.sidebarStatusFilterTitle', 'Filter by status'),
      content: t('uploadInfer.tour.sidebarStatusFilterBody', 'Show only files in one status here \u2014 independent of the Upload tab\u2019s own status filter.'),
      placement: 'right',
      onEnter: () => setActiveTab('infer'),
    },
    // ── Step 21: search bar ──
    {
      target: 'infer-search',
      title: t('uploadInfer.tour.sidebarSearchTitle', 'Search'),
      content: t('uploadInfer.tour.sidebarSearchBody', 'Find a file by name within this list.'),
      placement: 'right',
      onEnter: () => setActiveTab('infer'),
    },
    // ── Step 22: Sort By ──
    {
      target: 'infer-sort',
      title: t('uploadInfer.tour.sidebarSortTitle', 'Sort'),
      content: t('uploadInfer.tour.sidebarSortBody', 'Sort this list by ID, Name, Date, or Status \u2014 click again to flip ascending/descending.'),
      placement: 'right',
      onEnter: () => setActiveTab('infer'),
    },
    // ── Step 23: the file rows with checkboxes ──
    {
      target: 'infer-file-list',
      title: t('uploadInfer.tour.inferFileListTitle', 'Check off files'),
      content: t('uploadInfer.tour.inferFileListBody', 'Tick the checkbox on however many files you want to run inference on together \u2014 they\u2019ll all use the same settings below.'),
      placement: 'right',
      onEnter: () => setActiveTab('infer'),
    },
    // ── Step 24: the settings grid ──
    {
      target: 'infer-settings-grid',
      title: t('uploadInfer.tour.inferSettingsTitle', 'Choose what to generate'),
      content: t('uploadInfer.tour.inferSettingsBody', 'Turn on Summary, Keywords (with optional Keyword Insights), Assessment Questions, Short Answer, and True/False \u2014 each has its own customizable prompt. These options stay unchecked and disabled until you\u2019ve selected at least one file.'),
      onEnter: () => setActiveTab('infer'),
    },
    // ── Step 25: model, then one step per checkbox ──
    {
      target: 'infer-model',
      title: t('uploadInfer.tour.modelTitle', 'Pick a model'),
      content: t('uploadInfer.tour.modelBody', 'Choose which model runs the inference. Hit Refresh if the list looks out of date \u2014 you need a model selected before you can run anything.'),
      onEnter: () => setActiveTab('infer'),
    },
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
    // ── Step 26: Run Inference ──
    {
      target: 'infer-run',
      title: t('uploadInfer.tour.inferRunTitle', 'Run it'),
      content: t('uploadInfer.tour.inferRunBody', 'Once you\u2019ve picked files and settings, run the batch here. You can watch progress or switch tabs \u2014 it keeps running either way.'),
      placement: 'left',
      onEnter: () => setActiveTab('infer'),
    },
    // ── Step 27: View Results tab ──
    {
      target: 'tab-results',
      title: t('uploadInfer.tour.tabResultsTitle', 'View Results'),
      content: t('uploadInfer.tour.tabResultsBody', 'Once a file\u2019s finished processing, come here to browse everything that was generated for it.'),
      onEnter: () => setActiveTab('results'),
    },
    // ── Step 28: the sidebar container ──
    {
      target: 'results-sidebar',
      title: t('uploadInfer.tour.resultsSidebarTitle', 'Find a finished file'),
      content: t('uploadInfer.tour.resultsSidebarBody', 'Once a file\u2019s done, click it here to open its results. This list also has its own date range, status filter, search, and sort.'),
      placement: 'right',
      onEnter: () => setActiveTab('results'),
    },
    // ── Step 29: from/to date ──
    {
      target: 'results-date-filter',
      title: t('uploadInfer.tour.sidebarDateFilterTitle', 'Filter by date range'),
      content: t('uploadInfer.tour.sidebarDateFilterBody', 'Narrow this list down to a date range, then hit Apply \u2014 independent of the Upload tab\u2019s own date filter.'),
      placement: 'right',
      onEnter: () => setActiveTab('results'),
    },
    // ── Step 30: status dropdown ──
    {
      target: 'results-status-filter',
      title: t('uploadInfer.tour.sidebarStatusFilterTitle', 'Filter by status'),
      content: t('uploadInfer.tour.sidebarStatusFilterBody', 'Show only files in one status here \u2014 independent of the Upload tab\u2019s own status filter.'),
      placement: 'right',
      onEnter: () => setActiveTab('results'),
    },
    // ── Step 31: search box ──
    {
      target: 'results-search',
      title: t('uploadInfer.tour.sidebarSearchTitle', 'Search'),
      content: t('uploadInfer.tour.sidebarSearchBody', 'Find a file by name within this list.'),
      placement: 'right',
      onEnter: () => setActiveTab('results'),
    },
    // ── Step 32: Sort By ──
    {
      target: 'results-sort',
      title: t('uploadInfer.tour.sidebarSortTitle', 'Sort'),
      content: t('uploadInfer.tour.sidebarSortBody', 'Sort this list by ID, Name, Date, or Status \u2014 click again to flip ascending/descending.'),
      placement: 'right',
      onEnter: () => setActiveTab('results'),
    },
    // ── Step 33: the file rows (works whether the list is empty or not) ──
    {
      target: 'results-file-list',
      title: t('uploadInfer.tour.resultsFileListTitle', 'Your finished files'),
      content: t('uploadInfer.tour.resultsFileListBody', 'Every file that\u2019s been through inference shows up here \u2014 click one to load its results on the right. If nothing\u2019s finished yet, this list will simply be empty for now.'),
      placement: 'right',
      onEnter: () => setActiveTab('results'),
    },
    // ── Step 34: the results panel ──
    {
      target: 'results-panel',
      title: t('uploadInfer.tour.resultsContentTitle', 'Your notes, ready to use'),
      content: t('uploadInfer.tour.resultsContentBody', 'Summary, keywords, quiz questions, and more \u2014 all generated from the file you selected. You can edit and re-generate individual sections without re-running the whole batch.'),
      placement: 'left',
      onEnter: () => { setActiveTab('results'); workspacePanelRef.current?.setTab('summary'); },
    },
    // ── Step 35: one step per results tab ──
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

  // The card + its five prompt-icon steps only make sense once there's a
  // real card to spotlight; skip them when the Upload tab is empty rather
  // than showing dim, un-anchored tooltips in a row.
  const tourSteps: TourStep[] = tourHasFiles
    ? allTourSteps
    : allTourSteps.filter(s => s.target !== 'upload-cards' && !s.target.startsWith('upload-card-icon-'));

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
    { id: 'upload', label: t('uploadInfer.tabs.upload'), desc: t('uploadInfer.tabs.uploadDesc') },
    { id: 'infer', label: t('uploadInfer.tabs.infer'), desc: t('uploadInfer.tabs.inferDesc') },
    { id: 'results', label: t('uploadInfer.tabs.results'), desc: t('uploadInfer.tabs.resultsDesc') },
  ];

  return (
    <div className={styles.page}>
      {/* ── Header — title and self-explanatory tab cards share one row ── */}
      <div className={styles.headerBar}>
        <div className={styles.phTitleRow}>
          <div className={styles.phTitle} data-tour="ph-title">{t('uploadInfer.pageTitle')}</div>
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
              data-tour={`tab-${tab.id}`}
              className={`${styles.tabBtn} ${activeTab === tab.id ? styles.tabBtnActive : ''}`}
              onClick={() => setActiveTab(tab.id)}
              title={tab.desc}
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

















// ═══════════════════════════════════════════════
// pages/UploadInfer/FilePanel.tsx
// Content Analytics · Step-1 upload + uploaded files list
// ═══════════════════════════════════════════════
import React, { useRef, useState, useCallback, useEffect, forwardRef, useImperativeHandle } from 'react';
import { useTranslation } from 'react-i18next';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import {
  setDateFrom, setDateTo, setStatusFilter,
  serverFilesLoading, serverFilesSuccess, serverFilesFailure,
  setFilesPage, setFilesPageSize, setFilesSort, setFilesSearch,
  toggleServerFileSelection, toggleSelectAllServerFiles, clearServerSelection,
  type FilesSortBy, type SortOrder, type FileStatus, type ServerFile,
} from '../../store/uploadSlice';
import api from '../../services/api';
import styles from './FilePanel.module.scss';
import DictionaryAssociationModal from './DictionaryAssociationModal';
import PromptTemplateAssociationModal from './PromptTemplateAssociationModal';

// ── Types ────────────────────────────────────────
type UploadStatus = 'pending' | 'uploading' | 'success' | 'failed';
type PanelView = 'dropzone' | 'preview' | 'uploading';

interface BrowsedFile {
  id: string;
  file: File;
  name: string;
  size: string;
  ext: 'vtt' | 'srt';
  status: UploadStatus;
  error?: string;
}

// ── Helpers ──────────────────────────────────────
const ALLOWED = ['.vtt', '.srt'];
const isAllowed = (n: string) => ALLOWED.some(e => n.toLowerCase().endsWith(e));
const getExt = (n: string): 'vtt' | 'srt' => n.toLowerCase().endsWith('.srt') ? 'srt' : 'vtt';
const uid = () => Math.random().toString(36).slice(2);
const formatSize = (b: number) => b < 1024 ? `${b} B` : b < 1048576 ? `${(b / 1024).toFixed(1)} KB` : `${(b / 1048576).toFixed(1)} MB`;

// ── Status badge metadata (covers every value /files/by-date/ can return) ──
const STATUS_META: Record<FileStatus, { labelKey: string; cls: string }> = {
  waiting: { labelKey: 'uploadInfer.filePanel.statusWaiting', cls: 'bWaiting' },
  running: { labelKey: 'uploadInfer.filePanel.statusRunning', cls: 'bRunning' },
  completed: { labelKey: 'uploadInfer.filePanel.inferenced', cls: 'bInferred' },
  error: { labelKey: 'uploadInfer.filePanel.statusErrorBadge', cls: 'bError' },
  tbd: { labelKey: 'uploadInfer.filePanel.notInferenced', cls: 'bNotInferred' },
  queued: { labelKey: 'uploadInfer.filePanel.statusQueued', cls: 'bQueued' },
};

const STATUS_FILTER_OPTIONS: { value: FileStatus | ''; labelKey: string }[] = [
  { value: '', labelKey: 'uploadInfer.filePanel.statusAll' },
  { value: 'waiting', labelKey: 'uploadInfer.filePanel.statusWaiting' },
  { value: 'queued', labelKey: 'uploadInfer.filePanel.statusQueued' },
  { value: 'running', labelKey: 'uploadInfer.filePanel.statusRunning' },
  { value: 'completed', labelKey: 'uploadInfer.filePanel.inferenced' },
  { value: 'error', labelKey: 'uploadInfer.filePanel.statusError' },
  { value: 'tbd', labelKey: 'uploadInfer.filePanel.statusTbd' },
];

// ── Which of the 5 content prompts are customized for this file ──
const PROMPT_FIELDS: { key: keyof ServerFile; letter: string; labelKey: string; icon: string; tourId: string }[] = [
  { key: 'summary_prompt', letter: 'S', labelKey: 'uploadInfer.inferencePanel.summaryPrompt', icon: 'M3 2.5h7.5a1.5 1.5 0 011.5 1.5v9a.5.5 0 01-.5.5H4a1 1 0 01-1-1V2.5zM5.5 5.5h5M5.5 8h5M5.5 10.5h3', tourId: 'upload-card-icon-summary' },
  { key: 'keywords_prompt', letter: 'K', labelKey: 'uploadInfer.inferencePanel.keywordPrompt', icon: 'M2 8l4.5-4.5H12v5.5L7.5 13.5 2 8z M9.5 5.5h.01', tourId: 'upload-card-icon-keywords' },
  { key: 'faq_prompt', letter: 'Q', labelKey: 'uploadInfer.inferencePanel.questionPrompt', icon: 'M8 2.5a5.5 5.5 0 100 11 5.5 5.5 0 000-11zM6.2 6.3a1.8 1.8 0 013.4.7c0 1.2-1.6 1.4-1.6 2.6 M8 11.3v.1', tourId: 'upload-card-icon-questions' },
  // Speech-bubble-with-lines — reads as "a short written reply", distinct from the summary document icon above.
  { key: 'short_answer_prompt', letter: 'A', labelKey: 'uploadInfer.inferencePanel.shortAnswerPrompt', icon: 'M2.5 3.5h11a1 1 0 011 1v6a1 1 0 01-1 1H6.5L3.5 14.5V11.5h-1a1 1 0 01-1-1v-6a1 1 0 011-1z M5 7h6M5 9h3.5', tourId: 'upload-card-icon-answer' },
  // Toggle switch — a direct visual metaphor for a binary True/False choice, distinct from a generic "done" checkmark.
  { key: 'true_false_prompt', letter: 'T', labelKey: 'uploadInfer.inferencePanel.trueFalsePrompt', icon: 'M5.5 4h5a4 4 0 010 8h-5a4 4 0 010-8z M10.6 8a1.25 1.25 0 102.5 0 1.25 1.25 0 00-2.5 0z', tourId: 'upload-card-icon-truefalse' },
];

const filesToBrowsed = (files: File[]): BrowsedFile[] =>
  files.filter(f => isAllowed(f.name)).map(f => ({
    id: uid(), file: f, name: f.name, size: formatSize(f.size), ext: getExt(f.name), status: 'pending' as UploadStatus,
  }));

// Fixed 4-button sliding window around the current page — e.g. current=57,
// total=200 → [56, 57, 58, 59]. Keeps the row compact and predictable
// instead of a variable count with ellipsis gaps.
const PAGE_WINDOW = 4;
const getPageNumbers = (current: number, total: number): number[] => {
  if (total <= PAGE_WINDOW) return Array.from({ length: total }, (_, i) => i + 1);
  const start = Math.max(1, Math.min(current - 1, total - PAGE_WINDOW + 1));
  return Array.from({ length: PAGE_WINDOW }, (_, i) => start + i);
};

// ── Icons ────────────────────────────────────────
const IconPending: React.FC = () => <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round"><circle cx="8" cy="8" r="6" strokeDasharray="2 2" /></svg>;
const IconUploading: React.FC = () => <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round"><circle cx="8" cy="8" r="6" strokeOpacity="0.2" /><path d="M8 2a6 6 0 016 6" /></svg>;
const IconSuccess: React.FC = () => <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round"><circle cx="8" cy="8" r="6" /><path d="M5.5 8l2 2 3-3" /></svg>;
const IconFailed: React.FC = () => <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round"><circle cx="8" cy="8" r="6" /><path d="M6 6l4 4M10 6l-4 4" /></svg>;
const IconClose: React.FC = () => <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round"><path d="M4 4l8 8M12 4l-8 8" /></svg>;

// ── Checkbox ─────────────────────────────────────
interface CbProps { checked: boolean; indeterminate?: boolean; onChange: () => void; disabled?: boolean; }
const Checkbox: React.FC<CbProps> = ({ checked, indeterminate, onChange, disabled }) => {
  const ref = useRef<HTMLDivElement>(null);
  return (
    <div
      ref={ref}
      className={`${styles.cb} ${checked ? styles.cbChecked : ''} ${indeterminate ? styles.cbIndet : ''} ${disabled ? styles.cbDisabled : ''}`}
      onClick={e => { e.stopPropagation(); if (!disabled) onChange(); }}
      role="checkbox" aria-checked={indeterminate ? 'mixed' : checked} tabIndex={0}
      onKeyDown={e => { if (e.key === ' ' || e.key === 'Enter') { e.preventDefault(); if (!disabled) onChange(); } }}
    />
  );
};

// ── FilePanel ────────────────────────────────────
type SortKey = 'id' | 'name' | 'date' | 'status';
type SortDir = 'asc' | 'desc';

interface FilePanelProps {
  selectMode: boolean;
  onEnterSelectMode: () => void;
  onExitSelectMode: () => void;
  onDeleteComplete: (deletedIds: number[], all?: boolean) => void;
  // Whether the Upload tab is the one currently showing — triggers a
  // fresh fetch each time it becomes active, and clears stale state
  // when navigating away so a return visit doesn't flash old data.
  active: boolean;
  // Optional — lets the host switch to the Run inference tab once files
  // are picked here. FilePanel works fine without it (just no shortcut).
  onGoToInfer?: () => void;
}

// Exposed to the parent so the guided tour can programmatically open the
// actions popover for its "here's everything in here" step, without
// lifting actionsOpen into Redux just for that one use.
export interface FilePanelHandle {
  openActionsMenu: () => void;
  closeActionsMenu: () => void;
  openSearch: () => void;
  closeSearch: () => void;
}

const FilePanel = forwardRef<FilePanelHandle, FilePanelProps>(({ selectMode, onEnterSelectMode, onExitSelectMode, onDeleteComplete, active, onGoToInfer }, ref) => {
  const { t } = useTranslation();
  const dispatch = useAppDispatch();
  const {
    serverFiles,
    serverFilesLoading: filesLoading, serverFilesError: filesError,
    selectedServerIds, isBatchRunning,
    dateFrom, dateTo, statusFilter,
    lastBatchFinishedAt,
    filesPage, filesPageSize, filesTotal, filesTotalPages,
    filesSortBy, filesSortOrder, filesSearch,
  } = useAppSelector(s => s.upload);

  const [view, setView] = useState<PanelView>('dropzone');
  const [browsed, setBrowsed] = useState<BrowsedFile[]>([]);
  const [isDragOver, setIsDragOver] = useState(false);

  // ── Delete mode ──────────────────────────────
  const [deleteMode, setDeleteMode] = useState(false);
  const [deleteSelectedIds, setDeleteSelectedIds] = useState<number[]>([]);
  const [isDeleting, setIsDeleting] = useState(false);

  // ── Delete ALL (danger zone — wipes every file on the account) ──
  const [deleteAllConfirmOpen, setDeleteAllConfirmOpen] = useState(false);
  const [isDeletingAll, setIsDeletingAll] = useState(false);
  const [deleteAllError, setDeleteAllError] = useState<string | null>(null);

  // ── Export mode ──────────────────────────────
  const [exportMode, setExportMode] = useState(false);
  const [exportSelectedIds, setExportSelectedIds] = useState<number[]>([]);
  const [isExporting, setIsExporting] = useState(false);

  // ── Search ───────────────────────────────────
  const [searchOpen, setSearchOpen] = useState(false);
  const [searchQuery, setSearchQuery] = useState('');
  const searchInputRef = useRef<HTMLInputElement>(null);

  // ── Actions row (dictionary / template / search / export / delete) ──
  // Now always rendered inline rather than behind a popover. `actionsOpen`
  // is kept only so FilePanelHandle stays backward compatible with the
  // guided tour's openActionsMenu/closeActionsMenu calls — it's now inert.
  const [actionsOpen, setActionsOpen] = useState(false);

  useImperativeHandle(ref, () => ({
    openActionsMenu: () => setActionsOpen(true),
    closeActionsMenu: () => setActionsOpen(false),
    openSearch: () => openSearch(),
    closeSearch: () => closeSearch(),
  }), []);

  // ── Dictionary association modal ─────────────
  const [dictModalOpen, setDictModalOpen] = useState(false);

  // ── Prompt template association modal ────────
  const [templateModalOpen, setTemplateModalOpen] = useState(false);

  // ── Per-file prompt viewer popup (opened from a card's prompt icons) ──
  const [promptViewer, setPromptViewer] = useState<{ file: ServerFile; fieldIdx: number } | null>(null);

  const openSearch = () => {
    setSearchOpen(true);
    setTimeout(() => searchInputRef.current?.focus(), 50);
  };
  const closeSearch = () => {
    setSearchOpen(false);
    setSearchQuery('');
  };

  const enterDeleteMode = () => { setDeleteMode(true); setDeleteSelectedIds([]); };
  const exitDeleteMode = () => { setDeleteMode(false); setDeleteSelectedIds([]); closeDeleteAllConfirm(); };

  const toggleDeleteSelection = (id: number) => {
    setDeleteSelectedIds(prev =>
      prev.includes(id) ? prev.filter(x => x !== id) : [...prev, id]
    );
  };

  // Checked/indeterminate reflect only the CURRENT PAGE's files — a file
  // selected on another page doesn't count toward "all selected" here.
  const pageDeleteSelectedCount = serverFiles.filter(f => deleteSelectedIds.includes(f.id)).length;
  const allDeleteSelected = serverFiles.length > 0 && pageDeleteSelectedCount === serverFiles.length;
  const someDeleteSelected = pageDeleteSelectedCount > 0 && !allDeleteSelected;

  const toggleDeleteSelectAll = () => {
    const pageIds = serverFiles.map(f => f.id);
    setDeleteSelectedIds(prev => {
      if (allDeleteSelected) {
        // Unchecking here only removes THIS page's ids — selections made on
        // other pages are left untouched.
        const pageIdSet = new Set(pageIds);
        return prev.filter(id => !pageIdSet.has(id));
      }
      // Checking here adds THIS page's ids on top of whatever's already
      // selected elsewhere, without duplicating.
      const existing = new Set(prev);
      return [...prev, ...pageIds.filter(id => !existing.has(id))];
    });
  };

  const handleConfirmDelete = async () => {
    if (!deleteSelectedIds.length) return;
    setIsDeleting(true);
    try {
      const res = await api.post('/files/delete', { fileID: deleteSelectedIds, all: false });
      if ((res.data as any)?.status === 'success' || res.status === 200) {
        const deleted = [...deleteSelectedIds];
        dispatch(clearServerSelection());
        await fetchFiles();
        exitDeleteMode();
        onDeleteComplete(deleted);
      }
    } catch {
      // silently ignore — stay in delete mode so user can retry
    } finally {
      setIsDeleting(false);
    }
  };

  // ── Delete ALL — wipes every file on the account, ignores fileID ──
  const openDeleteAllConfirm = () => {
    setDeleteAllError(null);
    setDeleteAllConfirmOpen(true);
  };
  const closeDeleteAllConfirm = () => {
    setDeleteAllConfirmOpen(false);
    setDeleteAllError(null);
  };
  const handleConfirmDeleteAll = async () => {
    setIsDeletingAll(true);
    setDeleteAllError(null);
    try {
      const res = await api.post('/files/delete', {
        all: true,
        start_date: dateFrom,
        end_date: dateTo,
      });
      if ((res.data as any)?.status === 'success' || res.status === 200) {
        dispatch(clearServerSelection());
        closeDeleteAllConfirm();
        exitDeleteMode();
        await fetchFiles({ page: 1 });
        onDeleteComplete([], true);
      } else {
        setDeleteAllError(t('uploadInfer.filePanel.deleteAllFailed'));
      }
    } catch (err) {
      setDeleteAllError(err instanceof Error ? err.message : t('uploadInfer.filePanel.deleteAllFailed'));
    } finally {
      setIsDeletingAll(false);
    }
  };

  // ── Export handlers ───────────────────────────
  const enterExportMode = () => { setExportMode(true); setExportSelectedIds([]); setExportAllError(null); };
  const exitExportMode = () => { setExportMode(false); setExportSelectedIds([]); setExportAllError(null); };

  const toggleExportSelection = (id: number) => {
    setExportSelectedIds(prev =>
      prev.includes(id) ? prev.filter(x => x !== id) : [...prev, id]
    );
  };

  // Checked/indeterminate reflect only the CURRENT PAGE's files — a file
  // selected on another page doesn't count toward "all selected" here.
  const pageExportSelectedCount = serverFiles.filter(f => exportSelectedIds.includes(f.id)).length;
  const allExportSelected = serverFiles.length > 0 && pageExportSelectedCount === serverFiles.length;
  const someExportSelected = pageExportSelectedCount > 0 && !allExportSelected;

  const toggleExportSelectAll = () => {
    const pageIds = serverFiles.map(f => f.id);
    setExportSelectedIds(prev => {
      if (allExportSelected) {
        const pageIdSet = new Set(pageIds);
        return prev.filter(id => !pageIdSet.has(id));
      }
      const existing = new Set(prev);
      return [...prev, ...pageIds.filter(id => !existing.has(id))];
    });
  };

  const handleConfirmExport = async () => {
    if (!exportSelectedIds.length) return;
    setIsExporting(true);
    try {
      const res = await api.post(
        '/export_excel',
        { file_ids: exportSelectedIds, all: false },
        { responseType: 'blob' },
      );

      // Use the original file name (without extension) + .xlsx
      // For a single file: "CS401_Week03.vtt" → "CS401_Week03.xlsx"
      // For multiple files: "export_2026-05-07.xlsx"
      let filename: string;
      if (exportSelectedIds.length === 1) {
        const f = serverFiles.find(sf => sf.id === exportSelectedIds[0]);
        const base = f ? f.original_name.replace(/\.[^.]+$/, '') : String(exportSelectedIds[0]);
        filename = `${base}.xlsx`;
      } else {
        const today = new Date().toISOString().slice(0, 10);
        filename = `export_${today}.xlsx`;
      }

      const url = URL.createObjectURL(new Blob([res.data]));
      const a = document.createElement('a');
      a.href = url;
      a.download = filename;
      document.body.appendChild(a);
      a.click();
      a.remove();
      URL.revokeObjectURL(url);

      exitExportMode();
    } catch {
      // silently ignore — stay in export mode so user can retry
    } finally {
      setIsExporting(false);
    }
  };

  // ── Export ALL — every file in the current date range, ignores file_ids ──
  const [isExportingAll, setIsExportingAll] = useState(false);
  const [exportAllError, setExportAllError] = useState<string | null>(null);

  const handleExportAll = async () => {
    setIsExportingAll(true);
    setExportAllError(null);
    try {
      const res = await api.post(
        '/export_excel',
        { all: true, start_date: dateFrom, end_date: dateTo },
        { responseType: 'blob' },
      );

      const filename = `export_all_${dateFrom}_${dateTo}.xlsx`;
      const url = URL.createObjectURL(new Blob([res.data]));
      const a = document.createElement('a');
      a.href = url;
      a.download = filename;
      document.body.appendChild(a);
      a.click();
      a.remove();
      URL.revokeObjectURL(url);

      exitExportMode();
    } catch (err) {
      setExportAllError(err instanceof Error ? err.message : t('uploadInfer.filePanel.exportAllFailed'));
    } finally {
      setIsExportingAll(false);
    }
  };

  const fileInputRef = useRef<HTMLInputElement>(null);
  const folderInputRef = useRef<HTMLInputElement>(null);

  // ── Fetch server files (paginated, sorted, searched) ──
  // Any param not passed in `opts` falls back to the current redux value, so
  // callers only need to specify what's actually changing (e.g. just `page`).
  // In-flight guard: set synchronously before the await, so a second
  // overlapping call (e.g. StrictMode's intentional double mount-effect
  // invocation in dev, or two effects both wanting a fresh fetch at once)
  // bails out instead of firing a second real /files/by-date/ request.
  const filesFetchInFlightRef = useRef(false);
  // Extra guard for the non-overlapping case: two effects can each fire a
  // fetch with identical params moments apart (one finishes just before
  // the other starts), which the in-flight flag alone won't catch since
  // it's already been reset by then. Skip a repeat of the exact same
  // request if it was made very recently.
  const lastFilesFetchRef = useRef<{ sig: string; at: number } | null>(null);
  const fetchFiles = useCallback(async (opts?: {
    from?: string; to?: string;
    page?: number; pageSize?: number;
    sortBy?: FilesSortBy; sortOrder?: SortOrder;
    search?: string;
    statusFilter?: FileStatus | '';
  }) => {
    if (filesFetchInFlightRef.current) return;
    const from = opts?.from ?? dateFrom;
    const to = opts?.to ?? dateTo;
    const page = opts?.page ?? filesPage;
    const pageSize = opts?.pageSize ?? filesPageSize;
    const sortBy = opts?.sortBy ?? filesSortBy;
    const sortOrder = opts?.sortOrder ?? filesSortOrder;
    const search = opts?.search ?? filesSearch;
    const statusF = opts?.statusFilter ?? statusFilter;

    const sig = JSON.stringify({ from, to, page, pageSize, sortBy, sortOrder, search, statusF });
    const now = Date.now();
    if (lastFilesFetchRef.current && lastFilesFetchRef.current.sig === sig && now - lastFilesFetchRef.current.at < 800) {
      return;
    }
    lastFilesFetchRef.current = { sig, at: now };
    filesFetchInFlightRef.current = true;

    dispatch(serverFilesLoading());
    try {
      const res = await api.post('/files/by-date/', {
        start_date: from,
        end_date: to,
        page,
        page_size: pageSize,
        sort_by: sortBy,
        sort_order: sortOrder,
        ...(search ? { search } : {}),
        ...(statusF ? { status_filter: statusF } : {}),
      });
      const d = (res.data as any)?.data ?? {};
      dispatch(serverFilesSuccess({
        files: d.data ?? [],
        total: d.total ?? 0,
        page: d.page ?? page,
        pageSize: d.page_size ?? pageSize,
        totalPages: d.total_pages ?? 1,
        sortBy, sortOrder, search,
      }));
    } catch (err) {
      dispatch(serverFilesFailure(err instanceof Error ? err.message : 'Failed to load files.'));
    } finally {
      filesFetchInFlightRef.current = false;
    }
  }, [dispatch, dateFrom, dateTo, filesPage, filesPageSize, filesSortBy, filesSortOrder, filesSearch, statusFilter]);

  // Re-fetch /by-date/ whenever a batch finishes so completed statuses are up to date
  useEffect(() => {
    if (lastBatchFinishedAt !== null) {
      fetchFiles();
    }
  }, [lastBatchFinishedAt]); // eslint-disable-line

  // ── Pagination / page-size handlers ──
  const goToPage = useCallback((p: number) => {
    if (p < 1 || p > filesTotalPages || p === filesPage || filesLoading) return;
    fetchFiles({ page: p });
  }, [filesTotalPages, filesPage, filesLoading, fetchFiles]);

  const handlePageSizeChange = useCallback((size: number) => {
    dispatch(setFilesPageSize(size));
    fetchFiles({ pageSize: size, page: 1 });
  }, [dispatch, fetchFiles]);

  // ── Browse / drop ────────────────────────────
  const handleFiles = useCallback((files: File[]) => {
    const valid = filesToBrowsed(files);
    if (!valid.length) return;
    setBrowsed(valid);
    setView('preview');
  }, []);

  const onFileChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    if (e.target.files) handleFiles(Array.from(e.target.files));
    e.target.value = '';
  };

  const onDragOver = (e: React.DragEvent) => { e.preventDefault(); setIsDragOver(true); };
  const onDragLeave = () => setIsDragOver(false);
  const onDrop = (e: React.DragEvent) => {
    e.preventDefault(); setIsDragOver(false);
    const files: File[] = [];
    if (e.dataTransfer.items) {
      const traverse = (entry: FileSystemEntry) => {
        if (entry.isFile) (entry as FileSystemFileEntry).file(f => { if (isAllowed(f.name)) files.push(f); });
        else if (entry.isDirectory) { const r = (entry as FileSystemDirectoryEntry).createReader(); r.readEntries(es => es.forEach(traverse)); }
      };
      Array.from(e.dataTransfer.items).forEach(item => { const e = item.webkitGetAsEntry?.(); if (e) traverse(e); });
      setTimeout(() => { if (files.length) handleFiles(files); }, 200);
    } else handleFiles(Array.from(e.dataTransfer.files));
  };

  // ── Upload actions ───────────────────────────
  const removeFile = (id: string) => { const n = browsed.filter(f => f.id !== id); setBrowsed(n); if (!n.length) setView('dropzone'); };
  const handleCancel = () => { setBrowsed([]); setView('dropzone'); };

  const handleUpload = async () => {
    setView('uploading');
    for (const bf of browsed) {
      setBrowsed(prev => prev.map(f => f.id === bf.id ? { ...f, status: 'uploading' } : f));
      try {
        const fd = new FormData(); fd.append('file', bf.file);
        const res = await api.post('/upload', fd, { headers: { 'Content-Type': 'multipart/form-data' } });
        const ok = res.status === 200 && (res.data as any)?.status === 'Success';
        setBrowsed(prev => prev.map(f => f.id === bf.id ? { ...f, status: ok ? 'success' : 'failed', error: ok ? undefined : 'Upload failed' } : f));
      } catch (err: any) {
        const apiMessage = err?.response?.data?.details?.result;
        const errorMsg = err?.response?.status === 403 && apiMessage
          ? apiMessage
          : err instanceof Error ? err.message : 'Upload failed';
        setBrowsed(prev => prev.map(f => f.id === bf.id ? { ...f, status: 'failed', error: errorMsg } : f));
      }
    }
  };

  useEffect(() => {
    if (view !== 'uploading') return;
    if (!browsed.every(f => f.status === 'success' || f.status === 'failed')) return;
    fetchFiles({ page: 1 });
    const allOk = browsed.every(f => f.status === 'success');
    if (allOk) {
      // All files uploaded successfully — clear immediately
      setBrowsed([]);
      setView('dropzone');
      return;
    }
    // Some failed — leave the list visible briefly so the user can read the error
    const t = setTimeout(() => { setBrowsed([]); setView('dropzone'); }, 2500);
    return () => clearTimeout(t);
  }, [browsed, view]); // eslint-disable-line

  const handleApply = () => fetchFiles({ page: 1 });
  const handleStatusFilterChange = (value: FileStatus | '') => {
    dispatch(setStatusFilter(value));
    fetchFiles({ page: 1, statusFilter: value });
  };

  // Select-all state for uploaded files (scoped to the current page)
  // Checked/indeterminate reflect only the CURRENT PAGE's files — files
  // selected on other pages (kept in redux) don't affect this checkbox.
  const pageSelectedCount = serverFiles.filter(f => selectedServerIds.includes(f.id)).length;
  const allSelected = serverFiles.length > 0 && pageSelectedCount === serverFiles.length;
  const someSelected = pageSelectedCount > 0 && !allSelected;

  // ── Sort (server-side — maps UI key to the API's sort_by values) ──────
  const SORT_KEY_TO_API: Record<SortKey, FilesSortBy> = {
    id: 'id', name: 'original_name', date: 'inserted_at', status: 'status',
  };
  const API_TO_SORT_KEY: Record<FilesSortBy, SortKey> = {
    id: 'id', original_name: 'name', inserted_at: 'date', status: 'status',
  };
  const sort = { key: API_TO_SORT_KEY[filesSortBy], dir: filesSortOrder as SortDir };

  // ── exitSelectMode — clears selections then notifies parent ──
  const exitSelectMode = useCallback(() => {
    dispatch(clearServerSelection());
    onExitSelectMode();
  }, [dispatch, onExitSelectMode]);

  const handleSort = useCallback((key: SortKey) => {
    const apiKey = SORT_KEY_TO_API[key];
    const dir: SortDir = filesSortBy === apiKey ? (filesSortOrder === 'asc' ? 'desc' : 'asc') : 'asc';
    dispatch(setFilesSort({ sortBy: apiKey, sortOrder: dir }));
    fetchFiles({ sortBy: apiKey, sortOrder: dir, page: 1 });
  }, [filesSortBy, filesSortOrder, dispatch, fetchFiles]); // eslint-disable-line

  // ── Search (debounced, server-side) ──────────────────────────────────
  const skipNextSearchEffect = useRef(true);
  useEffect(() => {
    if (skipNextSearchEffect.current) { skipNextSearchEffect.current = false; return; }
    const q = searchQuery.trim();
    const handle = setTimeout(() => {
      dispatch(setFilesSearch(q));
      fetchFiles({ search: q, page: 1 });
    }, 400);
    return () => clearTimeout(handle);
  }, [searchQuery]); // eslint-disable-line

  // Own fresh fetch every time the Upload tab becomes active — and when
  // it goes inactive (navigated away from), clear the list and reset
  // search/sort/status-filter/page so a return visit shows a clean
  // loading state instead of the previous stale results. Also reset any
  // in-progress delete/export/select-files action, so switching tabs and
  // coming back never leaves a stale selection or an open confirm panel
  // behind.
  useEffect(() => {
    if (active) {
      fetchFiles({ page: 1 });
    } else {
      skipNextSearchEffect.current = true;
      setSearchQuery('');
      dispatch(setStatusFilter(''));
      dispatch(serverFilesSuccess({
        files: [], total: 0, page: 1, pageSize: filesPageSize, totalPages: 1,
        sortBy: 'id', sortOrder: 'desc', search: '',
      }));

      setDeleteMode(false);
      setDeleteSelectedIds([]);

      setExportMode(false);
      setExportSelectedIds([]);

      dispatch(clearServerSelection());
      if (selectMode) onExitSelectMode?.();
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [active]);

  const displayedFiles = serverFiles;

  const done = browsed.filter(f => f.status === 'success').length;
  const failed = browsed.filter(f => f.status === 'failed').length;
  const finished = done + failed;

  return (
    <div className={styles.panel}>
      <input ref={fileInputRef} type="file" accept=".vtt,.srt" multiple style={{ display: 'none' }} onChange={onFileChange} />
      <input ref={folderInputRef} type="file" accept=".vtt,.srt" multiple style={{ display: 'none' }} onChange={onFileChange} {...{ webkitdirectory: 'true' }} />

      {/* ══ SECTION 1: Step-1 ══ */}
      <div className={styles.step1} data-tour="upload-dropzone">
        <div className={styles.step1Card} data-tour="upload-step1-card">
        <div className={styles.step1Bar}>
          <span className={styles.slbl}>{t('uploadInfer.filePanel.step1Label')}</span>
          {(view === 'preview' || view === 'uploading') && (
            <span className={`${styles.badge} ${styles.bInfo}`}>{t('uploadInfer.filePanel.selected', { count: browsed.length })}</span>
          )}
        </div>

        <div className={styles.step1Content}>
          {view === 'dropzone' && (
            <div className={`${styles.dropzone} ${isDragOver ? styles.dragOver : ''}`}
              onDragOver={onDragOver} onDragLeave={onDragLeave} onDrop={onDrop}
              onClick={() => fileInputRef.current?.click()}>
              <div className={styles.dzIc}>
                <svg viewBox="0 0 18 18" fill="none" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round" stroke="var(--blue)">
                  <path d="M9 12V4M6 7l3-3 3 3" /><path d="M2 15h14" />
                </svg>
              </div>
              <div className={styles.dzTitle}>{isDragOver ? t('uploadInfer.filePanel.dropTitle') : t('uploadInfer.filePanel.dropTitleDefault')}</div>
              <div className={styles.dzSub}>{t('uploadInfer.filePanel.dropSub')}</div>
              <div className={styles.dzActions} onClick={e => e.stopPropagation()}>
                <button className={`${styles.btn} ${styles.btnP} ${styles.btnSm}`} onClick={() => fileInputRef.current?.click()}>{t('uploadInfer.filePanel.browseFiles')}</button>
                <button className={`${styles.btn} ${styles.btnSm}`} onClick={() => folderInputRef.current?.click()}>{t('uploadInfer.filePanel.folder')}</button>
              </div>
            </div>
          )}

          {(view === 'preview' || view === 'uploading') && (
            <div className={styles.previewWrap}>
              <div className={styles.previewList}>
                {browsed.map(bf => (
                  <div key={bf.id} className={`${styles.fileCard} ${styles[bf.status]}`}>
                    <div className={`${styles.extBadge} ${styles[bf.ext]}`}>{bf.ext.toUpperCase()}</div>
                    <div className={styles.fileInfo}>
                      <div className={styles.fileName}>{bf.name}</div>
                      <div className={styles.fileMeta}>
                        <span className={styles.fileSizeChip}>{bf.size}</span>
                        {bf.status === 'failed' && bf.error && <span className={styles.fileError}>{bf.error}</span>}
                        {bf.status === 'pending' && <span className={styles.fileStatusText}>{t('uploadInfer.filePanel.fileStatus.ready')}</span>}
                        {bf.status === 'uploading' && <span className={styles.fileStatusText}>{t('uploadInfer.filePanel.fileStatus.uploading')}</span>}
                        {bf.status === 'success' && <span className={styles.fileStatusTextSuccess}>{t('uploadInfer.filePanel.fileStatus.uploaded')}</span>}
                      </div>
                    </div>
                    <div className={`${styles.statusIc} ${styles[bf.status]}`}>
                      {bf.status === 'pending' && <IconPending />}
                      {bf.status === 'uploading' && <IconUploading />}
                      {bf.status === 'success' && <IconSuccess />}
                      {bf.status === 'failed' && <IconFailed />}
                    </div>
                    {view === 'preview' && <button className={styles.removeBtn} onClick={() => removeFile(bf.id)}><IconClose /></button>}
                  </div>
                ))}
              </div>
            </div>
          )}
        </div>

        {/* ── Fixed footer — stays put once files are browsed, doesn't scroll away with the list ── */}
        {view === 'preview' && (
          <div className={styles.step1Footer}>
            <button className={`${styles.btn} ${styles.btnP} ${styles.step1FooterBtn}`} onClick={handleUpload}>
              <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                <path d="M8 11V4M5 7l3-3 3 3" /><path d="M2.5 13.5h11" />
              </svg>
              {t('uploadInfer.filePanel.uploadBtn', { count: browsed.length })}
            </button>
            <button className={`${styles.btn} ${styles.btnDanger}`} onClick={handleCancel}>{t('uploadInfer.filePanel.cancelBtn')}</button>
          </div>
        )}
        {view === 'uploading' && (
          <div className={styles.step1Footer}>
            <div className={styles.uploadSummary}>
              <div className={styles.uploadProgressBar}>
                <div className={styles.uploadProgressFill} style={{ width: `${browsed.length ? (finished / browsed.length) * 100 : 0}%` }} />
              </div>
              <div className={styles.uploadProgressLabel}>
                {t('uploadInfer.filePanel.complete', { finished, total: browsed.length })}
                {failed > 0 && <span className={styles.failCount}>{t('uploadInfer.filePanel.failed', { count: failed })}</span>}
              </div>
            </div>
          </div>
        )}
        </div>
      </div>

      {/* ══ SECTION 2: Uploaded files ══ */}
      <div className={styles.section2}>

        {/* Header */}
        <div className={styles.section2Header}>

          {/* ── Title row (always visible) ── */}
          <div className={styles.section2TitleRow} data-tour="upload-section2-title-row">
            {/* Select-all checkbox — inference mode */}
            {selectMode && !isBatchRunning && (
              <Checkbox
                checked={allSelected}
                indeterminate={someSelected}
                onChange={() => dispatch(toggleSelectAllServerFiles(serverFiles.map(f => f.id)))}
                disabled={filesTotal === 0}
              />
            )}

            {/* Select-all checkbox — export mode */}
            {exportMode && !isBatchRunning && (
              <Checkbox
                checked={allExportSelected}
                indeterminate={someExportSelected}
                onChange={toggleExportSelectAll}
                disabled={filesTotal === 0}
              />
            )}

            {/* Select-all checkbox — delete mode */}
            {deleteMode && !isBatchRunning && (
              <Checkbox
                checked={allDeleteSelected}
                indeterminate={someDeleteSelected}
                onChange={toggleDeleteSelectAll}
                disabled={filesTotal === 0}
              />
            )}

            <div className={styles.section2Title}>
              {t('uploadInfer.filePanel.uploadedFiles')}
              {!filesLoading && filesTotal > 0 && (
                <span className={styles.filesCount}>{filesTotal}</span>
              )}
              {selectMode && !isBatchRunning && selectedServerIds.length > 0 && (
                <span className={styles.selectedTotalHint}>
                  {t('uploadInfer.filePanel.selectedTotal', { count: selectedServerIds.length, total: filesTotal })}
                </span>
              )}
              {deleteMode && !isBatchRunning && deleteSelectedIds.length > 0 && (
                <span className={styles.selectedTotalHint}>
                  {t('uploadInfer.filePanel.selectedTotal', { count: deleteSelectedIds.length, total: filesTotal })}
                </span>
              )}
              {exportMode && !isBatchRunning && exportSelectedIds.length > 0 && (
                <span className={styles.selectedTotalHint}>
                  {t('uploadInfer.filePanel.selectedTotal', { count: exportSelectedIds.length, total: filesTotal })}
                </span>
              )}
            </div>

            <div className={styles.sortHeader} data-tour="upload-sort">
              <span className={styles.sortHeaderLabel}>
                <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                  <path d="M2 4h10M4 7h6M6 10h2" />
                </svg>
                {t('uploadInfer.filePanel.sortBy')}
              </span>
              <div className={styles.sortCols}>
                {([
                  { key: 'id' as SortKey, label: t('uploadInfer.filePanel.sortId') },
                  { key: 'name' as SortKey, label: t('uploadInfer.filePanel.sortName') },
                  { key: 'date' as SortKey, label: t('uploadInfer.filePanel.sortDate') },
                  { key: 'status' as SortKey, label: t('uploadInfer.filePanel.sortStatus') },
                ]).map(({ key, label }) => (
                  <button
                    key={key}
                    className={`${styles.sortCol} ${sort.key === key ? styles.sortColActive : ''}`}
                    onClick={() => handleSort(key)}
                  >
                    {label}
                    <svg viewBox="0 0 10 12" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round"
                      className={sort.key === key ? (sort.dir === 'asc' ? styles.sortAsc : styles.sortDesc) : styles.sortInactive}>
                      <path d="M5 1v10M2 8l3 3 3-3" />
                    </svg>
                  </button>
                ))}
              </div>
            </div>

            <span className={styles.vSep} aria-hidden="true" />

            {/* ── Date filter — moved here so it's always visible, not just in a filter row ── */}
            <div className={styles.dateFilter} data-tour="upload-date-filter">
              <svg className={styles.dateIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                <rect x="2" y="3" width="12" height="11" rx="1.5" />
                <path d="M2 6.5h12M5 2v2.5M11 2v2.5" />
              </svg>
              <span className={styles.filterBarLabel}>{t('uploadInfer.filePanel.dateRangeLabel', 'Date Range')}</span>
              <input type="date" className={styles.dateInput} value={dateFrom} max={dateTo}
                disabled={isBatchRunning}
                aria-label={t('uploadInfer.filePanel.dateFrom')}
                onChange={e => dispatch(setDateFrom(e.target.value))} />
              <span className={styles.dateSep}>–</span>
              <input type="date" className={styles.dateInput} value={dateTo} min={dateFrom}
                disabled={isBatchRunning}
                aria-label={t('uploadInfer.filePanel.dateTo')}
                onChange={e => dispatch(setDateTo(e.target.value))} />
              <button className={styles.applyBtn} onClick={handleApply} disabled={filesLoading || isBatchRunning}>{t('uploadInfer.filePanel.applyDate')}</button>
            </div>

            {/* Search stays in title row for select mode only; export/delete have their own row */}
            {selectMode && !isBatchRunning && (
              <div className={styles.headerActions}>
                <button
                  className={`${styles.searchIconBtn} ${searchOpen ? styles.searchIconBtnActive : ''}`}
                  onClick={openSearch}
                  title={t('uploadInfer.filePanel.searchBtn')}
                >
                  <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                    <circle cx="6.5" cy="6.5" r="4" />
                    <path d="M11 11l2.5 2.5" />
                  </svg>
                </button>
              </div>
            )}
          </div>

          {/* ── Export mode action row ── */}
          {exportMode && !isBatchRunning && (
            <div className={styles.modeActionsRow}>
              <div className={styles.modeActionsLeft}>
                <button
                  className={`${styles.modeTile} ${styles.modeTileSearch} ${searchOpen ? styles.modeTileSearchActive : ''}`}
                  onClick={openSearch}
                >
                  <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                    <circle cx="6.5" cy="6.5" r="4" />
                    <path d="M11 11l2.5 2.5" />
                  </svg>
                  {t('uploadInfer.filePanel.searchBtn')}
                </button>
                <button
                  className={styles.modeTileExportAll}
                  onClick={handleExportAll}
                  disabled={filesTotal === 0 || isExportingAll || isExporting}
                  title={t('uploadInfer.filePanel.exportAllBtn')}
                >
                  {isExportingAll ? (
                    <div className={styles.miniSpinnerGreen} />
                  ) : (
                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                      <path d="M9 2H4a1 1 0 00-1 1v10a1 1 0 001 1h8a1 1 0 001-1V6z" />
                      <path d="M9 2v4h4" />
                      <path d="M6 9.5l1.5 2 2.5-3" />
                    </svg>
                  )}
                  {t('uploadInfer.filePanel.exportAllBtn')}
                </button>
                {exportAllError && <span className={styles.modeActionsError}>{exportAllError}</span>}
              </div>
              <div className={styles.modeActionsRight}>
                <button
                  className={`${styles.modeTile} ${styles.modeTileConfirmExport} ${exportSelectedIds.length === 0 ? styles.modeTileDisabled : ''}`}
                  onClick={handleConfirmExport}
                  disabled={exportSelectedIds.length === 0 || isExporting}
                >
                  {isExporting ? (
                    <div className={styles.miniSpinnerGreen} />
                  ) : (
                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                      <path d="M9 2H4a1 1 0 00-1 1v10a1 1 0 001 1h8a1 1 0 001-1V6z" />
                      <path d="M9 2v4h4" />
                      <path d="M6 9.5l1.5 2 2.5-3" />
                    </svg>
                  )}
                  {exportSelectedIds.length > 0
                    ? t('uploadInfer.filePanel.exportCount', { count: exportSelectedIds.length })
                    : t('uploadInfer.filePanel.exportBtn')}
                </button>
                <button
                  className={`${styles.modeTile} ${styles.modeTileCancel}`}
                  onClick={exitExportMode}
                >
                  <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                    <path d="M4 4l8 8M12 4l-8 8" />
                  </svg>
                  {t('uploadInfer.filePanel.cancelBtn')}
                </button>
              </div>
            </div>
          )}

          {/* ── Select mode action row (choosing files for inference) ── */}
          {selectMode && !isBatchRunning && (
            <div className={styles.modeActionsRow}>
              <div className={styles.modeActionsLeft}>
                <button
                  className={`${styles.modeTile} ${styles.modeTileSearch} ${searchOpen ? styles.modeTileSearchActive : ''}`}
                  onClick={openSearch}
                >
                  <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                    <circle cx="6.5" cy="6.5" r="4" />
                    <path d="M11 11l2.5 2.5" />
                  </svg>
                  {t('uploadInfer.filePanel.searchBtn')}
                </button>
              </div>
              <div className={styles.modeActionsRight}>
                <button
                  className={`${styles.modeTile} ${styles.modeTileConfirmExport} ${selectedServerIds.length === 0 ? styles.modeTileDisabled : ''}`}
                  onClick={onGoToInfer}
                  disabled={selectedServerIds.length === 0 || !onGoToInfer}
                >
                  <svg viewBox="0 0 16 16" fill="currentColor" stroke="none" width="12" height="12">
                    <path d="M8 1l1.8 4.4L14 6.2l-3.3 2.5 1.2 4.3L8 10.8 4.1 13l1.2-4.3L2 6.2l4.2-.8z" />
                  </svg>
                  {selectedServerIds.length > 0
                    ? t('uploadInfer.filePanel.continueToInferCount', { count: selectedServerIds.length })
                    : t('uploadInfer.filePanel.continueToInfer')}
                </button>
                <button
                  className={`${styles.modeTile} ${styles.modeTileCancel}`}
                  onClick={exitSelectMode}
                >
                  <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                    <path d="M4 4l8 8M12 4l-8 8" />
                  </svg>
                  {t('uploadInfer.filePanel.cancelBtn')}
                </button>
              </div>
            </div>
          )}

          {/* ── Delete mode action row ── */}
          {deleteMode && !isBatchRunning && (
            <div className={styles.modeActionsRow}>
              <div className={styles.modeActionsLeft}>
                <button
                  className={`${styles.modeTile} ${styles.modeTileSearch} ${searchOpen ? styles.modeTileSearchActive : ''}`}
                  onClick={openSearch}
                >
                  <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                    <circle cx="6.5" cy="6.5" r="4" />
                    <path d="M11 11l2.5 2.5" />
                  </svg>
                  {t('uploadInfer.filePanel.searchBtn')}
                </button>
                <button
                  className={styles.modeTileDeleteAll}
                  onClick={openDeleteAllConfirm}
                  disabled={filesTotal === 0}
                  title={t('uploadInfer.filePanel.deleteAllBtn')}
                >
                  <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                    <path d="M8 1.5l6.5 11.5h-13z" />
                    <path d="M8 6v3.5M8 12v.1" />
                  </svg>
                  {t('uploadInfer.filePanel.deleteAllBtn')}
                </button>
              </div>
              <div className={styles.modeActionsRight}>
                <button
                  className={`${styles.modeTile} ${styles.modeTileConfirmDelete} ${deleteSelectedIds.length === 0 ? styles.modeTileDisabled : ''}`}
                  onClick={handleConfirmDelete}
                  disabled={deleteSelectedIds.length === 0 || isDeleting}
                >
                  {isDeleting ? (
                    <div className={styles.miniSpinner} />
                  ) : (
                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round" strokeLinejoin="round">
                      <path d="M3 4h10M6 4V3h4v1" />
                      <path d="M5 4l.5 8h5l.5-8" />
                      <path d="M7 7v3M9 7v3" />
                    </svg>
                  )}
                  {deleteSelectedIds.length > 0
                    ? t('uploadInfer.filePanel.deleteCount', { count: deleteSelectedIds.length })
                    : t('uploadInfer.filePanel.deleteBtn')}
                </button>
                <button
                  className={`${styles.modeTile} ${styles.modeTileCancel}`}
                  onClick={exitDeleteMode}
                >
                  <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                    <path d="M4 4l8 8M12 4l-8 8" />
                  </svg>
                  {t('uploadInfer.filePanel.cancelBtn')}
                </button>
              </div>
            </div>
          )}

        </div>

        <div className={styles.filterSortRow} data-tour="upload-filter-sort-row">
          <div className={styles.filterBarRow}>
            {/* ── Status filter + Actions ── */}
            <div className={styles.filterBarRight}>
              <div className={styles.statusFilterWrap} data-tour="upload-status-filter">
                <svg className={styles.dateIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                  <path d="M2 4h12M4.5 8h7M7 12h2" />
                </svg>
                <span className={styles.filterBarLabel}>{t('uploadInfer.filePanel.statusLabel', 'Status')}</span>
                <select
                  className={styles.statusFilterSelect}
                  value={statusFilter}
                  disabled={filesLoading || isBatchRunning}
                  onChange={e => handleStatusFilterChange(e.target.value as FileStatus | '')}
                >
                  {STATUS_FILTER_OPTIONS.map(opt => (
                    <option key={opt.value || 'all'} value={opt.value}>{t(opt.labelKey)}</option>
                  ))}
                </select>
              </div>

              {/* ── Actions — shown inline as a labeled button row, only in normal (idle) mode ── */}
              {!selectMode && !deleteMode && !exportMode && !isBatchRunning && (
                <>
                  <span className={styles.vSep} aria-hidden="true" />
                <div className={styles.actionsInlineWrap} data-tour="upload-actions">
                  <span className={styles.filterBarLabel}>{t('uploadInfer.filePanel.actionsLabel')}</span>

                  {/* Dictionary */}
                  <button
                    data-tour="upload-action-dictionary"
                    className={`${styles.actionTile} ${styles.actionTileDictionary}`}
                    onClick={() => setDictModalOpen(true)}
                    disabled={filesTotal === 0}
                    title={t('uploadInfer.filePanel.dictionaryBtn')}
                  >
                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
                      strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                      <path d="M3 2.5h7.5a1.5 1.5 0 011.5 1.5v9a.5.5 0 01-.5.5H4a1 1 0 01-1-1V2.5z" />
                      <path d="M3 11.5a1 1 0 011-1h8" />
                      <path d="M6 5.5h3" />
                    </svg>
                    <span className={styles.actionTileLabel}>{t('uploadInfer.filePanel.dictionaryBtn')}</span>
                  </button>

                  {/* Template */}
                  <button
                    data-tour="upload-action-template"
                    className={`${styles.actionTile} ${styles.actionTileTemplate}`}
                    onClick={() => setTemplateModalOpen(true)}
                    disabled={filesTotal === 0}
                    title={t('uploadInfer.filePanel.templateBtn')}
                  >
                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
                      strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                      <path d="M3 2.5h10v11H3z" />
                      <path d="M5.5 5.5h5M5.5 8h5M5.5 10.5h3" />
                    </svg>
                    <span className={styles.actionTileLabel}>{t('uploadInfer.filePanel.templateBtn')}</span>
                  </button>

                  {/* Search */}
                  <button
                    data-tour="upload-action-search"
                    className={`${styles.actionTile} ${styles.actionTileSearch} ${searchOpen ? styles.actionTileActive : ''}`}
                    onClick={openSearch}
                    title={t('uploadInfer.filePanel.searchBtn')}
                  >
                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                      <circle cx="6.5" cy="6.5" r="4" />
                      <path d="M11 11l2.5 2.5" />
                    </svg>
                    <span className={styles.actionTileLabel}>{t('uploadInfer.filePanel.searchBtn')}</span>
                  </button>

                  {/* Export */}
                  <button
                    data-tour="upload-action-export"
                    className={`${styles.actionTile} ${styles.actionTileExport}`}
                    onClick={enterExportMode}
                    disabled={filesTotal === 0}
                    title={t('uploadInfer.filePanel.exportBtn')}
                  >
                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
                      strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                      <path d="M9 2H4a1 1 0 00-1 1v10a1 1 0 001 1h8a1 1 0 001-1V6z" />
                      <path d="M9 2v4h4" />
                      <path d="M6 9.5l1.5 2 2.5-3" />
                    </svg>
                    <span className={styles.actionTileLabel}>{t('uploadInfer.filePanel.exportBtn')}</span>
                  </button>

                  {/* Delete */}
                  <button
                    data-tour="upload-action-delete"
                    className={`${styles.actionTile} ${styles.actionTileDelete}`}
                    onClick={enterDeleteMode}
                    disabled={filesTotal === 0}
                    title={t('uploadInfer.filePanel.deleteBtn')}
                  >
                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
                      strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                      <path d="M3 4h10M6 4V3h4v1" />
                      <path d="M5 4l.5 8h5l.5-8" />
                      <path d="M7 7v3M9 7v3" />
                    </svg>
                    <span className={styles.actionTileLabel}>{t('uploadInfer.filePanel.deleteBtn')}</span>
                  </button>
                </div>
                </>
              )}
            </div>
          </div>
        </div>

        {/* Search bar */}
        <div className={`${styles.searchBar} ${searchOpen ? styles.searchBarOpen : ''}`} data-tour="upload-search-bar">
          <div className={styles.searchBarRow}>
            <div className={styles.searchInner}>
              <svg className={styles.searchBarIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor"
                strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                <circle cx="6.5" cy="6.5" r="4" />
                <path d="M11 11l2.5 2.5" />
              </svg>
              <input
                ref={searchInputRef}
                type="text"
                className={styles.searchInput}
                placeholder={t('uploadInfer.filePanel.searchPlaceholder')}
                value={searchQuery}
                onChange={e => setSearchQuery(e.target.value)}
              />
              {filesSearch && (
                <span className={styles.searchCount}>
                  {t('uploadInfer.filePanel.matchCount', { count: filesTotal })}
                </span>
              )}
            </div>
            <button className={styles.searchCloseBtn} onClick={closeSearch} title={t('uploadInfer.filePanel.searchBtn')}>
              <svg viewBox="0 0 12 12" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round">
                <path d="M2 2l8 8M10 2l-8 8" />
              </svg>
            </button>
          </div>
        </div>

        {/* File list */}
        <div className={styles.uploadedBody} data-tour="upload-uploaded-body">
          {filesLoading && (
            <div className={styles.listState}><div className={styles.spinner} /><span>{t('uploadInfer.filePanel.loadingFiles')}</span></div>
          )}
          {!filesLoading && filesError && (
            <div className={`${styles.listState} ${styles.errorState}`}>
              <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                <circle cx="8" cy="8" r="6" /><path d="M8 5v3M8 11v.5" />
              </svg>
              {filesError}
            </div>
          )}
          {!filesLoading && !filesError && filesTotal === 0 && !filesSearch && (
            <div className={styles.listState}>
              <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                <rect x="2" y="2" width="12" height="12" rx="2" /><path d="M5 8h6M5 5.5h4M5 10.5h6" />
              </svg>
              {t('uploadInfer.filePanel.noFilesRange')}
            </div>
          )}
          {!filesLoading && !filesError && filesTotal === 0 && filesSearch && (
            <div className={styles.listState}>
              <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                <circle cx="6.5" cy="6.5" r="4" /><path d="M11 11l2.5 2.5" />
              </svg>
              {t('uploadInfer.filePanel.noFilesMatch', { query: filesSearch })}
            </div>
          )}
          {!filesLoading && !filesError && displayedFiles.map((f, idx) => {
            const ext = getExt(f.original_name);
            const isChecked = selectedServerIds.includes(f.id);
            const isDeleteChecked = deleteSelectedIds.includes(f.id);
            const isExportChecked = exportSelectedIds.includes(f.id);
            const selectable = selectMode && !isBatchRunning;
            const deletable = deleteMode && !isBatchRunning;
            const exportable = exportMode && !isBatchRunning;
            const checked = selectable ? isChecked : deletable ? isDeleteChecked : exportable ? isExportChecked : false;
            const toggle = () => {
              if (selectable) dispatch(toggleServerFileSelection(f.id));
              else if (deletable) toggleDeleteSelection(f.id);
              else if (exportable) toggleExportSelection(f.id);
              // No mode active — cards are no longer clickable to view results;
              // that navigation now only happens from the Workspace tab's own list.
            };
            const statusMeta = STATUS_META[f.status] ?? STATUS_META.tbd;
            const isFirstCard = idx === 0;
            const hasLinks = !!f.dictionary_id || !!f.prompt_template_id;
            return (
              <div
                key={f.id}
                data-tour={idx === 0 ? 'upload-cards' : undefined}
                className={`${styles.fcard} ${styles[`fcardBar_${statusMeta.cls}`] ?? ''}
                  ${selectable ? (isChecked ? styles.fcardActive : '') : ''}
                  ${deletable ? (isDeleteChecked ? styles.fcardActiveDelete : '') : ''}
                  ${exportable ? (isExportChecked ? styles.fcardActiveExport : '') : ''}
                  ${!selectable && !deletable && !exportable ? styles.fcardStatic : ''}
                `}
                onClick={toggle}
                title={f.original_name}
              >
                <div className={styles.fcardSweep} aria-hidden="true" />

                {(selectable || deletable || exportable) && (
                  <div className={styles.fcardCheck}>
                    <Checkbox checked={checked} onChange={toggle} />
                  </div>
                )}

                <div className={`${styles.fcardIcon} ${styles[ext]}`}>
                  {ext.toUpperCase()}
                </div>

                <div className={styles.fcardInfo}>
                  <div className={styles.fcardNameRow}>
                    <div className={styles.fcardName}>{f.original_name}</div>
                    {hasLinks && (
                      <div className={styles.fcardLinks}>
                        {f.dictionary_id && (
                          <span className={`${styles.fcardLinkIcon} ${styles.fcardLinkIconDict}`} title={t('uploadInfer.filePanel.dictionaryLinked')}>
                            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                              <path d="M8 3.7c-1.2-.9-2.9-1.2-4.7-.9v9c1.8-.3 3.5-.1 4.7.9 1.2-.9 2.9-1.2 4.7-.9v-9c-1.8-.3-3.5-.1-4.7.9z" />
                              <path d="M8 3.7v9" />
                            </svg>
                          </span>
                        )}
                        {f.prompt_template_id && (
                          <span className={`${styles.fcardLinkIcon} ${styles.fcardLinkIconTemplate}`} title={t('uploadInfer.filePanel.templateLinked')}>
                            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                              <path d="M5.5 2.5h5v2h-5z" />
                              <path d="M4.5 4h7a1 1 0 011 1v8a1 1 0 01-1 1h-7a1 1 0 01-1-1V5a1 1 0 011-1z" />
                              <path d="M6 8.3l1.3 1.3L10.2 7" />
                            </svg>
                          </span>
                        )}
                      </div>
                    )}
                  </div>

                  <div className={styles.fcardMeta}>
                    <span>{f.inserted_at} · #{f.id}</span>
                    <span
                      className={`${styles.badge} ${styles.fcardBadge} ${isChecked ? styles.bSelected :
                        isDeleteChecked ? styles.bDelete :
                          isExportChecked ? styles.bExport :
                            styles[statusMeta.cls]
                        }`}
                      title={!isChecked && !isDeleteChecked && !isExportChecked ? t(statusMeta.labelKey) : undefined}
                    >
                      {isChecked ? (
                        <svg viewBox="0 0 12 12" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" width="10" height="10">
                          <path d="M2 6l3 3 5-5" />
                        </svg>
                      ) : isDeleteChecked ? (
                        <svg viewBox="0 0 12 12" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" width="10" height="10">
                          <path d="M2 2l8 8M10 2l-8 8" />
                        </svg>
                      ) : isExportChecked ? (
                        <>
                          <svg viewBox="0 0 12 12" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" width="9" height="9">
                            <path d="M6 1v6M3.5 4.5L6 7l2.5-2.5" /><path d="M2 10h8" />
                          </svg>
                          {t('uploadInfer.filePanel.export')}
                        </>
                      ) : (
                        <>
                          {f.status === 'completed' && (
                            <svg viewBox="0 0 12 12" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" width="9" height="9">
                              <path d="M2 6l3 3 5-5" />
                            </svg>
                          )}
                          {(f.status === 'running' || f.status === 'queued') && (
                            <span className={styles.fcardBadgeDot} />
                          )}
                          {t(statusMeta.labelKey)}
                        </>
                      )}
                    </span>
                  </div>

                  <div className={styles.fcardPrompts} onClick={e => e.stopPropagation()}>
                    {PROMPT_FIELDS.map(({ key, labelKey, icon, tourId }, fieldIdx) => {
                      const set = !!(f[key] as string)?.trim();
                      return (
                        <button
                          key={key}
                          type="button"
                          data-tour={isFirstCard ? tourId : undefined}
                          className={`${styles.fcardPromptDot} ${set ? styles.fcardPromptDotSet : ''}`}
                          title={`${t(labelKey)}${set ? ' — ' + t('uploadInfer.filePanel.promptCustomized') : ' — ' + t('uploadInfer.filePanel.promptDefault')}`}
                          onClick={() => setPromptViewer({ file: f, fieldIdx })}
                        >
                          <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                            <path d={icon} />
                          </svg>
                        </button>
                      );
                    })}
                  </div>
                </div>
              </div>
            );
          })}
        </div>

        {/* Pagination footer */}
        {!filesLoading && !filesError && filesTotal > 0 && (
          <div className={styles.paginationBar}>
            <div className={styles.paginationTopRow}>
              <div className={styles.pageSizeGroup}>
                <span className={styles.pageSizeLabel}>{t('uploadInfer.filePanel.perPage')}</span>
                <select
                  className={styles.pageSizeSelect}
                  value={filesPageSize}
                  onChange={e => handlePageSizeChange(Number(e.target.value))}
                >
                  {[50, 75, 100].map(size => (
                    <option key={size} value={size}>{size}</option>
                  ))}
                </select>
              </div>

              <div className={styles.pageNav}>
                <button
                  className={styles.pageNavBtn}
                  onClick={() => goToPage(filesPage - 1)}
                  disabled={filesPage <= 1}
                  aria-label="Previous page"
                >
                  <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                    <path d="M10 3L6 8l4 5" />
                  </svg>
                </button>

                {(() => {
                  const pageWindow = getPageNumbers(filesPage, filesTotalPages);
                  const showFirst = pageWindow[0] > 1;
                  const showLast = pageWindow[pageWindow.length - 1] < filesTotalPages;
                  return (
                    <>
                      {showFirst && (
                        <>
                          <button className={styles.pageNumBtn} onClick={() => goToPage(1)}>1</button>
                          {pageWindow[0] > 2 && <span className={styles.pageEllipsis}>…</span>}
                        </>
                      )}
                      {pageWindow.map(p => (
                        <button
                          key={p}
                          className={`${styles.pageNumBtn} ${p === filesPage ? styles.pageNumBtnActive : ''}`}
                          onClick={() => goToPage(p)}
                        >
                          {p}
                        </button>
                      ))}
                      {showLast && (
                        <>
                          {pageWindow[pageWindow.length - 1] < filesTotalPages - 1 && <span className={styles.pageEllipsis}>…</span>}
                          <button className={styles.pageNumBtn} onClick={() => goToPage(filesTotalPages)}>{filesTotalPages}</button>
                        </>
                      )}
                    </>
                  );
                })()}

                <button
                  className={styles.pageNavBtn}
                  onClick={() => goToPage(filesPage + 1)}
                  disabled={filesPage >= filesTotalPages}
                  aria-label="Next page"
                >
                  <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                    <path d="M6 3l4 5-4 5" />
                  </svg>
                </button>
              </div>
            </div>

            <div className={styles.pageInfo}>
              {t('uploadInfer.filePanel.pageInfo', { page: filesPage, totalPages: filesTotalPages, total: filesTotal })}
            </div>
          </div>
        )}
      </div>

      <DictionaryAssociationModal
        open={dictModalOpen}
        onClose={() => setDictModalOpen(false)}
      />

      <PromptTemplateAssociationModal
        open={templateModalOpen}
        onClose={() => setTemplateModalOpen(false)}
      />

      {/* ── Per-file prompt viewer — opened from a card's prompt icons ── */}
      {promptViewer && (
        <div className={styles.dangerOverlay} onClick={() => setPromptViewer(null)}>
          <div className={styles.promptViewerModal} onClick={e => e.stopPropagation()}>
            <div className={styles.promptViewerHead}>
              <div>
                <div className={styles.promptViewerTitle}>{t(PROMPT_FIELDS[promptViewer.fieldIdx].labelKey)}</div>
                <div className={styles.promptViewerFile}>{promptViewer.file.original_name}</div>
              </div>
              <button className={styles.promptViewerClose} onClick={() => setPromptViewer(null)} aria-label="Close">
                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                  <path d="M4 4l8 8M12 4l-8 8" />
                </svg>
              </button>
            </div>
            <div className={styles.promptViewerBody}>
              {(promptViewer.file[PROMPT_FIELDS[promptViewer.fieldIdx].key] as string)?.trim()
                ? (promptViewer.file[PROMPT_FIELDS[promptViewer.fieldIdx].key] as string)
                : <span className={styles.promptViewerEmpty}>{t('uploadInfer.filePanel.promptDefaultFull')}</span>}
            </div>
          </div>
        </div>
      )}

      {/* ── Delete ALL confirmation — wipes every file on the account ── */}
      {deleteAllConfirmOpen && (
        <div className={styles.dangerOverlay} onClick={() => !isDeletingAll && closeDeleteAllConfirm()}>
          <div className={styles.dangerModal} onClick={e => e.stopPropagation()}>
            <div className={styles.dangerIcon}>
              <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round" strokeLinejoin="round">
                <path d="M12 2l10 18H2z" />
                <path d="M12 9v5M12 17v.1" />
              </svg>
            </div>
            <div className={styles.dangerTitle}>{t('uploadInfer.filePanel.deleteAllConfirmTitle')}</div>
            <div className={styles.dangerBody}>
              {t('uploadInfer.filePanel.deleteAllConfirmBody', { from: dateFrom, to: dateTo })}
            </div>
            {deleteAllError && <div className={styles.dangerError}>{deleteAllError}</div>}
            <div className={styles.dangerActions}>
              <button
                className={`${styles.btn} ${styles.btnFull}`}
                onClick={closeDeleteAllConfirm}
                disabled={isDeletingAll}
              >
                {t('uploadInfer.filePanel.cancelBtn')}
              </button>
              <button
                type="button"
                className={`${styles.btn} ${styles.btnDanger} ${styles.btnFull}`}
                onClick={handleConfirmDeleteAll}
                disabled={isDeletingAll}
                aria-disabled={isDeletingAll}
              >
                {isDeletingAll ? <div className={styles.miniSpinner} /> : t('uploadInfer.filePanel.deleteAllConfirmBtn')}
              </button>
            </div>
          </div>
        </div>
      )}
    </div>
  );
});

FilePanel.displayName = 'FilePanel';

export default FilePanel;














// ═══════════════════════════════════════════════
// pages/UploadInfer/FileSidebar.tsx
// Content Analytics · Shared 400px file sidebar
// Used by both the Infer tab (mode="select") and the Workspace tab
// (mode="view"). Deliberately does NOT read the Upload tab's shared
// `serverFiles` — it keeps its own local list, date filter, sort,
// search, and pagination, and fetches independently every time its
// tab becomes active. The only thing still shared with the rest of
// the app is `selectedServerIds`, since InferencePanel needs that to
// run a batch.
// ═══════════════════════════════════════════════
import React, { useCallback, useEffect, useRef, useState } from 'react';
import { useTranslation } from 'react-i18next';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import { toggleServerFileSelection, toggleSelectAllServerFiles, selectFilesSuccess, ServerFile, FilesSortBy, SortOrder, FileStatus } from '../../store/uploadSlice';
import api from '../../services/api';
import styles from './FileSidebar.module.scss';

const getExt = (n: string): 'vtt' | 'srt' => n.toLowerCase().endsWith('.srt') ? 'srt' : 'vtt';

function toDateStr(d: Date) { return d.toISOString().slice(0, 10); }
function defaultFrom() { const d = new Date(); d.setMonth(d.getMonth() - 1); return toDateStr(d); }
function defaultTo() { return toDateStr(new Date()); }

const PAGE_WINDOW = 4;
const getPageNumbers = (current: number, total: number): number[] => {
  if (total <= PAGE_WINDOW) return Array.from({ length: total }, (_, i) => i + 1);
  const start = Math.max(1, Math.min(current - 1, total - PAGE_WINDOW + 1));
  return Array.from({ length: PAGE_WINDOW }, (_, i) => start + i);
};

type SortKey = 'id' | 'name' | 'date' | 'status';
const SORT_KEY_TO_API: Record<SortKey, FilesSortBy> = {
  id: 'id', name: 'original_name', date: 'inserted_at', status: 'status',
};
const API_TO_SORT_KEY: Record<FilesSortBy, SortKey> = {
  id: 'id', original_name: 'name', inserted_at: 'date', status: 'status',
};

// ── Status → badge label/color — same 5-state mapping as the Upload tab ──
const STATUS_META: Record<FileStatus, { labelKey: string; cls: string }> = {
  waiting: { labelKey: 'uploadInfer.filePanel.statusWaiting', cls: 'bWaiting' },
  running: { labelKey: 'uploadInfer.filePanel.statusRunning', cls: 'bRunning' },
  completed: { labelKey: 'uploadInfer.filePanel.inferenced', cls: 'bInferred' },
  error: { labelKey: 'uploadInfer.filePanel.notInferenced', cls: 'bNotInferred' },
  tbd: { labelKey: 'uploadInfer.filePanel.notInferenced', cls: 'bNotInferred' },
  queued: { labelKey: 'uploadInfer.filePanel.statusQueued', cls: 'bQueued' },
};

const STATUS_FILTER_OPTIONS: { value: FileStatus | ''; labelKey: string }[] = [
  { value: '', labelKey: 'uploadInfer.filePanel.statusAll' },
  { value: 'waiting', labelKey: 'uploadInfer.filePanel.statusWaiting' },
  { value: 'queued', labelKey: 'uploadInfer.filePanel.statusQueued' },
  { value: 'running', labelKey: 'uploadInfer.filePanel.statusRunning' },
  { value: 'completed', labelKey: 'uploadInfer.filePanel.inferenced' },
  { value: 'error', labelKey: 'uploadInfer.filePanel.statusError' },
  { value: 'tbd', labelKey: 'uploadInfer.filePanel.statusTbd' },
];

interface CbProps { checked: boolean; indeterminate?: boolean; onChange: () => void; disabled?: boolean; }
const Checkbox: React.FC<CbProps> = ({ checked, indeterminate, onChange, disabled }) => (
  <div
    className={`${styles.cb} ${checked ? styles.cbChecked : ''} ${indeterminate ? styles.cbIndet : ''} ${disabled ? styles.cbDisabled : ''}`}
    onClick={e => { e.stopPropagation(); if (!disabled) onChange(); }}
    role="checkbox" aria-checked={indeterminate ? 'mixed' : checked} tabIndex={0}
    onKeyDown={e => { if (e.key === ' ' || e.key === 'Enter') { e.preventDefault(); if (!disabled) onChange(); } }}
  />
);

interface Props {
  // "select" — checkbox multi-select for building the inference batch (Infer tab)
  // "view"   — click a file to load its results (Workspace tab)
  mode: 'select' | 'view';
  // Whether this sidebar's tab is the one currently showing. A fresh
  // fetch fires each time this flips true — i.e. every time the user
  // navigates to the Infer or Result tab, not just on first mount.
  active: boolean;
  activeFileId?: number | null;
  onFileClick?: (fileId: number, fileName: string) => void;
}

const PAGE_SIZES = [25, 50, 75, 100];

const FileSidebar: React.FC<Props> = ({ mode, active, activeFileId = null, onFileClick }) => {
  const { t } = useTranslation();
  const dispatch = useAppDispatch();
  const selectedServerIds = useAppSelector(s => s.upload.selectedServerIds);
  const isBatchRunning = useAppSelector(s => s.upload.isBatchRunning);
  const lastFileCompletedAt = useAppSelector(s => s.upload.lastFileCompletedAt);

  // ── Independent local file list ──
  const [files, setFiles] = useState<ServerFile[]>([]);
  const [total, setTotal] = useState(0);
  const [totalPages, setTotalPages] = useState(1);
  const [page, setPage] = useState(1);
  const [pageSize, setPageSize] = useState(50);
  const [sortBy, setSortBy] = useState<FilesSortBy>('id');
  const [sortOrder, setSortOrder] = useState<SortOrder>('desc');
  const [search, setSearch] = useState('');
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const [dateFrom, setDateFrom] = useState(defaultFrom());
  const [dateTo, setDateTo] = useState(defaultTo());
  const [applying, setApplying] = useState(false);
  const [statusFilter, setStatusFilter] = useState<FileStatus | ''>('');

  const fetchFiles = useCallback(async (opts?: {
    page?: number; pageSize?: number; from?: string; to?: string;
    sortBy?: FilesSortBy; sortOrder?: SortOrder; search?: string;
    statusFilter?: FileStatus | '';
  }) => {
    const p = opts?.page ?? page;
    const ps = opts?.pageSize ?? pageSize;
    const from = opts?.from ?? dateFrom;
    const to = opts?.to ?? dateTo;
    const sBy = opts?.sortBy ?? sortBy;
    const sOrd = opts?.sortOrder ?? sortOrder;
    const q = opts?.search ?? search;
    const statusF = opts?.statusFilter ?? statusFilter;
    setLoading(true);
    setError(null);
    try {
      const res = await api.post('/files/by-date/', {
        start_date: from,
        end_date: to,
        page: p,
        page_size: ps,
        sort_by: sBy,
        sort_order: sOrd,
        ...(q ? { search: q } : {}),
        ...(statusF ? { status_filter: statusF } : {}),
      });
      const d = (res.data as any)?.data ?? {};
      setFiles(d.data ?? []);
      setTotal(d.total ?? 0);
      setPage(d.page ?? p);
      setPageSize(d.page_size ?? ps);
      setTotalPages(d.total_pages ?? 1);
      setSortBy(sBy);
      setSortOrder(sOrd);
      setStatusFilter(statusF);
      // select mode = Infer tab: this is the only live source of full file
      // objects (prompt defaults included) while that tab is active, since
      // the Upload tab's shared `serverFiles` gets cleared as soon as it's
      // not the active tab. view mode (Workspace tab) doesn't publish —
      // InferencePanel only needs this for its own selection.
      if (mode === 'select') dispatch(selectFilesSuccess(d.data ?? []));
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to load files.');
    } finally {
      setLoading(false);
    }
  }, [page, pageSize, dateFrom, dateTo, sortBy, sortOrder, search, statusFilter, mode, dispatch]);

  const handleApply = useCallback(async () => {
    setApplying(true);
    await fetchFiles({ page: 1 });
    setApplying(false);
  }, [fetchFiles]);

  const handleStatusFilterChange = (value: FileStatus | '') => {
    fetchFiles({ page: 1, statusFilter: value });
  };

  const goToPage = (p: number) => {
    if (p < 1 || p > totalPages || p === page || loading) return;
    fetchFiles({ page: p });
  };

  const handlePageSizeChange = (size: number) => {
    fetchFiles({ page: 1, pageSize: size });
  };

  const handleSort = useCallback((key: SortKey) => {
    const apiKey = SORT_KEY_TO_API[key];
    const dir: SortOrder = sortBy === apiKey ? (sortOrder === 'asc' ? 'desc' : 'asc') : 'asc';
    fetchFiles({ sortBy: apiKey, sortOrder: dir, page: 1 });
  }, [sortBy, sortOrder, fetchFiles]);
  const sort = { key: API_TO_SORT_KEY[sortBy], dir: sortOrder };

  // ── Search — server-side, debounced (matches the Upload tab's search) ──
  const [searchQuery, setSearchQuery] = useState('');
  const skipNextSearchEffect = useRef(true);
  useEffect(() => {
    if (skipNextSearchEffect.current) { skipNextSearchEffect.current = false; return; }
    const q = searchQuery.trim();
    const handle = setTimeout(() => {
      setSearch(q);
      fetchFiles({ search: q, page: 1 });
    }, 400);
    return () => clearTimeout(handle);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [searchQuery]);

  // Own API call, fired every time this tab becomes active — entirely
  // separate from whatever the Upload tab is doing. When the tab goes
  // inactive (navigated away from), clear the list and reset search/
  // sort/filter/page back to defaults so a return visit shows a clean
  // loading state and a genuinely fresh fetch, instead of the previous
  // tab's stale results flashing before the new data arrives.
  useEffect(() => {
    if (active) {
      fetchFiles({ page: 1 });
    } else {
      skipNextSearchEffect.current = true; // don't let the reset below trigger a second debounced fetch
      setFiles([]);
      if (mode === 'select') dispatch(selectFilesSuccess([]));
      setTotal(0);
      setTotalPages(1);
      setPage(1);
      setSortBy('id');
      setSortOrder('desc');
      setSearch('');
      setSearchQuery('');
      setStatusFilter('');
      setError(null);
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [active]);

  // Re-fetch /files/by-date/ whenever an individual file finishes inferring
  // (InferencePanel stamps this the moment a file lands in the completed
  // bucket), so this sidebar's statuses stay current without waiting for
  // the whole batch to drain. Only relevant while the tab is actually
  // active — skip the initial mount (timestamp starts as null).
  useEffect(() => {
    if (active && lastFileCompletedAt !== null) {
      fetchFiles();
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [lastFileCompletedAt]);

  const pageSelectedCount = files.filter(f => selectedServerIds.includes(f.id)).length;
  const allSelected = files.length > 0 && pageSelectedCount === files.length;
  const someSelected = pageSelectedCount > 0 && !allSelected;

  const handleRowClick = (fileId: number, fileName: string) => {
    if (mode === 'select') {
      if (isBatchRunning) return;
      dispatch(toggleServerFileSelection(fileId));
    } else {
      onFileClick?.(fileId, fileName);
    }
  };

  return (
    <div className={styles.sidebar} data-tour={mode === 'select' ? 'infer-sidebar' : 'results-sidebar'}>
      <div className={styles.dateFilter} data-tour={mode === 'select' ? 'infer-date-filter' : 'results-date-filter'}>
        <div className={styles.dateField}>
          <label className={styles.dateLabel}>{t('uploadInfer.filePanel.dateFrom')}</label>
          <input
            type="date" className={styles.dateInput} value={dateFrom} max={dateTo}
            disabled={applying}
            onChange={e => setDateFrom(e.target.value)}
          />
        </div>
        <div className={styles.dateSep}>—</div>
        <div className={styles.dateField}>
          <label className={styles.dateLabel}>{t('uploadInfer.filePanel.dateTo')}</label>
          <input
            type="date" className={styles.dateInput} value={dateTo} min={dateFrom}
            disabled={applying}
            onChange={e => setDateTo(e.target.value)}
          />
        </div>
        <button className={styles.applyBtn} onClick={handleApply} disabled={applying || loading}>
          {applying ? <span className={styles.miniSpinner} /> : t('uploadInfer.filePanel.applyDate')}
        </button>
      </div>

      <div className={styles.filterSearchRow}>
        <div className={styles.statusFilterWrap} data-tour={mode === 'select' ? 'infer-status-filter' : 'results-status-filter'}>
          <svg className={styles.statusFilterIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
            <path d="M2 4h12M4.5 8h7M7 12h2" />
          </svg>
          <select
            className={styles.statusFilterSelect}
            value={statusFilter}
            disabled={loading}
            onChange={e => handleStatusFilterChange(e.target.value as FileStatus | '')}
          >
            {STATUS_FILTER_OPTIONS.map(opt => (
              <option key={opt.value || 'all'} value={opt.value}>{t(opt.labelKey)}</option>
            ))}
          </select>
        </div>

        <div className={styles.searchWrap} data-tour={mode === 'select' ? 'infer-search' : 'results-search'}>
          <svg className={styles.searchIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
            <circle cx="6.5" cy="6.5" r="4" /><path d="M11 11l2.5 2.5" />
          </svg>
          <input
            type="text"
            className={styles.searchInput}
            placeholder={t('uploadInfer.workspace.searchFiles')}
            value={searchQuery}
            onChange={e => setSearchQuery(e.target.value)}
          />
        </div>
      </div>

      <div className={styles.head}>
        {mode === 'select' && (
          <Checkbox
            checked={allSelected}
            indeterminate={someSelected}
            onChange={() => dispatch(toggleSelectAllServerFiles(files.map(f => f.id)))}
            disabled={files.length === 0 || isBatchRunning}
          />
        )}
        <span className={styles.headTitle}>{t('uploadInfer.workspace.filesTitle')}</span>
        {total > 0 && <span className={styles.headCount}>{total}</span>}
        {loading && <span className={styles.headSpinner} aria-label="Loading" />}
        {mode === 'select' && selectedServerIds.length > 0 && (
          <span className={styles.headSelected}>
            {t('uploadInfer.filePanel.selectedTotal', { count: selectedServerIds.length, total })}
          </span>
        )}
      </div>

      {/* Sort header — same sort keys as the Upload tab */}
      <div className={styles.sortHeader} data-tour={mode === 'select' ? 'infer-sort' : 'results-sort'}>
        <span className={styles.sortHeaderLabel}>
          <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
            <path d="M2 4h10M4 7h6M6 10h2" />
          </svg>
          {t('uploadInfer.filePanel.sortBy')}
        </span>
        <div className={styles.sortCols}>
          {([
            { key: 'id' as SortKey, label: t('uploadInfer.filePanel.sortId') },
            { key: 'name' as SortKey, label: t('uploadInfer.filePanel.sortName') },
            { key: 'date' as SortKey, label: t('uploadInfer.filePanel.sortDate') },
            { key: 'status' as SortKey, label: t('uploadInfer.filePanel.sortStatus') },
          ]).map(({ key, label }) => (
            <button
              key={key}
              className={`${styles.sortCol} ${sort.key === key ? styles.sortColActive : ''}`}
              onClick={() => handleSort(key)}
              disabled={loading}
            >
              {label}
              <svg viewBox="0 0 10 12" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round"
                className={sort.key === key ? (sort.dir === 'asc' ? styles.sortAsc : styles.sortDesc) : styles.sortInactive}>
                <path d="M5 1v10M2 8l3 3 3-3" />
              </svg>
            </button>
          ))}
        </div>
      </div>

      <div className={styles.rowsWrap} data-tour={mode === 'select' ? 'infer-file-list' : 'results-file-list'}>
        <div className={styles.rows}>
        {error && (
          <div className={styles.empty}>{error}</div>
        )}
        {!error && loading && files.length === 0 && (
          <div className={styles.emptyLoading}>
            <span className={styles.rowsSpinner} />
            {t('uploadInfer.workspace.loadingFiles')}
          </div>
        )}
        {!error && !loading && files.length === 0 && (
          <div className={styles.empty}>
            {search ? t('uploadInfer.workspace.noFilesMatch') : t('uploadInfer.workspace.noFilesYet')}
          </div>
        )}
        {files.map(f => {
          const ext = getExt(f.original_name);
          const isChecked = mode === 'select' && selectedServerIds.includes(f.id);
          const isActiveView = mode === 'view' && f.id === activeFileId;
          const disabled = mode === 'select' && isBatchRunning;
          const statusMeta = STATUS_META[f.status] ?? STATUS_META.tbd;
          return (
            <div
              key={f.id}
              role="button"
              tabIndex={0}
              className={`${styles.hitm} ${styles.hitmSelectable}
                ${isChecked ? styles.active : ''}
                ${isActiveView ? styles.activeView : ''}
                ${disabled ? styles.rowDisabled : ''}`}
              onClick={() => !disabled && handleRowClick(f.id, f.original_name)}
              onKeyDown={e => { if (!disabled && (e.key === 'Enter' || e.key === ' ')) { e.preventDefault(); handleRowClick(f.id, f.original_name); } }}
            >
              {mode === 'select' && (
                <Checkbox checked={isChecked} onChange={() => handleRowClick(f.id, f.original_name)} disabled={isBatchRunning} />
              )}
              <div className={`${styles.ficon} ${styles[ext]}`}>{ext.toUpperCase()}</div>
              <div className={styles.hi}>
                <div className={styles.hn}>{f.original_name}</div>
                <div className={styles.hm}>{f.inserted_at} · #{f.id}</div>
              </div>
              <span className={`${styles.badge} ${isChecked ? styles.bSelected : styles[statusMeta.cls]}`}>
                {isChecked ? (
                  <svg viewBox="0 0 12 12" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" width="10" height="10">
                    <path d="M2 6l3 3 5-5" />
                  </svg>
                ) : (
                  <>
                    {f.status === 'completed' && (
                      <svg viewBox="0 0 12 12" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" width="9" height="9">
                        <path d="M2 6l3 3 5-5" />
                      </svg>
                    )}
                    {(f.status === 'running' || f.status === 'queued') && (
                      <span className={styles.badgeDot} />
                    )}
                    {t(statusMeta.labelKey)}
                  </>
                )}
              </span>
            </div>
          );
        })}
        </div>

        {/* Blocks interaction with the current list while a fetch is in
            flight (pagination, sort, search, filter) — only shown when
            there's already content on screen; the empty-state spinner
            above covers the first load. */}
        {loading && files.length > 0 && (
          <div className={styles.rowsLoadingOverlay}>
            <span className={styles.rowsSpinner} />
          </div>
        )}
      </div>

      {/* Pagination footer — same page-number layout as the Upload tab */}
      {!loading && !error && total > 0 && (
        <div className={styles.paginationBar}>
          <div className={styles.paginationTopRow}>
            <div className={styles.pageSizeGroup}>
              <span className={styles.pageSizeLabel}>{t('uploadInfer.filePanel.perPage')}</span>
              <select
                className={styles.pageSizeSelect}
                value={pageSize}
                onChange={e => handlePageSizeChange(Number(e.target.value))}
              >
                {PAGE_SIZES.map(size => (
                  <option key={size} value={size}>{size}</option>
                ))}
              </select>
            </div>

            <div className={styles.pageNav}>
              <button
                className={styles.pageNavBtn}
                onClick={() => goToPage(page - 1)}
                disabled={page <= 1}
                aria-label="Previous page"
              >
                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                  <path d="M10 3L6 8l4 5" />
                </svg>
              </button>

              {(() => {
                const pageWindow = getPageNumbers(page, totalPages);
                const showFirst = pageWindow[0] > 1;
                const showLast = pageWindow[pageWindow.length - 1] < totalPages;
                return (
                  <>
                    {showFirst && (
                      <>
                        <button className={styles.pageNumBtn} onClick={() => goToPage(1)}>1</button>
                        {pageWindow[0] > 2 && <span className={styles.pageEllipsis}>…</span>}
                      </>
                    )}
                    {pageWindow.map(p => (
                      <button
                        key={p}
                        className={`${styles.pageNumBtn} ${p === page ? styles.pageNumBtnActive : ''}`}
                        onClick={() => goToPage(p)}
                      >
                        {p}
                      </button>
                    ))}
                    {showLast && (
                      <>
                        {pageWindow[pageWindow.length - 1] < totalPages - 1 && <span className={styles.pageEllipsis}>…</span>}
                        <button className={styles.pageNumBtn} onClick={() => goToPage(totalPages)}>{totalPages}</button>
                      </>
                    )}
                  </>
                );
              })()}

              <button
                className={styles.pageNavBtn}
                onClick={() => goToPage(page + 1)}
                disabled={page >= totalPages}
                aria-label="Next page"
              >
                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                  <path d="M6 3l4 5-4 5" />
                </svg>
              </button>
            </div>
          </div>

          <div className={styles.pageInfo}>
            {t('uploadInfer.filePanel.pageInfo', { page, totalPages, total })}
          </div>
        </div>
      )}
    </div>
  );
};

export default FileSidebar;


















// ═══════════════════════════════════════════════
// pages/UploadInfer/InferencePanel.tsx
// Content Analytics · Inference configuration + batch status
// ═══════════════════════════════════════════════
import React, { useEffect, useRef, useCallback, useState } from 'react';
import { useTranslation } from 'react-i18next';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import {
  inferenceStatusSuccess, updateRunningProgress,
  updateSummaryPrompt, updateKeywordPrompt, updateQuestionPrompt,
  updateShortAnswerPrompt, updateTrueFalsePrompt, updateSettings,
  modelsLoading, modelsSuccess, modelsFailure, setSelectedModel,
  updateFilePrompts,
  type ServerFile, type ServerFilesData, type TimeInterval,
} from '../../store/uploadSlice';
import api from '../../services/api';
import styles from './InferencePanel.module.scss';
import { addToast } from '../../store/toastSlice';

// ── Batch status columns ──────────────────────────
const StatusCard: React.FC<{ file: ServerFile; variant: 'queued' | 'running' | 'completed'; onStop?: (id: number) => void; stopping?: boolean }> = ({ file, variant, onStop, stopping }) => {
  const { t } = useTranslation();
  const ext = file.original_name.toLowerCase().endsWith('.srt') ? 'srt' : 'vtt';
  return (
    <div className={`${styles.statusCardWrap} ${styles[variant + 'Wrap']}`}>
      <div className={`${styles.statusCard} ${styles[variant]}`}>

        {/* Left icon — ext badge for queued/running, green check for completed */}
        {variant === 'completed' ? (
          <div className={styles.completedCheck}>
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
              <path d="M3 8l3.5 3.5L13 5" />
            </svg>
          </div>
        ) : (
          <div className={`${styles.statusExt} ${styles[ext]}`}>{ext.toUpperCase()}</div>
        )}

        <div className={styles.statusInfo}>
          <div className={styles.statusNameRow}>
            <span className={`${styles.statusIdBadge} ${styles['statusIdBadge_' + variant]}`}>#{file.id}</span>
            <div className={styles.statusName}>{file.original_name}</div>
          </div>

          {variant === 'running' && typeof file.progress === 'number' && (
            <div className={styles.statusProgress}>
              <div className={styles.statusBar}>
                <div className={styles.statusFill} style={{ width: `${file.progress}%` }} />
              </div>
              <span className={styles.statusPct}>{file.progress}%</span>
            </div>
          )}

          {variant === 'queued' && (
            <div className={styles.queuedMeta}>
              <span className={styles.queuedDot} />
              {t('uploadInfer.inferencePanel.waitingQueue')}
            </div>
          )}

          {variant === 'completed' && (
            <div className={styles.completedMeta}>{file.inserted_at}</div>
          )}

          {variant === 'running' && typeof file.progress !== 'number' && (
            <div className={styles.statusDate}>{file.inserted_at}</div>
          )}
        </div>

        {/* Stop button — queued files only */}
        {variant === 'queued' && onStop && (
          <button
            className={styles.stopBtn}
            onClick={() => onStop(file.id)}
            disabled={stopping}
            title={t('uploadInfer.inferencePanel.stopFile')}
          >
            {stopping ? (
              <span className={styles.stopSpinner} />
            ) : (
              <svg viewBox="0 0 16 16" fill="currentColor" stroke="none">
                <rect x="4" y="4" width="8" height="8" rx="1.5" />
              </svg>
            )}
          </button>
        )}

      </div>
    </div>
  );
};

// ── InferencePanel ───────────────────────────────
interface InferencePanelProps {
  onClose?: () => void;
  minimized?: boolean;
  onToggleMinimize?: () => void;
}

const InferencePanel: React.FC<InferencePanelProps> = ({ onClose, minimized = false, onToggleMinimize }) => {
  const { t } = useTranslation();
  const dispatch = useAppDispatch();
  const {
    settings,
    selectedServerIds, isBatchRunning,
    selectFiles,
    models, modelsLoading: mlLoading, selectedModel,
    dateFrom, dateTo,
  } = useAppSelector(s => s.upload);

  const [running, setRunning] = React.useState(false);
  const [stoppingIds, setStoppingIds] = React.useState<Set<number>>(new Set());

  // ── Prompt save state ──────────────────────────
  const [promptSaving, setPromptSaving] = useState(false);
  const [promptSaveError, setPromptSaveError] = useState<string | null>(null);
  const promptSnapshot = useRef({ summary: '', keyword: '', question: '', shortAnswer: '', trueFalse: '' });
  const promptsDirty =
    settings.summaryPromptOverride !== promptSnapshot.current.summary ||
    settings.keywordPromptOverride !== promptSnapshot.current.keyword ||
    settings.questionPromptOverride !== promptSnapshot.current.question ||
    settings.shortAnswerPromptOverride !== promptSnapshot.current.shortAnswer ||
    settings.trueFalsePromptOverride !== promptSnapshot.current.trueFalse;

  const emptyBatch: ServerFilesData = { queued: [], running: [], completed: [], pending: [] };
  const [batchData, setBatchData] = React.useState<ServerFilesData>(emptyBatch);
  const [countdown, setCountdown] = React.useState(10);
  const countdownRef = useRef<ReturnType<typeof setInterval> | null>(null);
  const n = selectedServerIds.length;
  // When nothing is selected, every generation checkbox should read as
  // unchecked and be non-interactive — there's nothing to configure yet.
  // Settings themselves aren't reset (so a prior selection is restored if
  // the user re-selects the same files), only the display/interaction is.
  const noFilesSelected = n === 0;
  const isChecked = (v: boolean) => v && !noFilesSelected;
  const canRunInference = n > 0 && !isBatchRunning && !!selectedModel;
  const pollingRef = useRef<ReturnType<typeof setInterval> | null>(null);

  // ── Fetch models ────────────────────────────
  // In-flight guard: set synchronously before the await, so a second
  // overlapping call (e.g. StrictMode's intentional double mount-effect
  // invocation in dev) bails out instead of firing a second real request.
  const modelsFetchInFlightRef = useRef(false);
  const fetchModels = useCallback(async () => {
    if (modelsFetchInFlightRef.current) return;
    modelsFetchInFlightRef.current = true;
    dispatch(modelsLoading());
    try {
      const res = await api.get('/get_models');
      const result = (res.data as any)?.result ?? [];
      dispatch(modelsSuccess(result));
    } catch {
      dispatch(modelsFailure());
    } finally {
      modelsFetchInFlightRef.current = false;
    }
  }, [dispatch]); // eslint-disable-line

  useEffect(() => { fetchModels(); }, []); // eslint-disable-line

  // ── Polling helper ───────────────────────────
  // Same in-flight guard as fetchModels — this is the one that's called
  // from two different mount-time effects (the isBatchRunning effect and
  // the bootstrap effect below), so it's the most exposed to duplicate
  // overlapping calls.
  const pollFetchInFlightRef = useRef(false);
  const fetchFilesForPolling = useCallback(async () => {
    if (pollFetchInFlightRef.current) return false;
    pollFetchInFlightRef.current = true;
    try {
      const statusRes = await api.post('/files/by-progress/', { start_date: dateFrom, end_date: dateTo });
      const d = (statusRes.data as any)?.data;
      const data: ServerFilesData = {
        queued: d?.queued ?? [],
        completed: d?.completed ?? [],
        pending: d?.pending ?? [],
        running: d?.running ?? [],
      };
      dispatch(inferenceStatusSuccess(data));
      setBatchData(data);

      const stillRunning = data.running.length > 0 || data.queued.length > 0;

      if (data.running.length > 0) {
        const runningIds = data.running.map(f => f.id);
        try {
          const progressRes = await api.post('/files/progress/', { file_ids: runningIds });
          const progressMap = (progressRes.data as any)?.result ?? {};
          dispatch(updateRunningProgress(progressMap));
          setBatchData(prev => ({
            ...prev,
            running: prev.running.map(f => {
              const raw = progressMap[String(f.id)];
              if (raw === undefined) return f;
              const pct = typeof raw === 'string' ? parseFloat(raw) : raw;
              return { ...f, progress: isNaN(pct) ? f.progress : Math.min(100, Math.max(0, pct)) };
            }),
          }));
        } catch {
          // progress fetch failing is non-critical — silently ignore
        }
      }

      if (!stillRunning) setBatchData(emptyBatch);
      return stillRunning;
    } catch {
      return false;
    } finally {
      pollFetchInFlightRef.current = false;
    }
  }, [dispatch, dateFrom, dateTo]);

  const stopCountdown = useCallback(() => {
    if (countdownRef.current) {
      clearInterval(countdownRef.current);
      countdownRef.current = null;
    }
  }, []);

  const startCountdown = useCallback(() => {
    stopCountdown();
    setCountdown(10);
    countdownRef.current = setInterval(() => {
      setCountdown(prev => (prev <= 1 ? 10 : prev - 1));
    }, 1000);
  }, [stopCountdown]);

  const stopPolling = useCallback(() => {
    if (pollingRef.current) {
      clearInterval(pollingRef.current);
      pollingRef.current = null;
    }
    stopCountdown();
    setCountdown(10);
  }, [stopCountdown]);

  const startPolling = useCallback(() => {
    stopPolling();
    startCountdown();
    pollingRef.current = setInterval(async () => {
      const stillRunning = await fetchFilesForPolling();
      if (!stillRunning) stopPolling();
      else startCountdown();
    }, 10000);
  }, [fetchFilesForPolling, stopPolling, startCountdown]);

  useEffect(() => {
    if (isBatchRunning) {
      fetchFilesForPolling().then(stillRunning => {
        if (stillRunning) startPolling();
        else stopPolling();
      });
    } else {
      stopPolling();
    }
    return stopPolling;
  }, [isBatchRunning]); // eslint-disable-line

  // ── One-time bootstrap check on mount ──
  // /files/by-date/ is now paginated and no longer tells us whether a batch
  // is running (a running/queued file may simply be on another page), so we
  // can't rely on its response to seed isBatchRunning like before. Ask
  // /files/by-progress/ directly once on mount to catch a batch that was
  // already running before this page loaded (e.g. after a refresh).
  useEffect(() => {
    fetchFilesForPolling().then(stillRunning => { if (stillRunning) startPolling(); });
  }, []); // eslint-disable-line

  // ── Stop a queued file ───────────────────────
  const handleStop = useCallback(async (fileId: number) => {
    setStoppingIds(prev => new Set(prev).add(fileId));
    try {
      await api.patch('/files/stop', { fileID: [fileId] });
    } catch (err: any) {
      if (err?.response?.status === 403) {
        dispatch(addToast(t('uploadInfer.inferencePanel.stopAlready'), 'error'));
      } else {
        console.error('Stop file failed:', err);
      }
    } finally {
      await fetchFilesForPolling();
      setStoppingIds(prev => { const s = new Set(prev); s.delete(fileId); return s; });
    }
  }, [fetchFilesForPolling]);

  // ── Run inference ────────────────────────────
  const handleRun = async () => {
    if (!canRunInference) return;
    setRunning(true);
    try {
      await api.post('/batch_process', {
        all: false, // always false for now — file_ids-driven runs only
        file_ids: selectedServerIds,
        start_date: dateFrom,
        end_date: dateTo,
        model_name: selectedModel,
        summary_prompt: settings.summaryPromptOverride,
        faq_prompt: settings.questionPromptOverride,
        keywords_prompt: settings.keywordPromptOverride,
        short_answers_prompt: settings.shortAnswerPromptOverride,
        true_false_prompt: settings.trueFalsePromptOverride,
        generate_summary: settings.generateSummary,
        generate_keywords: settings.generateKeywords,
        generate_faq: settings.generateQuestions,
        generate_short_answer: settings.generateShortAnswer,
        generate_true_false: settings.generateTrueFalse,
        // Only meaningful — and only ever sent true — when Keywords itself is on.
        generate_keyword_insights: settings.generateKeywords && settings.generateKeywordInsights,
        timestamped_summary: settings.timestampedSummary,
        time_interval: settings.timeInterval,
      });
      const stillRunning = await fetchFilesForPolling();
      if (stillRunning) startPolling();
      dispatch(addToast(t('uploadInfer.inferencePanel.inferenceStarted'), 'success'));
    } catch (err) {
      console.error('Batch process failed:', err);
    } finally {
      setRunning(false);
    }
  };

  // ── Run button sub-label ─────────────────────
  const runBtnSubLabel = (() => {
    if (n === 0) return t('uploadInfer.inferencePanel.selectFilesFirst');
    if (!selectedModel) return t('uploadInfer.inferencePanel.noModelSelected');
    return `${n} file${n !== 1 ? 's' : ''} ready`;
  })();

  // ── Auto-fill prompts when exactly 1 file is selected ────────────
  // Keyed off the *actual selected file id* (or a stable "multi"/"none"
  // marker), not the selection count and not the selectedServerIds array
  // reference. Redux can hand us a brand-new array reference for the same
  // underlying selection on unrelated updates — keying off the array (or
  // count alone) made this effect refire and stomp on prompt edits the
  // user was still mid-typing, even though the selection hadn't changed.
  const singleSelectedId = n === 1 ? selectedServerIds[0] : null;
  const selectionKey = n === 1 ? `one:${singleSelectedId}` : n === 0 ? 'none' : 'multi';
  const prevSelectionKey = useRef<string | null>(null);
  useEffect(() => {
    if (selectionKey === prevSelectionKey.current) return;
    prevSelectionKey.current = selectionKey;

    if (n === 1) {
      const file = selectFiles.find(f => f.id === singleSelectedId);
      if (file) {
        const s = file.summary_prompt ?? '';
        const k = file.keywords_prompt ?? '';
        const q = file.faq_prompt ?? '';
        const sa = file.short_answer_prompt ?? '';
        const tf = file.true_false_prompt ?? '';
        dispatch(updateSummaryPrompt(s));
        dispatch(updateKeywordPrompt(k));
        dispatch(updateQuestionPrompt(q));
        dispatch(updateShortAnswerPrompt(sa));
        dispatch(updateTrueFalsePrompt(tf));
        promptSnapshot.current = { summary: s, keyword: k, question: q, shortAnswer: sa, trueFalse: tf };
      }
    } else {
      dispatch(updateSummaryPrompt(''));
      dispatch(updateKeywordPrompt(''));
      dispatch(updateQuestionPrompt(''));
      dispatch(updateShortAnswerPrompt(''));
      dispatch(updateTrueFalsePrompt(''));
      promptSnapshot.current = { summary: '', keyword: '', question: '', shortAnswer: '', trueFalse: '' };
    }
    setPromptSaveError(null);
  }, [selectionKey]); // eslint-disable-line

  // ── Per-file prompt preview expand/collapse ──
  const [summaryExpanded, setSummaryExpanded] = useState(false);
  const [keywordExpanded, setKeywordExpanded] = useState(false);
  const [questionExpanded, setQuestionExpanded] = useState(false);
  const [shortAnswerExpanded, setShortAnswerExpanded] = useState(false);
  const [trueFalseExpanded, setTrueFalseExpanded] = useState(false);

  const prevSelectedCount = useRef(selectedServerIds.length);
  useEffect(() => {
    if (selectedServerIds.length !== prevSelectedCount.current) {
      prevSelectedCount.current = selectedServerIds.length;
      setSummaryExpanded(false);
      setKeywordExpanded(false);
      setQuestionExpanded(false);
      setShortAnswerExpanded(false);
      setTrueFalseExpanded(false);
    }
  });

  const selectedFiles = selectFiles.filter(f => selectedServerIds.includes(f.id));

  const handlePromptCancel = () => {
    dispatch(updateSummaryPrompt(promptSnapshot.current.summary));
    dispatch(updateKeywordPrompt(promptSnapshot.current.keyword));
    dispatch(updateQuestionPrompt(promptSnapshot.current.question));
    dispatch(updateShortAnswerPrompt(promptSnapshot.current.shortAnswer));
    dispatch(updateTrueFalsePrompt(promptSnapshot.current.trueFalse));
    setPromptSaveError(null);
  };

  const handlePromptSave = async () => {
    setPromptSaving(true);
    setPromptSaveError(null);
    try {
      const payload: Record<string, unknown> = {
        file_ids: selectedServerIds,
      };
      if (settings.generateSummary) payload.summary_prompt = settings.summaryPromptOverride;
      if (settings.generateKeywords) payload.keywords_prompt = settings.keywordPromptOverride;
      if (settings.generateQuestions) payload.faq_prompt = settings.questionPromptOverride;
      if (settings.generateShortAnswer) payload.short_answers_prompt = settings.shortAnswerPromptOverride;
      if (settings.generateTrueFalse) payload.true_false_prompt = settings.trueFalsePromptOverride;

      await api.post('/prompt_update', payload);

      dispatch(updateFilePrompts({
        fileIds: selectedServerIds,
        ...(settings.generateSummary && { summaryPrompt: settings.summaryPromptOverride }),
        ...(settings.generateKeywords && { keywordsPrompt: settings.keywordPromptOverride }),
        ...(settings.generateQuestions && { faqPrompt: settings.questionPromptOverride }),
        ...(settings.generateShortAnswer && { shortAnswerPrompt: settings.shortAnswerPromptOverride }),
        ...(settings.generateTrueFalse && { trueFalsePrompt: settings.trueFalsePromptOverride }),
      }));

      promptSnapshot.current = {
        summary: settings.summaryPromptOverride,
        keyword: settings.keywordPromptOverride,
        question: settings.questionPromptOverride,
        shortAnswer: settings.shortAnswerPromptOverride,
        trueFalse: settings.trueFalsePromptOverride,
      };
    } catch {
      setPromptSaveError(t('uploadInfer.inferencePanel.promptSaveFail'));
    } finally {
      setPromptSaving(false);
    }
  };

  return (
    <div className={`${styles.infpanel} ${minimized ? styles.infpanelMinimized : ''}`}>

      {/* ── Minimized rail ── */}
      {minimized ? (
        <button
          type="button"
          className={styles.minRail}
          onClick={onToggleMinimize}
          title={t('uploadInfer.inferencePanel.expandStep2')}
          aria-label={t('uploadInfer.inferencePanel.expandStep2')}
        >
          <span className={styles.minRailIcon}>
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
              strokeWidth="1.6" strokeLinecap="round" strokeLinejoin="round">
              <path d="M6 4l4 4-4 4" />
            </svg>
          </span>
          <span className={styles.minRailLabel}>
            {t('uploadInfer.inferencePanel.step2Label').replace('Configuration', '').replace('2 —', '2 —')}
            {isBatchRunning && <span className={styles.minRailDot} />}
          </span>
        </button>
      ) : (
        <>

          {/* ── Header ── */}
          <div className={styles.infpanelHead}>
            <div>
              <div className={styles.slbl}>
                {isBatchRunning ? (
                  <span className={styles.inferenceRunningLabel}>{t('uploadInfer.inferencePanel.step2Running')}</span>
                ) : t('uploadInfer.inferencePanel.step2Label')}
              </div>
              <div className={styles.selSummary}>
                {t('uploadInfer.inferencePanel.filesSelected', { count: n })}
                {isBatchRunning && <span className={styles.batchRunPill}><span className={styles.batchRunDot} />{t('uploadInfer.inferencePanel.batchRunPill')}</span>}
              </div>
            </div>
            <div className={styles.headActions}>
              {!isBatchRunning && (
                <>
                  {/* ── Run Inference button — large, self-describing ── */}
                  <button
                    className={`${styles.runBtn} ${canRunInference ? styles.runBtnReady : styles.runBtnDisabled}`}
                    onClick={handleRun}
                    disabled={!canRunInference}
                    data-tour="infer-run"
                    aria-label={`Run inference on ${n} file${n !== 1 ? 's' : ''}`}
                  >
                    <span className={styles.runBtnIconWrap}>
                      <svg width="15" height="15" viewBox="0 0 16 16" aria-hidden="true">
                        <path d="M5 3l8 5-8 5V3z" fill="currentColor" />
                      </svg>
                    </span>
                    <span className={styles.runBtnText}>
                      <span className={styles.runBtnTitle}>Run inference</span>
                      <span className={styles.runBtnSub}>{runBtnSubLabel}</span>
                    </span>
                  </button>

                  {onToggleMinimize && (
                    <button
                      className={styles.minimizeStepBtn}
                      onClick={onToggleMinimize}
                      title={t('uploadInfer.inferencePanel.minimizeStep2')}
                      aria-label={t('uploadInfer.inferencePanel.minimizeStep2')}
                    >
                      <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
                        strokeWidth="1.6" strokeLinecap="round">
                        <path d="M3 8h10" />
                      </svg>
                    </button>
                  )}
                  {onClose && (
                    <button
                      className={styles.closeStepBtn}
                      onClick={onClose}
                      title={t('uploadInfer.inferencePanel.closeStep2')}
                    >
                      <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
                        <path d="M4 4l8 8M12 4l-8 8" />
                      </svg>
                    </button>
                  )}
                </>
              )}
              {isBatchRunning && onToggleMinimize && (
                <button
                  className={styles.minimizeStepBtn}
                  onClick={onToggleMinimize}
                  title={t('uploadInfer.inferencePanel.minimizeStep2')}
                  aria-label={t('uploadInfer.inferencePanel.minimizeStep2')}
                >
                  <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
                    strokeWidth="1.6" strokeLinecap="round">
                    <path d="M3 8h10" />
                  </svg>
                </button>
              )}
            </div>
          </div>


          {/* ── Main body ── */}
          <div className={`${styles.infpanelBody} ${isBatchRunning ? styles.infpanelBodyRunning : styles.infpanelBodyConfig}`}>

            {/* ── Submitting overlay — shown while awaiting /batch_process ── */}
            {running && (
              <div className={styles.submittingOverlay}>
                <div className={styles.submittingCard}>
                  <div className={styles.submittingSpinner} />
                  <div className={styles.submittingTitle}>{t('uploadInfer.inferencePanel.submitting')}</div>
                  <div className={styles.submittingDesc}>
                    {t('uploadInfer.inferencePanel.submittingDesc', { count: n }).split('\n').map((line, i) => <React.Fragment key={i}>{line}{i === 0 && <br />}</React.Fragment>)}
                  </div>
                  <div className={styles.submittingFiles}>
                    {selectedServerIds.slice(0, 5).map((id, i) => {
                      const f = selectFiles.find(sf => sf.id === id);
                      return f ? (
                        <div key={id} className={styles.submittingFile}>
                          <span className={styles.submittingDot} style={{ animationDelay: `${i * 0.15}s` }} />
                          {f.original_name}
                        </div>
                      ) : null;
                    })}
                    {selectedServerIds.length > 5 && (
                      <div className={styles.submittingMore}>+{selectedServerIds.length - 5} more</div>
                    )}
                  </div>
                </div>
              </div>
            )}

            {/* Selection banner — hidden while batch running or submitting */}
            {!isBatchRunning && !running && <div className={styles.selBanner}>
              <svg width="12" height="12" viewBox="0 0 16 16" fill="none" stroke="var(--blue)" strokeWidth="1.5" strokeLinecap="round">
                <path d="M2 8h12M8 3l5 5-5 5" />
              </svg>
              <span className={styles.selCt}>{n} file{n !== 1 ? 's' : ''}</span>
              <span className={styles.selNm}>
                {n === 0 ? t('uploadInfer.inferencePanel.selBannerEmpty') : `${n} file${n !== 1 ? 's' : ''} selected for inference`}
              </span>
            </div>}

            {/* ── Settings — hidden while batch running or submitting ── */}
            {!isBatchRunning && !running && <div className={styles.infSettingsWrap} data-tour="infer-settings">
              <div className={styles.settingsGrid} data-tour="infer-settings-grid">

                {/* Generate content card */}
                <div className={styles.card}>
                  <div className={styles.cardT}>{t('uploadInfer.inferencePanel.generateContent')}</div>

                  {/* Model dropdown — moved to the top; deliberately narrow rather than spanning the row */}
                  <div className={styles.modelFieldTop} data-tour="infer-model">
                    <div className={styles.fg}>
                      <div className={styles.modelLabelRow}>
                        <label className={styles.fl}>
                          {t('uploadInfer.inferencePanel.modelLabel')} {mlLoading && <span className={styles.loadingDot}>…</span>}
                        </label>
                        <button
                          className={styles.refreshBtn}
                          onClick={fetchModels}
                          disabled={mlLoading}
                          title={t('uploadInfer.inferencePanel.refreshModels')}
                        >
                          <svg
                            viewBox="0 0 16 16" fill="none" stroke="currentColor"
                            strokeWidth="1.6" strokeLinecap="round" strokeLinejoin="round"
                            className={mlLoading ? styles.spinning : undefined}
                          >
                            <path d="M13.5 8A5.5 5.5 0 1 1 10 3.07" />
                            <path d="M10 2v3h3" />
                          </svg>
                          Refresh
                        </button>
                      </div>
                      <select className={styles.fc} value={selectedModel}
                        onChange={e => dispatch(setSelectedModel(e.target.value))}
                        disabled={mlLoading || models.length === 0}>
                        {models.length === 0
                          ? <option value="">{t('uploadInfer.inferencePanel.noModelsOption')}</option>
                          : models.map(m => <option key={m} value={m}>{m}</option>)
                        }
                      </select>
                      {!mlLoading && models.length === 0 && (
                        <div className={styles.modelWarn}>
                          <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                            <path d="M8 2L14 13H2L8 2z" /><path d="M8 7v3M8 11.5v.1" />
                          </svg>
                          {t('uploadInfer.inferencePanel.noModels')}
                        </div>
                      )}
                      {!mlLoading && models.length > 0 && (
                        <div className={styles.modelOk}>
                          <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                            <circle cx="8" cy="8" r="5.5" /><path d="M5.5 8l2 2 3-3" />
                          </svg>
                          {t('uploadInfer.inferencePanel.modelsAvailable', { count: models.length })}
                        </div>
                      )}
                    </div>
                  </div>

                  <div className={styles.genContentCols}>

                  {/* ── Left column: all checkboxes ── */}
                  <div className={styles.checkboxCol}>

                    <div data-tour="infer-check-summary"
                      className={`${styles.cr} ${isChecked(settings.generateSummary) ? styles.ck : ''} ${styles.mb8} ${noFilesSelected ? styles.crDisabled : ''}`}
                      onClick={() => { if (noFilesSelected) return; dispatch(updateSettings({ generateSummary: !settings.generateSummary })); }}
                    >
                      <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.generateSummary')}</label>
                    </div>

                    <div data-tour="infer-check-keywords"
                      className={`${styles.cr} ${isChecked(settings.generateKeywords) ? styles.ck : ''} ${styles.mb8} ${noFilesSelected ? styles.crDisabled : ''}`}
                      onClick={() => {
                        if (noFilesSelected) return;
                        dispatch(updateSettings({
                          generateKeywords: !settings.generateKeywords,
                          // Force the nested toggle off when the parent turns off,
                          // so a stale "on" state can never leak into the request.
                          ...(settings.generateKeywords ? { generateKeywordInsights: false } : {}),
                        }));
                      }}
                    >
                      <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.generateKeywords')}</label>
                    </div>
                    {isChecked(settings.generateKeywords) && (
                      <div
                        className={`${styles.crNested} ${isChecked(settings.generateKeywordInsights) ? styles.ck : ''} ${styles.mb8}`}
                        onClick={(e) => { e.stopPropagation(); if (noFilesSelected) return; dispatch(updateSettings({ generateKeywordInsights: !settings.generateKeywordInsights })); }}
                      >
                        <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.generateKeywordInsights')}</label>
                      </div>
                    )}

                    <div data-tour="infer-check-questions"
                      className={`${styles.cr} ${isChecked(settings.generateQuestions) ? styles.ck : ''} ${styles.mb8} ${noFilesSelected ? styles.crDisabled : ''}`}
                      onClick={() => { if (noFilesSelected) return; dispatch(updateSettings({ generateQuestions: !settings.generateQuestions })); }}
                    >
                      <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.generateQuestions')}</label>
                    </div>

                    <div data-tour="infer-check-shortanswer"
                      className={`${styles.cr} ${isChecked(settings.generateShortAnswer) ? styles.ck : ''} ${styles.mb8} ${noFilesSelected ? styles.crDisabled : ''}`}
                      onClick={() => { if (noFilesSelected) return; dispatch(updateSettings({ generateShortAnswer: !settings.generateShortAnswer })); }}
                    >
                      <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.generateShortAnswer')}</label>
                    </div>

                    <div data-tour="infer-check-truefalse"
                      className={`${styles.cr} ${isChecked(settings.generateTrueFalse) ? styles.ck : ''} ${styles.mb8} ${noFilesSelected ? styles.crDisabled : ''}`}
                      onClick={() => { if (noFilesSelected) return; dispatch(updateSettings({ generateTrueFalse: !settings.generateTrueFalse })); }}
                    >
                      <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.generateTrueFalse')}</label>
                    </div>

                    <div data-tour="infer-check-timestamped"
                      className={`${styles.cr} ${isChecked(settings.timestampedSummary) ? styles.ck : ''} ${styles.mb8} ${noFilesSelected ? styles.crDisabled : ''}`}
                      onClick={() => { if (noFilesSelected) return; dispatch(updateSettings({ timestampedSummary: !settings.timestampedSummary })); }}
                    >
                      <div className={styles.cb} /><label>{t('uploadInfer.inferencePanel.timestampedSummary')}</label>
                    </div>
                    {isChecked(settings.timestampedSummary) && (
                      <div className={styles.nestedField} onClick={(e) => e.stopPropagation()}>
                        <label>{t('uploadInfer.inferencePanel.timeInterval')}</label>
                        <select
                          className={styles.fc}
                          value={settings.timeInterval}
                          onChange={e => dispatch(updateSettings({ timeInterval: Number(e.target.value) as TimeInterval }))}
                        >
                          {([5, 10, 15, 20, 30, 45, 60] as const).map(min => (
                            <option key={min} value={min}>{t('uploadInfer.inferencePanel.minutesOption', { count: min })}</option>
                          ))}
                        </select>
                      </div>
                    )}

                  </div>

                  {/* ── Right column: prompt panel for whichever checkboxes are on ── */}
                  <div className={styles.promptCol}>

                    {!isChecked(settings.generateSummary) && !isChecked(settings.generateKeywords)
                      && !isChecked(settings.generateQuestions) && !isChecked(settings.generateShortAnswer)
                      && !isChecked(settings.generateTrueFalse) && (
                      <div className={styles.noPromptHint}>
                        {noFilesSelected
                          ? t('uploadInfer.inferencePanel.selBannerEmpty')
                          : t('uploadInfer.inferencePanel.noPromptHint', 'Check an option on the left to set its prompt')}
                      </div>
                    )}

                    {isChecked(settings.generateSummary) && (
                      <div className={styles.promptCard}>
                        <div className={styles.fg}>
                          <div className={styles.flRow}>
                            <label className={styles.fl}>
                              {t('uploadInfer.inferencePanel.summaryPrompt')} <span className={styles.optTag}>{t('uploadInfer.inferencePanel.optional')}</span>
                            </label>
                            {n > 1 && (
                              <button
                                className={`${styles.perFileBtn} ${summaryExpanded ? styles.perFileBtnActive : ''}`}
                                onClick={(e) => { e.stopPropagation(); setSummaryExpanded(v => !v); }}
                                title={summaryExpanded ? 'Collapse per-file prompts' : 'View per-file prompts'}
                              >
                                <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                  <rect x="1.5" y="2" width="11" height="10" rx="2" />
                                  <path d="M4 5h6M4 7.5h4" />
                                </svg>
                                {summaryExpanded ? 'Hide' : t('uploadInfer.inferencePanel.perFile')}
                                <svg className={`${styles.perFileChevron} ${summaryExpanded ? styles.perFileChevronOpen : ''}`} viewBox="0 0 10 10" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                                  <path d="M2 3.5l3 3 3-3" />
                                </svg>
                              </button>
                            )}
                          </div>
                          <div className={`${styles.perFilePanel} ${summaryExpanded ? styles.perFilePanelOpen : ''}`}>
                            <div className={styles.perFilePanelInner}>
                              {selectedFiles.map(f => (
                                <div key={f.id} className={styles.perFileRow}>
                                  <div className={styles.perFileName}>{f.original_name}</div>
                                  <div className={styles.perFilePrompt}>{f.summary_prompt || <span className={styles.perFileEmpty}>{t('uploadInfer.inferencePanel.noPromptSet')}</span>}</div>
                                </div>
                              ))}
                            </div>
                          </div>
                          <textarea className={styles.fc} rows={3}
                            value={settings.summaryPromptOverride}
                            placeholder={n === 1 ? t('uploadInfer.inferencePanel.autofillPlaceholder') : t('uploadInfer.inferencePanel.manualPlaceholder')}
                            onChange={e => dispatch(updateSummaryPrompt(e.target.value))} />
                        </div>
                      </div>
                    )}

                    {isChecked(settings.generateKeywords) && (
                      <div className={styles.promptCard}>
                        <div className={styles.fg}>
                          <div className={styles.flRow}>
                            <label className={styles.fl}>
                              {t('uploadInfer.inferencePanel.keywordPrompt')} <span className={styles.optTag}>{t('uploadInfer.inferencePanel.optional')}</span>
                            </label>
                            {n > 1 && (
                              <button
                                className={`${styles.perFileBtn} ${keywordExpanded ? styles.perFileBtnActive : ''}`}
                                onClick={(e) => { e.stopPropagation(); setKeywordExpanded(v => !v); }}
                                title={keywordExpanded ? 'Collapse per-file prompts' : 'View per-file prompts'}
                              >
                                <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                  <rect x="1.5" y="2" width="11" height="10" rx="2" />
                                  <path d="M4 5h6M4 7.5h4" />
                                </svg>
                                {keywordExpanded ? t('uploadInfer.inferencePanel.hide') : t('uploadInfer.inferencePanel.perFile')}
                                <svg className={`${styles.perFileChevron} ${keywordExpanded ? styles.perFileChevronOpen : ''}`} viewBox="0 0 10 10" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                                  <path d="M2 3.5l3 3 3-3" />
                                </svg>
                              </button>
                            )}
                          </div>
                          <div className={`${styles.perFilePanel} ${keywordExpanded ? styles.perFilePanelOpen : ''}`}>
                            <div className={styles.perFilePanelInner}>
                              {selectedFiles.map(f => (
                                <div key={f.id} className={styles.perFileRow}>
                                  <div className={styles.perFileName}>{f.original_name}</div>
                                  <div className={styles.perFilePrompt}>{f.keywords_prompt || <span className={styles.perFileEmpty}>No prompt set</span>}</div>
                                </div>
                              ))}
                            </div>
                          </div>
                          <textarea className={styles.fc} rows={3}
                            value={settings.keywordPromptOverride}
                            placeholder={n === 1 ? 'Autofilled from file — edit to override…' : 'Leave blank to use per-file prompts…'}
                            onChange={e => dispatch(updateKeywordPrompt(e.target.value))} />
                        </div>
                      </div>
                    )}

                    {isChecked(settings.generateQuestions) && (
                      <div className={styles.promptCard}>
                        <div className={styles.fg}>
                          <div className={styles.flRow}>
                            <label className={styles.fl}>
                              {t('uploadInfer.inferencePanel.questionPrompt')} <span className={styles.optTag}>{t('uploadInfer.inferencePanel.optional')}</span>
                            </label>
                            {n > 1 && (
                              <button
                                className={`${styles.perFileBtn} ${questionExpanded ? styles.perFileBtnActive : ''}`}
                                onClick={(e) => { e.stopPropagation(); setQuestionExpanded(v => !v); }}
                                title={questionExpanded ? 'Collapse per-file prompts' : 'View per-file prompts'}
                              >
                                <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                  <rect x="1.5" y="2" width="11" height="10" rx="2" />
                                  <path d="M4 5h6M4 7.5h4" />
                                </svg>
                                {questionExpanded ? t('uploadInfer.inferencePanel.hide') : t('uploadInfer.inferencePanel.perFile')}
                                <svg className={`${styles.perFileChevron} ${questionExpanded ? styles.perFileChevronOpen : ''}`} viewBox="0 0 10 10" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                                  <path d="M2 3.5l3 3 3-3" />
                                </svg>
                              </button>
                            )}
                          </div>
                          <div className={`${styles.perFilePanel} ${questionExpanded ? styles.perFilePanelOpen : ''}`}>
                            <div className={styles.perFilePanelInner}>
                              {selectedFiles.map(f => (
                                <div key={f.id} className={styles.perFileRow}>
                                  <div className={styles.perFileName}>{f.original_name}</div>
                                  <div className={styles.perFilePrompt}>{f.faq_prompt || <span className={styles.perFileEmpty}>No prompt set</span>}</div>
                                </div>
                              ))}
                            </div>
                          </div>
                          <textarea className={styles.fc} rows={3}
                            value={settings.questionPromptOverride}
                            placeholder={n === 1 ? 'Autofilled from file — edit to override…' : 'Leave blank to use per-file prompts…'}
                            onChange={e => dispatch(updateQuestionPrompt(e.target.value))} />
                        </div>
                      </div>
                    )}

                    {isChecked(settings.generateShortAnswer) && (
                      <div className={styles.promptCard}>
                        <div className={styles.fg}>
                          <div className={styles.flRow}>
                            <label className={styles.fl}>
                              {t('uploadInfer.inferencePanel.shortAnswerPrompt')} <span className={styles.optTag}>{t('uploadInfer.inferencePanel.optional')}</span>
                            </label>
                            {n > 1 && (
                              <button
                                className={`${styles.perFileBtn} ${shortAnswerExpanded ? styles.perFileBtnActive : ''}`}
                                onClick={(e) => { e.stopPropagation(); setShortAnswerExpanded(v => !v); }}
                                title={shortAnswerExpanded ? t('uploadInfer.inferencePanel.hide') : t('uploadInfer.inferencePanel.viewPerFile')}
                              >
                                <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                  <rect x="1.5" y="2" width="11" height="10" rx="2" />
                                  <path d="M4 5h6M4 7.5h4" />
                                </svg>
                                {shortAnswerExpanded ? t('uploadInfer.inferencePanel.hide') : t('uploadInfer.inferencePanel.perFile')}
                                <svg className={`${styles.perFileChevron} ${shortAnswerExpanded ? styles.perFileChevronOpen : ''}`} viewBox="0 0 10 10" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                                  <path d="M2 3.5l3 3 3-3" />
                                </svg>
                              </button>
                            )}
                          </div>
                          <div className={`${styles.perFilePanel} ${shortAnswerExpanded ? styles.perFilePanelOpen : ''}`}>
                            <div className={styles.perFilePanelInner}>
                              {selectedFiles.map(f => (
                                <div key={f.id} className={styles.perFileRow}>
                                  <div className={styles.perFileName}>{f.original_name}</div>
                                  <div className={styles.perFilePrompt}>{f.short_answer_prompt || <span className={styles.perFileEmpty}>{t('uploadInfer.inferencePanel.noPromptSet')}</span>}</div>
                                </div>
                              ))}
                            </div>
                          </div>
                          <textarea className={styles.fc} rows={3}
                            value={settings.shortAnswerPromptOverride}
                            placeholder={n === 1 ? t('uploadInfer.inferencePanel.autofillPlaceholder') : t('uploadInfer.inferencePanel.manualPlaceholder')}
                            onChange={e => dispatch(updateShortAnswerPrompt(e.target.value))} />
                        </div>
                      </div>
                    )}

                    {isChecked(settings.generateTrueFalse) && (
                      <div className={styles.promptCard}>
                        <div className={styles.fg}>
                          <div className={styles.flRow}>
                            <label className={styles.fl}>
                              {t('uploadInfer.inferencePanel.trueFalsePrompt')} <span className={styles.optTag}>{t('uploadInfer.inferencePanel.optional')}</span>
                            </label>
                            {n > 1 && (
                              <button
                                className={`${styles.perFileBtn} ${trueFalseExpanded ? styles.perFileBtnActive : ''}`}
                                onClick={(e) => { e.stopPropagation(); setTrueFalseExpanded(v => !v); }}
                                title={trueFalseExpanded ? t('uploadInfer.inferencePanel.hide') : t('uploadInfer.inferencePanel.viewPerFile')}
                              >
                                <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                  <rect x="1.5" y="2" width="11" height="10" rx="2" />
                                  <path d="M4 5h6M4 7.5h4" />
                                </svg>
                                {trueFalseExpanded ? t('uploadInfer.inferencePanel.hide') : t('uploadInfer.inferencePanel.perFile')}
                                <svg className={`${styles.perFileChevron} ${trueFalseExpanded ? styles.perFileChevronOpen : ''}`} viewBox="0 0 10 10" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                                  <path d="M2 3.5l3 3 3-3" />
                                </svg>
                              </button>
                            )}
                          </div>
                          <div className={`${styles.perFilePanel} ${trueFalseExpanded ? styles.perFilePanelOpen : ''}`}>
                            <div className={styles.perFilePanelInner}>
                              {selectedFiles.map(f => (
                                <div key={f.id} className={styles.perFileRow}>
                                  <div className={styles.perFileName}>{f.original_name}</div>
                                  <div className={styles.perFilePrompt}>{f.true_false_prompt || <span className={styles.perFileEmpty}>{t('uploadInfer.inferencePanel.noPromptSet')}</span>}</div>
                                </div>
                              ))}
                            </div>
                          </div>
                          <textarea className={styles.fc} rows={3}
                            value={settings.trueFalsePromptOverride}
                            placeholder={n === 1 ? t('uploadInfer.inferencePanel.autofillPlaceholder') : t('uploadInfer.inferencePanel.manualPlaceholder')}
                            onChange={e => dispatch(updateTrueFalsePrompt(e.target.value))} />
                        </div>
                      </div>
                    )}

                  </div>

                  </div>

                  {/* Save / Cancel bar — persists any edited prompt overrides above */}
                  {n > 0 && (settings.generateSummary || settings.generateKeywords || settings.generateQuestions
                    || settings.generateShortAnswer || settings.generateTrueFalse) && (
                    <div className={styles.promptActions}>
                      {promptSaveError && (
                        <span className={styles.promptSaveError}>{promptSaveError}</span>
                      )}
                      <button
                        className={`${styles.btn} ${styles.btnSm} ${styles.btnGhost}`}
                        onClick={handlePromptCancel}
                        disabled={!promptsDirty || promptSaving}
                      >
                        Cancel
                      </button>
                      <button
                        className={`${styles.btn} ${styles.btnSm} ${styles.btnPrimary}`}
                        onClick={handlePromptSave}
                        disabled={!promptsDirty || promptSaving}
                      >
                        {promptSaving ? (
                          <><span className={styles.btnSpinner} />{t('uploadInfer.inferencePanel.saving')}</>
                        ) : t('uploadInfer.inferencePanel.savePrompts')}
                      </button>
                    </div>
                  )}
                </div>

              </div>
            </div>}

            {/* ══ 3-Column batch status ══ */}
            {isBatchRunning && (
              <div className={styles.batchSection}>
                <div className={styles.batchSectionTitle}>
                  <span className={styles.liveLabel}>{t('uploadInfer.inferencePanel.inferenceStatus')}</span>
                  {isBatchRunning && (
                    <span className={styles.liveStatus}>
                      <span className={styles.liveDot} />
                      Live
                      <span className={styles.liveSep}>·</span>
                      {t('uploadInfer.inferencePanel.refreshingIn', { sec: countdown })}
                      {batchData.running.length > 0 && (
                        <><span className={styles.liveSep}>·</span>
                          <span className={styles.liveFiles}>
                            {t('uploadInfer.inferencePanel.filesRunning', { count: batchData.running.length })}
                          </span></>
                      )}
                    </span>
                  )}
                </div>
                <div className={styles.batchColumns}>

                  {/* Queued */}
                  <div className={styles.batchCol}>
                    <div className={`${styles.batchColHead} ${styles.headQueued}`}>
                      <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round">
                        <circle cx="7" cy="7" r="5" /><path d="M7 4v3.5l2 1.5" />
                      </svg>
                      Queued
                      <span className={styles.batchColCount}>{batchData.queued.length}</span>
                    </div>
                    <div className={styles.batchColBody}>
                      {batchData.queued.length === 0
                        ? <div className={styles.batchEmpty}>—</div>
                        : batchData.queued.map(f => (
                          <StatusCard
                            key={f.id}
                            file={f}
                            variant="queued"
                            onStop={handleStop}
                            stopping={stoppingIds.has(f.id)}
                          />
                        ))
                      }
                    </div>
                  </div>

                  {/* Running */}
                  <div className={styles.batchCol}>
                    <div className={`${styles.batchColHead} ${styles.headRunning}`}>
                      <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round">
                        <circle cx="7" cy="7" r="5" /><path d="M7 4v3M7 9.5v.2" />
                      </svg>
                      Running
                      <span className={styles.batchColCount}>{batchData.running.length}</span>
                    </div>
                    <div className={styles.batchColBody}>
                      {batchData.running.length === 0
                        ? <div className={styles.batchEmpty}>—</div>
                        : batchData.running.map(f => <StatusCard key={f.id} file={f} variant="running" />)
                      }
                    </div>
                  </div>

                  {/* Completed */}
                  <div className={styles.batchCol}>
                    <div className={`${styles.batchColHead} ${styles.headCompleted}`}>
                      <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                        <circle cx="7" cy="7" r="5" /><path d="M4.5 7l2 2 3-3" />
                      </svg>
                      Completed
                      <span className={styles.batchColCount}>{batchData.completed.length}</span>
                    </div>
                    <div className={styles.batchColBody}>
                      {batchData.completed.length === 0
                        ? <div className={styles.batchEmpty}>—</div>
                        : batchData.completed.map(f => <StatusCard key={f.id} file={f} variant="completed" />)
                      }
                    </div>
                  </div>

                </div>
              </div>
            )}

          </div>
        </>
      )}
    </div>
  );
};

export default InferencePanel;





















// ═══════════════════════════════════════════════
// pages/UploadInfer/WorkspacePanel.tsx
// LectureAI · Step-3 Workspace Result panel
// ═══════════════════════════════════════════════
import React, { useState, useMemo, useRef, useEffect, useCallback, forwardRef, useImperativeHandle } from 'react';
import { useTranslation } from 'react-i18next';
import { marked } from 'marked';
import {
    ReactFlow, Background, Controls, MiniMap, Handle, Position, applyNodeChanges,
    type Node as RFNode, type Edge as RFEdge, type NodeChange,
} from '@xyflow/react';
import '@xyflow/react/dist/style.css';
import dagre from 'dagre';
import api from '../../services/api';
import type { FileResult, WordCloudData, ImportanceComplexityData, GlossaryData, KeywordInsights, TimelineData, TimelineTimeUnit } from './UploadInfer';
import styles from './WorkspacePanel.module.scss';

marked.setOptions({ breaks: false, gfm: true });

interface Props {
    step2Visible?: boolean;
    step2Minimized?: boolean;
    fileResult: FileResult | null;
    fileLoading: boolean;
    activeFileId: number | null;
    onResultUpdate?: (patch: Partial<Pick<FileResult, 'summary' | 'keywords'>>) => void;
}

// TABS labels are now driven by i18n — see tabLabels() inside WorkspacePanel
const TAB_IDS = ['summary', 'keywords', 'assessment', 'shortAnswer', 'trueFalse', 'timestampedSummary', 'keywordInsights'] as const;
type TabId = typeof TAB_IDS[number];

// Exposed to the parent so the guided tour can drive which tab is showing
// without lifting activeTab into Redux just for that one use.
export interface WorkspacePanelHandle {
    setTab: (id: TabId) => void;
}


interface FaqItem {
    question: string;
    options: Record<string, string>;
    correct_answer: string;
    explanation: string;
}

function parseFaq(raw: string): FaqItem[] {
    if (!raw || raw === '[]') return [];
    try {
        // eslint-disable-next-line no-eval
        const result = eval(raw);
        if (Array.isArray(result)) return result as FaqItem[];
        return [];
    } catch {
        // Fallback: try direct JSON parse (if API returns valid JSON)
        try { return JSON.parse(raw) as FaqItem[]; } catch { }
        return [];
    }
}

// ── Permissive parser for the newer python-repr-style fields (short_answer,
//    true_false, timestamped_summary). Handles True/False/None literals that
//    plain eval() can't, and tries strict JSON first since that's cheaper
//    and safer whenever the backend does send valid JSON. ──
function parsePyList<T>(raw: string): T[] {
    if (!raw || raw === '[]') return [];
    try {
        const result = JSON.parse(raw);
        if (Array.isArray(result)) return result as T[];
    } catch { /* not valid JSON — fall through to python-ish eval */ }
    try {
        const normalized = raw
            .replace(/\bTrue\b/g, 'true')
            .replace(/\bFalse\b/g, 'false')
            .replace(/\bNone\b/g, 'null');
        // eslint-disable-next-line no-eval
        const result = eval('(' + normalized + ')');
        if (Array.isArray(result)) return result as T[];
    } catch { /* give up */ }
    return [];
}

interface ShortAnswerItem { question: string; answer: string; }
interface TrueFalseItem { statement: string; is_true: boolean; explanation: string; }
interface TimestampSegment { start_time: string; end_time: string; summary: string; }

const CHIP_COLORS = [
    'linear-gradient(135deg, #667eea 0%, #764ba2 100%)',
    'linear-gradient(135deg, #f093fb 0%, #f5576c 100%)',
    'linear-gradient(135deg, #4facfe 0%, #00f2fe 100%)',
    'linear-gradient(135deg, #43e97b 0%, #38f9d7 100%)',
    'linear-gradient(135deg, #fa709a 0%, #fee140 100%)',
    'linear-gradient(135deg, #30cfd0 0%, #330867 100%)',
    'linear-gradient(135deg, #a8edea 0%, #fed6e3 100%)',
    'linear-gradient(135deg, #ff9a56 0%, #ff6a88 100%)',
    'linear-gradient(135deg, #ffecd2 0%, #fcb69f 100%)',
    'linear-gradient(135deg, #a1c4fd 0%, #c2e9fb 100%)',
];

function seededColorIndex(str: string): number {
    let hash = 0;
    for (let i = 0; i < str.length; i++) hash = (hash * 31 + str.charCodeAt(i)) >>> 0;
    return hash % CHIP_COLORS.length;
}

// ── Copy / Download helpers ───────────────────────────────
function copyText(text: string) {
    if (navigator.clipboard) { navigator.clipboard.writeText(text).catch(() => fallbackCopy(text)); }
    else fallbackCopy(text);
}
function fallbackCopy(text: string) {
    const ta = document.createElement('textarea');
    ta.value = text; ta.style.cssText = 'position:fixed;top:0;left:0;opacity:0';
    document.body.appendChild(ta); ta.select();
    try { document.execCommand('copy'); } catch { }
    document.body.removeChild(ta);
}
function downloadFile(content: string, filename: string, mime: string) {
    const blob = new Blob([content], { type: mime });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url; a.download = filename;
    document.body.appendChild(a); a.click();
    document.body.removeChild(a); URL.revokeObjectURL(url);
}
function wrapHtml(body: string, title: string): string {
    return `<!DOCTYPE html><html lang="en"><head><meta charset="UTF-8"><title>${title}</title><style>body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;max-width:900px;margin:40px auto;padding:20px;background:#f8f9fa;color:#333}</style></head><body>${body}</body></html>`;
}

// ── Format helpers ────────────────────────────────────────
type Formatted = { content: string; filename: string; mime: string };

function formatSummary(summary: string, fmt: string, base = 'summary'): Formatted {
    const ts = new Date().toISOString().slice(0, 10);
    if (fmt === 'html') return { content: wrapHtml(marked.parse(summary) as string, 'Summary'), filename: `${base}_summary_${ts}.html`, mime: 'text/html' };
    if (fmt === 'json') return { content: JSON.stringify({ type: 'summary', content: summary, timestamp: new Date().toISOString() }, null, 2), filename: `${base}_summary_${ts}.json`, mime: 'application/json' };
    if (fmt === 'md') return { content: summary, filename: `${base}_summary_${ts}.md`, mime: 'text/markdown' };
    return { content: summary, filename: `${base}_summary_${ts}.txt`, mime: 'text/plain' };
}

function formatKeywords(keywords: string[], fmt: string, base = 'keywords'): Formatted {
    const ts = new Date().toISOString().slice(0, 10);
    if (fmt === 'html') return { content: wrapHtml(`<div style="display:flex;flex-wrap:wrap;gap:10px">${keywords.map(k => `<span style="padding:6px 14px;border-radius:20px;background:linear-gradient(135deg,#8b5cf6,#a78bfa);color:#fff;font-weight:600">${k}</span>`).join('')}</div>`, 'Keywords'), filename: `${base}_keywords_${ts}.html`, mime: 'text/html' };
    if (fmt === 'json') return { content: JSON.stringify({ type: 'keywords', keywords, timestamp: new Date().toISOString() }, null, 2), filename: `${base}_keywords_${ts}.json`, mime: 'application/json' };
    if (fmt === 'md') return { content: '# Keywords\n\n' + keywords.map(k => `- ${k}`).join('\n'), filename: `${base}_keywords_${ts}.md`, mime: 'text/markdown' };
    return { content: keywords.join('\n'), filename: `${base}_keywords_${ts}.txt`, mime: 'text/plain' };
}

function formatAssessment(faqRaw: string, fmt: string, base = 'assessment'): Formatted {
    const ts = new Date().toISOString().slice(0, 10);
    const items = parseFaq(faqRaw);
    if (fmt === 'json') return { content: JSON.stringify(items, null, 2), filename: `${base}_assessment_${ts}.json`, mime: 'application/json' };
    if (fmt === 'html') {
        const body = items.map((q, i) =>
            `<div style="background:#fff;border-radius:12px;padding:24px;margin-bottom:20px;box-shadow:0 2px 8px rgba(0,0,0,.08);border-left:4px solid #8b5cf6">
             <div style="font-weight:700;font-size:16px;margin-bottom:14px"><span style="background:linear-gradient(135deg,#8b5cf6,#a78bfa);color:#fff;padding:3px 10px;border-radius:6px;margin-right:10px;font-size:13px">Q${i + 1}</span>${q.question}</div>
             ${Object.entries(q.options).map(([k, v]) => `<div style="padding:10px 14px;margin:6px 0;border-radius:8px;border:1px solid ${k === q.correct_answer ? '#10b981' : '#e5e7eb'};background:${k === q.correct_answer ? 'rgba(16,185,129,.08)' : '#f9fafb'}"><b style="color:${k === q.correct_answer ? '#10b981' : '#8b5cf6'}">${k}.</b> ${v}${k === q.correct_answer ? ' <span style="background:#10b981;color:#fff;padding:1px 7px;border-radius:4px;font-size:11px">✓</span>' : ''}</div>`).join('')}
             <div style="margin-top:14px;padding:12px 16px;background:rgba(139,92,246,.06);border-left:3px solid #8b5cf6;border-radius:0 8px 8px 0;font-size:13px"><b style="color:#8b5cf6;font-size:11px;text-transform:uppercase">Explanation</b><br/>${q.explanation}</div>
             </div>`
        ).join('');
        return { content: wrapHtml(body, 'Assessment Questions'), filename: `${base}_assessment_${ts}.html`, mime: 'text/html' };
    }
    // txt
    const txt = items.map((q, i) =>
        `Q${i + 1}. ${q.question}\n` +
        Object.entries(q.options).map(([k, v]) => `  ${k}. ${v}${k === q.correct_answer ? ' ✓' : ''}`).join('\n') +
        `\n\nAnswer: ${q.correct_answer}\nExplanation: ${q.explanation}`
    ).join('\n\n' + '─'.repeat(60) + '\n\n');
    return { content: txt, filename: `${base}_assessment_${ts}.txt`, mime: 'text/plain' };
}

function formatShortAnswer(raw: string, fmt: string, base = 'short_answer'): Formatted {
    const ts = new Date().toISOString().slice(0, 10);
    const items = parsePyList<ShortAnswerItem>(raw);
    if (fmt === 'json') return { content: JSON.stringify(items, null, 2), filename: `${base}_${ts}.json`, mime: 'application/json' };
    if (fmt === 'html') {
        const body = items.map((item, i) =>
            `<div style="background:#fff;border-radius:12px;padding:20px;margin-bottom:16px;box-shadow:0 2px 8px rgba(0,0,0,.08);border-left:4px solid #4facfe">
             <div style="font-weight:700;font-size:15px;margin-bottom:10px"><span style="background:#4facfe;color:#fff;padding:3px 10px;border-radius:6px;margin-right:10px;font-size:13px">Q${i + 1}</span>${item.question}</div>
             <div style="padding:10px 14px;background:rgba(79,172,254,.08);border-radius:8px;font-size:13px"><b style="color:#4facfe">Answer:</b> ${item.answer}</div>
             </div>`
        ).join('');
        return { content: wrapHtml(body, 'Short Answer Questions'), filename: `${base}_${ts}.html`, mime: 'text/html' };
    }
    const txt = items.map((item, i) => `Q${i + 1}. ${item.question}\nA: ${item.answer}`).join('\n\n' + '─'.repeat(60) + '\n\n');
    return { content: txt, filename: `${base}_${ts}.txt`, mime: 'text/plain' };
}

function formatTrueFalse(raw: string, fmt: string, base = 'true_false'): Formatted {
    const ts = new Date().toISOString().slice(0, 10);
    const items = parsePyList<TrueFalseItem>(raw);
    if (fmt === 'json') return { content: JSON.stringify(items, null, 2), filename: `${base}_${ts}.json`, mime: 'application/json' };
    if (fmt === 'html') {
        const body = items.map((item, i) =>
            `<div style="background:#fff;border-radius:12px;padding:20px;margin-bottom:16px;box-shadow:0 2px 8px rgba(0,0,0,.08);border-left:4px solid #43e97b">
             <div style="font-weight:700;font-size:15px;margin-bottom:10px"><span style="background:#43e97b;color:#fff;padding:3px 10px;border-radius:6px;margin-right:10px;font-size:13px">${i + 1}</span>${item.statement}</div>
             <div style="padding:10px 14px;background:rgba(67,233,123,.08);border-radius:8px;font-size:13px"><b style="color:#43e97b">Answer:</b> ${item.is_true ? 'True' : 'False'}</div>
             <div style="margin-top:10px;font-size:13px;color:#666"><b>Explanation:</b> ${item.explanation}</div>
             </div>`
        ).join('');
        return { content: wrapHtml(body, 'True / False Questions'), filename: `${base}_${ts}.html`, mime: 'text/html' };
    }
    const txt = items.map((item, i) => `${i + 1}. ${item.statement}\nAnswer: ${item.is_true ? 'True' : 'False'}\nExplanation: ${item.explanation}`).join('\n\n' + '─'.repeat(60) + '\n\n');
    return { content: txt, filename: `${base}_${ts}.txt`, mime: 'text/plain' };
}

function formatTimestampedSummary(raw: string, fmt: string, base = 'timestamped_summary'): Formatted {
    const ts = new Date().toISOString().slice(0, 10);
    const items = parsePyList<TimestampSegment>(raw);
    if (fmt === 'json') return { content: JSON.stringify(items, null, 2), filename: `${base}_${ts}.json`, mime: 'application/json' };
    if (fmt === 'md') return { content: items.map(s => `### ${s.start_time} – ${s.end_time}\n\n${s.summary}`).join('\n\n'), filename: `${base}_${ts}.md`, mime: 'text/markdown' };
    const txt = items.map(s => `[${s.start_time} – ${s.end_time}]\n${s.summary}`).join('\n\n');
    return { content: txt, filename: `${base}_${ts}.txt`, mime: 'text/plain' };
}


const SUMMARY_FMTS = [{ k: 'txt', l: 'Text' }, { k: 'md', l: 'Markdown' }, { k: 'html', l: 'HTML' }, { k: 'json', l: 'JSON' }];
const KEYWORDS_FMTS = [{ k: 'txt', l: 'Text' }, { k: 'md', l: 'Markdown' }, { k: 'html', l: 'HTML' }, { k: 'json', l: 'JSON' }];
const ASSESSMENT_FMTS = [{ k: 'txt', l: 'Plain Text' }, { k: 'html', l: 'HTML' }, { k: 'json', l: 'JSON' }];
const SHORT_ANSWER_FMTS = [{ k: 'txt', l: 'Plain Text' }, { k: 'html', l: 'HTML' }, { k: 'json', l: 'JSON' }];
const TRUE_FALSE_FMTS = [{ k: 'txt', l: 'Plain Text' }, { k: 'html', l: 'HTML' }, { k: 'json', l: 'JSON' }];
const TIMESTAMPED_SUMMARY_FMTS = [{ k: 'txt', l: 'Text' }, { k: 'md', l: 'Markdown' }, { k: 'json', l: 'JSON' }];

// ── ActionBtn ─────────────────────────────────────────────
const ActionBtn: React.FC<{ title: string; onClick: () => void; active?: boolean; children: React.ReactNode }> = ({ title, onClick, active, children }) => (
    <button className={`${styles.actionBtn} ${active ? styles.actionBtnActive : ''}`} title={title} onClick={e => { e.stopPropagation(); onClick(); }}>
        {children}
    </button>
);

// ── FormatIcon ────────────────────────────────────────────
// Small colored glyph rendered next to each format option in the
// Copy / Download dropdowns. Each format has a recognisable hue.
const FORMAT_ICONS: Record<string, { color: string; bg: string; node: React.ReactNode }> = {
    txt: {
        color: '#64748b',
        bg: 'rgba(100, 116, 139, 0.12)',
        node: (
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
                strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                <path d="M3 2.5h6.5L13 6v7.5A1 1 0 0112 14.5H3a1 1 0 01-1-1v-10a1 1 0 011-1z" />
                <path d="M9 2.5V6h4" />
                <path d="M5 9h6M5 11.5h4" />
            </svg>
        ),
    },
    md: {
        color: '#3b82f6',
        bg: 'rgba(59, 130, 246, 0.14)',
        node: (
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
                strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                <rect x="1.5" y="3.5" width="13" height="9" rx="1.5" />
                <path d="M4 10.5V6l1.75 2.5L7.5 6v4.5" />
                <path d="M10.25 6v4.5M8.75 9l1.5 1.5L11.75 9" />
            </svg>
        ),
    },
    html: {
        color: '#e34f26',
        bg: 'rgba(227, 79, 38, 0.14)',
        node: (
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
                strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                <path d="M2.5 2l1 12 4.5 1.3L12.5 14l1-12z" />
                <path d="M5 5.5h6L10.6 11l-2.6.8L5.4 11l-.15-2" />
            </svg>
        ),
    },
    json: {
        color: '#f59e0b',
        bg: 'rgba(245, 158, 11, 0.14)',
        node: (
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
                strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                <path d="M6 2.5C4.5 2.5 4 3.5 4 5v1.5C4 7.5 3 8 2 8c1 0 2 .5 2 1.5V11c0 1.5.5 2.5 2 2.5" />
                <path d="M10 2.5c1.5 0 2 1 2 2.5v1.5C12 7.5 13 8 14 8c-1 0-2 .5-2 1.5V11c0 1.5-.5 2.5-2 2.5" />
            </svg>
        ),
    },
};

const FormatIcon: React.FC<{ fmt: string }> = ({ fmt }) => {
    const def = FORMAT_ICONS[fmt];
    if (!def) return null;
    return (
        <span
            className={styles.fmtIcon}
            style={{ color: def.color, background: def.bg }}
            aria-hidden="true"
        >
            {def.node}
        </span>
    );
};

// ── Dropdown ──────────────────────────────────────────────
const Dropdown: React.FC<{
    icon: React.ReactNode; title: string; label: string;
    fmts: { k: string; l: string }[];
    onSelect: (k: string) => void; active?: boolean;
}> = ({ icon, title, label, fmts, onSelect, active }) => {
    const [open, setOpen] = useState(false);
    const ref = useRef<HTMLDivElement>(null);
    useEffect(() => {
        const h = (e: MouseEvent) => { if (ref.current && !ref.current.contains(e.target as Node)) setOpen(false); };
        document.addEventListener('mousedown', h); return () => document.removeEventListener('mousedown', h);
    }, []);
    return (
        <div className={styles.dropdownWrap} ref={ref}>
            <ActionBtn title={title} onClick={() => setOpen(o => !o)} active={open || active}>{icon}</ActionBtn>
            {open && (
                <div className={styles.dropdown}>
                    <div className={styles.dropdownLabel}>{label}</div>
                    {fmts.map(f => (
                        <button
                            key={f.k}
                            className={styles.dropdownItem}
                            onClick={() => { onSelect(f.k); setOpen(false); }}
                        >
                            <FormatIcon fmt={f.k} />
                            <span className={styles.dropdownItemLabel}>{f.l}</span>
                        </button>
                    ))}
                </div>
            )}
        </div>
    );
};

// ── Icons ─────────────────────────────────────────────────
const IcoEdit = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <path d="M7 2.5H3.5A1.5 1.5 0 002 4v8.5A1.5 1.5 0 003.5 14H12a1.5 1.5 0 001.5-1.5V9" />
        <path d="M11.5 1.5a1.414 1.414 0 012 2L8 9l-2.5.5.5-2.5 5.5-5.5z" />
    </svg>
);
const IcoCopy = ({ success }: { success: boolean }) => success ? (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" className={styles.successIcon}><path d="M3 8l3.5 3.5L13 4" /></svg>
) : (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <rect x="5" y="5" width="8" height="9" rx="1.5" />
        <path d="M10 5V4a1 1 0 00-1-1H4a1.5 1.5 0 00-1.5 1.5V11a1 1 0 001 1h1" />
    </svg>
);
const IcoDownload = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <path d="M8 2v8M5 7l3 3 3-3" />
        <path d="M2 12v1.5A1.5 1.5 0 003.5 15h9a1.5 1.5 0 001.5-1.5V12" />
    </svg>
);

// ── TabToolbar: edit + copy + download ───────────────────
const TabToolbar: React.FC<{
    onEdit?: () => void;
    fmts: { k: string; l: string }[];
    onCopy: (k: string) => void;
    onDownload: (k: string) => void;
    copied: boolean;
}> = ({ onEdit, fmts, onCopy, onDownload, copied }) => {
    const { t } = useTranslation();
    return (
        <div className={styles.tabToolbar}>
            {onEdit && <ActionBtn title={t('uploadInfer.workspacePanel.edit')} onClick={onEdit}><IcoEdit /></ActionBtn>}
            <Dropdown icon={<IcoCopy success={copied} />} title={t('uploadInfer.workspacePanel.copy')} label={t('uploadInfer.workspacePanel.copyAs')} fmts={fmts} onSelect={onCopy} active={copied} />
            <Dropdown icon={<IcoDownload />} title={t('uploadInfer.workspacePanel.download')} label={t('uploadInfer.workspacePanel.downloadAs')} fmts={fmts} onSelect={onDownload} />
        </div>
    );
};

// ── Stable keyword chip ───────────────────────────────────
const KeywordChip: React.FC<{ kw: string; isNew: boolean }> = ({ kw, isNew }) => (
    <span className={`${styles.keywordPill} ${isNew ? styles.keywordPillNew : ''}`}
        style={{ background: CHIP_COLORS[seededColorIndex(kw)] }}>
        {kw}
    </span>
);

const ChipGrid: React.FC<{ kws: string[]; prevSet?: Set<string> }> = ({ kws, prevSet }) => {
    const { t } = useTranslation();
    return kws.length > 0 ? (
        <div className={styles.keywordGrid}>
            {kws.map(kw => <KeywordChip key={kw} kw={kw} isNew={prevSet ? !prevSet.has(kw) : false} />)}
        </div>
    ) : <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noKeywords')}</div>;
};

// ── Tab: Summary ──────────────────────────────────────────
const TabSummary: React.FC<{ summary: string; fileId: number; onSaved: (s: string) => void }> = ({ summary, fileId, onSaved }) => {
    const { t } = useTranslation();
    const [editing, setEditing] = useState(false);
    const [draft, setDraft] = useState(summary);
    const [saving, setSaving] = useState(false);
    const [copied, setCopied] = useState(false);

    const html = useMemo(() => marked.parse(draft) as string, [draft]);
    const viewHtml = useMemo(() => marked.parse(summary) as string, [summary]);

    const handleSave = async () => {
        setSaving(true);
        try { await api.post('/files/update', { fileID: String(fileId), summary: draft }); onSaved(draft); setEditing(false); }
        finally { setSaving(false); }
    };
    const handleCopy = (fmt: string) => { copyText(formatSummary(summary, fmt).content); setCopied(true); setTimeout(() => setCopied(false), 1500); };
    const handleDownload = (fmt: string) => { const f = formatSummary(summary, fmt); downloadFile(f.content, f.filename, f.mime); };

    if (!summary && !editing) return <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noSummary')}</div>;

    return (
        <div className={`${styles.tabContent} ${editing ? styles.editMode : ''}`}>
            {!editing ? (
                <>
                    <TabToolbar onEdit={() => { setDraft(summary); setEditing(true); }} fmts={SUMMARY_FMTS} onCopy={handleCopy} onDownload={handleDownload} copied={copied} />
                    <div className={styles.summaryMd} dangerouslySetInnerHTML={{ __html: viewHtml }} />
                </>
            ) : (
                <div className={styles.editLayout}>
                    <div className={styles.editCol}>
                        <div className={styles.editColHeader}><span className={styles.editColLabel}>{t('uploadInfer.workspacePanel.editColPreview')}</span><span className={styles.editColTag}>{t('uploadInfer.workspacePanel.markdownRenderer')}</span></div>
                        <div className={styles.editPreview}><div className={styles.summaryMd} dangerouslySetInnerHTML={{ __html: html }} /></div>
                    </div>
                    <div className={styles.editCol}>
                        <div className={styles.editColHeader}><span className={styles.editColLabel}>{t('uploadInfer.workspacePanel.editColEdit')}</span><span className={styles.editColTag}>{t('uploadInfer.workspacePanel.markdownTag')}</span></div>
                        <textarea className={styles.editTextarea} value={draft} onChange={e => setDraft(e.target.value)} spellCheck={false} />
                    </div>
                    <div className={styles.editFooter}>
                        <button className={styles.cancelBtn} onClick={() => setEditing(false)} disabled={saving}>{t('uploadInfer.workspacePanel.cancel')}</button>
                        <button className={styles.saveBtn} onClick={handleSave} disabled={saving}>
                            {saving ? <><span className={styles.inlineSpinner} />{t('uploadInfer.workspacePanel.saving')}</> : t('uploadInfer.workspacePanel.save')}
                        </button>
                    </div>
                </div>
            )}
        </div>
    );
};

// ── Tab: Keywords ─────────────────────────────────────────
const TabKeywords: React.FC<{ keywords: string[]; fileId: number; onSaved: (kws: string[]) => void }> = ({ keywords, fileId, onSaved }) => {
    const { t } = useTranslation();
    const [editing, setEditing] = useState(false);
    const [draft, setDraft] = useState(keywords.join('\n'));
    const [saving, setSaving] = useState(false);
    const [copied, setCopied] = useState(false);

    // prevSetSnapshot holds the keyword set from the PREVIOUS render of ChipGrid
    // so only truly new chips get the pop animation
    const prevSetSnapshot = useRef<Set<string>>(new Set(keywords));

    const draftKeywords = useMemo(() => draft.split('\n').map(k => k.trim()).filter(Boolean), [draft]);

    // After each render, schedule an update of the snapshot so the *next*
    // render can compare against what's currently visible
    useEffect(() => {
        const id = setTimeout(() => { prevSetSnapshot.current = new Set(draftKeywords); }, 0);
        return () => clearTimeout(id);
    }, [draftKeywords]);

    const handleSave = async () => {
        setSaving(true);
        try { await api.post('/files/update', { fileID: String(fileId), keywords: draftKeywords }); onSaved(draftKeywords); setEditing(false); }
        finally { setSaving(false); }
    };
    const handleCopy = (fmt: string) => { copyText(formatKeywords(keywords, fmt).content); setCopied(true); setTimeout(() => setCopied(false), 1500); };
    const handleDownload = (fmt: string) => { const f = formatKeywords(keywords, fmt); downloadFile(f.content, f.filename, f.mime); };

    if (!keywords.length && !editing) return <div className={styles.tabEmpty}>No keywords available.</div>;

    return (
        <div className={`${styles.tabContent} ${editing ? styles.editMode : ''}`}>
            {!editing ? (
                <>
                    <TabToolbar onEdit={() => { setDraft(keywords.join('\n')); prevSetSnapshot.current = new Set(keywords); setEditing(true); }} fmts={KEYWORDS_FMTS} onCopy={handleCopy} onDownload={handleDownload} copied={copied} />
                    <ChipGrid kws={keywords} />
                </>
            ) : (
                <div className={styles.editLayout}>
                    <div className={styles.editCol}>
                        <div className={styles.editColHeader}><span className={styles.editColLabel}>{t('uploadInfer.workspacePanel.editColPreview')}</span><span className={styles.editColTag}>{t('uploadInfer.workspacePanel.chipView')}</span></div>
                        <div className={styles.editPreview}>
                            <ChipGrid kws={draftKeywords} prevSet={prevSetSnapshot.current} />
                        </div>
                    </div>
                    <div className={styles.editCol}>
                        <div className={styles.editColHeader}><span className={styles.editColLabel}>{t('uploadInfer.workspacePanel.editColEdit')}</span><span className={styles.editColTag}>{t('uploadInfer.workspacePanel.onePerLine')}</span></div>
                        <textarea className={styles.editTextarea} value={draft} onChange={e => setDraft(e.target.value)} spellCheck={false} placeholder={t('uploadInfer.workspacePanel.keywordPlaceholder')} />
                    </div>
                    <div className={styles.editFooter}>
                        <button className={styles.cancelBtn} onClick={() => setEditing(false)} disabled={saving}>{t('uploadInfer.workspacePanel.cancel')}</button>
                        <button className={styles.saveBtn} onClick={handleSave} disabled={saving}>
                            {saving ? <><span className={styles.inlineSpinner} />{t('uploadInfer.workspacePanel.saving')}</> : t('uploadInfer.workspacePanel.save')}
                        </button>
                    </div>
                </div>
            )}
        </div>
    );
};

// ── Tab: Assessment ───────────────────────────────────────
type AssessMode = 'mcq' | 'all';

const TabAssessment: React.FC<{ faq: string }> = ({ faq }) => {
    const { t } = useTranslation();
    const items = useMemo(() => parseFaq(faq), [faq]);

    // Always default to MCQ; reset when faq changes (new file)
    const [mode, setMode] = useState<AssessMode>('mcq');
    const [current, setCurrent] = useState(0);
    const [selected, setSelected] = useState<string | null>(null);
    const [revealed, setRevealed] = useState(false);
    const [complete, setComplete] = useState(false);
    const [copied, setCopied] = useState(false);

    const prevFaq = useRef(faq);
    useEffect(() => {
        if (faq !== prevFaq.current) {
            prevFaq.current = faq;
            setMode('mcq');
            setCurrent(0); setSelected(null); setRevealed(false); setComplete(false);
        }
    }, [faq]);

    const handleCopy = (fmt: string) => { copyText(formatAssessment(faq, fmt).content); setCopied(true); setTimeout(() => setCopied(false), 1500); };
    const handleDownload = (fmt: string) => { const f = formatAssessment(faq, fmt); downloadFile(f.content, f.filename, f.mime); };

    if (!items.length) return <div className={`${styles.tabContent} ${styles.tabEmpty}`}>{t('uploadInfer.workspacePanel.noQuestions')}</div>;

    // ── MCQ helpers ───────────────────────────────────────
    const q = items[current];
    const isLast = current === items.length - 1;
    const progress = Math.round(((current + (complete ? 1 : 0)) / items.length) * 100);

    const handleSelect = (key: string) => { if (revealed) return; setSelected(key); setRevealed(true); };
    const handleNext = () => { if (isLast) setComplete(true); else { setCurrent(c => c + 1); setSelected(null); setRevealed(false); } };
    const handleRestart = () => { setCurrent(0); setSelected(null); setRevealed(false); setComplete(false); };
    const getOptClass = (key: string) => {
        if (!revealed) return selected === key ? styles.faqOptSelected : '';
        if (key === q.correct_answer && selected === key) return styles.faqOptCorrect;
        if (key === q.correct_answer) return styles.faqOptCorrectAlt;
        if (key === selected) return styles.faqOptWrong;
        return '';
    };

    return (
        <div className={styles.tabContent}>
            {/* Toolbar row: mode tabs left, copy/download right */}
            <div className={styles.assessHeader}>
                <div className={styles.assessModeTabs}>
                    <button
                        className={`${styles.assessModeTab} ${mode === 'mcq' ? styles.assessModeTabActive : ''}`}
                        onClick={() => setMode('mcq')}
                    >
                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                            <circle cx="8" cy="8" r="6" />
                            <path d="M6 8.5l1.5 1.5L10.5 6" />
                        </svg>
                        MCQ
                    </button>
                    <button
                        className={`${styles.assessModeTab} ${mode === 'all' ? styles.assessModeTabActive : ''}`}
                        onClick={() => setMode('all')}
                    >
                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                            <path d="M2 4h12M2 8h12M2 12h8" />
                        </svg>
                        View All
                    </button>
                </div>
                <TabToolbar fmts={ASSESSMENT_FMTS} onCopy={handleCopy} onDownload={handleDownload} copied={copied} />
            </div>

            {/* ── MCQ mode ── */}
            {mode === 'mcq' && (
                complete ? (
                    <div className={styles.assessComplete}>
                        <div className={styles.assessCompleteIcon}>
                            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                <path d="M9 12.75L11.25 15 15 9.75M21 12a9 9 0 11-18 0 9 9 0 0118 0z" />
                            </svg>
                        </div>
                        <div className={styles.assessCompleteTitle}>{t('uploadInfer.workspacePanel.assessCompleteTitle')}</div>
                        <div className={styles.assessCompleteDesc}>{t('uploadInfer.workspacePanel.assessCompleteDesc', { count: items.length })}</div>
                        <button className={styles.assessRestartBtn} onClick={handleRestart}>{t('uploadInfer.workspacePanel.restart')}</button>
                    </div>
                ) : (
                    <>
                        <div className={styles.assessProgress}>
                            <div className={styles.assessProgressTrack}>
                                <div className={styles.assessProgressFill} style={{ width: `${progress}%` }} />
                            </div>
                            <span className={styles.assessProgressLabel}>{current + 1} / {items.length}</span>
                        </div>
                        <div className={styles.faqCard}>
                            <div className={styles.faqQ}>
                                <span className={styles.faqNum}>Q{current + 1}</span>
                                {q.question}
                            </div>
                            <div className={styles.faqOptions}>
                                {Object.entries(q.options).map(([key, val]) => (
                                    <button key={key} className={`${styles.faqOpt} ${getOptClass(key)}`} onClick={() => handleSelect(key)} disabled={revealed}>
                                        <span className={styles.faqOptKey}>{key}</span>
                                        <span className={styles.faqOptVal}>{val}</span>
                                        {revealed && key === q.correct_answer && <svg className={styles.faqCheckIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round"><path d="M3 8l3.5 3.5L13 4" /></svg>}
                                        {revealed && key === selected && key !== q.correct_answer && <svg className={styles.faqXIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round"><path d="M4 4l8 8M12 4l-8 8" /></svg>}
                                    </button>
                                ))}
                            </div>
                            {revealed && (
                                <div className={styles.faqExplain}>
                                    <div className={styles.faqExplainLabel}>
                                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round"><circle cx="8" cy="8" r="6" /><path d="M8 7v4M8 5.5v.5" /></svg>
                                        Explanation
                                    </div>
                                    <p className={styles.faqExplainText}>{q.explanation}</p>
                                </div>
                            )}
                        </div>
                        {revealed && (
                            <button className={`${styles.assessNextBtn} ${isLast ? styles.assessDoneBtn : ''}`} onClick={handleNext}>
                                {isLast ? t('uploadInfer.workspacePanel.finish') : t('uploadInfer.workspacePanel.next')}
                                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round">
                                    {isLast ? <path d="M3 8l3.5 3.5L13 4" /> : <path d="M3 8h10M9 4l4 4-4 4" />}
                                </svg>
                            </button>
                        )}
                    </>
                )
            )}

            {/* ── View All mode ── */}
            {mode === 'all' && (
                <div className={styles.viewAllList}>
                    {items.map((item, idx) => (
                        <div key={idx} className={styles.viewAllCard}>
                            {/* Question */}
                            <div className={styles.viewAllQ}>
                                <span className={styles.faqNum}>Q{idx + 1}</span>
                                {item.question}
                            </div>

                            {/* Options */}
                            <div className={styles.faqOptions}>
                                {Object.entries(item.options).map(([key, val]) => (
                                    <div
                                        key={key}
                                        className={`${styles.faqOpt} ${key === item.correct_answer ? styles.faqOptCorrectAlt : styles.viewAllOptNeutral}`}
                                    >
                                        <span className={`${styles.faqOptKey} ${key === item.correct_answer ? styles.faqOptKeyCorrect : ''}`}>{key}</span>
                                        <span className={styles.faqOptVal}>{val}</span>
                                        {key === item.correct_answer && (
                                            <svg className={styles.faqCheckIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round">
                                                <path d="M3 8l3.5 3.5L13 4" />
                                            </svg>
                                        )}
                                    </div>
                                ))}
                            </div>

                            {/* Explanation */}
                            <div className={styles.faqExplain}>
                                <div className={styles.faqExplainLabel}>
                                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                                        <circle cx="8" cy="8" r="6" /><path d="M8 7v4M8 5.5v.5" />
                                    </svg>
                                    Explanation
                                </div>
                                <p className={styles.faqExplainText}>{item.explanation}</p>
                            </div>
                        </div>
                    ))}
                </div>
            )}
        </div>
    );
};

// ── Tab: Short Answer ──────────────────────────────────────
const TabShortAnswer: React.FC<{ raw: string }> = ({ raw }) => {
    const { t } = useTranslation();
    const items = useMemo(() => parsePyList<ShortAnswerItem>(raw), [raw]);
    const [revealedIds, setRevealedIds] = useState<Set<number>>(new Set());
    const [copied, setCopied] = useState(false);

    const prevRaw = useRef(raw);
    useEffect(() => { if (raw !== prevRaw.current) { prevRaw.current = raw; setRevealedIds(new Set()); } }, [raw]);

    const toggleReveal = (idx: number) => setRevealedIds(prev => {
        const next = new Set(prev);
        if (next.has(idx)) next.delete(idx); else next.add(idx);
        return next;
    });
    const allRevealed = items.length > 0 && revealedIds.size === items.length;
    const toggleAll = () => setRevealedIds(allRevealed ? new Set() : new Set(items.map((_, i) => i)));

    const handleCopy = (fmt: string) => { copyText(formatShortAnswer(raw, fmt).content); setCopied(true); setTimeout(() => setCopied(false), 1500); };
    const handleDownload = (fmt: string) => { const f = formatShortAnswer(raw, fmt); downloadFile(f.content, f.filename, f.mime); };

    if (!items.length) return <div className={`${styles.tabContent} ${styles.tabEmpty}`}>{t('uploadInfer.workspacePanel.noShortAnswer')}</div>;

    return (
        <div className={styles.tabContent}>
            <div className={styles.assessHeader}>
                <button className={styles.revealAllBtn} onClick={toggleAll}>
                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                        <path d="M1.5 8s2.5-4.5 6.5-4.5S14.5 8 14.5 8s-2.5 4.5-6.5 4.5S1.5 8 1.5 8z" /><circle cx="8" cy="8" r="2" />
                    </svg>
                    {allRevealed ? t('uploadInfer.workspacePanel.hideAllAnswers') : t('uploadInfer.workspacePanel.showAllAnswers')}
                </button>
                <TabToolbar fmts={SHORT_ANSWER_FMTS} onCopy={handleCopy} onDownload={handleDownload} copied={copied} />
            </div>
            <div className={styles.viewAllList}>
                {items.map((item, idx) => {
                    const revealed = revealedIds.has(idx);
                    return (
                        <div key={idx} className={styles.viewAllCard}>
                            <div className={styles.viewAllQ}>
                                <span className={styles.faqNum}>Q{idx + 1}</span>
                                {item.question}
                            </div>
                            {revealed ? (
                                <div className={styles.shortAnswerBox}>
                                    <div className={styles.shortAnswerLabel}>{t('uploadInfer.workspacePanel.answer')}</div>
                                    <p className={styles.shortAnswerText}>{item.answer}</p>
                                </div>
                            ) : (
                                <button className={styles.revealBtn} onClick={() => toggleReveal(idx)}>
                                    {t('uploadInfer.workspacePanel.revealAnswer')}
                                </button>
                            )}
                        </div>
                    );
                })}
            </div>
        </div>
    );
};

// ── Tab: True / False ────────────────────────────────────────
type TfMode = 'quiz' | 'all';

const TabTrueFalse: React.FC<{ raw: string }> = ({ raw }) => {
    const { t } = useTranslation();
    const items = useMemo(() => parsePyList<TrueFalseItem>(raw), [raw]);

    const [mode, setMode] = useState<TfMode>('quiz');
    const [current, setCurrent] = useState(0);
    const [selected, setSelected] = useState<'true' | 'false' | null>(null);
    const [revealed, setRevealed] = useState(false);
    const [complete, setComplete] = useState(false);
    const [copied, setCopied] = useState(false);

    const prevRaw = useRef(raw);
    useEffect(() => {
        if (raw !== prevRaw.current) {
            prevRaw.current = raw;
            setMode('quiz');
            setCurrent(0); setSelected(null); setRevealed(false); setComplete(false);
        }
    }, [raw]);

    const handleCopy = (fmt: string) => { copyText(formatTrueFalse(raw, fmt).content); setCopied(true); setTimeout(() => setCopied(false), 1500); };
    const handleDownload = (fmt: string) => { const f = formatTrueFalse(raw, fmt); downloadFile(f.content, f.filename, f.mime); };

    if (!items.length) return <div className={`${styles.tabContent} ${styles.tabEmpty}`}>{t('uploadInfer.workspacePanel.noTrueFalse')}</div>;

    const q = items[current];
    const correctKey: 'true' | 'false' = q.is_true ? 'true' : 'false';
    const isLast = current === items.length - 1;
    const progress = Math.round(((current + (complete ? 1 : 0)) / items.length) * 100);

    const handleSelect = (key: 'true' | 'false') => { if (revealed) return; setSelected(key); setRevealed(true); };
    const handleNext = () => { if (isLast) setComplete(true); else { setCurrent(c => c + 1); setSelected(null); setRevealed(false); } };
    const handleRestart = () => { setCurrent(0); setSelected(null); setRevealed(false); setComplete(false); };
    const getOptClass = (key: 'true' | 'false') => {
        if (!revealed) return selected === key ? styles.faqOptSelected : '';
        if (key === correctKey && selected === key) return styles.faqOptCorrect;
        if (key === correctKey) return styles.faqOptCorrectAlt;
        if (key === selected) return styles.faqOptWrong;
        return '';
    };

    return (
        <div className={styles.tabContent}>
            <div className={styles.assessHeader}>
                <div className={styles.assessModeTabs}>
                    <button className={`${styles.assessModeTab} ${mode === 'quiz' ? styles.assessModeTabActive : ''}`} onClick={() => setMode('quiz')}>
                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                            <circle cx="8" cy="8" r="6" /><path d="M6 8.5l1.5 1.5L10.5 6" />
                        </svg>
                        {t('uploadInfer.workspacePanel.quizMode')}
                    </button>
                    <button className={`${styles.assessModeTab} ${mode === 'all' ? styles.assessModeTabActive : ''}`} onClick={() => setMode('all')}>
                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                            <path d="M2 4h12M2 8h12M2 12h8" />
                        </svg>
                        {t('uploadInfer.workspacePanel.viewAll')}
                    </button>
                </div>
                <TabToolbar fmts={TRUE_FALSE_FMTS} onCopy={handleCopy} onDownload={handleDownload} copied={copied} />
            </div>

            {mode === 'quiz' && (
                complete ? (
                    <div className={styles.assessComplete}>
                        <div className={styles.assessCompleteIcon}>
                            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                <path d="M9 12.75L11.25 15 15 9.75M21 12a9 9 0 11-18 0 9 9 0 0118 0z" />
                            </svg>
                        </div>
                        <div className={styles.assessCompleteTitle}>{t('uploadInfer.workspacePanel.assessCompleteTitle')}</div>
                        <div className={styles.assessCompleteDesc}>{t('uploadInfer.workspacePanel.assessCompleteDesc', { count: items.length })}</div>
                        <button className={styles.assessRestartBtn} onClick={handleRestart}>{t('uploadInfer.workspacePanel.restart')}</button>
                    </div>
                ) : (
                    <>
                        <div className={styles.assessProgress}>
                            <div className={styles.assessProgressTrack}><div className={styles.assessProgressFill} style={{ width: `${progress}%` }} /></div>
                            <span className={styles.assessProgressLabel}>{current + 1} / {items.length}</span>
                        </div>
                        <div className={styles.faqCard}>
                            <div className={styles.faqQ}>
                                <span className={styles.faqNum}>{current + 1}</span>
                                {q.statement}
                            </div>
                            <div className={styles.tfOptions}>
                                {(['true', 'false'] as const).map(key => (
                                    <button key={key} className={`${styles.tfOpt} ${getOptClass(key)}`} onClick={() => handleSelect(key)} disabled={revealed}>
                                        <span className={styles.tfOptLabel}>{key === 'true' ? t('uploadInfer.workspacePanel.true') : t('uploadInfer.workspacePanel.false')}</span>
                                        {revealed && key === correctKey && <svg className={styles.faqCheckIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round"><path d="M3 8l3.5 3.5L13 4" /></svg>}
                                        {revealed && key === selected && key !== correctKey && <svg className={styles.faqXIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round"><path d="M4 4l8 8M12 4l-8 8" /></svg>}
                                    </button>
                                ))}
                            </div>
                            {revealed && (
                                <div className={styles.faqExplain}>
                                    <div className={styles.faqExplainLabel}>
                                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round"><circle cx="8" cy="8" r="6" /><path d="M8 7v4M8 5.5v.5" /></svg>
                                        {t('uploadInfer.workspacePanel.explanation')}
                                    </div>
                                    <p className={styles.faqExplainText}>{q.explanation}</p>
                                </div>
                            )}
                        </div>
                        {revealed && (
                            <button className={`${styles.assessNextBtn} ${isLast ? styles.assessDoneBtn : ''}`} onClick={handleNext}>
                                {isLast ? t('uploadInfer.workspacePanel.finish') : t('uploadInfer.workspacePanel.next')}
                                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round">
                                    {isLast ? <path d="M3 8l3.5 3.5L13 4" /> : <path d="M3 8h10M9 4l4 4-4 4" />}
                                </svg>
                            </button>
                        )}
                    </>
                )
            )}

            {mode === 'all' && (
                <div className={styles.viewAllList}>
                    {items.map((item, idx) => {
                        const ck: 'true' | 'false' = item.is_true ? 'true' : 'false';
                        return (
                            <div key={idx} className={styles.viewAllCard}>
                                <div className={styles.viewAllQ}>
                                    <span className={styles.faqNum}>{idx + 1}</span>
                                    {item.statement}
                                </div>
                                <div className={styles.tfOptions}>
                                    {(['true', 'false'] as const).map(key => (
                                        <div key={key} className={`${styles.tfOpt} ${key === ck ? styles.faqOptCorrectAlt : styles.viewAllOptNeutral}`}>
                                            <span className={styles.tfOptLabel}>{key === 'true' ? t('uploadInfer.workspacePanel.true') : t('uploadInfer.workspacePanel.false')}</span>
                                            {key === ck && <svg className={styles.faqCheckIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round"><path d="M3 8l3.5 3.5L13 4" /></svg>}
                                        </div>
                                    ))}
                                </div>
                                <div className={styles.faqExplain}>
                                    <div className={styles.faqExplainLabel}>
                                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round"><circle cx="8" cy="8" r="6" /><path d="M8 7v4M8 5.5v.5" /></svg>
                                        {t('uploadInfer.workspacePanel.explanation')}
                                    </div>
                                    <p className={styles.faqExplainText}>{item.explanation}</p>
                                </div>
                            </div>
                        );
                    })}
                </div>
            )}
        </div>
    );
};

// ── Tab: Timestamped Summary ─────────────────────────────────
const TabTimestampedSummary: React.FC<{ raw: string }> = ({ raw }) => {
    const { t } = useTranslation();
    const items = useMemo(() => parsePyList<TimestampSegment>(raw), [raw]);
    const [copied, setCopied] = useState(false);

    const handleCopy = (fmt: string) => { copyText(formatTimestampedSummary(raw, fmt).content); setCopied(true); setTimeout(() => setCopied(false), 1500); };
    const handleDownload = (fmt: string) => { const f = formatTimestampedSummary(raw, fmt); downloadFile(f.content, f.filename, f.mime); };

    if (!items.length) return <div className={`${styles.tabContent} ${styles.tabEmpty}`}>{t('uploadInfer.workspacePanel.noTimestampedSummary')}</div>;

    return (
        <div className={styles.tabContent}>
            <TabToolbar fmts={TIMESTAMPED_SUMMARY_FMTS} onCopy={handleCopy} onDownload={handleDownload} copied={copied} />
            <div className={styles.tsList}>
                {items.map((seg, idx) => (
                    <div key={idx} className={styles.tsRow}>
                        <div className={styles.tsRail}>
                            <div className={styles.tsDot} />
                            {idx < items.length - 1 && <div className={styles.tsLine} />}
                        </div>
                        <div className={styles.tsCard}>
                            <div className={styles.tsRange}>{seg.start_time} <span className={styles.tsArrow}>→</span> {seg.end_time}</div>
                            <p className={styles.tsText}>{seg.summary}</p>
                        </div>
                    </div>
                ))}
            </div>
        </div>
    );
};


// ── Keyword Insights: shared node-link graph (used for Knowledge Graph) ──
interface GNode { id: string; label: string; value?: number; }
interface GEdge { source: string; target: string; label?: string; }

function buildGraphNodes(nodes: GNode[], edges: GEdge[]): GNode[] {
    const norm = (s: string) => s.trim().toLowerCase();
    const byKey = new Map<string, GNode>();
    const byId = new Map<string, GNode>();
    nodes.forEach(n => {
        byKey.set(norm(n.id), n); byKey.set(norm(n.label), n); byId.set(n.id, n);
    });
    edges.forEach(e => {
        [e.source, e.target].forEach(ref => {
            const k = norm(ref);
            if (!byKey.has(k)) {
                const synth: GNode = { id: ref, label: ref, value: 1 };
                byKey.set(k, synth); byId.set(ref, synth);
            }
        });
    });
    return Array.from(byId.values());
}

// Custom node — a labeled circle sized by `value`, with an invisible
// handle on all 4 sides (each doubling as source+target) so edges can
// connect from whichever side actually faces the other node.
// Multi-word labels wrap onto multiple lines (capped at LABEL_MAX_W).
// Single unbroken words (no spaces/hyphens — common for keyword nodes)
// are NOT force-wrapped: breaking a word with no natural break point
// inside a narrow box is what produced near-vertical stacks of 1-2
// characters per line. Those get their own width instead, up to
// LABEL_HARD_MAX_W.
const LABEL_MAX_W = 150;
const LABEL_HARD_MAX_W = 220;
const LABEL_CHAR_W = 6.1; // ~px per Latin character at the label's font size
const LABEL_CJK_CHAR_W = 12.5; // ~px per CJK character (Hangul/Kanji/Kana glyphs render roughly 2x as wide as Latin at the same font-size)
const LABEL_LINE_H = 14;

// Matches Hangul syllables/Jamo, CJK Unified Ideographs, and Hiragana/Katakana.
const CJK_CHAR_RE = /[\u3040-\u30ff\u3400-\u4dbf\u4e00-\u9fff\uac00-\ud7a3\uf900-\ufaff]/;

function estimateTextWidth(label: string): number {
    let width = 8;
    for (const ch of label) width += CJK_CHAR_RE.test(ch) ? LABEL_CJK_CHAR_W : LABEL_CHAR_W;
    return width;
}

// NOTE: dagre's setNode() specifically reads `node.width` / `node.height` to
// reserve space for a node in the layout. The previous version of this
// function returned `{ w, h, labelWidth }`, which meant every node was
// silently treated by dagre as a 0×0 point — ranks were still spaced apart
// via ranksep/nodesep, but nothing accounted for how wide/tall a node with
// a long label actually is, so nodes and their labels overlapped each
// other and adjacent edges. Returning `width`/`height` (matching what the
// edge-label reservation code below already does) is the actual fix.
function estimateNodeBox(label: string, circleSize: number) {
    const hasBreakPoint = /[\s-]/.test(label);
    const rawTextWidth = estimateTextWidth(label);
    const capWidth = hasBreakPoint ? LABEL_MAX_W : LABEL_HARD_MAX_W;
    const labelWidth = Math.min(capWidth, Math.max(rawTextWidth, 44));
    const lines = hasBreakPoint ? Math.max(1, Math.ceil(rawTextWidth / LABEL_MAX_W)) : 1;
    const width = Math.max(circleSize, labelWidth, 72) + 16;
    const height = circleSize + 16 + lines * LABEL_LINE_H;
    return { width, height, labelWidth };
}

// The node's DOM box is now sized to `width`×`height` (see NodeLinkGraph,
// where these are passed onto the React Flow node), and the circle +
// label are laid out top-to-bottom in normal flow inside that box —
// instead of the label being position:absolute below a free-floating
// circle. That keeps the visual box in sync with the box dagre reserved,
// so nothing drifts as labels get longer.
const MindMapNode: React.FC<{ data: { label: string; value?: number; labelWidth?: number } }> = ({ data }) => {
    const size = 34 + Math.min(26, (data.value ?? 1) * 4);
    const hasBreakPoint = /[\s-]/.test(data.label);
    return (
        <div className={styles.rfNodeBox}>
            <div className={styles.rfNode} style={{ width: size, height: size }} title={data.label}>
                <Handle type="target" position={Position.Top} id="top-target" className={styles.rfHandle} />
                <Handle type="source" position={Position.Bottom} id="bottom-source" className={styles.rfHandle} />
                <Handle type="target" position={Position.Left} id="left-target" className={styles.rfHandle} />
                <Handle type="source" position={Position.Right} id="right-source" className={styles.rfHandle} />
            </div>
            <span
                className={styles.rfNodeLabel}
                style={{ maxWidth: data.labelWidth ?? LABEL_MAX_W, whiteSpace: hasBreakPoint ? 'normal' : 'nowrap' }}
            >
                {data.label}
            </span>
        </div>
    );
};
const RF_NODE_TYPES = { mindmap: MindMapNode };

// ── Dagre auto-layout — replaces naive circular placement, which packed
//    nodes on top of each other once a graph had more than a handful of
//    them. Dagre lays nodes out in ranked layers with guaranteed spacing,
//    so nothing overlaps regardless of graph size, AS LONG AS the nodes
//    passed to it actually carry width/height (see estimateNodeBox above —
//    that mismatch was the actual source of the overlap bug). ──
function computeDagreLayout(nodes: GNode[], edges: GEdge[]) {
    const g = new dagre.graphlib.Graph();
    g.setGraph({ rankdir: 'TB', nodesep: 72, ranksep: 150, marginx: 60, marginy: 60, acyclicer: 'greedy' });
    g.setDefaultEdgeLabel(() => ({}));

    const dims = new Map<string, { width: number; height: number; labelWidth: number }>();
    nodes.forEach(n => {
        const size = 34 + Math.min(26, (n.value ?? 1) * 4);
        const dim = estimateNodeBox(n.label, size);
        dims.set(n.id, dim);
        g.setNode(n.id, dim);
    });

    const norm = (s: string) => s.trim().toLowerCase();
    const byKey = new Map<string, string>();
    nodes.forEach(n => { byKey.set(norm(n.id), n.id); byKey.set(norm(n.label), n.id); });

    // Dedupe edges — repeated source/target pairs don't add information but
    // do add extra crowded lines between the same two nodes.
    const seenEdges = new Set<string>();
    const resolvedEdges: { source: string; target: string; label?: string }[] = [];
    edges.forEach(e => {
        const s = byKey.get(norm(e.source));
        const tg = byKey.get(norm(e.target));
        if (!s || !tg || s === tg) return;
        const key = `${s}→${tg}`;
        if (seenEdges.has(key)) return;
        seenEdges.add(key);
        // Reserve space for the edge's own label too — without this, dagre
        // has no idea the label text exists and will happily route another
        // node right where that text needs to sit.
        const label = e.label?.trim();
        g.setEdge(s, tg, label ? { width: Math.min(120, estimateTextWidth(label) + 4), height: LABEL_LINE_H + 8, labelpos: 'c' } : {});
        resolvedEdges.push({ source: s, target: tg, label: e.label });
    });

    dagre.layout(g);

    const positioned = nodes.map(n => {
        const pos = g.node(n.id);
        const dim = dims.get(n.id)!;
        // Dagre positions are centers — React Flow expects top-left.
        return { ...n, x: pos.x - dim.width / 2, y: pos.y - dim.height / 2, labelWidth: dim.labelWidth, w: dim.width, h: dim.height };
    });
    return { positioned, resolvedEdges };
}

// ── Forest layout — each disconnected tree gets its own horizontal
//    (root → children growing left-to-right) dagre layout, then the trees
//    are stacked vertically underneath one another. Labels stay perfectly
//    horizontal either way (that's just CSS text direction, unaffected by
//    which way the tree branches), but growing each tree sideways instead
//    of downward gives multi-word labels far more horizontal room before
//    they need to wrap, and keeps unrelated trees visually separated
//    instead of interleaved in one shared vertical ranking. ──
function computeForestLayout(nodes: GNode[], edges: GEdge[]) {
    const norm = (s: string) => s.trim().toLowerCase();
    const byKey = new Map<string, string>();
    nodes.forEach(n => { byKey.set(norm(n.id), n.id); byKey.set(norm(n.label), n.id); });

    // Reduce to one parent per node — same rule as treeMode above, so each
    // node belongs to exactly one tree with no crossing multi-parent edges.
    const seenEdges = new Set<string>();
    const hasParent = new Set<string>();
    const treeEdges: { source: string; target: string; label?: string }[] = [];
    edges.forEach(e => {
        const s = byKey.get(norm(e.source));
        const tg = byKey.get(norm(e.target));
        if (!s || !tg || s === tg) return;
        const key = `${s}→${tg}`;
        if (seenEdges.has(key) || hasParent.has(tg)) return;
        seenEdges.add(key);
        hasParent.add(tg);
        treeEdges.push({ source: s, target: tg, label: e.label });
    });

    // Union-find to group nodes into connected components (one per tree,
    // including singleton nodes with no edges at all).
    const uf = new Map<string, string>();
    nodes.forEach(n => uf.set(n.id, n.id));
    const find = (x: string): string => {
        let root = x;
        while (uf.get(root) !== root) root = uf.get(root)!;
        while (uf.get(x) !== root) { const next = uf.get(x)!; uf.set(x, root); x = next; }
        return root;
    };
    const union = (a: string, b: string) => { const ra = find(a), rb = find(b); if (ra !== rb) uf.set(ra, rb); };
    treeEdges.forEach(e => union(e.source, e.target));

    const groups = new Map<string, GNode[]>();
    nodes.forEach(n => {
        const root = find(n.id);
        if (!groups.has(root)) groups.set(root, []);
        groups.get(root)!.push(n);
    });

    const dims = new Map<string, { width: number; height: number; labelWidth: number }>();
    nodes.forEach(n => {
        const size = 34 + Math.min(26, (n.value ?? 1) * 4);
        dims.set(n.id, estimateNodeBox(n.label, size));
    });

    const TREE_GAP = 56;
    let yOffset = 0;
    const positioned: (GNode & { x: number; y: number; labelWidth: number; w: number; h: number })[] = [];

    for (const groupNodes of groups.values()) {
        const g = new dagre.graphlib.Graph();
        g.setGraph({ rankdir: 'LR', nodesep: 28, ranksep: 130, marginx: 20, marginy: 20, acyclicer: 'greedy' });
        g.setDefaultEdgeLabel(() => ({}));
        groupNodes.forEach(n => g.setNode(n.id, dims.get(n.id)!));

        const idsInGroup = new Set(groupNodes.map(n => n.id));
        treeEdges.forEach(e => {
            if (!idsInGroup.has(e.source) || !idsInGroup.has(e.target)) return;
            const label = e.label?.trim();
            g.setEdge(e.source, e.target, label ? { width: Math.min(120, estimateTextWidth(label) + 4), height: LABEL_LINE_H + 8, labelpos: 'c' } : {});
        });

        dagre.layout(g);

        let minX = Infinity, maxX = -Infinity, minY = Infinity, maxY = -Infinity;
        groupNodes.forEach(n => {
            const pos = g.node(n.id);
            const dim = dims.get(n.id)!;
            minX = Math.min(minX, pos.x - dim.width / 2);
            maxX = Math.max(maxX, pos.x + dim.width / 2);
            minY = Math.min(minY, pos.y - dim.height / 2);
            maxY = Math.max(maxY, pos.y + dim.height / 2);
        });
        const shiftX = -minX;
        const shiftY = yOffset - minY;

        groupNodes.forEach(n => {
            const pos = g.node(n.id);
            const dim = dims.get(n.id)!;
            positioned.push({ ...n, x: pos.x - dim.width / 2 + shiftX, y: pos.y - dim.height / 2 + shiftY, labelWidth: dim.labelWidth, w: dim.width, h: dim.height });
        });

        yOffset += (maxY - minY) + TREE_GAP;
    }

    return { positioned, resolvedEdges: treeEdges };
}

const NodeLinkGraph: React.FC<{ nodes: GNode[]; edges: GEdge[]; treeMode?: boolean }> = ({ nodes, edges, treeMode = false }) => {
    const { t } = useTranslation();
    const allNodes = useMemo(() => buildGraphNodes(nodes, edges), [nodes, edges]);

    const initial = useMemo(() => {
        const { positioned, resolvedEdges } = treeMode
            ? computeForestLayout(allNodes, edges)
            : computeDagreLayout(allNodes, edges);

        const rfNodes: RFNode[] = positioned.map(n => ({
            id: n.id,
            type: 'mindmap',
            position: { x: n.x, y: n.y },
            // Give React Flow the same box dagre reserved, so its own
            // internal bookkeeping (fitView, minimap, drag bounds) matches
            // what was actually laid out.
            width: n.w,
            height: n.h,
            data: { label: n.label, value: n.value, labelWidth: n.labelWidth },
        }));

        // Forest/tree mode lays out left→right (rankdir: 'LR'), so edges
        // should leave from the right side of a node and arrive on the
        // left of the next — using bottom/top handles there forced edges
        // to arc around the node instead of running straight, which read
        // as extra crossing/overlap. The plain knowledge-graph layout is
        // still top→bottom, so it keeps the original top/bottom handles.
        const sourceHandle = treeMode ? 'right-source' : 'bottom-source';
        const targetHandle = treeMode ? 'left-target' : 'top-target';

        const rfEdges: RFEdge[] = resolvedEdges.map((e, i) => ({
            id: `e${i}-${e.source}-${e.target}`,
            source: e.source,
            target: e.target,
            sourceHandle,
            targetHandle,
            type: 'smoothstep',
            label: e.label,
            style: { stroke: 'var(--bdr2)' },
            labelStyle: { fill: 'var(--t2)', fontSize: 10 },
            labelBgStyle: { fill: 'var(--bg1)' },
        }));
        return { rfNodes, rfEdges };
    }, [allNodes, edges, treeMode]);

    const [rfNodes, setRfNodes] = useState<RFNode[]>(initial.rfNodes);
    useEffect(() => { setRfNodes(initial.rfNodes); }, [initial.rfNodes]);
    const onNodesChange = useCallback((changes: NodeChange[]) => setRfNodes(nds => applyNodeChanges(changes, nds)), []);

    if (allNodes.length === 0) return <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>;

    return (
        <div className={styles.graphWrap}>
            <ReactFlow
                nodes={rfNodes}
                edges={initial.rfEdges}
                onNodesChange={onNodesChange}
                nodeTypes={RF_NODE_TYPES}
                fitView
                fitViewOptions={{ padding: 0.25 }}
                minZoom={0.1}
                maxZoom={1.5}
                proOptions={{ hideAttribution: true }}
            >
                <Background gap={16} size={1} color="var(--bdr)" />
                <Controls showInteractive={false} />
                {allNodes.length > 8 && <MiniMap pannable zoomable style={{ background: 'var(--bg0)' }} />}
            </ReactFlow>
        </div>
    );
};

// ── Keyword Insights: timeline — heatmap-style grid with time labels
//    running horizontally across the top header row (guaranteed via
//    inline styles: nowrap + horizontal writing-mode) and one row per
//    keyword running down the left. ──
const TIME_UNIT_LABEL: Record<TimelineTimeUnit, string> = {
    minutes: 'uploadInfer.workspacePanel.timeUnitMinutes',
    hours: 'uploadInfer.workspacePanel.timeUnitHours',
    seconds: 'uploadInfer.workspacePanel.timeUnitSeconds',
};
const TIME_UNIT_SHORT: Record<TimelineTimeUnit, string> = {
    minutes: 'uploadInfer.workspacePanel.timeUnitMinShort',
    hours: 'uploadInfer.workspacePanel.timeUnitHrShort',
    seconds: 'uploadInfer.workspacePanel.timeUnitSecShort',
};

const TimelineView: React.FC<{ data: TimelineData }> = ({ data }) => {
    const { t } = useTranslation();
    const labels = data.labels ?? [];
    const datasets = data.datasets ?? [];
    if (!labels.length || !datasets.length) return <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>;

    const max = Math.max(1, ...datasets.flatMap(d => d.data.filter(v => typeof v === 'number')));
    const ROW_LABEL_W = 140;
    const COL_W = 56;
    const timeUnit = data.time_unit ?? 'minutes';
    const unitLabel = t(TIME_UNIT_LABEL[timeUnit]);
    // When labels are already actual clock timestamps (e.g. "0:00–5:00"),
    // the format itself conveys the unit — don't also append a suffix.
    // Only append the short unit to plain numeric bucket labels.
    const unitShort = t(TIME_UNIT_SHORT[timeUnit]);
    const formatTimeLabel = (lab: string) => data.timestamped ? lab : `${lab} ${unitShort}`;

    // Text guaranteed to stay horizontal, left-to-right, single line —
    // independent of whatever the surrounding stylesheet does elsewhere.
    const horizontalLabel: React.CSSProperties = {
        writingMode: 'horizontal-tb',
        textOrientation: 'mixed',
        whiteSpace: 'nowrap',
        overflow: 'hidden',
        textOverflow: 'ellipsis',
    };

    return (
        <div className={styles.tabContent}>
            <div className={styles.timelineAxisHint}>
                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                    <circle cx="8" cy="8" r="6.25" />
                    <path d="M8 7.2v3.6M8 5.2v.1" />
                </svg>
                <span>
                    {t('uploadInfer.workspacePanel.timelineAxisHint', { unit: unitLabel })}
                </span>
            </div>
        <div style={{ overflowX: 'auto', paddingBottom: 4 }}>
            <div style={{ display: 'grid', gridTemplateColumns: `${ROW_LABEL_W}px repeat(${labels.length}, minmax(${COL_W}px, 1fr))`, width: 'max-content', minWidth: '100%' }}>
                {/* Corner cell — axis labels */}
                <div style={{ ...horizontalLabel, fontSize: 9.5, fontWeight: 700, color: 'var(--t2)', textTransform: 'uppercase', letterSpacing: '0.06em', display: 'flex', alignItems: 'flex-end', padding: '0 10px 8px 0' }}>
                    {t('uploadInfer.workspacePanel.timelineYAxis', 'Keyword')}
                </div>
                {/* Time labels — header row, horizontal across the top */}
                {labels.map((lab, li) => (
                    <div
                        key={li}
                        title={formatTimeLabel(lab)}
                        style={{
                            ...horizontalLabel,
                            fontSize: 10.5, fontWeight: 600, color: 'var(--t2)',
                            textAlign: 'center', padding: '0 4px 8px',
                        }}
                    >
                        {formatTimeLabel(lab)}
                    </div>
                ))}
                {/* One row per keyword */}
                {datasets.map((ds, di) => (
                    <React.Fragment key={di}>
                        <div style={{ ...horizontalLabel, fontSize: 12, fontWeight: 600, color: 'var(--t1)', padding: '6px 10px 6px 0', display: 'flex', alignItems: 'center' }} title={ds.label}>
                            {ds.label}
                        </div>
                        {labels.map((lab, li) => {
                            const v = ds.data[li] ?? 0;
                            const alpha = v > 0 ? Math.min(1, 0.22 + (v / max) * 0.78) : 0;
                            return (
                                <div
                                    key={li}
                                    title={`${ds.label} \u00b7 ${formatTimeLabel(lab)}`}
                                    style={{
                                        height: 32, margin: 2, borderRadius: 4,
                                        background: `rgba(91, 164, 239, ${alpha})`,
                                    }}
                                />
                            );
                        })}
                    </React.Fragment>
                ))}
            </div>
        </div>
        </div>
    );
};

// ── Keyword Insights: word cloud ──
// ── Pan/zoom image viewer — same interaction model as the Knowledge Graph
// tab (drag anywhere to pan, scroll to zoom in/out, buttons for
// zoom in/out/reset), but self-contained (no react-flow dependency needed
// for a single static image). ──
const ZOOM_MIN = 1;
const ZOOM_MAX = 4;
const ZOOM_STEP = 0.35;

const ZoomPanImage: React.FC<{ src: string; alt: string }> = ({ src, alt }) => {
    const { t } = useTranslation();
    const containerRef = useRef<HTMLDivElement>(null);
    const [scale, setScale] = useState(1);
    const [pan, setPan] = useState({ x: 0, y: 0 });
    const dragRef = useRef<{ startX: number; startY: number; startPanX: number; startPanY: number } | null>(null);
    const [dragging, setDragging] = useState(false);

    const clampScale = (s: number) => Math.min(ZOOM_MAX, Math.max(ZOOM_MIN, s));

    // Zoom while keeping the point under the cursor/center visually fixed —
    // same "zoom toward where you're pointing" feel as the graph view.
    const zoomAt = (nextScaleRaw: number, clientX?: number, clientY?: number) => {
        const nextScale = clampScale(nextScaleRaw);
        const el = containerRef.current;
        if (!el) { setScale(nextScale); return; }
        const rect = el.getBoundingClientRect();
        const cx = clientX ?? rect.left + rect.width / 2;
        const cy = clientY ?? rect.top + rect.height / 2;
        const originX = cx - rect.left - rect.width / 2;
        const originY = cy - rect.top - rect.height / 2;
        setPan(prev => {
            if (nextScale === ZOOM_MIN) return { x: 0, y: 0 };
            const ratio = nextScale / scale;
            return {
                x: originX - (originX - prev.x) * ratio,
                y: originY - (originY - prev.y) * ratio,
            };
        });
        setScale(nextScale);
    };

    const handleWheel = (e: React.WheelEvent) => {
        e.preventDefault();
        const delta = e.deltaY < 0 ? ZOOM_STEP : -ZOOM_STEP;
        zoomAt(scale + delta, e.clientX, e.clientY);
    };

    const handleMouseDown = (e: React.MouseEvent) => {
        if (scale <= ZOOM_MIN) return;
        dragRef.current = { startX: e.clientX, startY: e.clientY, startPanX: pan.x, startPanY: pan.y };
        setDragging(true);
    };
    useEffect(() => {
        if (!dragging) return;
        const onMove = (e: MouseEvent) => {
            const d = dragRef.current;
            if (!d) return;
            setPan({ x: d.startPanX + (e.clientX - d.startX), y: d.startPanY + (e.clientY - d.startY) });
        };
        const onUp = () => { setDragging(false); dragRef.current = null; };
        window.addEventListener('mousemove', onMove);
        window.addEventListener('mouseup', onUp);
        return () => {
            window.removeEventListener('mousemove', onMove);
            window.removeEventListener('mouseup', onUp);
        };
    }, [dragging]);

    const zoomIn = () => zoomAt(scale + ZOOM_STEP);
    const zoomOut = () => zoomAt(scale - ZOOM_STEP);
    const reset = () => { setScale(1); setPan({ x: 0, y: 0 }); };

    return (
        <div
            ref={containerRef}
            className={styles.zoomPanWrap}
            onWheel={handleWheel}
            onMouseDown={handleMouseDown}
            onDoubleClick={reset}
            style={{ cursor: scale > ZOOM_MIN ? (dragging ? 'grabbing' : 'grab') : 'default' }}
        >
            <img
                src={src}
                alt={alt}
                className={styles.wordCloudImage}
                draggable={false}
                style={{
                    transform: `translate(${pan.x}px, ${pan.y}px) scale(${scale})`,
                    transition: dragging ? 'none' : 'transform 0.15s ease-out',
                }}
            />
            <div className={styles.zoomControls}>
                <button type="button" className={styles.zoomControlBtn} onClick={zoomIn} disabled={scale >= ZOOM_MAX} title={t('uploadInfer.workspacePanel.zoomIn')}>
                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round" strokeLinejoin="round">
                        <circle cx="7" cy="7" r="5" /><path d="M13.5 13.5L10.8 10.8M7 4.5v5M4.5 7h5" />
                    </svg>
                </button>
                <button type="button" className={styles.zoomControlBtn} onClick={zoomOut} disabled={scale <= ZOOM_MIN} title={t('uploadInfer.workspacePanel.zoomOut')}>
                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round" strokeLinejoin="round">
                        <circle cx="7" cy="7" r="5" /><path d="M13.5 13.5L10.8 10.8M4.5 7h5" />
                    </svg>
                </button>
                <button type="button" className={styles.zoomControlBtn} onClick={reset} disabled={scale === ZOOM_MIN && pan.x === 0 && pan.y === 0} title={t('uploadInfer.workspacePanel.zoomReset')}>
                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round" strokeLinejoin="round">
                        <path d="M13.5 8a5.5 5.5 0 11-1.6-3.9" /><path d="M13.5 2.5v3h-3" />
                    </svg>
                </button>
            </div>
        </div>
    );
};

const WordCloudView: React.FC<{ data: WordCloudData }> = ({ data }) => {
    const { t } = useTranslation();
    if (!data.wordcloud_image) return <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>;
    return (
        <div className={styles.wordCloudImageWrap}>
            <ZoomPanImage src={data.wordcloud_image} alt={t('uploadInfer.workspacePanel.kiTabs.wordcloud')} />
        </div>
    );
};

// ── Keyword Insights: importance vs complexity scatter ──
// Complexity → accent color, reused for the column header, bars, and dots.
const COMPLEXITY_COLOR: Record<string, string> = {
    Easy: 'var(--green)',
    Medium: 'var(--amber)',
    Hard: 'var(--red)',
};
const complexityColor = (c: string) => COMPLEXITY_COLOR[c] ?? 'var(--blue)';

const ImportanceComplexityScatter: React.FC<{ data: ImportanceComplexityData }> = ({ data }) => {
    const { t } = useTranslation();
    const items = data.data ?? [];
    if (!items.length) return <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>;

    const order = ['Easy', 'Medium', 'Hard'];
    const complexities = Array.from(new Set(items.map(it => it.complexity)))
        .sort((a, b) => {
            const ai = order.indexOf(a), bi = order.indexOf(b);
            if (ai === -1 && bi === -1) return a.localeCompare(b);
            if (ai === -1) return 1;
            if (bi === -1) return -1;
            return ai - bi;
        });

    // Importance is always on a fixed 0–10 scale from the backend, so the
    // bar fill is a direct out-of-10 progress value rather than scaled
    // against whatever the largest item in this particular file happens
    // to be — a 6 always fills 60% of the bar, on any file.
    const IMPORTANCE_MAX = 10;

    return (
        <div className={styles.tabContent}>
            <div className={styles.importanceHint}>
                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                    <circle cx="8" cy="8" r="6.25" />
                    <path d="M8 7.2v3.6M8 5.2v.1" />
                </svg>
                <span>{t('uploadInfer.workspacePanel.importanceHint')}</span>
            </div>
            <div className={styles.importanceColumns}>
            {complexities.map(c => {
                const colItems = items.filter(it => it.complexity === c).sort((a, b) => b.importance - a.importance);
                const color = complexityColor(c);
                return (
                    <div key={c} className={styles.importanceColumn}>
                        <div className={styles.importanceColumnHead} style={{ color, borderColor: color }}>
                            {c}
                            <span className={styles.importanceColumnCount}>{colItems.length}</span>
                        </div>
                        <div className={styles.importanceRows}>
                            {colItems.map((it, i) => (
                                <div key={i} className={styles.importanceRow} title={it.reason}>
                                    <span className={styles.importanceDot} style={{ background: color }} />
                                    <span className={styles.importanceKeyword}>{it.keyword}</span>
                                    <div className={styles.importanceBarTrack}>
                                        <div
                                            className={styles.importanceBarFill}
                                            style={{ width: `${Math.min(100, (it.importance / IMPORTANCE_MAX) * 100)}%`, background: color }}
                                        />
                                    </div>
                                    <span className={styles.importanceMeta}>
                                        <span className={styles.importanceValue}>{it.importance}/{IMPORTANCE_MAX}</span>
                                        <span
                                            className={styles.importanceFrequency}
                                            title={t('uploadInfer.workspacePanel.frequencyTitle')}
                                        >
                                            {t('uploadInfer.workspacePanel.frequencyCount', { count: it.frequency })}
                                        </span>
                                    </span>
                                </div>
                            ))}
                        </div>
                    </div>
                );
            })}
            </div>
        </div>
    );
};

// ── Keyword Insights: glossary ──
const GlossaryView: React.FC<{ data: GlossaryData }> = ({ data }) => {
    const { t } = useTranslation();
    const items = data.glossary ?? [];
    if (!items.length) return <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>;
    const fmtTime = (ms: number) => {
        const totalSec = Math.floor(ms / 1000);
        const m = Math.floor(totalSec / 60), s = totalSec % 60;
        return `${m}:${String(s).padStart(2, '0')}`;
    };
    return (
        <div className={styles.glossaryList}>
            {items.map((it, i) => (
                <div key={i} className={styles.glossaryRow}>
                    <div className={styles.glossaryTerm}>{it.term}<span className={styles.glossaryTime}>{fmtTime(it.first_mentioned_ms)}</span></div>
                    <div className={styles.glossaryDef}>{it.definition}</div>
                </div>
            ))}
        </div>
    );
};

// ── Tab: Keyword Insights ────────────────────────────────────
const KI_SUBTABS = ['graph', 'wordcloud', 'timeline', 'importance', 'glossary'] as const;
type KiSubTab = typeof KI_SUBTABS[number];

const TabKeywordInsights: React.FC<{ data: KeywordInsights | null }> = ({ data }) => {
    const { t } = useTranslation();
    const [sub, setSub] = useState<KiSubTab>('graph');

    if (!data) return <div className={`${styles.tabContent} ${styles.tabEmpty}`}>{t('uploadInfer.workspacePanel.noKeywordInsights')}</div>;

    return (
        <div className={styles.tabContent}>
            <ScrollableTabRow
                ids={KI_SUBTABS}
                activeId={sub}
                onChange={setSub}
                labelFor={id => t(`uploadInfer.workspacePanel.kiTabs.${id}`)}
                itemClassName={styles.kiSubTab}
                activeClassName={styles.kiSubTabActive}
                wrapClassName={styles.kiSubNavWrap}
                trackClassName={styles.kiSubNav}
            />
            <div className={styles.kiBody}>
                {sub === 'graph' && (data.knowledge_graph
                    ? <NodeLinkGraph nodes={data.knowledge_graph.nodes} edges={data.knowledge_graph.edges.map(e => ({ source: e.source, target: e.target, label: e.type }))} treeMode />
                    : <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>)}
                {sub === 'wordcloud' && (data.word_cloud ? <WordCloudView data={data.word_cloud} /> : <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>)}
                {sub === 'timeline' && (data.timeline
                    ? <TimelineView data={data.timeline} />
                    : <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>)}
                {sub === 'importance' && (data.importance_complexity ? <ImportanceComplexityScatter data={data.importance_complexity} /> : <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>)}
                {sub === 'glossary' && (data.glossary ? <GlossaryView data={data.glossary} /> : <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>)}
            </div>
        </div>
    );
};

// ── Scrollable tab row — generic horizontal-scroll tab strip with arrow
//    buttons + edge fades, reused for both the main tab bar and any
//    sub-navigation row (e.g. Keyword Insights) that could overflow. ──
function ScrollableTabRow<T extends string>({
    ids, activeId, onChange, labelFor,
    itemClassName, activeClassName, wrapClassName, trackClassName,
    dataTourFor,
}: {
    ids: readonly T[];
    activeId: T;
    onChange: (id: T) => void;
    labelFor: (id: T) => string;
    itemClassName: string;
    activeClassName: string;
    wrapClassName: string;
    trackClassName: string;
    // Optional — lets a specific caller (e.g. the main results tab bar)
    // tag each individual tab button for the guided tour, without
    // affecting other callers that reuse this same component (e.g. the
    // Keyword Insights sub-tab row).
    dataTourFor?: (id: T) => string | undefined;
}) {
    const trackRef = useRef<HTMLDivElement>(null);
    const [canScrollLeft, setCanScrollLeft] = useState(false);
    const [canScrollRight, setCanScrollRight] = useState(false);

    const updateScrollState = () => {
        const el = trackRef.current;
        if (!el) return;
        setCanScrollLeft(el.scrollLeft > 2);
        setCanScrollRight(el.scrollLeft < el.scrollWidth - el.clientWidth - 2);
    };

    useEffect(() => {
        updateScrollState();
        const el = trackRef.current;
        if (!el) return;
        const onResize = () => updateScrollState();
        window.addEventListener('resize', onResize);
        el.addEventListener('scroll', updateScrollState);
        return () => { window.removeEventListener('resize', onResize); el.removeEventListener('scroll', updateScrollState); };
    }, []); // eslint-disable-line

    // Keep the active item in view when it changes
    useEffect(() => {
        const el = trackRef.current;
        if (!el) return;
        const activeEl = el.querySelector<HTMLElement>(`[data-tab-id="${activeId}"]`);
        activeEl?.scrollIntoView({ behavior: 'smooth', block: 'nearest', inline: 'nearest' });
        const id = setTimeout(updateScrollState, 300);
        return () => clearTimeout(id);
    }, [activeId]); // eslint-disable-line

    const scrollBy = (dx: number) => trackRef.current?.scrollBy({ left: dx, behavior: 'smooth' });

    return (
        <div className={wrapClassName}>
            {canScrollLeft && (
                <button className={`${styles.tabScrollBtn} ${styles.tabScrollBtnLeft}`} onClick={() => scrollBy(-160)} aria-label="Scroll left">
                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round"><path d="M10 3L6 8l4 5" /></svg>
                </button>
            )}
            <div className={trackClassName} ref={trackRef}>
                {ids.map(id => (
                    <button
                        key={id}
                        data-tab-id={id}
                        data-tour={dataTourFor?.(id)}
                        className={`${itemClassName} ${activeId === id ? activeClassName : ''}`}
                        onClick={() => onChange(id)}
                    >
                        {labelFor(id)}
                    </button>
                ))}
            </div>
            {canScrollLeft && <div className={`${styles.tabFade} ${styles.tabFadeLeft}`} />}
            {canScrollRight && <div className={`${styles.tabFade} ${styles.tabFadeRight}`} />}
            {canScrollRight && (
                <button className={`${styles.tabScrollBtn} ${styles.tabScrollBtnRight}`} onClick={() => scrollBy(160)} aria-label="Scroll right">
                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round"><path d="M6 3l4 5-4 5" /></svg>
                </button>
            )}
        </div>
    );
}

// ── Main tab bar — the fixed-width flex row silently clipped tabs once
//    there were more than ~3; ScrollableTabRow fixes that. ──
const ScrollableTabs: React.FC<{ activeTab: TabId; onChange: (id: TabId) => void }> = ({ activeTab, onChange }) => {
    const { t } = useTranslation();
    return (
        <div data-tour="results-tabbar">
            <ScrollableTabRow
                ids={TAB_IDS}
                activeId={activeTab}
                onChange={onChange}
                labelFor={id => t(`uploadInfer.workspacePanel.tabs.${id}`)}
                itemClassName={styles.tab}
                activeClassName={styles.active}
                wrapClassName={styles.wsptabsWrap}
                trackClassName={styles.wsptabs}
                dataTourFor={id => `results-tab-${id}`}
            />
        </div>
    );
};

// ── Main panel ────────────────────────────────────────────
const WorkspacePanel = forwardRef<WorkspacePanelHandle, Props>(({ step2Visible = true, step2Minimized = false, fileResult, fileLoading, activeFileId, onResultUpdate }, ref) => {
    const { t } = useTranslation();
    const [activeTab, setActiveTab] = useState<TabId>('summary');

    useImperativeHandle(ref, () => ({
        setTab: (id: TabId) => setActiveTab(id),
    }), []);

    return (
        <div className={`${styles.wspanel} ${(!step2Visible || step2Minimized) ? styles.wspanelExpanded : ''} ${step2Visible ? styles.wspanelWithStep2 : ''}`} data-tour="results-panel">
            <div className={styles.wspanelHead}>
                <div className={styles.headLeft}>
                    {fileResult ? (
                        <>
                            <div className={styles.wsftitle}>{fileResult.fileName}</div>
                            <div className={styles.wsmeta}>
                                <span className={styles.wsmetaId}>#{fileResult.fileId}</span>
                                {fileResult.insertedAt && (<><span className={styles.wsmetaSep}>·</span><span className={styles.wsmetaDate}>{fileResult.insertedAt}</span></>)}
                            </div>
                        </>
                    ) : (
                        <>
                            <div className={styles.wsftitleEmpty}>—</div>
                            <div className={styles.wsmeta}><span className={styles.wsmetaHint}>{t('uploadInfer.workspacePanel.clickToView')}</span></div>
                        </>
                    )}
                </div>
            </div>

            <ScrollableTabs activeTab={activeTab} onChange={setActiveTab} />

            <div className={styles.wsbody}>
                {fileLoading && (
                    <div className={styles.wsSpinner}><div className={styles.spinner} /><span>{t('uploadInfer.workspacePanel.loadingFile')}</span></div>
                )}
                {!fileLoading && !fileResult && (
                    <div className={styles.wsEmpty}>
                        <svg viewBox="0 0 48 48" fill="none" stroke="currentColor" strokeWidth="1.2" strokeLinecap="round" strokeLinejoin="round">
                            <rect x="8" y="6" width="32" height="36" rx="3" /><path d="M16 16h16M16 23h16M16 30h10" />
                        </svg>
                        <div className={styles.wsEmptyTitle}>{t('uploadInfer.workspacePanel.noFileSelected')}</div>
                        <div className={styles.wsEmptyDesc}>{t('uploadInfer.workspacePanel.noFileDesc')}</div>
                    </div>
                )}
                {!fileLoading && fileResult && (
                    <>
                        {activeTab === 'summary' && <TabSummary summary={fileResult.summary} fileId={fileResult.fileId} onSaved={s => onResultUpdate?.({ summary: s })} />}
                        {activeTab === 'keywords' && <TabKeywords keywords={fileResult.keywords} fileId={fileResult.fileId} onSaved={kw => onResultUpdate?.({ keywords: kw })} />}
                        {activeTab === 'assessment' && <TabAssessment faq={fileResult.faq} />}
                        {activeTab === 'shortAnswer' && <TabShortAnswer raw={fileResult.shortAnswer} />}
                        {activeTab === 'trueFalse' && <TabTrueFalse raw={fileResult.trueFalse} />}
                        {activeTab === 'timestampedSummary' && <TabTimestampedSummary raw={fileResult.timestampedSummary} />}
                        {activeTab === 'keywordInsights' && <TabKeywordInsights data={fileResult.keywordInsights} />}
                    </>
                )}
            </div>
        </div>
    );
});

WorkspacePanel.displayName = 'WorkspacePanel';

export default WorkspacePanel;
