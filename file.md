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

// Create always POSTs the full CustomModelRequest. Editing normally only
// POSTs {model_id, name, description} against the existing model — but if
// discovery finds a *different* model id than the one being edited, we
// treat that as registering a brand-new model instead, so the caller needs
// to know which branch happened.
export type CustomModelSubmitPayload =
  | { kind: 'create'; payload: CustomModelRequestWithParams }
  | { kind: 'update'; model_id: string; name: string; description: string; request_params?: Record<string, unknown> };

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

  // Full validation (name/id/category/base URL/context window) applies to a
  // real create — either CREATE mode outright, or an edit that discovered a
  // different model and will therefore be submitted as a new registration.
  const needsFullValidation = mode === 'create' || willCreateNew;
  const valid = (needsFullValidation
    ? Boolean(name.trim() && modelId.trim() && category.trim() && baseUrl.trim() && contextWindowValid)
    : Boolean(name.trim()))
    && !requestParamsError
    && paramsVerified;

  const submitLabel = needsFullValidation
    ? (submitting ? 'Registering…' : (willCreateNew ? 'Register as New Model' : 'Register Model'))
    : (submitting ? 'Saving…' : 'Update Model');

  const handleSubmit = () => {
    if (!valid) return;
    const parsed = parseRequestParams(requestParamsText);
    const requestParams = parsed.ok && Object.keys(parsed.value).length > 0 ? parsed.value : undefined;

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
          ...(requestParams ? { request_params: requestParams } : {}),
        },
      });
      return;
    }
    // Plain update — name/description, plus request_params if the user set any.
    onSubmit({
      kind: 'update',
      model_id: (initialModel?.id ?? modelId).trim(),
      name: name.trim(),
      description: description.trim(),
      ...(requestParams ? { request_params: requestParams } : {}),
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
  background: linear-gradient(165deg, $wash 0%, $paper 65%);
  border: 1px solid rgba($signal, 0.16);
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
  background: $signal;
  color: #fff;
  box-shadow: 0 4px 10px -3px rgba($signal, 0.5);
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
  border: 1px solid $signal;
  background: $signal;
  color: #fff;
  font-size: 0.9615em; // 12.5px / 13px
  font-weight: 700;
  cursor: pointer;
  box-shadow: 0 2px 6px -2px rgba($signal, 0.5);
  transition: background 0.15s ease, border-color 0.15s ease, transform 0.12s ease, box-shadow 0.15s ease;

  &:hover:not(:disabled) { background: $signal-2; border-color: $signal-2; transform: translateY(-1px); box-shadow: 0 4px 10px -2px rgba($signal, 0.55); }
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

  &:hover { color: $signal; }
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

// ---- request params: mandatory-verify card --------------------------------------
// Uses the same ink/paper/signal token system as Providers.module.scss (shared
// via _variables.scss) rather than the drawer's older $indigo/$amber tokens, so
// this card's colors stay consistent with the rest of the Providers surface
// and pick up theme changes automatically.
.params-card {
  margin-bottom: 16px;
  padding: 14px;
  border-radius: 12px;
  background: $paper;
  border: 1px solid $line;
}
.params-card-hdr {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  margin-bottom: 4px;
}
.params-card-title {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  font-size: 1.0385em; // 13.5px / 13px
  font-weight: 750;
  color: $ink;

  svg { color: $signal; }
}
.params-required-tag {
  flex-shrink: 0;
  padding: 2px 8px;
  border-radius: 999px;
  background: $wash;
  color: $signal;
  font-size: 0.8077em; // 10.5px / 13px
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.03em;
  white-space: nowrap;
}
.params-card-text {
  font-size: 0.9231em; // 12px / 13px
  line-height: 1.5;
  color: $ink-2;
  margin: 0 0 12px;
}
.json-input {
  width: 100%;
  padding: 10px 12px;
  border: 1.5px solid $line;
  border-radius: 10px;
  background: $card;
  color: $ink;
  font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
  font-size: 0.9231em; // 12px / 13px
  line-height: 1.5;
  resize: vertical;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;

  &::placeholder { color: $ink-3; }
  &:focus { outline: none; border-color: $signal; box-shadow: 0 0 0 3px $wash; }

  &--error {
    border-color: $danger;
    &:focus { box-shadow: 0 0 0 3px $danger-wash; }
  }
}
.verify-row {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-top: 10px;
}
.verify-btn {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 8px 14px;
  border-radius: 9px;
  border: 1px solid $signal;
  background: $signal;
  color: #fff;
  font-size: 0.9615em; // 12.5px / 13px
  font-weight: 700;
  cursor: pointer;
  transition: background 0.15s ease, border-color 0.15s ease, transform 0.12s ease;

  &:hover:not(:disabled) { background: $signal-2; border-color: $signal-2; transform: translateY(-1px); }
  &:disabled { opacity: 0.5; cursor: not-allowed; transform: none; }
}
.params-gate-hint {
  margin-right: auto;
  font-size: 0.9231em; // 12px / 13px
  font-weight: 600;
  color: $ink-3;
}

// ---- drawer submit button (Register / Update Model) -------------------------------
// Standard button color: mirrors Providers' foot-btn--primary via the shared
// $signal token, rather than the drawer's legacy $indigo.
.drawer-submit-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 130px;
  padding: 9px 18px;
  border-radius: 9px;
  border: 1px solid $signal;
  background: $signal;
  color: #fff;
  font-size: 0.9615em; // 12.5px / 13px
  font-weight: 700;
  cursor: pointer;
  transition: background 0.15s ease, border-color 0.15s ease, transform 0.12s ease;

  &:hover:not(:disabled) { background: $signal-2; border-color: $signal-2; transform: translateY(-1px); }
  &:disabled { opacity: 0.6; cursor: not-allowed; transform: none; }
}

// ---- verify result banners -------------------------------------------------------
.verify-banner--success,
.verify-banner--warning,
.verify-banner--error {
  display: flex;
  align-items: flex-start;
  gap: 8px;
  margin-top: 12px;
  padding: 10px 13px;
  border-radius: 10px;
  font-size: 0.9231em; // 12px / 13px
  font-weight: 550;
  line-height: 1.5;

  svg { flex-shrink: 0; margin-top: 1px; }
}
.verify-banner--success {
  background: $ok-wash;
  border: 1px solid rgba($ok, 0.25);
  color: $ok;
  svg { color: $ok; }
}
.verify-banner--warning {
  background: $amber-pale;
  border: 1px solid rgba($amber-dark, 0.3);
  color: #92400E;
  svg { color: $amber-dark; }
}
.verify-banner--error {
  background: $danger-wash;
  border: 1px solid rgba($danger, 0.3);
  color: $danger;
  svg { color: $danger; }
}
.verify-warning-list {
  margin: 0;
  padding-left: 16px;
  li { margin-bottom: 2px; }
}
.verify-unsupported-note {
  margin-bottom: 4px;
  font-weight: 650;
}
.verify-skipped {
  margin-top: 4px;
  code {
    font-family: monospace;
    font-size: 0.92em;
    background: rgba(0, 0, 0, 0.06);
    padding: 1px 4px;
    border-radius: 4px;
  }
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
