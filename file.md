//Providermodelssidebar.tsx
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


















//Providers.module.scss
@use '../../styles/_variables' as *;

// ===========================================================================
// Providers — matches the Run Console / Dashboard design system:
// ink/paper palette, ultramarine signal accent, mono instrument labels,
// hover-lift cards. Sidebar block keys are kept stable (shared with
// ProviderModelsSidebar) but recolored to the same tokens.
//
// Neutral/status tokens ($ink, $paper, $card, $line, $signal, $wash, $ok,
// $danger, etc.) now come from the shared "ink" block in _variables.scss
// (theme-aware via _theme.scss custom properties) — only font aliases and
// shadow tokens specific to this file are declared locally below.
//
// Font scaling: `.providers` sets a single base font-size. All descendant
// font-sizes are expressed in `em` (relative to that base), so bumping
// `.providers`'s font-size (e.g. on wide screens) scales the whole
// component proportionally from one place — same convention as Sidebar.
// ===========================================================================

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;

$soft: 0 1px 2px rgba(20, 22, 27, 0.05);
$lift: 0 14px 30px -14px rgba(20, 22, 27, 0.22);

// base font-size the providers page's internal `em` scale is built on
$providers-base-font: 0.8125rem;

%micro {
  font-family: $mono;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}

.providers {
  // master scale control — every em-based font-size below responds to this
  font-size: $providers-base-font;

  @media (min-width: 1800px) {
    font-size: 1rem;
  }

  // ---- header -----------------------------------------------------------
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
  }

  &__header-meta {
    flex-shrink: 0;
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
    margin-bottom: 3px;
  }

  // ---- toolbar ------------------------------------------------------------
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
    max-width: 340px;
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
      &:focus {
        outline: none;
        border-color: $signal;
        background: $card;
      }
    }
  }

  &__toolbar-right {
    display: flex;
    align-items: center;
    gap: 14px;
    flex-wrap: wrap;
  }

  &__filter-group {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    padding: 4px;
    background: $paper;
    border: 1px solid $line;
    border-radius: 999px;
  }

  &__toolbar-label {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 5px 10px 5px 11px;
    @extend %micro;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    color: $ink-3;
    white-space: nowrap;
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

  &__toolbar-divider {
    flex-shrink: 0;
    width: 1px;
    height: 26px;
    background: $line;
  }

  &__add-btn {
    display: inline-flex;
    align-items: center;
    gap: 7px;
    padding: 9px 15px;
    border: 1px solid $signal;
    border-radius: 10px;
    background: $signal;
    color: #fff;
    font-family: $sans;
    font-size: 1em; // 0.8125rem / 0.8125rem (base)
    font-weight: 650;
    cursor: pointer;
    box-shadow: $soft;
    transition: background 0.16s ease, border-color 0.16s ease, transform 0.16s ease, box-shadow 0.16s ease;

    &:hover { background: $signal-2; border-color: $signal-2; transform: translateY(-1px); box-shadow: $lift; }
  }

  // ---- provider card grid --------------------------------------------------
  &__grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(380px, 1fr));
    gap: 12px;
  }

  &__card {
    position: relative;
    display: flex;
    flex-direction: column;
    height: 100%;
    padding: 17px 18px;
    border: 1.5px solid $line;
    border-radius: 16px;
    background: $card;
    transition: border-color 0.16s ease, box-shadow 0.16s ease, transform 0.16s ease;

    &:hover {
      border-color: $ink-3;
      box-shadow: $lift;
      transform: translateY(-2px);
    }
  }

  &__card-hdr {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 10px;
  }

  &__card-id {
    display: flex;
    align-items: center;
    gap: 13px;
    min-width: 0;
  }

  &__icon {
    flex-shrink: 0;
    width: 42px;
    height: 42px;
    border-radius: 12px;
    display: grid;
    place-items: center;
    background: $paper;
    border: 1px solid $line;
    color: $ink;
    font-family: $display;
    font-weight: 800;
    font-size: 1.3077em; // 1.0625rem / 0.8125rem

    img { width: 24px; height: 24px; object-fit: contain; }
  }

  &__name {
    font-family: $display;
    font-size: 1.1538em; // 0.9375rem / 0.8125rem
    font-weight: 700;
    color: $ink;
    line-height: 1.25;
  }

  &__count {
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    color: $ink-3;
    margin-top: 2px;
  }

  &__card-top-actions {
    display: flex;
    align-items: center;
    gap: 6px;
    flex-shrink: 0;
  }

  &__icon-btn {
    display: grid;
    place-items: center;
    width: 28px;
    height: 28px;
    border: 1px solid $line;
    border-radius: 8px;
    background: $card;
    color: $ink-2;
    cursor: pointer;
    transition: border-color 0.14s ease, color 0.14s ease, background 0.14s ease;

    &:hover { border-color: $ink-3; color: $ink; background: $paper; }
  }

  &__badge-connected {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 4px 10px 4px 8px;
    border-radius: 999px;
    font-family: $mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: $ok;
    background: $ok-wash;
    white-space: nowrap;

    &::before { content: ''; width: 5px; height: 5px; border-radius: 50%; background: $ok; }
  }

  &__badge-idle {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 4px 11px;
    border-radius: 999px;
    font-size: 0.8462em; // 0.6875rem / 0.8125rem
    font-weight: 600;
    color: $ink-3;
    background: transparent;
    border: 1px dashed $line;
    white-space: nowrap;

    &::before {
      content: '';
      flex-shrink: 0;
      width: 6px;
      height: 6px;
      border-radius: 50%;
      background: $ink-3;
      opacity: 0.7;
    }
  }

  &__desc {
    flex: 1;
    margin-top: 11px;
    font-size: 1em; // 0.8125rem / 0.8125rem (base)
    color: $ink-2;
    line-height: 1.5;
  }

  // ---- inline API key form -------------------------------------------------
  &__key-form {
    display: flex;
    gap: 8px;
    margin-top: 12px;
  }

  &__key-input {
    flex: 1;
    border: 1.5px solid $line;
    border-radius: 9px;
    padding: 8px 11px;
    font-size: 1em; // 0.8125rem / 0.8125rem (base)
    font-family: $sans;
    color: $ink;
    background: $paper;
    transition: border-color 0.15s ease, background 0.15s ease;

    &::placeholder { color: $ink-3; }
    &:focus { outline: none; border-color: $signal; background: $card; }
  }

  &__key-actions {
    display: flex;
    gap: 6px;
    flex-shrink: 0;
  }

  // ---- footer action row ---------------------------------------------------
  &__foot-actions {
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 6px;
    margin-top: 13px;
  }

  &__foot-btn {
    flex: 0 0 auto;
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 6px 11px;
    border-radius: 8px;
    border: 1px solid transparent;
    font-family: $sans;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    font-weight: 650;
    white-space: nowrap;
    cursor: pointer;
    transition: background 0.15s ease, border-color 0.15s ease, color 0.15s ease, transform 0.12s ease;

    &:disabled { cursor: not-allowed; opacity: 0.55; }

    &--primary {
      border-color: $signal;
      background: $signal;
      color: #fff;
      &:not(:disabled):hover { background: $signal-2; border-color: $signal-2; transform: translateY(-1px); }
    }

    &--accent {
      background: $signal;
      color: #fff;
      &:not(:disabled):hover { background: $signal-2; transform: translateY(-1px); }
    }

    &--ghost {
      background: $card;
      border-color: $line;
      color: $ink-2;
      &:not(:disabled):hover { border-color: $ink-3; color: $ink; background: $paper; }
    }

    &--danger {
      background: $danger-wash;
      color: $danger;
      &:not(:disabled):hover { background: rgba($danger, 0.16); }
    }
  }

  &__spin { animation: providers-spin 0.8s linear infinite; }

  &__empty {
    grid-column: 1 / -1;
    padding: 40px 20px;
    text-align: center;
    color: $ink-3;
    font-size: 1.0385em; // 0.84375rem / 0.8125rem
    border: 1px dashed $line;
    border-radius: 14px;
  }

  // ===========================================================================
  // Provider models sidebar — keys kept stable for ProviderModelsSidebar,
  // recolored to the ink/paper/signal system.
  // ===========================================================================
  &__sidebar-overlay {
    position: fixed;
    inset: 0;
    background: rgba(20, 22, 27, 0.45);
    z-index: 40;
  }

  &__sidebar {
    position: fixed;
    top: 0;
    right: 0;
    bottom: 0;
    width: min(420px, 100vw);
    background: $card;
    border-left: 1px solid $line;
    box-shadow: -20px 0 40px -16px rgba(20, 22, 27, 0.28);
    z-index: 41;
    display: flex;
    flex-direction: column;
    animation: providers-sidebar-in 0.22s cubic-bezier(0.22, 0.72, 0.16, 1);
  }

  &__sidebar-header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 12px;
    padding: 20px 20px 16px;
    border-bottom: 1px solid $line;
  }

  &__sidebar-title {
    font-family: $display;
    font-size: 1.3077em; // 1.0625rem / 0.8125rem
    font-weight: 800;
    letter-spacing: -0.01em;
    color: $ink;
  }

  &__sidebar-subtitle {
    margin-top: 3px;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    color: $ink-3;
  }

  &__sidebar-body {
    flex: 1;
    overflow-y: auto;
    padding: 16px 20px 24px;
    display: flex;
    flex-direction: column;
    gap: 12px;
  }

  &__sidebar-empty {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 8px;
    padding: 48px 12px;
    color: $ink-3;
    font-size: 1em; // 0.8125rem / 0.8125rem (base)
    text-align: center;
  }

  &__model-row {
    border: 1px solid $line;
    border-radius: 12px;
    padding: 14px;
    background: $paper;
    transition: opacity 0.15s ease;

    &--deleting {
      opacity: 0.5;
      pointer-events: none;
    }
  }

  &__model-row-head {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
  }

  &__model-row-head-actions {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 6px;
  }

  &__model-row-delete {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 24px;
    height: 24px;
    border-radius: 7px;
    border: 1px solid $line;
    background: $card;
    color: $ink-3;
    cursor: pointer;
    transition: border-color 0.14s ease, color 0.14s ease, background 0.14s ease;

    &:hover:not(:disabled) {
      border-color: rgba($danger, 0.35);
      color: $danger;
      background: $danger-wash;
    }

    &:disabled { cursor: not-allowed; opacity: 0.6; }
  }

  &__model-row-edit {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 24px;
    height: 24px;
    border-radius: 7px;
    border: 1px solid $line;
    background: $card;
    color: $ink-3;
    cursor: pointer;
    transition: border-color 0.14s ease, color 0.14s ease, background 0.14s ease;

    &:hover:not(:disabled) {
      border-color: $signal;
      color: $signal;
      background: $wash;
    }
  }

  &__model-row-desc {
    margin: 6px 0 0;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    line-height: 1.5;
    color: $ink-2;
  }

  &__model-row-name {
    font-family: $display;
    font-weight: 700;
    font-size: 1.0769em; // 0.875rem / 0.8125rem
    color: $ink;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    min-width: 0;
  }

  &__model-row-tags {
    margin-top: 10px;
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  &__tag-group {
    display: flex;
    align-items: baseline;
    flex-wrap: wrap;
    gap: 8px;
  }

  &__tag-group-label {
    @extend %micro;
    flex-shrink: 0;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    color: $ink-3;
  }

  &__tag-group-pills {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
  }

  // Category is a single classifying value — a filled, accent-colored pill
  // makes it read as "this model's category", distinct from the list below.
  &__category-badge {
    display: inline-flex;
    align-items: center;
    padding: 2px 9px;
    border-radius: 999px;
    background: $signal;
    color: #fff;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    font-weight: 700;
  }

  // Capabilities are a set of tags — neutral outline pills keep them
  // visually separate from the single filled category badge above.
  &__capability-badge {
    display: inline-flex;
    align-items: center;
    padding: 2px 9px;
    border-radius: 999px;
    background: $card;
    border: 1px solid $line;
    color: $ink-2;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    font-weight: 600;
  }

  // ---- inline model edit form ------------------------------------------------
  &__model-edit {
    display: flex;
    flex-direction: column;
    gap: 10px;
  }

  &__model-edit-field {
    display: flex;
    flex-direction: column;
    gap: 4px;

    label {
      font-size: 0.7692em; // 0.625rem / 0.8125rem
      font-weight: 700;
      letter-spacing: 0.04em;
      text-transform: uppercase;
      color: $ink-3;
    }
  }

  &__model-edit-input,
  &__model-edit-textarea {
    width: 100%;
    border: 1.5px solid $line;
    border-radius: 8px;
    padding: 7px 10px;
    font-size: 1em; // 0.8125rem / 0.8125rem (base)
    font-family: $sans;
    color: $ink;
    background: $card;
    resize: vertical;
    transition: border-color 0.15s ease, box-shadow 0.15s ease;

    &::placeholder { color: $ink-3; }
    &:focus { outline: none; border-color: $signal; box-shadow: 0 0 0 3px $wash; }
    &:disabled { opacity: 0.6; cursor: not-allowed; }
  }

  &__model-edit-actions {
    display: flex;
    justify-content: flex-end;
    gap: 6px;
  }

  &__model-edit-cancel,
  &__model-edit-save {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 6px 11px;
    border-radius: 8px;
    font-family: $sans;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: background 0.15s ease, border-color 0.15s ease, color 0.15s ease;

    &:disabled { cursor: not-allowed; opacity: 0.6; }
  }

  &__model-edit-cancel {
    border: 1px solid $line;
    background: $card;
    color: $ink-2;

    &:hover:not(:disabled) { border-color: $ink-3; color: $ink; }
  }

  &__model-edit-save {
    border: 1px solid $signal;
    background: $signal;
    color: #fff;

    &:hover:not(:disabled) { background: $signal-2; border-color: $signal-2; }
  }

  &__model-row-meta {
    margin-top: 10px;
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 8px 12px;

    > div {
      display: flex;
      flex-direction: column;
      gap: 2px;
      font-size: 1em; // 0.8125rem / 0.8125rem (base)
      color: $ink;
    }
  }

  &__model-row-meta-label {
    @extend %micro;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    color: $ink-3;
  }

  &__model-row-url {
    margin-top: 10px;
    padding-top: 10px;
    border-top: 1px dashed $line;
    font-family: $mono;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    color: $ink-2;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }
}

@keyframes providers-spin {
  to { transform: rotate(360deg); }
}

@keyframes providers-sidebar-in {
  from { transform: translateX(100%); }
  to { transform: translateX(0); }
}

@media (max-width: 768px) {
  .providers__header { padding: 20px 18px 16px; flex-direction: column; align-items: flex-start; gap: 10px; }
  .providers__toolbar { padding: 12px 18px; }
  .providers__grid { grid-template-columns: 1fr; }
}
