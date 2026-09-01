import { useEffect, useRef, useState } from 'react';
import { X, Loader2, Inbox, Trash2, Pencil } from 'lucide-react';
import type { Provider, Model } from '../../types';
import ConfirmDialog from './ConfirmDialog';
import AddCustomModelDrawer, { type CustomModelSubmitPayload, type EditableModel } from './AddCustomModelDrawer';
import styles from './Providers.module.scss';

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

export default function ProviderModelsSidebar({
  provider,
  models = [],
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

  return (
    <>
      <div className={styles['providers__sidebar-overlay']} onClick={onClose} />
      <aside className={styles['providers__sidebar']}>
        <div className={styles['providers__sidebar-header']}>
          <div>
            <div className={styles['providers__sidebar-title']}>{provider?.name ?? 'Provider'}</div>
            <div className={styles['providers__sidebar-subtitle']}>
              {models.length} model{models.length === 1 ? '' : 's'} available
            </div>
          </div>
          <button className="btn btn-sm btn-ghost" onClick={onClose} aria-label="Close">
            <X size={16} />
          </button>
        </div>

        <div className={styles['providers__sidebar-body']}>
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

          {status === 'succeeded' && models.map((raw) => {
            const m = raw as EditableModel;
            const isDeleting = deletingId === m.id;

            return (
              <div
                key={m.id}
                className={`${styles['providers__model-row']} ${isDeleting ? styles['providers__model-row--deleting'] : ''}`}
              >
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
                          onClick={() => setEditingModel(m)}
                          title="Edit model"
                          aria-label={`Edit ${m.name ?? m.id}`}
                        >
                          <Pencil size={13} />
                        </button>
                        <button
                          type="button"
                          className={styles['providers__model-row-delete']}
                          onClick={() => setPendingDelete(m)}
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

                <div className={styles['providers__model-row-tags']}>
                  {m.category && <span className="tag tag-ind">{m.category}</span>}
                  {(m.capabilities ?? []).map((c) => (
                    <span key={c} className="tag tag-ind">{c}</span>
                  ))}
                </div>

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
          })}
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
