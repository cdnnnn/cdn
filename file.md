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
  Sparkles,
  PenLine,
} from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchModelCategories } from '../../store/slices/modelsSlice';
import { modelsApi, type DiscoveredModel } from '../../api/endpoints/models';
import type { CustomModelRequest } from '../../types';
import styles from './AddCustomModelDrawer.module.scss';

interface Props {
  onClose: () => void;
  onSubmit: (payload: CustomModelRequest) => void;
  submitting: boolean;
}

// Fields that describe the actual model (name/id/context window) start out
// hidden — the user either runs discovery against the base URL to auto-fill
// them, or opts into filling them in by hand.
type FieldsMode = 'hidden' | 'manual' | 'discovered';

export default function AddCustomModelDrawer({ onClose, onSubmit, submitting }: Props) {
  const dispatch = useAppDispatch();
  const categories = useAppSelector((s) => s.models.categories);
  const categoriesStatus = useAppSelector((s) => s.models.categoriesStatus);

  const [baseUrl, setBaseUrl] = useState('');
  const [apiKey, setApiKey] = useState('');
  const [category, setCategory] = useState('');
  const [categoryOpen, setCategoryOpen] = useState(false);
  const categoryRef = useRef<HTMLDivElement>(null);
  const [description, setDescription] = useState('');

  const [fieldsMode, setFieldsMode] = useState<FieldsMode>('hidden');
  const [name, setName] = useState('');
  const [modelId, setModelId] = useState('');
  const [contextWindowInput, setContextWindowInput] = useState('');
  const [contextWindowLocked, setContextWindowLocked] = useState(false);
  const [autoDetected, setAutoDetected] = useState(false);

  const [discoverStatus, setDiscoverStatus] = useState<'idle' | 'loading' | 'error'>('idle');
  const [discoverError, setDiscoverError] = useState<string | null>(null);
  const [discoveredModels, setDiscoveredModels] = useState<DiscoveredModel[]>([]);
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

  const resetModelFields = () => {
    setFieldsMode('hidden');
    setName('');
    setModelId('');
    setContextWindowInput('');
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

  const valid =
    name.trim() &&
    modelId.trim() &&
    category.trim() &&
    baseUrl.trim() &&
    contextWindowValid;

  const handleSubmit = () => {
    if (!valid) return;
    onSubmit({
      name: name.trim(),
      model_id: modelId.trim(),
      category: category.trim(),
      base_url: baseUrl.trim(),
      api_key: apiKey,
      context_window: Number(contextWindowInput),
      description: description.trim(),
    });
  };

  return (
    <div className={styles['drawer-overlay']} onClick={onClose}>
      <div className={styles.drawer} onClick={(e) => e.stopPropagation()}>
        <div className={styles['drawer__hdr']}>
          <h2>Register Custom Model</h2>
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

          {/* ---- discovery panel: hidden until the user opts into discover-or-manual ---- */}
          {fieldsMode === 'hidden' && (
            <div className={styles['discover-panel']}>
              <div className={styles['discover-panel-icon']}>
                <Sparkles size={16} />
              </div>
              <div className={styles['discover-panel-body']}>
                <div className={styles['discover-panel-title']}>Auto-detect this model</div>
                <p className={styles['discover-panel-text']}>
                  We can probe the base URL to pull in the model name, ID, and context window automatically.
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
                  <button type="button" className={styles['discover-link']} onClick={() => setFieldsMode('manual')}>
                    <PenLine size={12} /> or enter details manually
                  </button>
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

          {fieldsMode === 'discovered' && autoDetected && selectedModelId && (
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
                      {m.already_added && <span className={styles['already-added-badge']}>Already added</span>}
                    </div>
                    {selectedModelId === m.id && <CheckCircle2 size={16} className={styles['discovered-row-check']} />}
                  </button>
                ))}
              </div>
            </div>
          )}

          {(fieldsMode === 'manual' || (fieldsMode === 'discovered' && selectedModelId)) && (
            <>
              <div className="fg">
                <label className="fl">Model Name</label>
                <input className="fi" value={name} onChange={(e) => setName(e.target.value)} placeholder="e.g. My Fine-tuned Llama" />
              </div>

              <div className="fg">
                <label className="fl">
                  Model ID {fieldsMode === 'discovered' && <span className={styles['locked-tag']}>Locked from discovery</span>}
                </label>
                <input
                  className="fi"
                  value={modelId}
                  onChange={(e) => setModelId(e.target.value)}
                  placeholder="e.g. llama-3-70b-custom"
                  disabled={fieldsMode === 'discovered'}
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

              <button type="button" className={styles['reset-link']} onClick={resetModelFields}>
                <RotateCcw size={12} /> {fieldsMode === 'discovered' ? 'Use a different endpoint' : 'Try auto-discovery instead'}
              </button>
            </>
          )}

          <div className="fg">
            <label className="fl">Description <span className="opt">(optional)</span></label>
            <textarea className="fi" rows={3} value={description} onChange={(e) => setDescription(e.target.value)} />
          </div>
        </div>

        <div className={styles['drawer__foot']}>
          <button className="btn btn-ghost" onClick={onClose}>Cancel</button>
          <button className="btn btn-ind" disabled={!valid || submitting} onClick={handleSubmit}>{submitting ? 'Registering…' : 'Register Model'}</button>
        </div>
      </div>
    </div>
  );
}




















