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
    <div className={styles['drawer-overlay']} onClick={onClose}>
      <div className={styles.drawer} onClick={(e) => e.stopPropagation()}>
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


















//Addcustommodeldrawer.module.scss
@use '../../styles/_variables' as *;

// Font scaling: `.drawer` sets a single base font-size. All descendant
// font-sizes are expressed in `em` (relative to that base), so bumping
// `.drawer`'s font-size on wide screens scales the whole drawer
// proportionally from one place — same convention as Sidebar, Providers,
// and Model Catalog.

// base font-size the drawer's internal `em` scale is built on
$drawer-base-font: 13px;

.drawer-overlay {
  position: fixed; top: 0; left: 0; right: 0; bottom: $footer-height;
  background: rgba(17, 24, 39, .4); z-index: 100;
  display: flex; justify-content: flex-end;
}
.drawer {
  width: 420px; max-width: 100%; height: calc(100% - 30px); background: $surface; box-shadow: $shadow-4;
  display: flex; flex-direction: column; animation: drawerIn .25s ease both;

  // master scale control — every em-based font-size below responds to this
  font-size: $drawer-base-font;

  @media (min-width: 1800px) {
    font-size: 16px;
  }
}
@keyframes drawerIn { from { transform: translateX(24px); opacity: 0; } to { transform: translateX(0); opacity: 1; } }
.drawer__hdr {
  display: flex; justify-content: space-between; align-items: center; padding: 20px 24px; border-bottom: 1px solid $border-light;
  h2 { font-size: 1.3846em; font-weight: 700; } // 18px / 13px
}
.drawer__close { background: none; border: none; cursor: pointer; color: $text-muted; }
.drawer__body { flex: 1; overflow-y: auto; padding: 24px; }
.drawer__foot { display: flex; justify-content: flex-end; gap: 10px; padding: 16px 24px; border-top: 1px solid $border-light; }

