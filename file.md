//Datasets.tsx
import { useEffect, useMemo, useState, useRef, useCallback } from 'react';
import {
  RefreshCw, Search, Layers, AlertTriangle, Database, ListFilter, X,
  Check, Boxes, ArrowRight, Filter, ChevronsUpDown, Upload, Loader2,
  FileUp, AlertCircle, CheckCircle2, Trash2, Eye, ChevronLeft, ChevronRight, HelpCircle,
} from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import {
  fetchTestSuites, uploadDataset, resetUploadStatus, deleteTestSuite, resetDeleteError,
  fetchDatasetPreview, resetPreview,
} from '../../store/slices/testSuitesSlice';
import type { TestSuite, PreviewState } from '../../store/slices/testSuitesSlice';
import { SUPPORTED_UPLOAD_EXTENSIONS } from '../../api/endpoints/testSuites';
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
      if (tagFilter.length && !tagFilter.some((t) => tags.includes(t))) return false;
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
    setTagFilter((prev) => (prev.includes(t) ? prev.filter((x) => x !== t) : [...prev, t]));
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
  // details drawer.
  useEffect(() => {
    if (!previewTarget) return;
    const onKey = (e: KeyboardEvent) => {
      if (e.key === 'Escape') closePreview();
    };
    window.addEventListener('keydown', onKey);
    return () => window.removeEventListener('keydown', onKey);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [previewTarget]);

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
            const col = hashColor(t);
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
                          <span className={styles['datasets__row-type']} style={{ color: accent.fg }}>
                            {dsType}
                          </span>
                          <span className={styles['datasets__row-dots']}>
                            {tags.slice(0, 4).map((t, ti) => (
                              <i key={t || ti} style={{ background: hashColor(t).fg }} title={t || undefined} />
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
          />
        )}
      </div>
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
            <span className={styles['datasets__source-badge']}>{datasetType}</span>
            <button
              type="button"
              className={styles['datasets__preview-btn']}
              onClick={onPreviewClick}
              title="Preview questions"
            >
              <Eye size={14} /> Preview
            </button>
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
              click to find similar datasets <ArrowRight size={11} />
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
              const col = hashColor(t);
              const active = tagFilter.includes(t);
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
}

function PreviewDrawer({ dataset, preview, onClose, onChangeOffset, onChangeLimit }: PreviewDrawerProps) {
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
          return (
            <div key={q?.id ?? i} className={styles['datasets__preview-card']}>
              <div className={styles['datasets__preview-card-hdr']}>
                <span className={styles['datasets__preview-card-num']}>Q{offset + i + 1}</span>
                {q?.category && <span className={styles['datasets__preview-card-cat']}>{q.category}</span>}
              </div>
              <div className={styles['datasets__preview-field']}>
                <span className={styles['datasets__preview-field-label']}>Prompt</span>
                <div className={styles['datasets__preview-field-text']}>{prompt || '—'}</div>
              </div>
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
