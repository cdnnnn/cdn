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
  XCircle,
  Sparkles,
  PenLine,
  SlidersHorizontal,
  ShieldCheck,
} from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchModelCategories } from '../../store/slices/modelsSlice';
import { modelsApi, type DiscoveredModel, type CustomModelRequestWithParams, type VerifyParamsResponse } from '../../api/endpoints/models';
import type { Model } from '../../types';
import styles from './AddCustomModelDrawer.module.scss';

// The update endpoint returns `description`, which the shared `Model` type
// doesn't declare — extend it locally for the edit-prefill case.
export type EditableModel = Model & { description?: string; request_params?: Record<string, unknown> };

// Both create and a plain edit now POST the exact same full payload
// (name, model_id, category, base_url, api_key, context_window, description,
// request_params) — the only difference the caller needs is whether this is
// a brand-new registration or an update to the model being edited, which
// matters for a discovery-mismatch (editing one model but discovery found a
// different id, so it's registered as new rather than mutating the original).
export type CustomModelSubmitPayload =
  | { kind: 'create'; payload: CustomModelRequestWithParams }
  | { kind: 'update'; payload: CustomModelRequestWithParams };

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

  // ---- request params: always-visible, mandatory-verify step ----
  // Auto-filled with a sensible starting template so the user sees a working
  // example immediately; they're free to edit or clear it before saving.
  const DEFAULT_REQUEST_PARAMS_TEXT = '{\n  "chat_template_kwargs": { "thinking": true },\n  "reasoning_effort": "high"\n}';
  const [requestParamsText, setRequestParamsText] = useState(
    initialModel?.request_params
      ? JSON.stringify(initialModel.request_params, null, 2)
      : DEFAULT_REQUEST_PARAMS_TEXT
  );
  const [requestParamsError, setRequestParamsError] = useState<string | null>(null);
  const [verifyStatus, setVerifyStatus] = useState<'idle' | 'loading' | 'success' | 'warning' | 'error'>('idle');
  const [verifyResult, setVerifyResult] = useState<VerifyParamsResponse | null>(null);
  const [verifyError, setVerifyError] = useState<string | null>(null);

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

  // The model id to verify/submit params against — whichever field is
  // currently authoritative for the mode/state combination.
  const effectiveModelId = (isEdit ? (initialModel?.id ?? modelId) : modelId).trim();

  // Any change to the target (endpoint, key, or which model) invalidates a
  // previously-verified result — force the user to re-verify before saving.
  useEffect(() => {
    if (verifyStatus !== 'idle') {
      setVerifyStatus('idle');
      setVerifyResult(null);
      setVerifyError(null);
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [baseUrl, apiKey, effectiveModelId]);

  const selectedCategory = categories.find((c) => c.value === category) || null;

  // Re-parses on every keystroke so the Verify button's enabled state and
  // any JSON error message stay in sync with what's actually in the field.
  const parseRequestParams = (text: string): { ok: true; value: Record<string, unknown> } | { ok: false } => {
    const trimmed = text.trim();
    if (!trimmed) return { ok: true, value: {} };
    try {
      const parsed = JSON.parse(trimmed);
      if (parsed === null || typeof parsed !== 'object' || Array.isArray(parsed)) {
        return { ok: false };
      }
      return { ok: true, value: parsed as Record<string, unknown> };
    } catch {
      return { ok: false };
    }
  };

  const handleRequestParamsChange = (text: string) => {
    setRequestParamsText(text);
    const parsed = parseRequestParams(text);
    setRequestParamsError(parsed.ok ? null : 'Enter valid JSON, e.g. {"reasoning_effort": "high"}');
    // Any edit invalidates a prior verification result.
    if (verifyStatus !== 'idle') {
      setVerifyStatus('idle');
      setVerifyResult(null);
      setVerifyError(null);
    }
  };

  const parsedRequestParams = parseRequestParams(requestParamsText);
  // Whether there's actually something to verify — an empty box or `{}`
  // needs no verification and won't block saving.
  const requestParamsNonEmpty = parsedRequestParams.ok && Object.keys(parsedRequestParams.value).length > 0;
  const needsVerification = requestParamsNonEmpty;
  const paramsVerified = !needsVerification || verifyStatus === 'success' || verifyStatus === 'warning';

  const handleVerifyParams = async () => {
    const parsed = parseRequestParams(requestParamsText);
    if (!parsed.ok) {
      setRequestParamsError('Enter valid JSON, e.g. {"reasoning_effort": "high"}');
      return;
    }
    if (!baseUrl.trim() || !effectiveModelId) return;

    setVerifyStatus('loading');
    setVerifyError(null);
    setVerifyResult(null);
    try {
      const res = await modelsApi.verifyParams({
        base_url: baseUrl.trim(),
        api_key: apiKey.trim() || undefined,
        model_id: effectiveModelId,
        request_params: parsed.value,
      });
      setVerifyResult(res);
      // Any 200 response counts as "verified" and unblocks Save — warnings
      // (or even an unsupported result) are surfaced to the user but don't
      // stop them from saving. Only an actual failed call (network error /
      // non-200, handled in the catch below) blocks saving.
      const hasIssues =
        !res.supported ||
        (res.warning && res.warning.length > 0) ||
        (res.skipped_params && res.skipped_params.length > 0);
      setVerifyStatus(hasIssues ? 'warning' : 'success');
    } catch (err) {
      setVerifyStatus('error');
      setVerifyError(err instanceof Error ? err.message : 'Could not verify these parameters against this endpoint.');
    }
  };

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

  // Full validation (name/id/category/base URL/context window) always
  // applies now — an update sends exactly the same full payload as a
  // create, so it needs the same fields filled in.
  const willRegisterNew = mode === 'create' || willCreateNew;
  const valid = Boolean(name.trim() && modelId.trim() && category.trim() && baseUrl.trim() && contextWindowValid)
    && !requestParamsError
    && paramsVerified;

  const submitLabel = willRegisterNew
    ? (submitting ? 'Registering…' : (willCreateNew ? 'Register as New Model' : 'Register Model'))
    : (submitting ? 'Saving…' : 'Update Model');

  const handleSubmit = () => {
    if (!valid) return;
    const parsed = parseRequestParams(requestParamsText);
    const requestParams = parsed.ok && Object.keys(parsed.value).length > 0 ? parsed.value : undefined;

    const payload: CustomModelRequestWithParams = {
      name: name.trim(),
      model_id: modelId.trim(),
      category: category.trim(),
      base_url: baseUrl.trim(),
      api_key: apiKey,
      context_window: Number(contextWindowInput),
      description: description.trim(),
      ...(requestParams ? { request_params: requestParams } : {}),
    };

    onSubmit({ kind: willRegisterNew ? 'create' : 'update', payload });
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

              {/* ---- request params: always visible, verification is mandatory before saving ---- */}
              <div className={styles['params-card']}>
                <div className={styles['params-card-hdr']}>
                  <div className={styles['params-card-title']}>
                    <SlidersHorizontal size={13} />
                    Request Parameters
                  </div>
                  <span className={styles['params-required-tag']}>
                    {paramsVerified && verifyStatus !== 'idle' ? 'Verified' : 'Verify before saving'}
                  </span>
                </div>
                <p className={styles['params-card-text']}>
                  Extra parameters sent with every request to this model — edit as needed, then verify they're
                  actually supported by this endpoint before saving.
                </p>

                <textarea
                  className={`${styles['json-input']} ${requestParamsError ? styles['json-input--error'] : ''}`}
                  rows={5}
                  spellCheck={false}
                  value={requestParamsText}
                  onChange={(e) => handleRequestParamsChange(e.target.value)}
                  placeholder={'{\n  "chat_template_kwargs": { "thinking": true },\n  "reasoning_effort": "high"\n}'}
                />
                {requestParamsError && (
                  <div className={styles['field-hint--error']}>
                    <AlertCircle size={12} /> {requestParamsError}
                  </div>
                )}

                <div className={styles['verify-row']}>
                  <button
                    type="button"
                    className={styles['verify-btn']}
                    disabled={!baseUrl.trim() || !effectiveModelId || !requestParamsNonEmpty || verifyStatus === 'loading'}
                    onClick={handleVerifyParams}
                  >
                    {verifyStatus === 'loading' ? <Loader2 size={14} className={styles['spin']} /> : <ShieldCheck size={14} />}
                    {verifyStatus === 'loading' ? 'Verifying…' : 'Verify Parameters'}
                  </button>
                  {!effectiveModelId && (
                    <span className={styles['field-hint']}>Select a model above to verify.</span>
                  )}
                  {effectiveModelId && !requestParamsNonEmpty && (
                    <span className={styles['field-hint']}>Box is empty — nothing to verify, this model will use default params.</span>
                  )}
                </div>

                {verifyStatus === 'success' && verifyResult && (
                  <div className={styles['verify-banner--success']}>
                    <CheckCircle2 size={15} />
                    <span>
                      Verified — all parameters are supported by this model.
                      {verifyResult.sample_output && (
                        <>
                          {' '}Sample response: <em>&ldquo;{verifyResult.sample_output}&rdquo;</em>
                        </>
                      )}
                    </span>
                  </div>
                )}

                {verifyStatus === 'warning' && verifyResult && (
                  <div className={styles['verify-banner--warning']}>
                    <AlertTriangle size={15} />
                    <div>
                      {!verifyResult.supported && (
                        <div className={styles['verify-unsupported-note']}>
                          This model doesn't fully support the given parameters — you can still save, but requests may ignore some of them.
                        </div>
                      )}
                      {verifyResult.warning && verifyResult.warning.length > 0 ? (
                        <ul className={styles['verify-warning-list']}>
                          {verifyResult.warning.map((w, i) => <li key={i}>{w}</li>)}
                        </ul>
                      ) : (
                        verifyResult.supported && <span>Some parameters were ignored by this model.</span>
                      )}
                      {verifyResult.skipped_params && verifyResult.skipped_params.length > 0 && (
                        <div className={styles['verify-skipped']}>
                          Skipped: <code>{verifyResult.skipped_params.join(', ')}</code>
                        </div>
                      )}
                    </div>
                  </div>
                )}

                {verifyStatus === 'error' && (
                  <div className={styles['verify-banner--error']}>
                    <XCircle size={15} />
                    <span>{verifyError || 'Could not verify these parameters against this endpoint.'}</span>
                  </div>
                )}

                {needsVerification && !paramsVerified && verifyStatus === 'idle' && (
                  <div className={styles['field-hint--error']}>
                    <AlertCircle size={12} /> Verify these parameters before you can save.
                  </div>
                )}
              </div>
            </>
          )}

          <div className="fg">
            <label className="fl">Description <span className="opt">(optional)</span></label>
            <textarea className="fi" rows={3} value={description} onChange={(e) => setDescription(e.target.value)} />
          </div>
        </div>

        <div className={styles['drawer__foot']}>
          {needsVerification && !paramsVerified && (
            <span className={styles['params-gate-hint']}>Verify request parameters to continue</span>
          )}
          <button className="btn btn-ghost" onClick={onClose}>Cancel</button>
          <button className={styles['drawer-submit-btn']} disabled={!valid || submitting} onClick={handleSubmit}>{submitLabel}</button>
        </div>
      </div>
    </div>
  );
}














