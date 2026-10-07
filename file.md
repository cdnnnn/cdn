import { memo, useCallback, useEffect, useMemo, useRef, useState } from 'react';
import { X, Loader2, Inbox, Trash2, Pencil, Search } from 'lucide-react';
import type { Provider, Model } from '../../types';
import ConfirmDialog from './ConfirmDialog';
import AddCustomModelDrawer, { type CustomModelSubmitPayload, type EditableModel } from './AddCustomModelDrawer';
import styles from './Providers.module.scss';

// Providers can have hundreds of models (e.g. 500+). Rendering every card at
// once blocks the main thread, so cards are rendered in batches and more are
// appended as the user scrolls near the bottom.
const BATCH_SIZE = 20;
const SEARCH_THRESHOLD = 10;
const EMPTY_MODELS: Model[] = [];

interface ProviderModelsSidebarProps {
  provider: Provider;
  models: Model[];
  status: 'idle' | 'loading' | 'succeeded' | 'failed';
  onClose: () => void;
  /** Edit + delete affordances only make sense for the Custom provider. */
  canManage?: boolean;
  deletingId?: string | null;
  updatingId?: string | null;
  /** True while a "register as new model" submission (from a mismatched edit) is in flight. */
  creatingNew?: boolean;
  onDelete?: (modelId: string) => void;
  /** Returning a Promise lets the sidebar close the edit drawer once the dispatch resolves. */
  onEditSubmit?: (payload: CustomModelSubmitPayload) => Promise<unknown> | void;
}

interface ModelRowProps {
  model: EditableModel;
  canManage: boolean;
  isDeleting: boolean;
  onEdit: (m: EditableModel) => void;
  onRemove: (m: EditableModel) => void;
}

// Memoized so that unrelated state changes (search typing, delete spinner on
// another row, loading more batches) don't re-render every card already shown.
const ModelRow = memo(function ModelRow({ model: m, canManage, isDeleting, onEdit, onRemove }: ModelRowProps) {
  return (
    <div className={`${styles['providers__model-row']} ${isDeleting ? styles['providers__model-row--deleting'] : ''}`}>
      <div className={styles['providers__model-row-head']}>
        <span className={styles['providers__model-row-name']} title={m.name ?? 'Unnamed model'}>{m.name ?? 'Unnamed model'}</span>
        <div className={styles['providers__model-row-head-actions']}>
          <span className={`badge ${m.is_active ? 'badge-green' : 'badge-gray'}`}>
            {m.is_active ? 'Active' : 'Inactive'}
          </span>
          {canManage && (
            <>
              <button
                type="button"
                className={styles['providers__model-row-edit']}
                onClick={() => onEdit(m)}
                title="Edit model"
                aria-label={`Edit ${m.name ?? m.id}`}
              >
                <Pencil size={13} />
              </button>
              <button
                type="button"
                className={styles['providers__model-row-delete']}
                onClick={() => onRemove(m)}
                disabled={isDeleting}
                title="Remove model"
                aria-label={`Remove ${m.name ?? m.id}`}
              >
                {isDeleting ? (
                  <Loader2 size={13} style={{ animation: 'spin 1.5s linear infinite' }} />
                ) : (
                  <Trash2 size={13} />
                )}
              </button>
            </>
          )}
        </div>
      </div>

      {m.description && (
        <p className={styles['providers__model-row-desc']}>{m.description}</p>
      )}

      {(m.category || (m.capabilities ?? []).length > 0) && (
        <div className={styles['providers__model-row-tags']}>
          {m.category && (
            <div className={styles['providers__tag-group']}>
              <span className={styles['providers__tag-group-label']}>Category</span>
              <span className={styles['providers__category-badge']}>{m.category}</span>
            </div>
          )}
          {(m.capabilities ?? []).length > 0 && (
            <div className={styles['providers__tag-group']}>
              <span className={styles['providers__tag-group-label']}>Capabilities</span>
              <div className={styles['providers__tag-group-pills']}>
                {(m.capabilities ?? []).map((c) => (
                  <span key={c} className={styles['providers__capability-badge']}>{c}</span>
                ))}
              </div>
            </div>
          )}
        </div>
      )}

      <div className={styles['providers__model-row-meta']}>
        <div>
          <span className={styles['providers__model-row-meta-label']}>Context</span>
          <span>{(m.context_window ?? 0).toLocaleString()}</span>
        </div>
        <div>
          <span className={styles['providers__model-row-meta-label']}>Price (in/out)</span>
          <span>
            {m.input_price != null ? `$${m.input_price.toFixed(2)}` : '—'} / {m.output_price != null ? `$${m.output_price.toFixed(2)}` : '—'}
          </span>
        </div>
        <div>
          <span className={styles['providers__model-row-meta-label']}>Accuracy</span>
          <span>{m.accuracy_score != null ? `${m.accuracy_score}%` : '—'}</span>
        </div>
        <div>
          <span className={styles['providers__model-row-meta-label']}>Agent Score</span>
          <span>{m.agent_score != null ? `${m.agent_score}%` : '—'}</span>
        </div>
      </div>

      {m.base_url && (
        <div className={styles['providers__model-row-url']} title={m.base_url}>
          {m.base_url}
        </div>
      )}
    </div>
  );
});

