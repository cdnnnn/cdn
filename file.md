//testsuites.ts
import { apiClient } from '../client';
import type { TestSuite } from '../../store/slices/testSuitesSlice';

interface TestSuitesResponse {
  datasets: TestSuite[];
}

export type EvalType = 'model' | 'agent' | 'rag';

export interface UploadDatasetParams {
  file: File;
  name: string;
  description: string;
  evalType: EvalType;
}

// Extensions routed to POST /datasets/upload
const STRUCTURED_EXTENSIONS = ['json', 'arrow', 'parquet'];
// Extension routed to POST /datasets/upload-jsonl
const JSONL_EXTENSION = 'jsonl';

export const SUPPORTED_UPLOAD_EXTENSIONS = [...STRUCTURED_EXTENSIONS, JSONL_EXTENSION];

function getExtension(filename: string): string {
  const idx = filename.lastIndexOf('.');
  return idx >= 0 ? filename.slice(idx + 1).toLowerCase() : '';
}

export interface DeleteDatasetResponse {
  status: string;
  dataset_id: string;
}

export interface DatasetQuestion {
  id: string;
  category?: string | null;
  question_type?: string | null;
  input?: {
    prompt?: string | null;
    language?: string | null;
    // MinIO object keys, e.g. "datasets/ds-001/images/image1.jpg" — not
    // directly-fetchable URLs. Resolved on demand via getImageBlobUrl()
    // below, only when the user opens the lightbox for a given image.
    images?: string[] | null;
  } | null;
  expected?: { answer?: string | null } | null;
}

export interface DatasetPreviewResponse {
  dataset_id: string;
  questions: DatasetQuestion[];
  total: number;
}

export const testSuitesApi = {
  // GET /datasets
  list: async (): Promise<TestSuite[]> => {
    const { data } = await apiClient.get<TestSuitesResponse>('/datasets');
    return data?.datasets ?? [];
  },

  // Routes to POST /datasets/upload (.json, .arrow, .parquet) or
  // POST /datasets/upload-jsonl (.jsonl) based on the file extension.
  // Both endpoints return a bare 200 on success — no dataset body — so the
  // caller is expected to re-fetch the list afterward.
  upload: async ({ file, name, description, evalType }: UploadDatasetParams): Promise<void> => {
    const ext = getExtension(file.name);

    if (!SUPPORTED_UPLOAD_EXTENSIONS.includes(ext)) {
      throw new Error('Unsupported file type. Please upload a .json, .jsonl, .arrow, or .parquet file.');
    }

    const formData = new FormData();
    formData.append('file', file);

    if (ext === JSONL_EXTENSION) {
      // Agent uploads additionally require category="Agents" per the API
      // contract; Model and RAG uploads omit the field entirely.
      const params: Record<string, string> = { eval_type: evalType, name, description };
      if (evalType === 'agent') params.category = 'Agents';

      await apiClient.post('/datasets/upload-jsonl', formData, { params });
      return;
    }

    await apiClient.post('/datasets/upload', formData, { params: { eval_type: evalType, name, description } });
  },

  // DELETE /datasets/{dataset_id} — only offered in the UI for custom
  // datasets (dataset_type === 'custom'); non-custom datasets aren't
  // user-deletable.
  remove: async (datasetId: string): Promise<DeleteDatasetResponse> => {
    const { data } = await apiClient.delete<DeleteDatasetResponse>(`/datasets/${datasetId}`);
    return data;
  },

  // GET /datasets/{dataset_id}/preview?limit=&offset= — paginated question
  // preview shown in the slide-over drawer.
  preview: async (datasetId: string, limit: number, offset: number): Promise<DatasetPreviewResponse> => {
    const { data } = await apiClient.get<DatasetPreviewResponse>(`/datasets/${datasetId}/preview`, {
      params: { limit, offset },
    });
    return data;
  },

  // POST /datasets/image — resolves a MinIO object key to the actual image
  // bytes. Response is a raw blob, so this is fetched with responseType
  // 'blob' and turned into an object URL for the <img> tag; the lightbox
  // revokes it on close.
  getImageBlobUrl: async (imagePath: string): Promise<string> => {
    const { data } = await apiClient.post('/datasets/image', { path: imagePath }, {
      responseType: 'blob',
    });
    return URL.createObjectURL(data as Blob);
  },
};






















//Datasets.tsx
//Datasets.tsx
import { useEffect, useMemo, useState, useRef, useCallback } from 'react';
import {
  RefreshCw, Search, Layers, AlertTriangle, Database, ListFilter, X,
  Check, Boxes, ArrowRight, Filter, ChevronsUpDown, Upload, Loader2,
  FileUp, AlertCircle, CheckCircle2, Trash2, Eye, EyeOff, ChevronLeft, ChevronRight, HelpCircle,
  Image as ImageIcon, Download, RotateCw,
} from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import {
  fetchTestSuites, uploadDataset, resetUploadStatus, deleteTestSuite, resetDeleteError,
  fetchDatasetPreview, resetPreview,
} from '../../store/slices/testSuitesSlice';
import type { TestSuite, PreviewState } from '../../store/slices/testSuitesSlice';
import { testSuitesApi, SUPPORTED_UPLOAD_EXTENSIONS } from '../../api/endpoints/testSuites';
import type { EvalType } from '../../api/endpoints/testSuites';
import styles from './Datasets.module.scss';

// Deterministic color hash so the same category always gets the same pill
// color across renders, without a hardcoded lookup table. Palette matches
// the app's ink/paper/signal design tokens.
const PILL_COLORS = [
  { bg: '#ECEDFF', fg: '#2B2BF5' }, // signal
  { bg: '#FDF3E3', fg: '#C56A00' }, // amber
  { bg: '#E7F7EF', fg: '#0B8F58' }, // ok
  { bg: '#FDECEC', fg: '#C81E1E' }, // danger
  { bg: '#E6F4FB', fg: '#0369A1' }, // sky
  { bg: '#F1EDFB', fg: '#6D28D9' }, // violet
  { bg: '#EAF6EC', fg: '#3F7D20' }, // moss
];
function hashColor(label?: string | null) {
  const safe = label || '—';
  const sum = [...safe].reduce((acc, ch) => acc + ch.charCodeAt(0), 0);
  return PILL_COLORS[sum % PILL_COLORS.length];
}

// Categories/tags come back with inconsistent casing across datasets
// (e.g. "Chemistry" vs "chemistry") but should be treated as the same
// category for filtering, active-state highlighting, and color assignment.
function normalizeTag(t?: string | null): string {
  return (t || '').trim().toLowerCase();
}

// Renders a readable filename from a MinIO object key
// ("datasets/ds-001/images/image1.jpg" -> "image1.jpg").
function imageBasename(path: string): string {
  const parts = path.split('/');
  return parts[parts.length - 1] || path;
}

const EVAL_TYPE_OPTIONS: { value: EvalType; label: string }[] = [
  { value: 'model', label: 'Model' },
  { value: 'agent', label: 'Agent' },
];

const ACCEPT_ATTR = SUPPORTED_UPLOAD_EXTENSIONS.map((e) => `.${e}`).join(',');
const PREVIEW_PAGE_SIZE_OPTIONS = [10, 20, 50];

// Delete is only offered for user-uploaded custom datasets, not built-in
// benchmark suites.
function isCustomDataset(d: TestSuite | null | undefined): boolean {
  return (d?.dataset_type || '').toLowerCase() === 'custom';
}

// Question preview isn't available for agent-type benchmark datasets — the
// preview endpoint doesn't have question-level data for these.
function isPreviewUnavailable(d: TestSuite | null | undefined): boolean {
  return (d?.dataset_type || '').toLowerCase() === 'benchmark' && (d?.eval_type || '').toLowerCase() === 'agent';
}