//Models.ts
import api from '../axiosInstance';
import type { Model, CustomModelRequest } from '../../types';

export interface ModelCategory {
  value: string;
  label: string;
  description: string;
}

export interface DiscoveredModel {
  id: string;
  name: string;
  context_window: number | null;
  max_model_len: number | null;
  owned_by: string;
  already_added: boolean;
}

export interface DiscoverModelsRequest {
  base_url: string;
  api_key?: string;
}

export interface DiscoverModelsResponse {
  base_url: string;
  models: DiscoveredModel[];
  total: number;
  errors: string[];
}

export interface DeleteCustomModelResponse {
  status: string;
  model_id: string;
}

// A CustomModelRequest widened with the optional advanced `request_params`
// field. Defined locally rather than editing the shared `CustomModelRequest`
// type in ../../types, but stays fully compatible with it since the added
// field is optional — anywhere a CustomModelRequest is expected, this works.
export type CustomModelRequestWithParams = CustomModelRequest & {
  request_params?: Record<string, unknown>;
};

// Update now POSTs the exact same full payload as create (name, model_id,
// category, base_url, api_key, context_window, description, request_params)
// — the backend keys off model_id to know it's an update rather than a new
// registration, same endpoint either way.
export type UpdateCustomModelRequest = CustomModelRequestWithParams;