// ---- category custom dropdown ------------------------------------------------
.combo {
  position: relative;
}
.combo-trigger {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
  width: 100%;
  padding: 10px 12px;
  border: 1.5px solid $border;
  border-radius: 10px;
  background: $surface;
  cursor: pointer;
  text-align: left;
  transition: border-color 0.15s ease, box-shadow 0.15s ease, background 0.15s ease;

  &:hover:not(:disabled) { border-color: $indigo; }
  &:disabled { cursor: not-allowed; opacity: 0.6; }

  &--open {
    border-color: $indigo;
    box-shadow: 0 0 0 3px $indigo-pale;
  }
}
.combo-value {
  display: flex;
  align-items: center;
  min-width: 0;
}
.combo-value-label {
  font-size: 1em; // 13px / 13px (base)
  font-weight: 650;
  color: $text-primary;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.combo-placeholder {
  font-size: 1em; // 13px / 13px (base)
  color: $text-muted;
}
.combo-caret {
  flex-shrink: 0;
  color: $text-muted;
  transition: transform 0.18s ease;

  &--open { transform: rotate(180deg); color: $indigo; }
}
.combo-panel {
  position: absolute;
  z-index: 20;
  top: calc(100% + 6px);
  left: 0;
  right: 0;
  max-height: 280px;
  overflow-y: auto;
  padding: 6px;
  background: $surface;
  border: 1px solid $border;
  border-radius: 12px;
  box-shadow: $shadow-3;
  animation: combo-panel-in 0.14s ease both;
}
@keyframes combo-panel-in {
  from { opacity: 0; transform: translateY(-4px); }
  to { opacity: 1; transform: translateY(0); }
}
.combo-empty {
  padding: 14px 10px;
  font-size: 0.9615em; // 12.5px / 13px
  color: $text-muted;
  text-align: center;
}
.combo-option {
  display: flex;
  align-items: flex-start;
  gap: 8px;
  width: 100%;
  padding: 9px 10px;
  border: none;
  border-radius: 8px;
  background: none;
  cursor: pointer;
  text-align: left;
  transition: background 0.12s ease;

  &:hover { background: $surface-alt; }

  &--selected {
    background: $indigo-pale;

    &:hover { background: $indigo-pale; }
  }

  & + & { margin-top: 1px; }
}
.combo-option-check {
  flex-shrink: 0;
  width: 14px;
  height: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
  color: $indigo;
}
.combo-option-text {
  display: flex;
  flex-direction: column;
  gap: 2px;
  min-width: 0;
}
.combo-option-label {
  font-size: 1em; // 13px / 13px (base)
  font-weight: 650;
  color: $text-primary;
}
.combo-option-desc {
  font-size: 0.8846em; // 11.5px / 13px
  line-height: 1.4;
  color: $text-secondary;
}

// ---- hint / error text -------------------------------------------------------
.field-hint {
  margin-top: 6px;
  font-size: 0.9231em; // 12px / 13px
  line-height: 1.45;
  color: $text-secondary;
}
.field-hint--error {
  display: flex;
  align-items: center;
  gap: 6px;
  margin-top: 8px;
  font-size: 0.9231em; // 12px / 13px
  color: #DC2626;
}
.locked-tag {
  margin-left: 6px;
  padding: 1px 7px;
  border-radius: 999px;
  background: $emerald-pale;
  color: $emerald-dark;
  font-size: 0.8077em; // 10.5px / 13px
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.03em;
  vertical-align: middle;
}

// ---- optional-discovery panel -------------------------------------------------
.discover-panel {
  display: flex;
  gap: 12px;
  margin-bottom: 18px;
  padding: 16px;
  border-radius: 14px;
  background: linear-gradient(165deg, $indigo-pale 0%, $surface-alt 65%);
  border: 1px solid rgba($indigo, 0.16);
  position: relative;
  overflow: hidden;
}
.discover-panel-icon {
  flex-shrink: 0;
  width: 32px;
  height: 32px;
  border-radius: 9px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: $indigo;
  color: #fff;
  box-shadow: 0 4px 10px -3px rgba($indigo, 0.5);
}
.discover-panel-body {
  min-width: 0;
  flex: 1;
}
.discover-panel-title {
  font-size: 1.0385em; // 13.5px / 13px
  font-weight: 750;
  color: $text-primary;
  margin-bottom: 3px;
}
.discover-panel-text {
  font-size: 0.9231em; // 12px / 13px
  line-height: 1.5;
  color: $text-secondary;
  margin: 0 0 12px;
}
.discover-panel-actions {
  display: flex;
  align-items: center;
  gap: 14px;
  flex-wrap: wrap;
}
.discover-btn {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 8px 14px;
  border-radius: 9px;
  border: 1px solid $indigo;
  background: $indigo;
  color: #fff;
  font-size: 0.9615em; // 12.5px / 13px
  font-weight: 700;
  cursor: pointer;
  box-shadow: 0 2px 6px -2px rgba($indigo, 0.5);
  transition: background 0.15s ease, border-color 0.15s ease, transform 0.12s ease, box-shadow 0.15s ease;

  &:hover:not(:disabled) { background: $indigo-dark; border-color: $indigo-dark; transform: translateY(-1px); box-shadow: 0 4px 10px -2px rgba($indigo, 0.55); }
  &:disabled { opacity: 0.5; cursor: not-allowed; transform: none; box-shadow: none; }
}
.discover-link {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  padding: 0;
  border: none;
  background: none;
  font-size: 0.9231em; // 12px / 13px
  font-weight: 600;
  color: $text-secondary;
  cursor: pointer;
  transition: color 0.15s ease;

  &:hover { color: $indigo; }
}
.spin { animation: add-custom-model-spin 0.8s linear infinite; }
@keyframes add-custom-model-spin { to { transform: rotate(360deg); } }

// ---- discovery success banner --------------------------------------------------
.success-banner {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 16px;
  padding: 10px 13px;
  border-radius: 10px;
  background: $emerald-pale;
  border: 1px solid rgba($emerald, 0.25);
  color: $emerald-dark;
  font-size: 0.9615em; // 12.5px / 13px
  font-weight: 650;

  svg { flex-shrink: 0; }
}

// ---- edit-mode "different model detected" warning -------------------------------
.mismatch-banner {
  display: flex;
  align-items: flex-start;
  gap: 8px;
  margin-bottom: 16px;
  padding: 11px 13px;
  border-radius: 10px;
  background: $amber-pale;
  border: 1px solid rgba($amber-dark, 0.3);
  color: #92400E;
  font-size: 0.9231em; // 12px / 13px
  font-weight: 550;
  line-height: 1.5;

  svg { flex-shrink: 0; margin-top: 1px; color: $amber-dark; }
  strong { font-weight: 750; }
  code {
    font-family: monospace;
    font-size: 0.92em;
    background: rgba(0, 0, 0, 0.06);
    padding: 1px 4px;
    border-radius: 4px;
  }
}

.same-model-badge {
  padding: 1px 7px;
  border-radius: 999px;
  background: $indigo-pale;
  color: $indigo;
  font-weight: 700;
}

// ---- discovered model picker -------------------------------------------------
.discovered-list {
  display: flex;
  flex-direction: column;
  gap: 6px;
  max-height: 220px;
  overflow-y: auto;
  padding: 2px;
}
.discovered-row {
  position: relative;
  display: flex;
  flex-direction: column;
  gap: 4px;
  width: 100%;
  padding: 10px 34px 10px 12px;
  border: 1.5px solid $border;
  border-radius: 9px;
  background: $surface;
  cursor: pointer;
  text-align: left;
  transition: border-color 0.14s ease, background 0.14s ease;

  &:hover { border-color: $indigo; background: $indigo-pale; }

  &--selected {
    border-color: $indigo;
    background: $indigo-pale;
  }
}
.discovered-row-main {
  display: flex;
  align-items: baseline;
  gap: 8px;
  min-width: 0;
}
.discovered-row-name {
  font-weight: 700;
  font-size: 1em; // 13px / 13px (base)
  color: $text-primary;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.discovered-row-id {
  flex-shrink: 0;
  font-family: monospace;
  font-size: 0.8462em; // 11px / 13px
  color: $text-muted;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.discovered-row-meta {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  font-size: 0.8462em; // 11px / 13px
  color: $text-secondary;
}
.already-added-badge {
  padding: 1px 7px;
  border-radius: 999px;
  background: rgba(245, 158, 11, 0.16);
  color: #B45309;
  font-weight: 700;
}
.discovered-row-check {
  position: absolute;
  top: 50%;
  right: 10px;
  transform: translateY(-50%);
  color: $indigo;
}

// ---- reset / mode-switch link -------------------------------------------------
.reset-link {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  margin: -6px 0 16px;
  padding: 0;
  border: none;
  background: none;
  font-size: 0.9231em; // 12px / 13px
  font-weight: 650;
  color: $indigo;
  cursor: pointer;

  &:hover { text-decoration: underline; }
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
      (result as Promise<unknown>).then(() => setEditingModel(null));
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
import styles from './Providers.module.scss';
import type { Provider } from '../../types';

type Filter = 'all' | 'connected' | 'available';

const FILTERS: Filter[] = ['all', 'connected', 'available'];

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
            dispatch(createCustomModel(result.payload)).then(() => {
              setAddModelOpen(false);
              dispatch(fetchProviders());
              dispatch(fetchModelsByProvider(customProvider.id));
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
            dispatch(deleteCustomModel({ modelId, providerId: viewModelsProvider.id })).then(() => {
              dispatch(fetchProviders());
            });
          }}
          onEditSubmit={(result) => {
            if (result.kind === 'update') {
              return dispatch(updateCustomModel({
                model_id: result.model_id,
                name: result.name,
                description: result.description,
              }));
            }
            // Discovery found a different model id — register it as a new
            // model instead of mutating the one being edited.
            return dispatch(createCustomModel(result.payload)).then(() => {
              dispatch(fetchProviders());
              dispatch(fetchModelsByProvider(viewModelsProvider.id));
            });
          }}
        />
      )}
    </div>
  );
}