export default function Datasets() {
  const dispatch = useAppDispatch();
  const {
    items,
    status = 'idle',
    error = null,
    uploadStatus = 'idle',
    uploadError = null,
    deletingId = null,
    deleteError = null,
    preview,
  } = useAppSelector((s) => s.testSuites) ?? {};
  const safeItems = items ?? [];
  const previewState = preview ?? {
    datasetId: null, questions: [], total: 0, limit: 20, offset: 0, status: 'idle' as const, error: null,
  };

  const [search, setSearch] = useState('');
  const [datasetTypeFilter, setDatasetTypeFilter] = useState('All');
  const [tagFilter, setTagFilter] = useState<string[]>([]);       // active dataset_categories facets
  const [selectedId, setSelectedId] = useState<string | null>(null);
  const [uploadOpen, setUploadOpen] = useState(false);
  const [deleteTarget, setDeleteTarget] = useState<TestSuite | null>(null);
  const [previewTarget, setPreviewTarget] = useState<TestSuite | null>(null);
  const [lightboxPath, setLightboxPath] = useState<string | null>(null);
  const searchRef = useRef<HTMLInputElement>(null);

  useEffect(() => {
    dispatch(fetchTestSuites());
  }, [dispatch]);

  const datasetTypes = useMemo(
    () => ['All', ...new Set(safeItems.map((d) => d?.dataset_type).filter((t): t is string => Boolean(t)))],
    [safeItems]
  );

  const filtered = useMemo(() => {
    const q = search.trim().toLowerCase();
    return safeItems.filter((d) => {
      if (!d) return false;
      if (datasetTypeFilter !== 'All' && (d.dataset_type ?? '') !== datasetTypeFilter) return false;
      const tags = d.dataset_categories ?? [];
      if (tagFilter.length && !tagFilter.some((t) => tags.some((tag) => normalizeTag(tag) === normalizeTag(t)))) return false;
      if (!q) return true;
      const name = (d.name ?? '').toLowerCase();
      const desc = (d.description ?? '').toLowerCase();
      return name.includes(q) || desc.includes(q);
    });
  }, [safeItems, search, datasetTypeFilter, tagFilter]);

  // Keep a valid selection as the filtered set changes.
  useEffect(() => {
    if (!filtered.length) return;
    if (!filtered.some((d) => d?.id && d.id === selectedId)) {
      const first = filtered[0];
      if (first?.id) setSelectedId(first.id);
    }
  }, [filtered, selectedId]);

  const selected = safeItems.find((d) => d?.id && d.id === selectedId) ?? null;

  const toggleTag = useCallback((t: string) => {
    setTagFilter((prev) => {
      const normalized = normalizeTag(t);
      const exists = prev.some((x) => normalizeTag(x) === normalized);
      return exists ? prev.filter((x) => normalizeTag(x) !== normalized) : [...prev, t];
    });
  }, []);

  // Keyboard: ↑/↓ walk the list, "/" focuses search.
  useEffect(() => {
    const onKey = (e: KeyboardEvent) => {
      const el = document.activeElement as HTMLElement | null;
      const typing = el?.tagName === 'INPUT' || el?.tagName === 'TEXTAREA';
      if (e.key === '/' && !typing) {
        e.preventDefault();
        searchRef.current?.focus();
        return;
      }
      if (typing || (e.key !== 'ArrowDown' && e.key !== 'ArrowUp')) return;
      e.preventDefault();
      const idx = filtered.findIndex((d) => d?.id && d.id === selectedId);
      if (idx === -1) {
        const first = filtered[0];
        if (first?.id) setSelectedId(first.id);
        return;
      }
      const next = e.key === 'ArrowDown'
        ? Math.min(idx + 1, filtered.length - 1)
        : Math.max(idx - 1, 0);
      const nextItem = filtered[next];
      if (nextItem?.id) setSelectedId(nextItem.id);
    };
    window.addEventListener('keydown', onKey);
    return () => window.removeEventListener('keydown', onKey);
  }, [filtered, selectedId]);

  // Closes the modal + refreshes the list once an upload succeeds. The
  // upload endpoints return a bare 200 with no dataset body, so a re-fetch
  // is the only way to pick up the new row.
  useEffect(() => {
    if (uploadStatus !== 'succeeded') return;
    dispatch(fetchTestSuites());
    setUploadOpen(false);
    const t = setTimeout(() => dispatch(resetUploadStatus()), 300);
    return () => clearTimeout(t);
  }, [uploadStatus, dispatch]);

  const closeUploadModal = () => {
    setUploadOpen(false);
    if (uploadStatus !== 'idle') dispatch(resetUploadStatus());
  };

  const closeDeleteModal = () => {
    setDeleteTarget(null);
    if (deleteError) dispatch(resetDeleteError());
  };

  const confirmDelete = async () => {
    if (!deleteTarget?.id) return;
    try {
      await dispatch(deleteTestSuite(deleteTarget.id)).unwrap();
      if (selectedId === deleteTarget.id) setSelectedId(null);
      setDeleteTarget(null);
    } catch {
      // deleteError is already populated in state; the confirm modal stays
      // open and renders it.
    }
  };

  // Fetches page 1 as soon as the preview drawer opens for a dataset.
  const openPreview = (dataset: TestSuite) => {
    setPreviewTarget(dataset);
    if (dataset.id) {
      dispatch(fetchDatasetPreview({ datasetId: dataset.id, limit: previewState.limit || 20, offset: 0 }));
    }
  };

  const closePreview = () => {
    setPreviewTarget(null);
    dispatch(resetPreview());
  };

  const changePreviewOffset = (offset: number) => {
    if (!previewTarget?.id) return;
    dispatch(fetchDatasetPreview({ datasetId: previewTarget.id, limit: previewState.limit, offset }));
  };

  const changePreviewLimit = (limit: number) => {
    if (!previewTarget?.id) return;
    dispatch(fetchDatasetPreview({ datasetId: previewTarget.id, limit, offset: 0 }));
  };

  // Close the preview drawer with Escape — same pattern as History's
  // details drawer. Yields to the lightbox first: if an image is open on
  // top of the drawer, Escape should close just the image, not both.
  useEffect(() => {
    if (!previewTarget) return;
    const onKey = (e: KeyboardEvent) => {
      if (e.key !== 'Escape') return;
      if (lightboxPath) return;
      closePreview();
    };
    window.addEventListener('keydown', onKey);
    return () => window.removeEventListener('keydown', onKey);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [previewTarget, lightboxPath]);

  // Close the image lightbox with Escape.
  useEffect(() => {
    if (!lightboxPath) return;
    const onKey = (e: KeyboardEvent) => {
      if (e.key === 'Escape') setLightboxPath(null);
    };
    window.addEventListener('keydown', onKey);
    return () => window.removeEventListener('keydown', onKey);
  }, [lightboxPath]);

  return (
    <div className="page-enter pg-shell">
      {/* ---- header (unchanged, + upload button) --------------------------- */}
      <div className={styles['datasets__header']}>
        <div>
          <p className={styles['datasets__header-eyebrow']}>Datasets</p>
          <h1>Test Suite Library</h1>
          <p className={styles['datasets__header-sub']}>
            Browse every dataset available for evaluations, independent of any single wizard run.
          </p>
        </div>
        <div className={styles['datasets__header-meta']}>
          <span className={styles['datasets__header-count']}>
            <Database size={13} /> {safeItems.length} datasets available
          </span>
          <button className={styles['datasets__refresh-btn']} onClick={() => dispatch(fetchTestSuites())}>
            <RefreshCw size={14} /> Refresh
          </button>
          <button className={styles['datasets__upload-btn']} onClick={() => setUploadOpen(true)}>
            <Upload size={14} /> Upload dataset
          </button>
        </div>
      </div>

      {/* ---- toolbar (unchanged) ------------------------------------------ */}
      <div className={styles['datasets__toolbar']}>
        <div className={styles['datasets__search']}>
          <Search size={16} />
          <input
            ref={searchRef}
            placeholder="Search datasets…  (press /)"
            value={search}
            onChange={(e) => setSearch(e.target.value)}
          />
        </div>
        <div className={styles['datasets__filters']}>
          <span className={styles['datasets__toolbar-label']}>
            <ListFilter size={11} />
          </span>
          {datasetTypes.map((t) => (
            <button
              key={t}
              className={`${styles['datasets__filter-pill']} ${datasetTypeFilter === t ? styles['datasets__filter-pill--on'] : ''}`}
              onClick={() => setDatasetTypeFilter(t)}
            >
              {t}
            </button>
          ))}
        </div>
      </div>

      {/* ---- active tag facets --------------------------------------------- */}
      {tagFilter.length > 0 && (
        <div className={styles['datasets__facets']}>
          <Filter size={12} />
          <span className={styles['datasets__facets-lead']}>Showing datasets tagged</span>
          {tagFilter.map((t) => {
            const col = hashColor(normalizeTag(t));
            return (
              <button
                key={t}
                className={styles['datasets__facet']}
                style={{ background: col.bg, color: col.fg }}
                onClick={() => toggleTag(t)}
              >
                {t} <X size={11} />
              </button>
            );
          })}
          <button className={styles['datasets__facets-clear']} onClick={() => setTagFilter([])}>
            Clear all
          </button>
        </div>
      )}

      {/* ---- body: master / detail ---------------------------------------- */}
      <div className={styles['datasets__body']}>
        {status === 'failed' && (
          <div
            className={`${styles['datasets__state']} ${styles['datasets__state--error']}`}
            style={{ gridColumn: '1 / -1' }}
          >
            <AlertTriangle size={28} />
            <div>{error || 'Failed to load datasets.'}</div>
            <button className={styles['datasets__refresh-btn']} onClick={() => dispatch(fetchTestSuites())}>
              Retry
            </button>
          </div>
        )}

        {status !== 'failed' && (
          <>
            {/* LIST RAIL — bordered cards, one per dataset */}
            <aside className={styles['datasets__rail']}>
              <div className={styles['datasets__rail-head']}>
                <span>{filtered.length} of {safeItems.length}</span>
                <span className={styles['datasets__rail-hint']}>
                  <ChevronsUpDown size={11} /> ↑ ↓ to move
                </span>
              </div>
              <div className={styles['datasets__rail-scroll']}>
                {status === 'loading' &&
                  Array.from({ length: 7 }).map((_, i) => (
                    <div className={styles['datasets__skel-row']} key={i}>
                      <span className={styles['datasets__skel']} style={{ width: '55%' }} />
                      <span className={styles['datasets__skel']} style={{ width: '35%' }} />
                    </div>
                  ))}

                {status !== 'loading' && filtered.length === 0 && (
                  <div className={styles['datasets__empty-rail']}>
                    <Layers size={22} />
                    <p>No datasets match.<br />Loosen a filter to see more.</p>
                  </div>
                )}

                {status !== 'loading' &&
                  filtered.map((d, i) => {
                    if (!d) return null;
                    const rowKey = d.id ?? `row-${i}`;
                    const on = !!d.id && d.id === selectedId;
                    const dsType = d.dataset_type || 'Unknown';
                    const evalTypeLabel = d.eval_type || null;
                    const accent = hashColor(dsType);
                    const tags = d.dataset_categories ?? [];
                    const count = typeof d.question_count === 'number' ? d.question_count : 0;
                    return (
                      <button
                        key={rowKey}
                        className={`${styles['datasets__row']} ${on ? styles['datasets__row--on'] : ''}`}
                        onClick={() => d.id && setSelectedId(d.id)}
                        style={{ borderLeftColor: accent.fg }}
                      >
                        <div className={styles['datasets__row-top']}>
                          <span className={styles['datasets__row-name']}>{d.name || 'Untitled dataset'}</span>
                          <span className={styles['datasets__row-count']}>{count.toLocaleString()}</span>
                        </div>
                        <div className={styles['datasets__row-foot']}>
                          <span className={styles['datasets__row-type-group']}>
                            <span className={styles['datasets__row-type']} style={{ color: accent.fg }}>
                              {dsType}
                            </span>
                            {evalTypeLabel && (
                              <span className={styles['datasets__row-eval-tag']}>{evalTypeLabel}</span>
                            )}
                          </span>
                          <span className={styles['datasets__row-dots']}>
                            {tags.slice(0, 4).map((t, ti) => (
                              <i key={t || ti} style={{ background: hashColor(normalizeTag(t)).fg }} title={t || undefined} />
                            ))}
                          </span>
                        </div>
                      </button>
                    );
                  })}
              </div>
            </aside>

            {/* DETAIL */}
            <section className={styles['datasets__detail']}>
              {status === 'loading' ? (
                <div className={styles['datasets__detail-scroll']}>
                  <span className={styles['datasets__skel']} style={{ width: 120, height: 34, marginBottom: 22 }} />
                  <span className={styles['datasets__skel']} style={{ width: '90%', marginBottom: 8 }} />
                  <span className={styles['datasets__skel']} style={{ width: '70%' }} />
                </div>
              ) : !selected ? (
                <div className={styles['datasets__detail-empty']}>
                  <Boxes size={30} />
                  <p>Select a dataset to inspect its categories, questions, and eval type.</p>
                </div>
              ) : (
                <DetailView
                  dataset={selected}
                  tagFilter={tagFilter}
                  toggleTag={toggleTag}
                  onDeleteClick={() => setDeleteTarget(selected)}
                  onPreviewClick={() => openPreview(selected)}
                />
              )}
            </section>
          </>
        )}
      </div>

      {/* ---- upload dataset modal ------------------------------------------ */}
      {uploadOpen && (
        <UploadModal
          uploadStatus={uploadStatus}
          uploadError={uploadError}
          onClose={closeUploadModal}
          onSubmit={(params) => dispatch(uploadDataset(params))}
        />
      )}

      {/* ---- delete dataset confirmation ------------------------------------ */}
      {deleteTarget && (
        <DeleteConfirmModal
          dataset={deleteTarget}
          isDeleting={!!deleteTarget.id && deletingId === deleteTarget.id}
          deleteError={deleteError}
          onCancel={closeDeleteModal}
          onConfirm={confirmDelete}
        />
      )}

      {/* ---- question preview slide-over drawer ----------------------------- */}
      <div
        className={`${styles['datasets__preview-overlay']} ${previewTarget ? styles['datasets__preview-overlay--open'] : ''}`}
        onClick={closePreview}
      />
      <div
        className={`${styles['datasets__preview-drawer']} ${previewTarget ? styles['datasets__preview-drawer--open'] : ''}`}
        role="dialog"
        aria-hidden={!previewTarget}
        aria-label="Dataset question preview"
      >
        {previewTarget && (
          <PreviewDrawer
            dataset={previewTarget}
            preview={previewState}
            onClose={closePreview}
            onChangeOffset={changePreviewOffset}
            onChangeLimit={changePreviewLimit}
            onOpenImage={setLightboxPath}
          />
        )}
      </div>

      {/* ---- MinIO image lightbox — mounted only while an image is open,
          fetched lazily on demand rather than preloaded with the drawer */}
      {lightboxPath && (
        <ImageLightbox imagePath={lightboxPath} onClose={() => setLightboxPath(null)} />
      )}
    </div>
  );
}

// ---------------------------------------------------------------------------

interface DetailViewProps {
  dataset: TestSuite;
  tagFilter: string[];
  toggleTag: (t: string) => void;
  onDeleteClick: () => void;
  onPreviewClick: () => void;
}

function DetailView({ dataset: d, tagFilter, toggleTag, onDeleteClick, onPreviewClick }: DetailViewProps) {
  if (!d) return null;

  const category = d.category || 'Uncategorized';
  const datasetType = d.dataset_type || 'Unknown';
  const evalType = d.eval_type || '—';
  const questionCount = typeof d.question_count === 'number' ? d.question_count : 0;
  const accent = hashColor(category);
  const tags = d.dataset_categories ?? [];
  const deletable = isCustomDataset(d);
  const previewUnavailable = isPreviewUnavailable(d);

  return (
    <div className={styles['datasets__detail-scroll']} key={d.id ?? d.name ?? 'selected'}>
      <div className={styles['datasets__hero']}>
        <span className={styles['datasets__hero-bar']} style={{ background: accent.fg }} />
        <div className={styles['datasets__hero-top']}>
          <div>
            <span className={styles['datasets__hero-type']} style={{ color: accent.fg }}>{category}</span>
            <h2>{d.name || 'Untitled dataset'}</h2>
          </div>
          <div className={styles['datasets__hero-actions']}>
            <span className={styles['datasets__hero-badge-group']}>
              <span className={styles['datasets__hero-badge-label']}>Dataset Type</span>
              <span className={styles['datasets__source-badge']}>{datasetType}</span>
            </span>
            {d.eval_type && (
              <span className={styles['datasets__hero-badge-group']}>
                <span className={styles['datasets__hero-badge-label']}>Eval Type</span>
                <span className={styles['datasets__eval-badge']}>{d.eval_type}</span>
              </span>
            )}
            {!previewUnavailable && (
              <button
                type="button"
                className={styles['datasets__preview-btn']}
                onClick={onPreviewClick}
                title="Preview questions"
              >
                <Eye size={14} /> Preview
              </button>
            )}
            {deletable && (
              <button
                type="button"
                className={styles['datasets__delete-btn']}
                onClick={onDeleteClick}
                title="Delete this dataset"
              >
                <Trash2 size={14} />
              </button>
            )}
          </div>
        </div>
        <p className={styles['datasets__hero-desc']}>{d.description || 'No description provided.'}</p>

        {previewUnavailable && (
          <div className={styles['datasets__preview-unavailable']}>
            <span className={styles['datasets__preview-unavailable-icon']}>
              <EyeOff size={16} />
            </span>
            <div>
              <strong>Dataset preview not available</strong>
              <span>Agent benchmark datasets don't expose question-level data for preview.</span>
            </div>
          </div>
        )}
      </div>

      <div className={styles['datasets__stats']}>
        <Stat label="Questions" value={questionCount.toLocaleString()} />
        <Stat label="Categories" value={tags.length || '—'} />
        <Stat label="Eval Type" value={evalType} mono />
        <Stat label="Source" value={datasetType} mono />
      </div>

      <div className={styles['datasets__section']}>
        <div className={styles['datasets__section-head']}>
          <h3>Categories {tags.length > 0 && <em>{tags.length}</em>}</h3>
          {tags.length > 0 && (
            <span className={styles['datasets__section-hint']}>
              click a category to see other datasets with it <ArrowRight size={11} />
            </span>
          )}
        </div>

        {tags.length === 0 ? (
          <div className={styles['datasets__single']}>
            <Boxes size={15} />
            <div>
              <strong>No categories tagged.</strong>
              <span>This dataset isn't broken down into subject areas.</span>
            </div>
          </div>
        ) : (
          <div className={styles['datasets__caps']}>
            {tags.filter(Boolean).map((t) => {
              const col = hashColor(normalizeTag(t));
              const active = tagFilter.some((x) => normalizeTag(x) === normalizeTag(t));
              return (
                <button
                  key={t}
                  className={`${styles['datasets__cap']} ${active ? styles['datasets__cap--active'] : ''}`}
                  style={active
                    ? { background: col.fg, color: '#fff', borderColor: col.fg }
                    : { background: col.bg, color: col.fg, borderColor: 'transparent' }}
                  onClick={() => toggleTag(t)}
                >
                  {t}{active && <Check size={12} />}
                </button>
              );
            })}
          </div>
        )}
      </div>
    </div>
  );
}

function Stat({ label, value, mono }: { label: string; value: string | number | null | undefined; mono?: boolean }) {
  const display = value === null || value === undefined || value === '' ? '—' : value;
  return (
    <div className={styles['datasets__stat']}>
      <span className={`${styles['datasets__stat-val']} ${mono ? styles['datasets__stat-val--mono'] : ''}`}>
        {display}
      </span>
      <span className={styles['datasets__stat-label']}>{label}</span>
    </div>
  );
}

// ---------------------------------------------------------------------------
// Upload modal
// ---------------------------------------------------------------------------

interface UploadModalProps {
  uploadStatus: 'idle' | 'loading' | 'succeeded' | 'failed';
  uploadError: string | null;
  onClose: () => void;
  onSubmit: (params: { file: File; name: string; description: string; evalType: EvalType }) => void;
}

function getExtension(filename: string): string {
  const idx = filename.lastIndexOf('.');
  return idx >= 0 ? filename.slice(idx + 1).toLowerCase() : '';
}

function UploadModal({ uploadStatus, uploadError, onClose, onSubmit }: UploadModalProps) {
  const [evalType, setEvalType] = useState<EvalType>('model');
  const [name, setName] = useState('');
  const [description, setDescription] = useState('');
  const [file, setFile] = useState<File | null>(null);
  const [localError, setLocalError] = useState<string | null>(null);
  const fileInputRef = useRef<HTMLInputElement>(null);

  const isLoading = uploadStatus === 'loading';
  const ext = file ? getExtension(file.name) : '';
  const isJsonl = ext === 'jsonl';

  const pickFile = (f: File | null) => {
    setLocalError(null);
    if (!f) {
      setFile(null);
      return;
    }
    const fExt = getExtension(f.name);
    if (!SUPPORTED_UPLOAD_EXTENSIONS.includes(fExt)) {
      setFile(null);
      setLocalError(`Unsupported file type ".${fExt || '?'}". Please choose a .json, .jsonl, .arrow, or .parquet file.`);
      return;
    }
    setFile(f);
  };

  const handleSubmit = () => {
    setLocalError(null);
    if (!name.trim()) return setLocalError('Please enter a dataset name.');
    if (!description.trim()) return setLocalError('Please enter a description.');
    if (!file) return setLocalError('Please choose a file to upload.');
    onSubmit({ file, name: name.trim(), description: description.trim(), evalType });
  };

  const displayError = localError || (uploadStatus === 'failed' ? uploadError : null);

  return (
    <div className={styles['datasets__modal-overlay']}>
      <div
        className={styles['datasets__modal']}
        role="dialog"
        aria-modal="true"
        aria-label="Upload dataset"
      >
        <div className={styles['datasets__modal-hdr']}>
          <div>
            <p className={styles['datasets__modal-eyebrow']}>New test suite</p>
            <h3>Upload dataset</h3>
          </div>
          <button className={styles['datasets__modal-close']} onClick={onClose} disabled={isLoading} aria-label="Close">
            <X size={16} />
          </button>
        </div>

        <div className={styles['datasets__modal-body']}>
          <div className={styles['datasets__field']}>
            <label className={styles['datasets__field-label']}>Eval type</label>
            <div className={styles['datasets__eval-pills']}>
              {EVAL_TYPE_OPTIONS.map((opt) => (
                <button
                  key={opt.value}
                  type="button"
                  className={`${styles['datasets__eval-pill']} ${evalType === opt.value ? styles['datasets__eval-pill--on'] : ''}`}
                  onClick={() => setEvalType(opt.value)}
                  disabled={isLoading}
                >
                  {opt.label}
                </button>
              ))}
            </div>
          </div>

          <div className={styles['datasets__field']}>
            <label className={styles['datasets__field-label']} htmlFor="ds-upload-name">Name</label>
            <input
              id="ds-upload-name"
              className={styles['datasets__field-input']}
              placeholder="e.g. Internal QA Set v2"
              value={name}
              onChange={(e) => setName(e.target.value)}
              disabled={isLoading}
            />
          </div>

          <div className={styles['datasets__field']}>
            <label className={styles['datasets__field-label']} htmlFor="ds-upload-desc">Description</label>
            <textarea
              id="ds-upload-desc"
              className={styles['datasets__field-textarea']}
              placeholder="What does this dataset cover?"
              value={description}
              onChange={(e) => setDescription(e.target.value)}
              disabled={isLoading}
              rows={3}
            />
          </div>

          <div className={styles['datasets__field']}>
            <label className={styles['datasets__field-label']}>File</label>
            <button
              type="button"
              className={styles['datasets__dropzone']}
              onClick={() => fileInputRef.current?.click()}
              disabled={isLoading}
            >
              <FileUp size={18} />
              <span className={styles['datasets__dropzone-text']}>
                {file ? file.name : 'Choose a file to upload'}
              </span>
              <span className={styles['datasets__dropzone-hint']}>
                .json &nbsp;·&nbsp; .jsonl &nbsp;·&nbsp; .arrow &nbsp;·&nbsp; .parquet
              </span>
            </button>
            <input
              ref={fileInputRef}
              type="file"
              accept={ACCEPT_ATTR}
              className={styles['datasets__file-input']}
              onChange={(e) => pickFile(e.target.files?.[0] ?? null)}
              disabled={isLoading}
            />
            {isJsonl && evalType === 'agent' && (
              <p className={styles['datasets__field-note']}>
                JSONL uploads for Agent evals are automatically tagged with category “Agents”.
              </p>
            )}
          </div>

          {displayError && (
            <div className={styles['datasets__modal-error']}>
              <AlertCircle size={14} />
              {displayError}
            </div>
          )}

          {uploadStatus === 'succeeded' && (
            <div className={styles['datasets__modal-success']}>
              <CheckCircle2 size={14} />
              Dataset uploaded — refreshing the library…
            </div>
          )}
        </div>

        <div className={styles['datasets__modal-foot']}>
          <button className={styles['datasets__modal-cancel']} onClick={onClose} disabled={isLoading}>
            Cancel
          </button>
          <button className={styles['datasets__modal-submit']} onClick={handleSubmit} disabled={isLoading}>
            {isLoading ? <Loader2 size={14} className={styles['datasets__spin']} /> : <Upload size={14} />}
            {isLoading ? 'Uploading…' : 'Upload dataset'}
          </button>
        </div>
      </div>
    </div>
  );
}

// ---------------------------------------------------------------------------
// Delete confirmation modal
// ---------------------------------------------------------------------------

interface DeleteConfirmModalProps {
  dataset: TestSuite;
  isDeleting: boolean;
  deleteError: string | null;
  onCancel: () => void;
  onConfirm: () => void;
}

function DeleteConfirmModal({ dataset, isDeleting, deleteError, onCancel, onConfirm }: DeleteConfirmModalProps) {
  const name = dataset.name || 'this dataset';

  return (
    <div className={styles['datasets__modal-overlay']}>
      <div
        className={styles['datasets__modal']}
        role="dialog"
        aria-modal="true"
        aria-label="Delete dataset"
      >
        <div className={styles['datasets__modal-hdr']}>
          <div>
            <p className={`${styles['datasets__modal-eyebrow']} ${styles['datasets__modal-eyebrow--danger']}`}>
              This can't be undone
            </p>
            <h3>Delete dataset</h3>
          </div>
          <button className={styles['datasets__modal-close']} onClick={onCancel} disabled={isDeleting} aria-label="Close">
            <X size={16} />
          </button>
        </div>

        <div className={styles['datasets__modal-body']}>
          <p className={styles['datasets__delete-copy']}>
            Are you sure you want to delete <strong>{name}</strong>? This will permanently remove
            the dataset and it will no longer be available for evaluations.
          </p>

          {deleteError && (
            <div className={styles['datasets__modal-error']}>
              <AlertCircle size={14} />
              {deleteError}
            </div>
          )}
        </div>

        <div className={styles['datasets__modal-foot']}>
          <button className={styles['datasets__modal-cancel']} onClick={onCancel} disabled={isDeleting}>
            Cancel
          </button>
          <button className={styles['datasets__modal-danger']} onClick={onConfirm} disabled={isDeleting}>
            {isDeleting ? <Loader2 size={14} className={styles['datasets__spin']} /> : <Trash2 size={14} />}
            {isDeleting ? 'Deleting…' : 'Delete dataset'}
          </button>
        </div>
      </div>
    </div>
  );
}

// ---------------------------------------------------------------------------
// Question preview drawer — slides in from the right, same pattern as
// History's test-detail/metric-score drawer: a separate fade-only overlay,
// a fixed panel pinned to the right edge that translates in on open, its
// own header + scrollable body, and a pagination footer.
// ---------------------------------------------------------------------------

interface PreviewDrawerProps {
  dataset: TestSuite;
  preview: PreviewState;
  onClose: () => void;
  onChangeOffset: (offset: number) => void;
  onChangeLimit: (limit: number) => void;
  onOpenImage: (imagePath: string) => void;
}

function PreviewDrawer({ dataset, preview, onClose, onChangeOffset, onChangeLimit, onOpenImage }: PreviewDrawerProps) {
  const { questions, total, limit, offset, status, error } = preview;
  const safeLimit = limit > 0 ? limit : 20;
  const totalPages = Math.max(1, Math.ceil(total / safeLimit));
  const currentPage = Math.floor(offset / safeLimit) + 1;
  const rangeStart = total === 0 ? 0 : offset + 1;
  const rangeEnd = Math.min(offset + safeLimit, total);
  const isLoading = status === 'loading';

  return (
    <>
      <div className={styles['datasets__preview-header']}>
        <div>
          <p className={styles['datasets__preview-eyebrow']}>Question preview</p>
          <h3 className={styles['datasets__preview-title']}>{dataset.name || 'Untitled dataset'}</h3>
          <div className={styles['datasets__preview-sub']}>
            {total.toLocaleString()} question{total === 1 ? '' : 's'} total
          </div>
        </div>
        <button type="button" className={styles['datasets__preview-close']} onClick={onClose} title="Close">
          <X size={16} />
        </button>
      </div>

      <div className={styles['datasets__preview-body']}>
        {isLoading && questions.length === 0 && (
          <div className={styles['datasets__preview-loading']}>
            <Loader2 size={16} className={styles['datasets__spin']} /> Loading questions…
          </div>
        )}

        {status === 'failed' && questions.length === 0 && (
          <div className={styles['datasets__preview-loading']}>
            <AlertCircle size={16} /> {error || 'Failed to load questions.'}
          </div>
        )}

        {status === 'succeeded' && questions.length === 0 && (
          <div className={styles['datasets__preview-loading']}>
            <HelpCircle size={16} /> No questions found for this dataset.
          </div>
        )}

        {questions.map((q, i) => {
          const prompt = q?.input?.prompt;
          const answer = q?.expected?.answer;
          const language = q?.input?.language;
          const images = (q?.input?.images ?? []).filter(Boolean) as string[];
          return (
            <div key={q?.id ?? i} className={styles['datasets__preview-card']}>
              <div className={styles['datasets__preview-card-hdr']}>
                <span className={styles['datasets__preview-card-num']}>Q{offset + i + 1}</span>
                <span className={styles['datasets__preview-card-tags']}>
                  {language && <span className={styles['datasets__preview-card-lang']}>{language}</span>}
                  {q?.category && <span className={styles['datasets__preview-card-cat']}>{q.category}</span>}
                </span>
              </div>
              <div className={styles['datasets__preview-field']}>
                <span className={styles['datasets__preview-field-label']}>Prompt</span>
                <div className={styles['datasets__preview-field-text']}>{prompt || '—'}</div>
              </div>

              {images.length > 0 && (
                <div className={styles['datasets__preview-field']}>
                  <span className={styles['datasets__preview-field-label']}>
                    Image{images.length === 1 ? '' : `s (${images.length})`}
                  </span>
                  <div className={styles['datasets__preview-image-list']}>
                    {images.map((imgPath, imgIdx) => (
                      <button
                        key={imgPath || imgIdx}
                        type="button"
                        className={styles['datasets__preview-image-chip']}
                        onClick={() => onOpenImage(imgPath)}
                        title={`Preview ${imageBasename(imgPath)}`}
                      >
                        <ImageIcon size={13} />
                        <span>{imageBasename(imgPath)}</span>
                      </button>
                    ))}
                  </div>
                </div>
              )}

              <div className={styles['datasets__preview-field']}>
                <span className={styles['datasets__preview-field-label']}>Expected answer</span>
                <div
                  className={`${styles['datasets__preview-field-text']} ${answer == null ? styles['datasets__preview-field-text--empty'] : ''}`}
                >
                  {answer ?? 'No expected answer'}
                </div>
              </div>
            </div>
          );
        })}
      </div>

      {total > 0 && (
        <div className={styles['datasets__preview-pagination']}>
          <div className={styles['datasets__preview-pagination-info']}>
            {rangeStart}–{rangeEnd} of {total}
          </div>
          <div className={styles['datasets__preview-pagination-controls']}>
            <select
              className={styles['datasets__preview-size-select']}
              value={safeLimit}
              onChange={(e) => onChangeLimit(Number(e.target.value))}
              disabled={isLoading}
              title="Questions per page"
            >
              {PREVIEW_PAGE_SIZE_OPTIONS.map((n) => (
                <option key={n} value={n}>{n} / page</option>
              ))}
            </select>
            <button
              type="button"
              className={styles['datasets__preview-nav-btn']}
              onClick={() => onChangeOffset(Math.max(0, offset - safeLimit))}
              disabled={offset <= 0 || isLoading}
              title="Previous page"
            >
              <ChevronLeft size={14} />
            </button>
            <span className={styles['datasets__preview-page-label']}>{currentPage} / {totalPages}</span>
            <button
              type="button"
              className={styles['datasets__preview-nav-btn']}
              onClick={() => onChangeOffset(offset + safeLimit)}
              disabled={currentPage >= totalPages || isLoading}
              title="Next page"
            >
              <ChevronRight size={14} />
            </button>
          </div>
        </div>
      )}
    </>
  );
}

// ---------------------------------------------------------------------------
// MinIO image lightbox — mirrors the MinIO console's own object-preview
// experience: a near-black backdrop, the image centered and scaled to fit
// the viewport, a slim header bar with the filename and a download action,
// and the image itself only fetched once this is actually opened (never
// preloaded with the question list).
// ---------------------------------------------------------------------------

interface ImageLightboxProps {
  imagePath: string;
  onClose: () => void;
}

function ImageLightbox({ imagePath, onClose }: ImageLightboxProps) {
  const [blobUrl, setBlobUrl] = useState<string | null>(null);
  const [status, setStatus] = useState<'loading' | 'succeeded' | 'failed'>('loading');
  const [error, setError] = useState<string | null>(null);
  const [reloadKey, setReloadKey] = useState(0);

  useEffect(() => {
    let cancelled = false;
    let objectUrl: string | null = null;

    setStatus('loading');
    setError(null);

    testSuitesApi
      .getImageBlobUrl(imagePath)
      .then((url) => {
        if (cancelled) {
          URL.revokeObjectURL(url);
          return;
        }
        objectUrl = url;
        setBlobUrl(url);
        setStatus('succeeded');
      })
      .catch((err) => {
        if (cancelled) return;
        setError(err instanceof Error ? err.message : 'Failed to load image.');
        setStatus('failed');
      });

    return () => {
      cancelled = true;
      if (objectUrl) URL.revokeObjectURL(objectUrl);
    };
  }, [imagePath, reloadKey]);

  const filename = imageBasename(imagePath);

  return (
    <div className={styles['datasets__lightbox-overlay']} onClick={onClose}>
      <div className={styles['datasets__lightbox']} onClick={(e) => e.stopPropagation()}>
        <div className={styles['datasets__lightbox-hdr']}>
          <div className={styles['datasets__lightbox-hdr-info']}>
            <ImageIcon size={14} />
            <span className={styles['datasets__lightbox-filename']} title={imagePath}>{filename}</span>
          </div>
          <div className={styles['datasets__lightbox-hdr-actions']}>
            {blobUrl && (
              <a
                className={styles['datasets__lightbox-action']}
                href={blobUrl}
                download={filename}
                title="Download image"
              >
                <Download size={15} />
              </a>
            )}
            <button
              type="button"
              className={styles['datasets__lightbox-action']}
              onClick={onClose}
              title="Close"
            >
              <X size={16} />
            </button>
          </div>
        </div>

        <div className={styles['datasets__lightbox-stage']}>
          {status === 'loading' && (
            <div className={styles['datasets__lightbox-state']}>
              <Loader2 size={22} className={styles['datasets__spin']} />
              <span>Loading image…</span>
            </div>
          )}

          {status === 'failed' && (
            <div className={styles['datasets__lightbox-state']}>
              <AlertCircle size={22} />
              <span>{error || "Couldn't load this image."}</span>
              <button
                type="button"
                className={styles['datasets__lightbox-retry']}
                onClick={() => setReloadKey((k) => k + 1)}
              >
                <RotateCw size={13} /> Retry
              </button>
            </div>
          )}

          {status === 'succeeded' && blobUrl && (
            <img className={styles['datasets__lightbox-img']} src={blobUrl} alt={filename} />
          )}
        </div>
      </div>
    </div>
  );
}


















//Datasets.module.scss
//Datasets.module.scss
@use 'sass:color';
@use '../../styles/_variables' as *;

// ===========================================================================
// Datasets (Test Suite Library) — master/detail library browser.
// Header + toolbar are unchanged from the original. The body below replaces
// the card grid + modal with a scrollable list rail and a rich detail pane.
// Uses the app's existing font variables ($font-mono / $font-body /
// $font-display) — no font-family is introduced or overridden here.
// ===========================================================================

// ---------------------------------------------------------------------------
// Color tokens: intentionally NOT redeclared here. $ink, $ink-2, $ink-3,
// $paper, $card, $line, $line-2, $signal, $signal-2, $wash, $ok, $danger
// already come from the shared "ink" system in _variables.scss via the
// `@use` above, where the neutrals resolve to the themed CSS custom
// properties in _theme.scss and the accents are flat constants shared
// across light/dark. Redeclaring them locally as flat hex (as this file
// previously did) would silently break dark-mode theming for this
// component while looking identical in light mode. If a color this
// component needs doesn't exist in _variables.scss yet, add it there
// (and to _theme.scss if it should be theme-aware) rather than
// hardcoding it here — that's the single source of truth for the ink
// system.
// ---------------------------------------------------------------------------

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;

$soft: 0 1px 2px rgba(20, 22, 27, 0.05);

// Base font-size the whole Datasets component's internal `em` scale is
// built on. All descendant font-sizes in this file are expressed in `em`
// relative to this, so bumping it (e.g. on wide screens below) scales the
// whole component proportionally from one place — same convention as
// Model Catalog / Sidebar / Providers / History.
$datasets-base-font: 0.8125rem;

%micro {
  font-family: $mono;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}

.datasets {
  // master scale control — every em-based font-size below responds to this
  font-size: $datasets-base-font;

  @media (min-width: 1800px) {
    font-size: 1rem;
  }

  // ---- header (unchanged) ---------------------------------------------------
  &__header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    gap: 1rem;
    padding: 24px 32px 20px;
    margin-bottom: 20px;
    border-bottom: 1px solid $line;
    background: $card;

    h1 {
      font-family: $display;
      font-size: 1.8462em; // 1.5rem / 0.8125rem
      font-weight: 800;
      letter-spacing: -0.02em;
      color: $ink;
      line-height: 1.2;
    }
  }

  &__header-eyebrow {
    @extend %micro;
    display: flex;
    align-items: center;
    gap: 8px;
    color: $signal;
    margin-bottom: 6px;

    &::before {
      content: '';
      width: 16px;
      height: 2px;
      border-radius: 2px;
      background: $signal;
    }
  }

  &__header-sub {
    margin-top: 4px;
    font-size: 1.0385em; // 0.84375rem / 0.8125rem
    color: $ink-2;
    max-width: 52ch;
  }

  &__header-meta {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 10px;
    flex-wrap: wrap;
    margin-bottom: 3px;
  }

  &__header-count {
    display: inline-flex;
    align-items: center;
    gap: 7px;
    padding: 7px 13px;
    border-radius: 999px;
    border: 1px solid $line;
    background: $paper;
    font-family: $mono;
    font-size: 0.8846em; // 0.71875rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: $ink-2;
    white-space: nowrap;
  }

  &__refresh-btn {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 8px 13px;
    border: 1px solid $line;
    border-radius: 999px;
    background: $card;
    color: $ink-2;
    font-family: $sans;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

    &:hover { border-color: $ink-3; color: $ink; background: $paper; }
  }

  &__upload-btn {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 8px 14px;
    border: 1px solid transparent;
    border-radius: 999px;
    background: $signal;
    color: #fff;
    font-family: $sans;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: background 0.15s ease, transform 0.15s ease;

    &:hover { background: $signal-2; }
  }

  // ---- toolbar (unchanged) --------------------------------------------------
  &__toolbar {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 14px;
    padding: 14px 32px;
    background: $card;
    border-bottom: 1px solid $line;
    flex-wrap: wrap;
  }

  &__search {
    position: relative;
    flex: 1;
    max-width: 360px;
    min-width: 200px;

    svg {
      position: absolute;
      top: 50%;
      left: 13px;
      transform: translateY(-50%);
      color: $ink-3;
      pointer-events: none;
    }

    input {
      width: 100%;
      border: 1.5px solid $line;
      border-radius: 10px;
      padding: 9px 12px 9px 38px;
      font-size: 1.0385em; // 0.84375rem / 0.8125rem
      font-family: $sans;
      color: $ink;
      background: $paper;
      transition: border-color 0.15s ease, background 0.15s ease;

      &::placeholder { color: $ink-3; }
      &:focus { outline: none; border-color: $signal; background: $card; }
    }
  }

  &__toolbar-label {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 5px 8px 5px 9px;
    color: $ink-3;
  }

  &__filters {
    display: inline-flex;
    align-items: center;
    gap: 4px;
    padding: 4px;
    background: $paper;
    border: 1px solid $line;
    border-radius: 999px;
    flex-wrap: wrap;
  }

  &__filter-pill {
    padding: 6px 13px;
    border: 0;
    border-radius: 999px;
    background: transparent;
    color: $ink-2;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: all 0.15s ease;

    &:hover { color: $ink; }

    &--on {
      background: $card;
      color: $signal;
      box-shadow: $soft;
    }
  }

  // ---- capability facet bar -------------------------------------------------
  &__facets {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 8px;
    flex-wrap: wrap;
    padding: 10px 32px;
    background: $wash;
    border-bottom: 1px solid $line;
    color: $signal-2;
  }

  &__facets-lead {
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    font-weight: 650;
    color: $ink-2;
  }

  &__facet {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 4px 6px 4px 10px;
    border: 0;
    border-radius: 999px;
    font-family: $mono;
    font-size: 0.8462em; // 0.6875rem / 0.8125rem
    font-weight: 700;
    cursor: pointer;
    transition: filter 0.12s ease;

    &:hover { filter: brightness(0.96); }
  }

  &__facets-clear {
    margin-left: 2px;
    border: 0;
    background: none;
    color: $signal;
    font-family: $sans;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    font-weight: 700;
    cursor: pointer;

    &:hover { text-decoration: underline; }
  }

  // ---- body split -----------------------------------------------------------
  // NOTE: relies on the page shell (.pg-shell) being a flex column that gives
  // this element the remaining height. If your shell differs, set a height /
  // min-height on &__body instead of flex:1.
  &__body {
    flex: 1;
    min-height: 0;
    display: grid;
    grid-template-columns: minmax(320px, 380px) 1fr;
  }

  // ---- list rail ------------------------------------------------------------
  &__rail {
    display: flex;
    flex-direction: column;
    min-height: 0;
    border-right: 1px solid $line;
    background: $card;
  }

  &__rail-head {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 14px 20px;
    border-bottom: 1px solid $line-2;
    font-family: $mono;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: $ink-3;
  }

  &__rail-hint {
    display: inline-flex;
    align-items: center;
    gap: 5px;
  }

  &__rail-scroll {
    flex: 1;
    overflow-y: auto;
    padding: 12px;
    display: flex;
    flex-direction: column;
    gap: 10px;
  }

  &__row {
    text-align: left;
    width: 100%;
    cursor: pointer;
    background: $card;
    border: 1px solid $line;
    border-left-width: 4px;          // colored per type via inline borderLeftColor
    border-radius: 13px;
    padding: 14px 16px 14px 17px;
    display: flex;
    flex-direction: column;
    gap: 10px;
    font-family: $sans;
    transition: border-color 0.14s ease, box-shadow 0.14s ease, transform 0.14s ease, background 0.14s ease;

    &:hover {
      border-color: $ink-3;
      box-shadow: $soft;
      transform: translateY(-1px);
    }

    &--on {
      border-color: $signal;
      background: $wash;
      box-shadow: $soft;
    }
  }

  &__row-top {
    display: flex;
    align-items: baseline;
    justify-content: space-between;
    gap: 10px;
  }

  &__row-name {
    font-family: $display;
    font-size: 1.2308em; // 1rem / 0.8125rem
    font-weight: 650;
    color: $ink;
    line-height: 1.3;
  }

  &__row-count {
    font-family: $mono;
    font-size: 1.0769em; // 0.875rem / 0.8125rem
    font-weight: 700;
    color: $ink-3;
  }

  &__row-foot {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
  }

  &__row-type-group {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    min-width: 0;
  }

  &__row-type {
    font-family: $mono;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
  }

  &__row-eval-tag {
    font-family: $mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: $ink-3;
    background: $paper;
    border: 1px solid $line;
    border-radius: 999px;
    padding: 1px 7px;
    white-space: nowrap;
  }

  &__row-dots {
    display: inline-flex;
    gap: 5px;
    flex-shrink: 0;

    i { width: 7px; height: 7px; border-radius: 99px; display: block; }
  }

  &__empty-rail {
    margin: auto;
    text-align: center;
    color: $ink-3;
    padding: 40px 16px;

    svg { margin-bottom: 8px; }
    p { font-size: 1em; /* 0.8125rem / 0.8125rem (base) */ line-height: 1.5; margin: 0; }
  }

  // ---- loading skeleton -----------------------------------------------------
  &__skel-row {
    padding: 15px 17px;
    display: flex;
    flex-direction: column;
    gap: 10px;
  }

  &__skel {
    display: block;
    height: 11px;
    border-radius: 6px;
    background: linear-gradient(90deg, $line-2 25%, $paper 50%, $line-2 75%);
    background-size: 200% 100%;
    animation: datasets-shimmer 1.2s ease-in-out infinite;
  }

  // ---- detail ---------------------------------------------------------------
  &__detail {
    min-height: 0;
    display: flex;
    flex-direction: column;
  }

  &__detail-scroll {
    flex: 1;
    overflow-y: auto;
    padding: 26px 30px 40px;
    animation: datasets-detail-in 0.22s ease;
  }

  &__detail-empty {
    margin: auto;
    text-align: center;
    color: $ink-3;
    max-width: 280px;

    svg { margin-bottom: 10px; }
    p { font-size: 1.0769em; /* 0.875rem / 0.8125rem */ line-height: 1.5; }
  }

  &__hero {
    position: relative;
    padding-left: 18px;
    margin-bottom: 22px;
  }

  &__hero-bar {
    position: absolute;
    left: 0;
    top: 4px;
    bottom: 4px;
    width: 4px;
    border-radius: 3px;
  }

  &__hero-top {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 16px;
  }

  &__hero-type {
    font-family: $mono;
    font-size: 0.8462em; // 0.6875rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.1em;
    text-transform: uppercase;
  }

  &__hero {
    h2 {
      font-family: $display;
      font-size: 1.8462em; // 1.5rem / 0.8125rem
      font-weight: 700;
      letter-spacing: -0.02em;
      margin: 5px 0 0;
      line-height: 1.15;
      color: $ink;
    }
  }

  &__hero-desc {
    margin: 14px 0 0;
    font-size: 1.1538em; // 0.9375rem / 0.8125rem
    line-height: 1.6;
    color: $ink-2;
  }

  &__preview-unavailable {
    display: flex;
    align-items: flex-start;
    gap: 11px;
    margin-top: 16px;
    padding: 13px 15px;
    border: 1px dashed $line;
    border-radius: 12px;
    background: $paper;

    strong {
      display: block;
      font-size: 0.9231em; // 0.75rem / 0.8125rem
      font-weight: 700;
      color: $ink;
    }

    span {
      display: block;
      margin-top: 3px;
      font-size: 0.8846em; // 0.71875rem / 0.8125rem
      line-height: 1.5;
      color: $ink-3;
    }
  }

  &__preview-unavailable-icon {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 32px;
    height: 32px;
    border-radius: 10px;
    background: $ink-wash;
    color: $ink-3;
  }

  &__source-badge {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    font-family: $mono;
    font-size: 0.8462em; // 0.6875rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: $ink-2;
    padding: 5px 10px;
    border: 1px solid $line;
    border-radius: 999px;
    background: $paper;
    white-space: nowrap;
  }

  &__eval-badge {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    font-family: $mono;
    font-size: 0.8462em; // 0.6875rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: $signal;
    padding: 5px 10px;
    border: 1px solid rgba($signal, 0.22);
    border-radius: 999px;
    background: $wash;
    white-space: nowrap;
  }

  &__hero-actions {
    flex-shrink: 0;
    display: inline-flex;
    align-items: flex-end;
    gap: 10px;
    flex-wrap: wrap;
  }

  &__hero-badge-group {
    display: inline-flex;
    flex-direction: column;
    align-items: flex-start;
    gap: 4px;
  }

  &__hero-badge-label {
    font-family: $mono;
    font-size: 0.6154em; // 0.5rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.08em;
    text-transform: uppercase;
    color: $ink-3;
    padding-left: 2px;
  }

  &__preview-btn {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 7px 14px;
    border-radius: 999px;
    border: 1px solid transparent;
    background: $signal;
    color: #fff;
    font-family: $sans;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: background 0.15s ease, transform 0.15s ease, box-shadow 0.15s ease;
    box-shadow: 0 4px 12px -6px rgba(43, 43, 245, 0.6);

    &:hover { background: $signal-2; transform: translateY(-1px); }
  }

  &__delete-btn {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 30px;
    height: 30px;
    border-radius: 999px;
    border: 1px solid $line;
    background: $paper;
    color: $ink-3;
    cursor: pointer;
    transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

    &:hover { border-color: rgba($danger, 0.3); color: $danger; background: $danger-wash; }
  }

  &__stats {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 1px;
    background: $line;
    border: 1px solid $line;
    border-radius: 14px;
    overflow: hidden;
    margin-bottom: 26px;
  }

  &__stat {
    background: $card;
    padding: 15px 16px;
    display: flex;
    flex-direction: column;
    gap: 3px;
  }

  &__stat-val {
    font-family: $display;
    font-size: 1.6923em; // 1.375rem / 0.8125rem
    font-weight: 700;
    color: $ink;
    letter-spacing: -0.02em;
    line-height: 1;

    &--mono { font-family: $mono; font-size: 1.1538em; /* 0.9375rem / 0.8125rem */ font-weight: 700; }
  }

  &__stat-label {
    font-family: $mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.1em;
    text-transform: uppercase;
    color: $ink-3;
  }

  &__section {
    margin-bottom: 26px;
  }

  &__section-head {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
    margin-bottom: 12px;

    h3 {
      font-family: $display;
      font-size: 1.1538em; // 0.9375rem / 0.8125rem
      font-weight: 700;
      color: $ink;
      margin: 0;
      display: flex;
      align-items: center;
      gap: 8px;

      em {
        font-family: $mono;
        font-style: normal;
        font-size: 0.8462em; // 0.6875rem / 0.8125rem
        font-weight: 700;
        color: $ink-3;
        background: $paper;
        border: 1px solid $line;
        border-radius: 99px;
        padding: 2px 8px;
      }
    }
  }

  &__section-hint {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    font-size: 0.8846em; // 0.71875rem / 0.8125rem
    color: $ink-3;
  }

  &__caps {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
  }

  &__cap {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 6px 12px;
    border: 1px solid;
    border-radius: 8px;
    font-family: $mono;
    font-size: 0.8846em; // 0.71875rem / 0.8125rem
    font-weight: 700;
    cursor: pointer;
    transition: transform 0.13s ease;

    &:hover { transform: translateY(-1px); }

    &--active { box-shadow: $soft; }
  }

  &__single {
    display: flex;
    gap: 11px;
    align-items: flex-start;
    padding: 14px 16px;
    border: 1px dashed $line;
    border-radius: 12px;
    background: $paper;
    color: $ink-3;

    strong { display: block; font-size: 1em; /* 0.8125rem / 0.8125rem (base) */ color: $ink; font-weight: 650; }
    span { display: block; font-size: 0.9615em; /* 0.78125rem / 0.8125rem */ color: $ink-2; margin-top: 2px; }
  }

  // ---- state banner (error) -------------------------------------------------
  &__state {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 12px;
    padding: 48px 24px;
    margin: 24px 32px;
    border: 1px dashed $line;
    border-radius: 16px;
    background: $paper;
    color: $ink-2;
    font-size: 1.0769em; // 0.875rem / 0.8125rem
    text-align: center;

    svg { color: $ink-3; }
  }

  &__state--error svg { color: $danger; }

  // ---- upload dataset modal --------------------------------------------------
  &__modal-overlay {
    position: fixed;
    inset: 0;
    z-index: 60;
    background: rgba(20, 22, 27, 0.45);
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 24px;
    animation: datasets-fade-in 0.15s ease;
  }

  &__modal {
    width: min(460px, 100%);
    max-height: 88vh;
    display: flex;
    flex-direction: column;
    background: $card;
    border: 1px solid $line;
    border-radius: 18px;
    box-shadow: 0 24px 60px -20px rgba(20, 22, 27, 0.4);
    overflow: hidden;
    animation: datasets-modal-in 0.18s cubic-bezier(0.22, 1, 0.36, 1);
  }

  &__modal-hdr {
    flex-shrink: 0;
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 12px;
    padding: 20px 20px 16px;
    border-bottom: 1px solid $line;

    h3 {
      font-family: $display;
      font-size: 1.3846em; // 1.125rem / 0.8125rem
      font-weight: 700;
      color: $ink;
      margin: 3px 0 0;
    }
  }

  &__modal-eyebrow {
    @extend %micro;
    color: $signal;

    &--danger { color: $danger; }
  }

  &__modal-close {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 28px;
    height: 28px;
    border-radius: 8px;
    border: 1px solid $line;
    background: $paper;
    color: $ink-2;
    cursor: pointer;
    transition: border-color 0.15s ease, color 0.15s ease;

    &:hover:not(:disabled) { border-color: $ink-3; color: $ink; }
    &:disabled { opacity: 0.4; cursor: not-allowed; }
  }

  &__modal-body {
    flex: 1;
    overflow-y: auto;
    padding: 18px 20px 4px;
    display: flex;
    flex-direction: column;
    gap: 16px;
  }

  &__field {
    display: flex;
    flex-direction: column;
    gap: 7px;
  }

  &__field-label {
    font-family: $mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: $ink-3;
  }

  &__field-input,
  &__field-textarea {
    width: 100%;
    border: 1.5px solid $line;
    border-radius: 10px;
    padding: 9px 12px;
    font-size: 1em; /* 0.8125rem / 0.8125rem (base) */
    font-family: $sans;
    color: $ink;
    background: $paper;
    transition: border-color 0.15s ease, background 0.15s ease;
    resize: none;

    &::placeholder { color: $ink-3; }
    &:focus { outline: none; border-color: $signal; background: $card; }
    &:disabled { opacity: 0.6; cursor: not-allowed; }
  }

  &__field-note {
    font-size: 0.8846em; // 0.71875rem / 0.8125rem
    color: $ink-3;
    line-height: 1.5;
  }

  &__eval-pills {
    display: inline-flex;
    align-items: center;
    gap: 4px;
    padding: 4px;
    background: $paper;
    border: 1px solid $line;
    border-radius: 999px;
    width: fit-content;
  }

  &__eval-pill {
    padding: 6px 16px;
    border: 0;
    border-radius: 999px;
    background: transparent;
    color: $ink-2;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: all 0.15s ease;

    &:hover:not(:disabled) { color: $ink; }
    &:disabled { cursor: not-allowed; opacity: 0.6; }

    &--on {
      background: $card;
      color: $signal;
      box-shadow: $soft;
    }
  }

  &__dropzone {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 4px;
    width: 100%;
    padding: 20px 16px;
    border: 1.5px dashed $line;
    border-radius: 12px;
    background: $paper;
    color: $ink-2;
    cursor: pointer;
    text-align: center;
    transition: border-color 0.15s ease, background 0.15s ease;

    &:hover:not(:disabled) { border-color: $signal; background: $wash; }
    &:disabled { opacity: 0.6; cursor: not-allowed; }

    svg { color: $ink-3; margin-bottom: 2px; }
  }

  &__dropzone-text {
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    font-weight: 650;
    color: $ink;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    max-width: 100%;
  }

  &__dropzone-hint {
    font-family: $mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    color: $ink-3;
  }

  &__file-input {
    display: none;
  }

  &__modal-error {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 10px 12px;
    border-radius: 10px;
    background: $danger-wash;
    color: $danger;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    font-weight: 600;

    svg { flex-shrink: 0; }
  }

  &__modal-success {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 10px 12px;
    border-radius: 10px;
    background: $ok-wash;
    color: $ok;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    font-weight: 600;

    svg { flex-shrink: 0; }
  }

  &__modal-foot {
    flex-shrink: 0;
    display: flex;
    justify-content: flex-end;
    gap: 10px;
    padding: 16px 20px 20px;
  }

  &__modal-cancel {
    padding: 9px 16px;
    border: 1px solid $line;
    border-radius: 10px;
    background: $card;
    color: $ink-2;
    font-family: $sans;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

    &:hover:not(:disabled) { border-color: $ink-3; color: $ink; background: $paper; }
    &:disabled { opacity: 0.5; cursor: not-allowed; }
  }

  &__modal-submit {
    display: inline-flex;
    align-items: center;
    gap: 7px;
    padding: 9px 18px;
    border: 1px solid transparent;
    border-radius: 10px;
    background: $signal;
    color: #fff;
    font-family: $sans;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: background 0.15s ease;

    &:hover:not(:disabled) { background: $signal-2; }
    &:disabled { opacity: 0.6; cursor: not-allowed; }
  }

  &__modal-danger {
    display: inline-flex;
    align-items: center;
    gap: 7px;
    padding: 9px 18px;
    border: 1px solid transparent;
    border-radius: 10px;
    background: $danger;
    color: #fff;
    font-family: $sans;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: background 0.15s ease;

    &:hover:not(:disabled) { background: color.scale($danger, $lightness: -8%); }
    &:disabled { opacity: 0.6; cursor: not-allowed; }
  }

  &__delete-copy {
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    line-height: 1.6;
    color: $ink-2;

    strong { color: $ink; font-weight: 700; }
  }

  &__spin {
    animation: datasets-spin 0.8s linear infinite;
  }

  // ---- question preview slide-over drawer ------------------------------------
  // Same pattern as History's .drawer-overlay/.drawer: a fade-only scrim
  // plus a fixed panel pinned to the right edge that translates in on open.
  &__preview-overlay {
    position: fixed;
    inset: 0;
    // Deliberately hardcoded, not themed — an overlay scrim must stay dark
    // in both themes; the themed $ink would go near-white in dark mode.
    background: rgba(20, 22, 27, 0.32);
    opacity: 0;
    pointer-events: none;
    transition: opacity 0.22s ease;
    z-index: 55;

    &--open { opacity: 1; pointer-events: auto; }
  }

  &__preview-drawer {
    position: fixed;
    top: 0;
    right: 0;
    height: 100%;
    width: 480px;
    max-width: 92vw;
    background: $card;
    border-left: 1px solid $line;
    box-shadow: -18px 0 40px -20px rgba(20, 22, 27, 0.35);
    transform: translateX(100%);
    transition: transform 0.26s cubic-bezier(0.22, 1, 0.36, 1);
    z-index: 56;
    display: flex;
    flex-direction: column;
    min-height: 0;

    &--open { transform: translateX(0); }
  }

  &__preview-header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 12px;
    padding: 20px 20px 16px;
    border-bottom: 1px solid $line;
  }

  &__preview-eyebrow {
    @extend %micro;
    color: $signal;
    margin-bottom: 6px;
  }

  &__preview-title {
    font-family: $display;
    font-size: 1.3846em; // 1.125rem / 0.8125rem
    font-weight: 800;
    letter-spacing: -0.01em;
    color: $ink;
  }

  &__preview-sub {
    margin-top: 5px;
    font-family: $mono;
    font-size: 0.8846em; // 0.71875rem / 0.8125rem
    color: $ink-3;
  }

  &__preview-close {
    flex-shrink: 0;
    width: 30px;
    height: 30px;
    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: 8px;
    border: 1px solid $line;
    background: $paper;
    color: $ink-2;
    cursor: pointer;
    transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

    &:hover { border-color: $ink-3; color: $ink; }
  }

  &__preview-body {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding: 16px 20px 24px;
    display: flex;
    flex-direction: column;
    gap: 12px;
  }

  &__preview-loading {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 8px;
    padding: 40px 16px;
    color: $ink-3;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    text-align: center;
  }

  &__preview-card {
    border: 1px solid $line;
    border-radius: 12px;
    padding: 14px 16px;
    background: $paper;
  }

  &__preview-card-hdr {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 10px;
    margin-bottom: 10px;
  }

  &__preview-card-num {
    font-family: $mono;
    font-weight: 700;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    color: $ink-3;
  }

  &__preview-card-tags {
    display: inline-flex;
    align-items: center;
    gap: 6px;
  }

  &__preview-card-lang {
    font-family: $mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: $ink-3;
    background: $paper;
    border: 1px solid $line;
    border-radius: 6px;
    padding: 3px 8px;
    white-space: nowrap;
  }

  &__preview-card-cat {
    font-family: $mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: $signal;
    background: $wash;
    border-radius: 6px;
    padding: 3px 8px;
    white-space: nowrap;
  }

  &__preview-field {
    min-width: 0;

    & + & { margin-top: 10px; }
  }

  &__preview-field-label {
    @extend %micro;
    font-size: 0.6923em; // 0.5625rem / 0.8125rem
    color: $ink-3;
    display: block;
    margin-bottom: 4px;
  }

  &__preview-field-text {
    font-family: $mono;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    line-height: 1.5;
    color: $ink;
    white-space: pre-wrap;
    word-break: break-word;
    background: $card;
    border: 1px solid $line-2;
    border-radius: 8px;
    padding: 8px 10px;

    &--empty {
      color: $ink-3;
      font-style: italic;
      font-family: $sans;
    }
  }

  &__preview-image-list {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
  }

  &__preview-image-chip {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    max-width: 100%;
    padding: 6px 11px;
    border: 1px solid $line;
    border-radius: 999px;
    background: $card;
    color: $ink-2;
    font-family: $sans;
    font-size: 0.8462em; // 0.6875rem / 0.8125rem
    font-weight: 600;
    cursor: pointer;
    transition: border-color 0.14s ease, color 0.14s ease, background 0.14s ease;

    svg { flex-shrink: 0; color: $signal; }

    span {
      overflow: hidden;
      text-overflow: ellipsis;
      white-space: nowrap;
    }

    &:hover { border-color: $signal; color: $signal; background: $wash; }
  }

  // ---- preview pagination bar -------------------------------------------------
  &__preview-pagination {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 10px;
    flex-wrap: wrap;
    padding: 12px 20px;
    border-top: 1px solid $line;
    background: $paper;
  }

  &__preview-pagination-info {
    font-family: $mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    color: $ink-3;
    white-space: nowrap;
  }

  &__preview-pagination-controls {
    display: flex;
    align-items: center;
    gap: 6px;
  }

  &__preview-size-select {
    appearance: none;
    -webkit-appearance: none;
    font: inherit;
    font-family: $mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    font-weight: 650;
    color: $ink-2;
    background: $card;
    border: 1px solid $line;
    border-radius: 8px;
    padding: 5px 22px 5px 9px;
    cursor: pointer;
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='10' height='6' viewBox='0 0 10 6' fill='none'%3E%3Cpath d='M1 1L5 5L9 1' stroke='%238A909B' stroke-width='1.5' stroke-linecap='round' stroke-linejoin='round'/%3E%3C/svg%3E");
    background-repeat: no-repeat;
    background-position: right 8px center;

    &:hover { border-color: $ink-3; }
    &:focus { outline: none; border-color: $signal; box-shadow: 0 0 0 3px $wash; }
    &:disabled { opacity: 0.5; cursor: not-allowed; }
  }

  &__preview-nav-btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 26px;
    height: 26px;
    border-radius: 8px;
    border: 1px solid $line;
    background: $card;
    color: $ink-2;
    cursor: pointer;
    transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

    &:hover:not(:disabled) { border-color: $signal; color: $signal; background: $wash; }
    &:disabled { opacity: 0.4; cursor: not-allowed; }
  }

  &__preview-page-label {
    min-width: 44px;
    text-align: center;
    font-family: $mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    font-weight: 700;
    color: $ink;
  }

  // ---- MinIO image lightbox ---------------------------------------------------
  // Deliberately "always dark" — a photo/document viewer stage, not a
  // themed surface, same reasoning as the other scrims in this file: a
  // near-black canvas is what makes images (light or dark) read cleanly,
  // in both app themes.
  &__lightbox-overlay {
    position: fixed;
    inset: 0;
    z-index: 70;
    background: rgba(10, 11, 14, 0.82);
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 32px;
    animation: datasets-fade-in 0.15s ease;
  }

  &__lightbox {
    width: min(880px, 100%);
    max-height: 88vh;
    display: flex;
    flex-direction: column;
    background: #14161B;
    border-radius: 14px;
    overflow: hidden;
    box-shadow: 0 30px 70px -20px rgba(0, 0, 0, 0.6);
    animation: datasets-modal-in 0.18s cubic-bezier(0.22, 1, 0.36, 1);
  }

  &__lightbox-hdr {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
    padding: 12px 14px 12px 16px;
    background: rgba(255, 255, 255, 0.04);
    border-bottom: 1px solid rgba(255, 255, 255, 0.08);
  }

  &__lightbox-hdr-info {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    min-width: 0;
    color: rgba(255, 255, 255, 0.55);
  }

  &__lightbox-filename {
    font-family: $mono;
    font-size: 0.8462em; // 0.6875rem / 0.8125rem
    font-weight: 600;
    color: rgba(255, 255, 255, 0.92);
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__lightbox-hdr-actions {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 4px;
  }

  &__lightbox-action {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 30px;
    height: 30px;
    border-radius: 8px;
    border: 1px solid transparent;
    background: transparent;
    color: rgba(255, 255, 255, 0.7);
    cursor: pointer;
    text-decoration: none;
    transition: background 0.15s ease, color 0.15s ease;

    &:hover { background: rgba(255, 255, 255, 0.1); color: #fff; }
  }

  &__lightbox-stage {
    flex: 1;
    min-height: 320px;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 24px;
    overflow: auto;
    // Faint checker pattern so transparent-background images (PNGs) are
    // distinguishable from the dark stage itself — same convention MinIO
    // and most image tools use.
    background-image:
      linear-gradient(45deg, rgba(255, 255, 255, 0.03) 25%, transparent 25%),
      linear-gradient(-45deg, rgba(255, 255, 255, 0.03) 25%, transparent 25%),
      linear-gradient(45deg, transparent 75%, rgba(255, 255, 255, 0.03) 75%),
      linear-gradient(-45deg, transparent 75%, rgba(255, 255, 255, 0.03) 75%);
    background-size: 20px 20px;
    background-position: 0 0, 0 10px, 10px -10px, -10px 0;
  }

  &__lightbox-img {
    max-width: 100%;
    max-height: 72vh;
    object-fit: contain;
    border-radius: 4px;
    box-shadow: 0 12px 30px -10px rgba(0, 0, 0, 0.5);
  }

  &__lightbox-state {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 10px;
    color: rgba(255, 255, 255, 0.55);
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    text-align: center;
  }

  &__lightbox-retry {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    margin-top: 4px;
    padding: 7px 14px;
    border-radius: 999px;
    border: 1px solid rgba(255, 255, 255, 0.16);
    background: rgba(255, 255, 255, 0.06);
    color: #fff;
    font-family: $sans;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: background 0.15s ease;

    &:hover { background: rgba(255, 255, 255, 0.14); }
  }
}

@keyframes datasets-shimmer {
  from { background-position: 200% 0; }
  to { background-position: -200% 0; }
}

@keyframes datasets-detail-in {
  from { opacity: 0; transform: translateY(6px); }
  to { opacity: 1; transform: none; }
}

@keyframes datasets-fade-in {
  from { opacity: 0; }
  to { opacity: 1; }
}

@keyframes datasets-modal-in {
  from { opacity: 0; transform: translateY(8px) scale(0.98); }
  to { opacity: 1; transform: none; }
}

@keyframes datasets-spin {
  to { transform: rotate(360deg); }
}

@media (max-width: 820px) {
  .datasets__header { padding: 20px 18px 16px; flex-direction: column; align-items: flex-start; gap: 10px; }
  .datasets__toolbar { padding: 12px 18px; }
  .datasets__facets { padding: 10px 18px; }
  .datasets__body { grid-template-columns: 1fr; grid-template-rows: minmax(180px, 34vh) 1fr; }
  .datasets__rail { border-right: 0; border-bottom: 1px solid $line; }
  .datasets__detail-scroll { padding: 20px 18px 32px; }
  .datasets__stats { grid-template-columns: repeat(2, 1fr); }
  .datasets__hero h2 { font-size: 1.5385em; /* 1.25rem / 0.8125rem */ }
  .datasets__modal { width: 100%; max-width: 100%; border-radius: 16px; }
  .datasets__modal-overlay { padding: 14px; }
  .datasets__preview-drawer { width: 100%; max-width: 100vw; }
  .datasets__lightbox-overlay { padding: 12px; }
  .datasets__lightbox { max-height: 92vh; }
  .datasets__lightbox-img { max-height: 60vh; }
}

@media (prefers-reduced-motion: reduce) {
  .datasets__skel,
  .datasets__detail-scroll,
  .datasets__modal-overlay,
  .datasets__modal,
  .datasets__lightbox-overlay,
  .datasets__lightbox,
  .datasets__spin { animation: none; }
  .datasets__row,
  .datasets__cap,
  .datasets__upload-btn,
  .datasets__dropzone,
  .datasets__eval-pill,
  .datasets__modal-submit,
  .datasets__modal-danger,
  .datasets__delete-btn,
  .datasets__preview-btn,
  .datasets__preview-drawer,
  .datasets__preview-overlay,
  .datasets__preview-nav-btn,
  .datasets__preview-image-chip,
  .datasets__lightbox-action,
  .datasets__lightbox-retry,
  .datasets__modal-cancel { transition: none; }
}
