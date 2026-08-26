//Toast.tsx
import {
  createContext,
  useCallback,
  useContext,
  useMemo,
  useRef,
  useState,
  type ReactNode,
} from 'react';
import { CheckCircle2, XCircle, AlertTriangle, Info, X } from 'lucide-react';
import styles from './Toast.module.scss';

export type ToastTone = 'success' | 'error' | 'warning' | 'info';

export interface ToastOptions {
  /** How long the toast stays up before auto-dismissing, in ms. Pass 0 to disable. */
  duration?: number;
  title?: string;
}

interface ToastItem {
  id: number;
  tone: ToastTone;
  message: string;
  title?: string;
  duration: number;
}

interface ToastContextValue {
  /** Generic entry point — pick a tone explicitly. */
  showToast: (message: string, tone?: ToastTone, options?: ToastOptions) => void;
  success: (message: string, options?: ToastOptions) => void;
  error: (message: string, options?: ToastOptions) => void;
  warning: (message: string, options?: ToastOptions) => void;
  info: (message: string, options?: ToastOptions) => void;
  dismiss: (id: number) => void;
}

const ToastContext = createContext<ToastContextValue | null>(null);

const DEFAULT_DURATION = 4000;

const ICONS: Record<ToastTone, typeof CheckCircle2> = {
  success: CheckCircle2,
  error: XCircle,
  warning: AlertTriangle,
  info: Info,
};

/**
 * Wrap the app (or a section of it) with <ToastProvider> once, then call
 * useToast() anywhere below it to fire toasts:
 *
 *   const toast = useToast();
 *   toast.success('Model registered');
 *   toast.error('Something went wrong');
 */
export function ToastProvider({ children }: { children: ReactNode }) {
  const [toasts, setToasts] = useState<ToastItem[]>([]);
  const idRef = useRef(0);

  const dismiss = useCallback((id: number) => {
    setToasts((prev) => prev.filter((t) => t.id !== id));
  }, []);

  const showToast = useCallback(
    (message: string, tone: ToastTone = 'info', options?: ToastOptions) => {
      const id = ++idRef.current;
      const duration = options?.duration ?? DEFAULT_DURATION;
      setToasts((prev) => [...prev, { id, tone, message, title: options?.title, duration }]);
      if (duration > 0) {
        window.setTimeout(() => dismiss(id), duration);
      }
    },
    [dismiss]
  );

  const value = useMemo<ToastContextValue>(
    () => ({
      showToast,
      success: (message, options) => showToast(message, 'success', options),
      error: (message, options) => showToast(message, 'error', options),
      warning: (message, options) => showToast(message, 'warning', options),
      info: (message, options) => showToast(message, 'info', options),
      dismiss,
    }),
    [showToast, dismiss]
  );

  return (
    <ToastContext.Provider value={value}>
      {children}
      <ToastViewport toasts={toasts} onDismiss={dismiss} />
    </ToastContext.Provider>
  );
}

export function useToast(): ToastContextValue {
  const ctx = useContext(ToastContext);
  if (!ctx) {
    throw new Error('useToast must be used within a <ToastProvider>. Wrap your app root with it.');
  }
  return ctx;
}

function ToastViewport({ toasts, onDismiss }: { toasts: ToastItem[]; onDismiss: (id: number) => void }) {
  if (toasts.length === 0) return null;
  return (
    <div className={styles['toast-viewport']} role="region" aria-label="Notifications">
      {toasts.map((t) => {
        const Icon = ICONS[t.tone];
        return (
          <div
            key={t.id}
            className={`${styles['toast']} ${styles[`toast--${t.tone}`]}`}
            role={t.tone === 'error' ? 'alert' : 'status'}
            aria-live={t.tone === 'error' ? 'assertive' : 'polite'}
          >
            <div className={styles['toast__icon']}>
              <Icon size={17} />
            </div>
            <div className={styles['toast__body']}>
              {t.title && <div className={styles['toast__title']}>{t.title}</div>}
              <div className={styles['toast__message']}>{t.message}</div>
            </div>
            <button
              type="button"
              className={styles['toast__close']}
              onClick={() => onDismiss(t.id)}
              aria-label="Dismiss notification"
            >
              <X size={14} />
            </button>
          </div>
        );
      })}
    </div>
  );
}



















//Toast.module.scss
@use '../../styles/_variables' as *;

.toast-viewport {
  position: fixed;
  top: 20px;
  right: 20px;
  z-index: 200;
  display: flex;
  flex-direction: column;
  gap: 10px;
  width: min(360px, calc(100vw - 40px));
  pointer-events: none;
}