export default function ProviderModelsSidebar({
  provider,
  models = EMPTY_MODELS,
  status,
  onClose,
  canManage = false,
  deletingId = null,
  updatingId = null,
  creatingNew = false,
  onDelete,
  onEditSubmit,
}: ProviderModelsSidebarProps) {
  const [pendingDelete, setPendingDelete] = useState<EditableModel | null>(null);
  const [editingModel, setEditingModel] = useState<EditableModel | null>(null);
  const [query, setQuery] = useState('');
  const [visibleCount, setVisibleCount] = useState(BATCH_SIZE);

  const bodyRef = useRef<HTMLDivElement>(null);
  const sentinelRef = useRef<HTMLDivElement>(null);

  const confirmDelete = () => {
    if (pendingDelete && onDelete) onDelete(pendingDelete.id);
  };

  // Close the delete confirmation once the in-flight delete for the pending
  // model finishes.
  const prevDeletingId = useRef<string | null>(null);
  useEffect(() => {
    if (pendingDelete && prevDeletingId.current === pendingDelete.id && deletingId !== pendingDelete.id) {
      setPendingDelete(null);
    }
    prevDeletingId.current = deletingId;
  }, [deletingId, pendingDelete]);

  const handleEditSubmit = (payload: CustomModelSubmitPayload) => {
    const result = onEditSubmit?.(payload);
    if (result && typeof (result as Promise<unknown>).then === 'function') {
      // On success close the drawer; on failure (already toasted by the
      // caller) keep it open so the user can adjust and retry.
      (result as Promise<unknown>).then(() => setEditingModel(null)).catch(() => {});
    } else {
      setEditingModel(null);
    }
  };

  const editSubmitting = editingModel
    ? (updatingId === editingModel.id || creatingNew)
    : false;

  // ---- search + incremental rendering ----------------------------------------
  const filteredModels = useMemo(() => {
    const q = query.trim().toLowerCase();
    if (!q) return models;
    return models.filter((m) => {
      const e = m as EditableModel;
      return (
        (e.name ?? '').toLowerCase().includes(q) ||
        e.id.toLowerCase().includes(q) ||
        (e.category ?? '').toLowerCase().includes(q) ||
        (e.capabilities ?? []).some((c) => c.toLowerCase().includes(q))
      );
    });
  }, [models, query]);

  // Start from the first batch again whenever the search or provider changes.
  useEffect(() => {
    setVisibleCount(BATCH_SIZE);
    bodyRef.current?.scrollTo({ top: 0 });
  }, [query, provider?.id]);

  const visibleModels = useMemo(
    () => filteredModels.slice(0, visibleCount),
    [filteredModels, visibleCount]
  );
  const hasMore = visibleCount < filteredModels.length;

  // Append the next batch when the sentinel at the bottom scrolls into view.
  // Re-observing after each batch also handles the case where the sentinel is
  // still visible (tall viewport) once new rows have rendered.
  //
  // `status` must be a dependency: the sentinel only renders once status is
  // 'succeeded'. On re-open, the models are already cached in the store, so
  // `hasMore` is true while status is still 'loading' (no sentinel yet) —
  // without `status` here the effect would bail out and never attach later.
  useEffect(() => {
    if (status !== 'succeeded' || !hasMore) return;
    const node = sentinelRef.current;
    const root = bodyRef.current;
    if (!node || !root) return;
    const observer = new IntersectionObserver(
      (entries) => {
        if (entries[0]?.isIntersecting) setVisibleCount((c) => c + BATCH_SIZE);
      },
      { root, rootMargin: '300px' }
    );
    observer.observe(node);
    return () => observer.disconnect();
  }, [status, hasMore, visibleCount]);

  // Stable callbacks so memoized rows don't re-render needlessly.
  const handleEdit = useCallback((m: EditableModel) => setEditingModel(m), []);
  const handleRemove = useCallback((m: EditableModel) => setPendingDelete(m), []);

  const showSearch = status === 'succeeded' && models.length > SEARCH_THRESHOLD;
  const isFiltering = query.trim() !== '';

  return (
    <>
      <div className={styles['providers__sidebar-overlay']} onClick={onClose} />
      <aside className={styles['providers__sidebar']}>
        <div className={styles['providers__sidebar-header']}>
          <div>
            <div className={styles['providers__sidebar-title']}>{provider?.name ?? 'Provider'}</div>
            <div className={styles['providers__sidebar-subtitle']}>
              {isFiltering
                ? `${filteredModels.length} of ${models.length} models`
                : `${models.length} model${models.length === 1 ? '' : 's'} available`}
            </div>
          </div>
          <button className="btn btn-sm btn-ghost" onClick={onClose} aria-label="Close">
            <X size={16} />
          </button>
        </div>

        {showSearch && (
          <div className={styles['providers__sidebar-toolbar']}>
            <div className={styles['providers__sidebar-search']}>
              <Search size={14} />
              <input
                placeholder="Search models…"
                value={query}
                onChange={(e) => setQuery(e.target.value)}
              />
            </div>
          </div>
        )}

        <div className={styles['providers__sidebar-body']} ref={bodyRef}>
          {status === 'loading' && (
            <div className={styles['providers__sidebar-empty']}>
              <Loader2 size={18} style={{ animation: 'spin 1.5s linear infinite' }} />
              <span>Loading models…</span>
            </div>
          )}

          {status === 'failed' && (
            <div className={styles['providers__sidebar-empty']}>
              <span>Couldn't load models for this provider.</span>
            </div>
          )}

          {status === 'succeeded' && models.length === 0 && (
            <div className={styles['providers__sidebar-empty']}>
              <Inbox size={18} />
              <span>No models found for this provider yet.</span>
            </div>
          )}

          {status === 'succeeded' && models.length > 0 && filteredModels.length === 0 && (
            <div className={styles['providers__sidebar-empty']}>
              <Search size={18} />
              <span>No models match "{query.trim()}".</span>
            </div>
          )}

          {status === 'succeeded' && visibleModels.map((raw) => {
            const m = raw as EditableModel;
            return (
              <ModelRow
                key={m.id}
                model={m}
                canManage={canManage}
                isDeleting={deletingId === m.id}
                onEdit={handleEdit}
                onRemove={handleRemove}
              />
            );
          })}

          {status === 'succeeded' && hasMore && (
            <div ref={sentinelRef} className={styles['providers__sidebar-more']}>
              <Loader2 size={14} style={{ animation: 'spin 1.5s linear infinite' }} />
              <span>Showing {visibleModels.length} of {filteredModels.length} — loading more…</span>
            </div>
          )}
        </div>
      </aside>

      {pendingDelete && (
        <ConfirmDialog
          title="Remove this model?"
          message={`"${pendingDelete.name ?? pendingDelete.id}" will be permanently removed from ${provider?.name ?? 'this provider'}. This can't be undone.`}
          confirmLabel="Remove Model"
          tone="danger"
          loading={deletingId === pendingDelete.id}
          onCancel={() => setPendingDelete(null)}
          onConfirm={confirmDelete}
        />
      )}

      {editingModel && (
        <AddCustomModelDrawer
          mode="edit"
          initialModel={editingModel}
          submitting={editSubmitting}
          onClose={() => setEditingModel(null)}
          onSubmit={handleEditSubmit}
        />
      )}
    </>
  );
}