export interface UpdateCustomModelResponse {
  model_id: string;
  name?: string;
  description?: string;
}

export interface VerifyParamsRequest {
  base_url: string;
  api_key?: string;
  model_id: string;
  request_params: Record<string, unknown>;
}

export interface VerifyParamsResponse {
  supported: boolean;
  skipped_params: string[];
  warning?: string[];
  sample_output?: string;
}

export const modelsApi = {
  list: () => api.get<{ models: Model[] }>('/models').then((r) => r.data.models ?? []),

  createCustom: (payload: CustomModelRequestWithParams) =>
    api.post<void>('/models/custom', payload).then(() => undefined),

  // POST /models/custom — same endpoint as create, keyed by model_id: used here
  // to update an existing custom model with the same full payload create sends.
  updateCustom: (payload: UpdateCustomModelRequest) =>
    api.post<UpdateCustomModelResponse>('/models/custom', payload).then((r) => r.data),

  // DELETE /models/custom/:modelId — remove a previously registered custom model
  deleteCustom: (modelId: string) =>
    api.delete<DeleteCustomModelResponse>(`/models/custom/${modelId}`).then((r) => r.data),

  // GET /models/by-provider/:providerId — all models registered under a single provider
  listByProvider: (providerId: string) =>
    api
      .get<{ models: Model[]; total: number }>(`/models/by-provider/${providerId}`)
      .then((r) => ({ models: r.data.models ?? [], total: r.data.total ?? 0 })),

  // GET /models/categories — used to populate the Category dropdown
  listCategories: () =>
    api.get<{ categories: ModelCategory[] }>('/models/categories').then((r) => r.data.categories ?? []),

  // POST /models/discover — optional endpoint probe to auto-fill model name/id/context window
  discover: (payload: DiscoverModelsRequest) =>
    api.post<DiscoverModelsResponse>('/models/discover', payload).then((r) => ({
      base_url: r.data.base_url,
      models: r.data.models ?? [],
      total: r.data.total ?? 0,
      errors: r.data.errors ?? [],
    })),

  // POST /models/verify-params — probes the endpoint with a candidate set of
  // advanced request params (e.g. chat_template_kwargs, reasoning_effort) to
  // confirm the model actually supports them before saving.
  verifyParams: (payload: VerifyParamsRequest) =>
    api.post<VerifyParamsResponse>('/models/verify-params', payload).then((r) => r.data),
};