.toast {
  pointer-events: auto;
  display: flex;
  align-items: flex-start;
  gap: 10px;
  padding: 13px 14px;
  border-radius: 12px;
  background: $surface;
  box-shadow: $shadow-4;
  border: 1px solid $border;
  border-left: 3px solid $text-muted;
  animation: toast-in 0.18s cubic-bezier(0.22, 0.72, 0.16, 1) both;
  font-size: 13px;

  &--success { border-left-color: #10B981; }
  &--error { border-left-color: #DC2626; }
  &--warning { border-left-color: $amber-dark; }
  &--info { border-left-color: $indigo; }
}

@keyframes toast-in {
  from { opacity: 0; transform: translateX(16px); }
  to { opacity: 1; transform: translateX(0); }
}

.toast__icon {
  flex: 0 0 auto;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  margin-top: 1px;

  .toast--success & { color: #10B981; }
  .toast--error & { color: #DC2626; }
  .toast--warning & { color: $amber-dark; }
  .toast--info & { color: $indigo; }
}

.toast__body {
  flex: 1 1 auto;
  min-width: 0;
}

.toast__title {
  font-weight: 750;
  color: $text-primary;
  margin-bottom: 2px;
}

.toast__message {
  color: $text-secondary;
  line-height: 1.45;
  word-break: break-word;
}

.toast__close {
  flex: 0 0 auto;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 22px;
  height: 22px;
  margin-top: -2px;
  margin-right: -4px;
  border: none;
  border-radius: 6px;
  background: none;
  color: $text-muted;
  cursor: pointer;
  transition: background 0.14s ease, color 0.14s ease;

  &:hover { background: $surface-alt; color: $text-primary; }
}






















//Addcustommodeldrawer.tsx
import { useEffect, useRef, useState } from 'react';
import {
  X,
  Search,
  Loader2,
  ChevronDown,
  Check,
  CheckCircle2,
  RotateCcw,
  AlertCircle,
  AlertTriangle,
  Sparkles,
  PenLine,
} from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchModelCategories } from '../../store/slices/modelsSlice';
import { modelsApi, type DiscoveredModel } from '../../api/endpoints/models';
import type { CustomModelRequest, Model } from '../../types';
import styles from './AddCustomModelDrawer.module.scss';

// The update endpoint returns `description`, which the shared `Model` type
// doesn't declare — extend it locally for the edit-prefill case.
export type EditableModel = Model & { description?: string };

// Create always POSTs the full CustomModelRequest. Editing normally only
// POSTs {model_id, name, description} against the existing model — but if
// discovery finds a *different* model id than the one being edited, we
// treat that as registering a brand-new model instead, so the caller needs
// to know which branch happened.
export type CustomModelSubmitPayload =
  | { kind: 'create'; payload: CustomModelRequest }
  | { kind: 'update'; model_id: string; name: string; description: string };

interface Props {
  mode?: 'create' | 'edit';
  initialModel?: EditableModel;
  onClose: () => void;
  onSubmit: (payload: CustomModelSubmitPayload) => void;
  submitting: boolean;
}

// Fields that describe the actual model (name/id/context window) start out
// hidden in CREATE mode — the user either runs discovery against the base
// URL to auto-fill them, or opts into filling them in by hand. In EDIT mode
// they're always visible/pre-filled since we already have the model's data.
type FieldsMode = 'hidden' | 'manual' | 'discovered';

export default function AddCustomModelDrawer({ mode = 'create', initialModel, onClose, onSubmit, submitting }: Props) {
  const dispatch = useAppDispatch();
  const categories = useAppSelector((s) => s.models.categories);
  const categoriesStatus = useAppSelector((s) => s.models.categoriesStatus);
  const isEdit = mode === 'edit';

  const [baseUrl, setBaseUrl] = useState(initialModel?.base_url ?? '');
  const [apiKey, setApiKey] = useState('');
  const [category, setCategory] = useState(initialModel?.category ?? '');
  const [categoryOpen, setCategoryOpen] = useState(false);
  const categoryRef = useRef<HTMLDivElement>(null);
  const [description, setDescription] = useState(initialModel?.description ?? '');

  const [fieldsMode, setFieldsMode] = useState<FieldsMode>(isEdit ? 'manual' : 'hidden');
  const [name, setName] = useState(initialModel?.name ?? '');
  const [modelId, setModelId] = useState(initialModel?.id ?? '');
  const [contextWindowInput, setContextWindowInput] = useState(
    initialModel?.context_window != null ? String(initialModel.context_window) : ''
  );
  const [contextWindowLocked, setContextWindowLocked] = useState(false);
  const [autoDetected, setAutoDetected] = useState(false);

  const [discoverStatus, setDiscoverStatus] = useState<'idle' | 'loading' | 'error'>('idle');
  const [discoverError, setDiscoverError] = useState<string | null>(null);
  const [discoveredModels, setDiscoveredModels] = useState<DiscoveredModel[]>([]);
  // In edit mode, `selectedModelId` only gets set once discovery has been run
  // and a candidate chosen — that's what drives the same-model/new-model check.
  const [selectedModelId, setSelectedModelId] = useState<string | null>(null);

  useEffect(() => {
    if (categoriesStatus === 'idle') dispatch(fetchModelCategories());
  }, [dispatch, categoriesStatus]);

  // Close the custom category dropdown on outside click / Escape.
  useEffect(() => {
    if (!categoryOpen) return;
    const handleClick = (e: MouseEvent) => {
      if (categoryRef.current && !categoryRef.current.contains(e.target as Node)) setCategoryOpen(false);
    };
    const handleKey = (e: KeyboardEvent) => {
      if (e.key === 'Escape') setCategoryOpen(false);
    };
    document.addEventListener('mousedown', handleClick);
    document.addEventListener('keydown', handleKey);
    return () => {
      document.removeEventListener('mousedown', handleClick);
      document.removeEventListener('keydown', handleKey);
    };
  }, [categoryOpen]);

  const selectedCategory = categories.find((c) => c.value === category) || null;

  // Whether the currently-selected discovered candidate is a different model
  // than the one we started editing — only meaningful in edit mode, and only
  // once a discovery result has actually been applied.
  const willCreateNew = isEdit && !!initialModel && selectedModelId !== null && selectedModelId !== initialModel.id;
  const matchConfirmed = isEdit && !!initialModel && selectedModelId !== null && selectedModelId === initialModel.id;

  const resetModelFields = () => {
    if (isEdit && initialModel) {
      // "Use a different endpoint" in edit mode reverts to the original
      // model's own data rather than clearing everything out.
      setFieldsMode('manual');
      setName(initialModel.name ?? '');
      setModelId(initialModel.id ?? '');
      setContextWindowInput(initialModel.context_window != null ? String(initialModel.context_window) : '');
    } else {
      setFieldsMode('hidden');
      setName('');
      setModelId('');
      setContextWindowInput('');
    }
    setContextWindowLocked(false);
    setAutoDetected(false);
    setDiscoveredModels([]);
    setSelectedModelId(null);
    setDiscoverStatus('idle');
    setDiscoverError(null);
  };

  const selectDiscoveredModel = (m: DiscoveredModel) => {
    setSelectedModelId(m.id);
    setName(m.name ?? '');
    setModelId(m.id);
    const cw = m.context_window ?? m.max_model_len;
    if (cw != null) {
      setContextWindowInput(String(cw));
      setContextWindowLocked(true);
    } else {
      // Endpoint didn't report a context length — fall back to manual entry
      // for just this field since there's nothing authoritative to lock to.
      setContextWindowInput('');
      setContextWindowLocked(false);
    }
  };

  const handleDiscover = async () => {
    if (!baseUrl.trim()) return;
    setDiscoverStatus('loading');
    setDiscoverError(null);
    try {
      const res = await modelsApi.discover({
        base_url: baseUrl.trim(),
        api_key: apiKey.trim() || undefined,
      });
      if (res.models.length === 0) {
        setDiscoverStatus('error');
        setDiscoverError(res.errors[0] || 'No models were found at this endpoint.');
        return;
      }
      setDiscoverStatus('idle');
      setDiscoveredModels(res.models);
      setFieldsMode('discovered');
      if (res.models.length === 1) {
        selectDiscoveredModel(res.models[0]);
        setAutoDetected(true);
      } else {
        setSelectedModelId(null);
      }
    } catch (err) {
      setDiscoverStatus('error');
      setDiscoverError(err instanceof Error ? err.message : 'Could not reach this endpoint.');
    }
  };

  const contextWindowValid = contextWindowInput.trim() !== '' && !Number.isNaN(Number(contextWindowInput)) && Number(contextWindowInput) > 0;

  // Full validation (name/id/category/base URL/context window) applies to a
  // real create — either CREATE mode outright, or an edit that discovered a
  // different model and will therefore be submitted as a new registration.
  const needsFullValidation = mode === 'create' || willCreateNew;
  const valid = needsFullValidation
    ? Boolean(name.trim() && modelId.trim() && category.trim() && baseUrl.trim() && contextWindowValid)
    : Boolean(name.trim());

  const submitLabel = needsFullValidation
    ? (submitting ? 'Registering…' : (willCreateNew ? 'Register as New Model' : 'Register Model'))
    : (submitting ? 'Saving…' : 'Update Model');

  const handleSubmit = () => {
    if (!valid) return;
    if (needsFullValidation) {
      onSubmit({
        kind: 'create',
        payload: {
          name: name.trim(),
          model_id: modelId.trim(),
          category: category.trim(),
          base_url: baseUrl.trim(),
          api_key: apiKey,
          context_window: Number(contextWindowInput),
          description: description.trim(),
        },
      });
      return;
    }
    // Plain update — only name/description are persisted.
    onSubmit({
      kind: 'update',
      model_id: (initialModel?.id ?? modelId).trim(),
      name: name.trim(),
      description: description.trim(),
    });
  };

  return (
    <div className={styles['drawer-overlay']}>
      <div className={styles.drawer}>
        <div className={styles['drawer__hdr']}>
          <h2>{isEdit ? 'Edit Custom Model' : 'Register Custom Model'}</h2>
          <button className={styles['drawer__close']} onClick={onClose}><X size={18} /></button>
        </div>

        <div className={styles['drawer__body']}>
          <div className="fg">
            <label className="fl">Base URL</label>
            <input
              className="fi"
              value={baseUrl}
              onChange={(e) => setBaseUrl(e.target.value)}
              placeholder="https://…"
            />
          </div>

          <div className="fg">
            <label className="fl">API Key <span className="opt">(optional)</span></label>
            <input
              className="fi"
              type="password"
              value={apiKey}
              onChange={(e) => setApiKey(e.target.value)}
              placeholder="Paste API key…"
            />
          </div>

          {/* ---- custom category dropdown ---- */}
          <div className="fg">
            <label className="fl">Category</label>
            <div className={styles['combo']} ref={categoryRef}>
              <button
                type="button"
                className={`${styles['combo-trigger']} ${categoryOpen ? styles['combo-trigger--open'] : ''}`}
                onClick={() => setCategoryOpen((o) => !o)}
                disabled={categoriesStatus === 'loading'}
              >
                {categoriesStatus === 'loading' ? (
                  <span className={styles['combo-placeholder']}>Loading categories…</span>
                ) : selectedCategory ? (
                  <span className={styles['combo-value']}>
                    <span className={styles['combo-value-label']}>{selectedCategory.label}</span>
                  </span>
                ) : (
                  <span className={styles['combo-placeholder']}>Select a category</span>
                )}
                <ChevronDown size={15} className={`${styles['combo-caret']} ${categoryOpen ? styles['combo-caret--open'] : ''}`} />
              </button>

              {categoryOpen && (
                <div className={styles['combo-panel']} role="listbox">
                  {categories.length === 0 && categoriesStatus !== 'loading' && (
                    <div className={styles['combo-empty']}>No categories available.</div>
                  )}
                  {categories.map((c) => (
                    <button
                      type="button"
                      key={c.value}
                      role="option"
                      aria-selected={category === c.value}
                      className={`${styles['combo-option']} ${category === c.value ? styles['combo-option--selected'] : ''}`}
                      onClick={() => { setCategory(c.value); setCategoryOpen(false); }}
                    >
                      <div className={styles['combo-option-check']}>
                        {category === c.value && <Check size={14} />}
                      </div>
                      <div className={styles['combo-option-text']}>
                        <span className={styles['combo-option-label']}>{c.label}</span>
                        {c.description && <span className={styles['combo-option-desc']}>{c.description}</span>}
                      </div>
                    </button>
                  ))}
                </div>
              )}
            </div>
            {categoriesStatus === 'failed' && (
              <div className={styles['field-hint--error']}><AlertCircle size={12} /> Couldn't load categories. Try again shortly.</div>
            )}
          </div>

          {/* ---- discovery panel ----
              CREATE: hidden until the user opts into discover-or-manual.
              EDIT: always available as an optional "re-check" action. */}
          {(mode === 'create' ? fieldsMode === 'hidden' : true) && (
            <div className={styles['discover-panel']}>
              <div className={styles['discover-panel-icon']}>
                <Sparkles size={16} />
              </div>
              <div className={styles['discover-panel-body']}>
                <div className={styles['discover-panel-title']}>
                  {isEdit ? 'Re-check this endpoint' : 'Auto-detect this model'}
                </div>
                <p className={styles['discover-panel-text']}>
                  {isEdit
                    ? "We'll probe the base URL again to confirm this is still the same model — or catch it if the endpoint now serves something else."
                    : 'We can probe the base URL to pull in the model name, ID, and context window automatically.'}
                </p>

                <div className={styles['discover-panel-actions']}>
                  <button
                    type="button"
                    className={styles['discover-btn']}
                    disabled={!baseUrl.trim() || discoverStatus === 'loading'}
                    onClick={handleDiscover}
                  >
                    {discoverStatus === 'loading' ? (
                      <Loader2 size={14} className={styles['spin']} />
                    ) : (
                      <Search size={14} />
                    )}
                    {discoverStatus === 'loading' ? 'Discovering…' : 'Discover Models'}
                  </button>
                  {!isEdit && (
                    <button type="button" className={styles['discover-link']} onClick={() => setFieldsMode('manual')}>
                      <PenLine size={12} /> or enter details manually
                    </button>
                  )}
                </div>

                {!baseUrl.trim() && (
                  <div className={styles['field-hint']}>Add a base URL above to enable discovery.</div>
                )}
                {discoverStatus === 'error' && (
                  <div className={styles['field-hint--error']}>
                    <AlertCircle size={13} /> {discoverError}
                  </div>
                )}
              </div>
            </div>
          )}

          {/* Same-model confirmation (edit only) */}
          {matchConfirmed && (
            <div className={styles['success-banner']}>
              <CheckCircle2 size={15} />
              <span>Confirmed — this endpoint still serves the same model.</span>
            </div>
          )}

          {/* Different-model warning (edit only) */}
          {willCreateNew && initialModel && (
            <div className={styles['mismatch-banner']}>
              <AlertTriangle size={15} />
              <span>
                This endpoint now reports a different model ID (<code>{modelId}</code>) than the one you're
                editing (<code>{initialModel.id}</code>). Saving will register this as a <strong>new model</strong> —
                "{initialModel.name ?? initialModel.id}" itself will be left unchanged.
              </span>
            </div>
          )}

          {/* Create-mode auto-detect confirmation */}
          {!isEdit && fieldsMode === 'discovered' && autoDetected && selectedModelId && (
            <div className={styles['success-banner']}>
              <CheckCircle2 size={15} />
              <span>Model detected automatically from this endpoint.</span>
            </div>
          )}

          {fieldsMode === 'discovered' && discoveredModels.length > 1 && (
            <div className="fg">
              <label className="fl">
                Discovered Models <span className="opt">({discoveredModels.length} found — select one)</span>
              </label>
              <div className={styles['discovered-list']}>
                {discoveredModels.map((m) => (
                  <button
                    type="button"
                    key={m.id}
                    className={`${styles['discovered-row']} ${selectedModelId === m.id ? styles['discovered-row--selected'] : ''}`}
                    onClick={() => selectDiscoveredModel(m)}
                  >
                    <div className={styles['discovered-row-main']}>
                      <span className={styles['discovered-row-name']}>{m.name || m.id}</span>
                      <span className={styles['discovered-row-id']}>{m.id}</span>
                    </div>
                    <div className={styles['discovered-row-meta']}>
                      {(m.context_window ?? m.max_model_len) != null && (
                        <span>{(m.context_window ?? m.max_model_len)!.toLocaleString()} ctx</span>
                      )}
                      {m.owned_by && <span>{m.owned_by}</span>}
                      {isEdit && initialModel && m.id === initialModel.id && (
                        <span className={styles['same-model-badge']}>Currently editing</span>
                      )}
                      {m.already_added && m.id !== initialModel?.id && (
                        <span className={styles['already-added-badge']}>Already added</span>
                      )}
                    </div>
                    {selectedModelId === m.id && <CheckCircle2 size={16} className={styles['discovered-row-check']} />}
                  </button>
                ))}
              </div>
            </div>
          )}

          {(isEdit || fieldsMode === 'manual' || (fieldsMode === 'discovered' && selectedModelId)) && (
            <>
              <div className="fg">
                <label className="fl">Model Name</label>
                <input className="fi" value={name} onChange={(e) => setName(e.target.value)} placeholder="e.g. My Fine-tuned Llama" />
              </div>

              <div className="fg">
                <label className="fl">
                  Model ID {(isEdit || fieldsMode === 'discovered') && <span className={styles['locked-tag']}>{isEdit ? 'Not editable' : 'Locked from discovery'}</span>}
                </label>
                <input
                  className="fi"
                  value={modelId}
                  onChange={(e) => setModelId(e.target.value)}
                  placeholder="e.g. llama-3-70b-custom"
                  disabled={isEdit || fieldsMode === 'discovered'}
                />
              </div>

              <div className="fg">
                <label className="fl">
                  Context Window {contextWindowLocked && <span className={styles['locked-tag']}>Locked from discovery</span>}
                </label>
                <input
                  className="fi"
                  type="number"
                  min={1}
                  value={contextWindowInput}
                  onChange={(e) => setContextWindowInput(e.target.value)}
                  placeholder="e.g. 128000"
                  disabled={contextWindowLocked}
                />
                {fieldsMode === 'discovered' && !contextWindowLocked && (
                  <div className={styles['field-hint']}>This endpoint didn't report a context length — enter it manually.</div>
                )}
              </div>

              {(willCreateNew || (!isEdit && fieldsMode === 'discovered')) && (
                <button type="button" className={styles['reset-link']} onClick={resetModelFields}>
                  <RotateCcw size={12} /> {isEdit ? 'Cancel — keep editing the original model' : 'Use a different endpoint'}
                </button>
              )}
            </>
          )}

          <div className="fg">
            <label className="fl">Description <span className="opt">(optional)</span></label>
            <textarea className="fi" rows={3} value={description} onChange={(e) => setDescription(e.target.value)} />
          </div>
        </div>

        <div className={styles['drawer__foot']}>
          <button className="btn btn-ghost" onClick={onClose}>Cancel</button>
          <button className="btn btn-ind" disabled={!valid || submitting} onClick={handleSubmit}>{submitLabel}</button>
        </div>
      </div>
    </div>
  );
}





























//Providers.tsx
import { useEffect, useState } from 'react';
import { Search, Check, Plus, Settings, Unlink, Loader2, Cable, Trash2, RefreshCw, Eye, ListPlus, ListFilter, ListChecks } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import {
  fetchProviders,
  createProvider,
  deleteProvider,
  connectProvider,
  disconnectProvider,
  syncModels,
} from '../../store/slices/providersSlice';
import { fetchModelsByProvider, createCustomModel, deleteCustomModel, updateCustomModel } from '../../store/slices/modelsSlice';
import AddProviderDrawer from './AddProviderDrawer';
import AddCustomModelDrawer from './AddCustomModelDrawer';
import ProviderModelsSidebar from './ProviderModelsSidebar';
import { SkeletonCards } from '../common/Skeleton';
import { useToast } from '../common/Toast';
import styles from './Providers.module.scss';
import type { Provider } from '../../types';

type Filter = 'all' | 'connected' | 'available';

const FILTERS: Filter[] = ['all', 'connected', 'available'];

// Thunks rejected via axios surface as either an Error (network/message) or
// an object with a response payload — normalize both into a display string.
function getErrorMessage(err: unknown, fallback: string): string {
  if (err && typeof err === 'object') {
    const anyErr = err as { response?: { data?: { message?: string; detail?: string } }; message?: string };
    const serverMessage = anyErr.response?.data?.message || anyErr.response?.data?.detail;
    if (serverMessage) return serverMessage;
    if (anyErr.message) return anyErr.message;
  }
  return fallback;
}

export default function Providers() {
  const dispatch = useAppDispatch();
  const { items, status, mutatingId, creating, syncingId } = useAppSelector((s) => s.providers);
  const modelsByProvider = useAppSelector((s) => s.models.byProvider);
  const modelsByProviderStatus = useAppSelector((s) => s.models.byProviderStatus);
  const customModelCreating = useAppSelector((s) => s.models.creating);
  const customModelDeletingId = useAppSelector((s) => s.models.deletingId);
  const customModelUpdatingId = useAppSelector((s) => s.models.updatingId);
  const [search, setSearch] = useState('');
  const [filter, setFilter] = useState<Filter>('all');
  const [keyPromptFor, setKeyPromptFor] = useState<string | null>(null);
  const [apiKeyInput, setApiKeyInput] = useState('');
  const [drawerOpen, setDrawerOpen] = useState(false);
  const [addModelOpen, setAddModelOpen] = useState(false);
  const [viewModelsProvider, setViewModelsProvider] = useState<Provider | null>(null);
  const toast = useToast();

  useEffect(() => {
    dispatch(fetchProviders());
  }, [dispatch]);

  const connectedCount = items.filter((p) => p.status === 'connected').length;

  const filtered = items.filter((p) => {
    if (filter === 'connected' && p.status !== 'connected') return false;
    if (filter === 'available' && p.status === 'connected') return false;
    return !search || (p.name ?? '').toLowerCase().includes(search.toLowerCase());
  });

  const submitConnect = (providerId: string) => {
    if (!apiKeyInput.trim()) return;
    dispatch(connectProvider({ providerId, payload: { api_key: apiKeyInput } }));
    setKeyPromptFor(null);
    setApiKeyInput('');
  };

  const openModelsSidebar = (p: Provider) => {
    setViewModelsProvider(p);
    dispatch(fetchModelsByProvider(p.id));
  };

  const customProvider = items.find((p) => p.name === 'Custom') || null;

  return (
    <div className="page-enter pg-shell">
      <div className={styles.providers__header}>
        <div>
          <p className={styles['providers__header-eyebrow']}>Integrations</p>
          <h1>Providers</h1>
          <p className={styles['providers__header-sub']}>Manage your AI provider connections</p>
        </div>
        <div className={styles['providers__header-meta']}>
          <Cable size={13} />
          {connectedCount} of {items.length} connected
        </div>
      </div>

      <div className={styles['providers__toolbar']}>
        <div className={styles['providers__search']}>
          <Search size={16} />
          <input placeholder="Search providers…" value={search} onChange={(e) => setSearch(e.target.value)} />
        </div>

        <div className={styles['providers__toolbar-right']}>
          <div className={styles['providers__filter-group']}>
            <span className={styles['providers__toolbar-label']}>
              <ListFilter size={11} /> Status
            </span>
            {FILTERS.map((f) => (
              <button
                key={f}
                className={`${styles['providers__filter-pill']} ${filter === f ? styles['providers__filter-pill--on'] : ''}`}
                onClick={() => setFilter(f)}
              >
                {f[0].toUpperCase() + f.slice(1)}
              </button>
            ))}
          </div>
          <span className={styles['providers__toolbar-divider']} />
          <button className={styles['providers__add-btn']} onClick={() => setDrawerOpen(true)}>
            <Plus size={14} /> Add Provider
          </button>
        </div>
      </div>

      <div className="pg-body">
        <div className={styles['providers__grid']}>
          {status === 'loading' && <SkeletonCards count={6} />}
          {status !== 'loading' &&
            filtered.map((p) => {
              const isCustom = p.name === 'Custom';
              return (
                <div className={styles['providers__card']} key={p.id}>
                  <div className={styles['providers__card-hdr']}>
                    <div className={styles['providers__card-id']}>
                      <div className={styles['providers__icon']}>
                        {p.logo_url ? <img src={p.logo_url} alt={p.name ?? 'Provider'} /> : (p.name?.[0] ?? '?')}
                      </div>
                      <div style={{ minWidth: 0 }}>
                        <div className={styles['providers__name']}>{p.name ?? 'Unnamed provider'}</div>
                        <div className={styles['providers__count']}>{p.model_count ?? 0} models</div>
                      </div>
                    </div>
                    <div className={styles['providers__card-top-actions']}>
                      <button
                        className={styles['providers__icon-btn']}
                        onClick={() => openModelsSidebar(p)}
                        title={isCustom ? 'Manage models' : 'View models'}
                        aria-label={`${isCustom ? 'Manage' : 'View'} models for ${p.name ?? 'provider'}`}
                      >
                        {isCustom ? <ListChecks size={14} /> : <Eye size={14} />}
                      </button>
                      {p.status === 'connected' ? (
                        <span className={styles['providers__badge-connected']}>
                          <Check size={10} strokeWidth={3} /> Connected
                        </span>
                      ) : (
                        <span className={styles['providers__badge-idle']}>Not connected</span>
                      )}
                    </div>
                  </div>

                  <div className={styles['providers__desc']}>{p.description}</div>

                  {keyPromptFor === p.id ? (
                    <div className={styles['providers__key-form']}>
                      <input
                        className={styles['providers__key-input']}
                        type="password"
                        placeholder="Paste API key…"
                        value={apiKeyInput}
                        onChange={(e) => setApiKeyInput(e.target.value)}
                        autoFocus
                      />
                      <div className={styles['providers__key-actions']}>
                        <button
                          className={`${styles['providers__foot-btn']} ${styles['providers__foot-btn--primary']}`}
                          onClick={() => submitConnect(p.id)}
                        >
                          Save
                        </button>
                        <button
                          className={`${styles['providers__foot-btn']} ${styles['providers__foot-btn--ghost']}`}
                          onClick={() => setKeyPromptFor(null)}
                        >
                          Cancel
                        </button>
                      </div>
                    </div>
                  ) : (
                    <div className={styles['providers__foot-actions']}>
                      {isCustom && (
                        <button
                          className={`${styles['providers__foot-btn']} ${styles['providers__foot-btn--accent']}`}
                          onClick={() => setAddModelOpen(true)}
                        >
                          <ListPlus size={13} /> Add Model
                        </button>
                      )}
                      {p.status !== 'connected' && (
                        <button
                          className={`${styles['providers__foot-btn']} ${styles['providers__foot-btn--primary']}`}
                          disabled={mutatingId === p.id}
                          onClick={() => setKeyPromptFor(p.id)}
                        >
                          {mutatingId === p.id ? (
                            <Loader2 size={13} className={styles['providers__spin']} />
                          ) : (
                            <>
                              <Plus size={13} /> Connect
                            </>
                          )}
                        </button>
                      )}
                      {/* Configure button temporarily disabled per request
                      {p.status === 'connected' && (
                        <button
                          className={`${styles['providers__foot-btn']} ${styles['providers__foot-btn--ghost']}`}
                          disabled={mutatingId === p.id}
                          onClick={() => setKeyPromptFor(p.id)}
                        >
                          {mutatingId === p.id ? (
                            <Loader2 size={13} className={styles['providers__spin']} />
                          ) : (
                            <>
                              <Settings size={13} /> Configure
                            </>
                          )}
                        </button>
                      )}
                      */}
                      {p.status === 'connected' && (
                        <>
                          <button
                            className={`${styles['providers__foot-btn']} ${styles['providers__foot-btn--ghost']}`}
                            disabled={syncingId === p.id}
                            onClick={() => dispatch(syncModels(p.id))}
                          >
                            {syncingId === p.id ? (
                              <Loader2 size={13} className={styles['providers__spin']} />
                            ) : (
                              <>
                                <RefreshCw size={13} /> Sync
                              </>
                            )}
                          </button>
                          <button
                            className={`${styles['providers__foot-btn']} ${styles['providers__foot-btn--danger']}`}
                            disabled={mutatingId === p.id}
                            onClick={() => dispatch(disconnectProvider(p.id))}
                          >
                            <Unlink size={13} /> Disconnect
                          </button>
                        </>
                      )}
                      {p.status !== 'connected' && (
                        <button
                          className={`${styles['providers__foot-btn']} ${styles['providers__foot-btn--danger']}`}
                          disabled={mutatingId === p.id}
                          onClick={() => {
                            if (window.confirm(`Delete ${p.name ?? 'this provider'}? This cannot be undone.`)) {
                              dispatch(deleteProvider(p.id));
                            }
                          }}
                        >
                          <Trash2 size={13} /> Delete
                        </button>
                      )}
                    </div>
                  )}
                </div>
              );
            })}
          {status !== 'loading' && filtered.length === 0 && (
            <p className={styles['providers__empty']}>No providers match your search or filter.</p>
          )}
        </div>
      </div>

      {drawerOpen && (
        <AddProviderDrawer
          submitting={creating}
          onClose={() => setDrawerOpen(false)}
          onSubmit={(payload) => {
            dispatch(createProvider(payload)).then(() => setDrawerOpen(false));
          }}
        />
      )}

      {addModelOpen && customProvider && (
        <AddCustomModelDrawer
          mode="create"
          submitting={customModelCreating}
          onClose={() => setAddModelOpen(false)}
          onSubmit={(result) => {
            if (result.kind !== 'create') return;
            dispatch(createCustomModel(result.payload))
              .unwrap()
              .then(() => {
                setAddModelOpen(false);
                toast.success(`"${result.payload.name}" was registered successfully.`, { title: 'Model registered' });
                dispatch(fetchProviders());
                dispatch(fetchModelsByProvider(customProvider.id));
              })
              .catch((err) => {
                toast.error(getErrorMessage(err, 'Failed to register model.'), { title: 'Registration failed' });
              });
          }}
        />
      )}

      {viewModelsProvider && (
        <ProviderModelsSidebar
          provider={viewModelsProvider}
          models={modelsByProvider[viewModelsProvider.id] || []}
          status={modelsByProviderStatus[viewModelsProvider.id] || 'idle'}
          onClose={() => setViewModelsProvider(null)}
          canManage={viewModelsProvider.name === 'Custom'}
          deletingId={customModelDeletingId}
          updatingId={customModelUpdatingId}
          creatingNew={customModelCreating}
          onDelete={(modelId) => {
            dispatch(deleteCustomModel({ modelId, providerId: viewModelsProvider.id }))
              .unwrap()
              .then(() => {
                toast.success('Model removed successfully.', { title: 'Model removed' });
                dispatch(fetchProviders());
              })
              .catch((err) => {
                toast.error(getErrorMessage(err, 'Failed to remove model.'), { title: 'Removal failed' });
              });
          }}
          onEditSubmit={(result) => {
            if (result.kind === 'update') {
              return dispatch(updateCustomModel({
                model_id: result.model_id,
                name: result.name,
                description: result.description,
              }))
                .unwrap()
                .then((res) => {
                  toast.success(`"${res.name}" was updated successfully.`, { title: 'Model updated' });
                })
                .catch((err) => {
                  toast.error(getErrorMessage(err, 'Failed to update model.'), { title: 'Update failed' });
                  throw err;
                });
            }
            // Discovery found a different model id — register it as a new
            // model instead of mutating the one being edited.
            return dispatch(createCustomModel(result.payload))
              .unwrap()
              .then(() => {
                toast.success(`"${result.payload.name}" was registered as a new model.`, { title: 'Model registered' });
                dispatch(fetchProviders());
                dispatch(fetchModelsByProvider(viewModelsProvider.id));
              })
              .catch((err) => {
                toast.error(getErrorMessage(err, 'Failed to register model.'), { title: 'Registration failed' });
                throw err;
              });
          }}
        />
      )}
    </div>
  );
}
















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
                  <span className={styles['providers__model-row-name']}>{m.name ?? 'Unnamed model'}</span>
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