@use '../../styles/_variables' as *;

.drawer-overlay {
  position: fixed; top: 0; left: 0; right: 0; bottom: $footer-height;
  background: rgba(17, 24, 39, .4); z-index: 100;
  display: flex; justify-content: flex-end;
}
.drawer {
  width: 420px; max-width: 100%; height: calc(100% - 30px); background: $surface; box-shadow: $shadow-4;
  display: flex; flex-direction: column; animation: drawerIn .25s ease both;
}
@keyframes drawerIn { from { transform: translateX(24px); opacity: 0; } to { transform: translateX(0); opacity: 1; } }
.drawer__hdr {
  display: flex; justify-content: space-between; align-items: center; padding: 20px 24px; border-bottom: 1px solid $border-light;
  h2 { font-size: 18px; font-weight: 700; }
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
  font-size: 13px;
  font-weight: 650;
  color: $text-primary;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.combo-placeholder {
  font-size: 13px;
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
  font-size: 12.5px;
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
  font-size: 13px;
  font-weight: 650;
  color: $text-primary;
}
.combo-option-desc {
  font-size: 11.5px;
  line-height: 1.4;
  color: $text-secondary;
}

// ---- hint / error text -------------------------------------------------------
.field-hint {
  margin-top: 6px;
  font-size: 12px;
  line-height: 1.45;
  color: $text-secondary;
}
.field-hint--error {
  display: flex;
  align-items: center;
  gap: 6px;
  margin-top: 8px;
  font-size: 12px;
  color: #DC2626;
}
.locked-tag {
  margin-left: 6px;
  padding: 1px 7px;
  border-radius: 999px;
  background: $emerald-pale;
  color: $emerald-dark;
  font-size: 10.5px;
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
  font-size: 13.5px;
  font-weight: 750;
  color: $text-primary;
  margin-bottom: 3px;
}
.discover-panel-text {
  font-size: 12px;
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
  font-size: 12.5px;
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
  font-size: 12px;
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
  font-size: 12.5px;
  font-weight: 650;

  svg { flex-shrink: 0; }
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
  font-size: 13px;
  color: $text-primary;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.discovered-row-id {
  flex-shrink: 0;
  font-family: monospace;
  font-size: 11px;
  color: $text-muted;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.discovered-row-meta {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  font-size: 11px;
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
  font-size: 12px;
  font-weight: 650;
  color: $indigo;
  cursor: pointer;

  &:hover { text-decoration: underline; }
}