//Modelsslice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { modelsApi, type ModelCategory, type CustomModelRequestWithParams } from '../../api/endpoints/models';
import type { Model } from '../../types';

type FetchStatus = 'idle' | 'loading' | 'succeeded' | 'failed';

interface ModelsState {
  items: Model[];
  status: FetchStatus;
  error: string | null;
  creating: boolean;
  byProvider: Record<string, Model[]>;
  byProviderStatus: Record<string, FetchStatus>;
  categories: ModelCategory[];
  categoriesStatus: FetchStatus;
  deletingId: string | null;
  updatingId: string | null;
}

const initialState: ModelsState = {
  items: [],
  status: 'idle',
  error: null,
  creating: false,
  byProvider: {},
  byProviderStatus: {},
  categories: [],
  categoriesStatus: 'idle',
  deletingId: null,
  updatingId: null,
};

export const fetchModels = createAsyncThunk('models/fetchAll', () => modelsApi.list());

export const fetchModelsByProvider = createAsyncThunk(
  'models/fetchByProvider',
  async (providerId: string) => {
    const { models } = await modelsApi.listByProvider(providerId);
    return { providerId, models };
  }
);

export const fetchModelCategories = createAsyncThunk(
  'models/fetchCategories',
  () => modelsApi.listCategories()
);

export const createCustomModel = createAsyncThunk(
  'models/createCustom',
  async (payload: CustomModelRequestWithParams, { dispatch }) => {
    await modelsApi.createCustom(payload);
    // spec: no meaningful body returned, so refetch afterwards
    await dispatch(fetchModels());
  }
);

export const updateCustomModel = createAsyncThunk(
  'models/updateCustom',
  async (payload: CustomModelRequestWithParams) => {
    const res = await modelsApi.updateCustom(payload);
    // The backend may only echo back {model_id, name}; since we just sent
    // the full authoritative payload, use it (not the response) as the
    // source of truth for every other field.
    return {
      modelId: res.model_id || payload.model_id,
      name: res.name ?? payload.name,
      description: payload.description,
      category: payload.category,
      base_url: payload.base_url,
      context_window: payload.context_window,
      request_params: payload.request_params,
    };
  }
);

export const deleteCustomModel = createAsyncThunk(
  'models/deleteCustom',
  async ({ modelId, providerId }: { modelId: string; providerId: string }) => {
    const res = await modelsApi.deleteCustom(modelId);
    return { modelId: res.model_id || modelId, providerId };
  }
);

const modelsSlice = createSlice({
  name: 'models',
  initialState,
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(fetchModels.pending, (state) => {
        state.status = 'loading';
      })
      .addCase(fetchModels.fulfilled, (state, action) => {
        state.status = 'succeeded';
        state.items = action.payload ?? [];
      })
      .addCase(fetchModels.rejected, (state, action) => {
        state.status = 'failed';
        state.error = action.error.message || 'Failed to load models';
      })
      .addCase(createCustomModel.pending, (state) => {
        state.creating = true;
      })
      .addCase(createCustomModel.fulfilled, (state) => {
        state.creating = false;
      })
      .addCase(createCustomModel.rejected, (state, action) => {
        state.creating = false;
        state.error = action.error.message || 'Failed to register custom model';
      })
      .addCase(fetchModelsByProvider.pending, (state, action) => {
        state.byProviderStatus[action.meta.arg] = 'loading';
      })
      .addCase(fetchModelsByProvider.fulfilled, (state, action) => {
        state.byProviderStatus[action.payload.providerId] = 'succeeded';
        state.byProvider[action.payload.providerId] = action.payload.models ?? [];
      })
      .addCase(fetchModelsByProvider.rejected, (state, action) => {
        state.byProviderStatus[action.meta.arg] = 'failed';
      })
      .addCase(fetchModelCategories.pending, (state) => {
        state.categoriesStatus = 'loading';
      })
      .addCase(fetchModelCategories.fulfilled, (state, action) => {
        state.categoriesStatus = 'succeeded';
        state.categories = action.payload ?? [];
      })
      .addCase(fetchModelCategories.rejected, (state) => {
        state.categoriesStatus = 'failed';
      })
      .addCase(deleteCustomModel.pending, (state, action) => {
        state.deletingId = action.meta.arg.modelId;
      })
      .addCase(deleteCustomModel.fulfilled, (state, action) => {
        state.deletingId = null;
        const { modelId, providerId } = action.payload;
        state.items = state.items.filter((m) => m.id !== modelId);
        if (state.byProvider[providerId]) {
          state.byProvider[providerId] = state.byProvider[providerId].filter((m) => m.id !== modelId);
        }
      })
      .addCase(deleteCustomModel.rejected, (state, action) => {
        state.deletingId = null;
        state.error = action.error.message || 'Failed to delete model';
      })
      .addCase(updateCustomModel.pending, (state, action) => {
        state.updatingId = action.meta.arg.model_id;
      })
      .addCase(updateCustomModel.fulfilled, (state, action) => {
        state.updatingId = null;
        const { modelId, name, description, category, base_url, context_window, request_params } = action.payload;
        const patch = (m: Model) => {
          if (m.id !== modelId) return m;
          return {
            ...m,
            name,
            description,
            category,
            base_url,
            context_window,
            request_params,
          } as Model & { description: string; request_params?: Record<string, unknown> };
        };
        state.items = state.items.map(patch);
        for (const providerId of Object.keys(state.byProvider)) {
          state.byProvider[providerId] = state.byProvider[providerId].map(patch);
        }
      })
      .addCase(updateCustomModel.rejected, (state, action) => {
        state.updatingId = null;
        state.error = action.error.message || 'Failed to update model';
      });
  },
});

export default modelsSlice.reducer;













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
              return dispatch(updateCustomModel(result.payload))
                .unwrap()
                .then((res) => {
                  toast.success(`"${res.name}" was updated successfully.`, { title: 'Model updated' });
                  dispatch(fetchProviders());
                  dispatch(fetchModelsByProvider(viewModelsProvider.id));
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
