//Settings.tsx
// ═══════════════════════════════════════════════
// pages/Settings/Settings.tsx
// Content Analytics · User Dictionary (master-detail)
// ═══════════════════════════════════════════════
import React, { useState, useEffect, useMemo } from 'react';
import { useTranslation } from 'react-i18next';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import {
  fetchDictionaries,
  createDictionary,
  updateDictionary,
  deleteDictionary,
  selectDictionary,
} from '../../store/dictionarySlice';
import type { DictionaryTerm } from '../../services/dictionaryApi';
import TourGuide, { type TourStep } from '../UploadInfer/TourGuide';
import styles from './Settings.module.scss';

const DICT_NAME_MAX_LENGTH = 255;

// ─────────────────────────────────────────────
// Tour steps — built lazily so t() is available.
// Call getDictTourSteps(t) inside a component.
// ─────────────────────────────────────────────
const ensureEditorOpen = () => {
  if (!document.querySelector('[data-tour="dict-name-input"]')) {
    document.querySelector<HTMLElement>('[data-tour="dict-new-btn"]')?.click();
  }
};

export const getDictTourSteps = (t: (key: string) => string): TourStep[] => [
  {
    target: 'dict-list-panel',
    placement: 'right',
    title: t('settings.tour.step1Title'),
    content: t('settings.tour.step1Body'),
  },
  {
    target: 'dict-new-btn',
    placement: 'right',
    title: t('settings.tour.step2Title'),
    content: t('settings.tour.step2Body'),
  },
  {
    target: 'dict-name-input',
    placement: 'bottom',
    title: t('settings.tour.step3Title'),
    content: t('settings.tour.step3Body'),
    onEnter: ensureEditorOpen,
  },
  {
    target: 'dict-add-term-btn',
    placement: 'bottom',
    title: t('settings.tour.step4Title'),
    content: t('settings.tour.step4Body'),
    onEnter: ensureEditorOpen,
  },
  {
    target: 'dict-terms-table',
    placement: 'top',
    title: t('settings.tour.step5Title'),
    content: t('settings.tour.step5Body'),
    onEnter: ensureEditorOpen,
  },
  {
    target: 'dict-save-btn',
    placement: 'left',
    title: t('settings.tour.step6Title'),
    content: t('settings.tour.step6Body'),
    onEnter: ensureEditorOpen,
  },
];

type LocalTerm = DictionaryTerm & { _localId: string };

const makeLocalId = () => Math.random().toString(36).slice(2, 10);
const toLocal = (t: DictionaryTerm): LocalTerm => ({ ...t, _localId: makeLocalId() });
const stripLocal = (t: LocalTerm): DictionaryTerm => ({
  wrong_term: t.wrong_term,
  correct_term: t.correct_term,
});

const EMPTY_TERMS = (): LocalTerm[] => [
  { _localId: makeLocalId(), wrong_term: '', correct_term: '' },
];

// ─────────────────────────────────────────────
// Dictionary List Panel
// ─────────────────────────────────────────────
interface ListPanelProps {
  loading: boolean;
  items: { dictionary_id: number; dictionary_name: string; dictionary_terms: DictionaryTerm[] }[];
  selectedId: number | null;
  isCreating: boolean;
  onSelect: (id: number) => void;
  onNew: () => void;
}

const ListPanel: React.FC<ListPanelProps> = ({
  loading, items, selectedId, isCreating, onSelect, onNew,
}) => {
  const { t } = useTranslation();
  return (
    <aside className={styles.dictListPanel} data-tour="dict-list-panel">
      <div className={styles.dictListHead}>
        <div className={styles.dictListTitle}>{t('settings.dictionaries')}</div>
        <button className={styles.dictListAdd} onClick={onNew} title={t('settings.newDictTooltip')} data-tour="dict-new-btn">
          <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round">
            <path d="M8 3v10M3 8h10" />
          </svg>
        </button>
      </div>

      <div className={styles.dictList}>
        {loading && items.length === 0 ? (
          <div className={styles.dictListLoading}>{t('settings.loading')}</div>
        ) : (
          <>
            {isCreating && (
              <div className={`${styles.dictListItem} ${styles.dictListItemActive}`}>
                <div className={styles.dictListItemName}>{t('settings.newDict')}</div>
                <div className={styles.dictListItemMeta}>{t('settings.unsaved')}</div>
              </div>
            )}

            {items.length === 0 && !isCreating && (
              <div className={styles.dictListEmpty}>
                {t('settings.noDict').split('\n').map((line, i) => (
                  <React.Fragment key={i}>{line}{i === 0 && <br />}</React.Fragment>
                ))}
              </div>
            )}

            {items.map(d => (
              <button
                key={d.dictionary_id}
                className={`${styles.dictListItem} ${selectedId === d.dictionary_id && !isCreating ? styles.dictListItemActive : ''}`}
                onClick={() => onSelect(d.dictionary_id)}
              >
                <div className={styles.dictListItemName}>{d.dictionary_name}</div>
                <div className={styles.dictListItemMeta}>
                  {t('settings.term', { count: d.dictionary_terms.length })}
                </div>
              </button>
            ))}
          </>
        )}
      </div>
    </aside>
  );
};

// ─────────────────────────────────────────────
// Dictionary Detail Panel
// ─────────────────────────────────────────────
const UserDictionary: React.FC = () => {
  const dispatch = useAppDispatch();
  const { t } = useTranslation();
  const { items, selectedId, loading, saving, deleting, error } = useAppSelector(s => s.dictionary);

  const [isCreating,     setIsCreating]     = useState(false);
  const [name,           setName]           = useState('');
  const [terms,          setTerms]          = useState<LocalTerm[]>([]);
  const [search,         setSearch]         = useState('');
  const [dirty,          setDirty]          = useState(false);
  const [savedFlash,     setSavedFlash]     = useState(false);
  const [confirmDelete,  setConfirmDelete]  = useState(false);

  useEffect(() => {
    if (items.length === 0 && !loading) {
      dispatch(fetchDictionaries());
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  const selected = useMemo(
    () => items.find(d => d.dictionary_id === selectedId) ?? null,
    [items, selectedId],
  );

  useEffect(() => {
    if (isCreating) return;
    if (selected) {
      setName(selected.dictionary_name);
      setTerms(selected.dictionary_terms.map(toLocal));
      setDirty(false);
      setConfirmDelete(false);
    } else {
      setName('');
      setTerms([]);
      setDirty(false);
      setConfirmDelete(false);
    }
  }, [selected, isCreating]);

  const handleSelect = (id: number) => {
    if (dirty && !window.confirm(t('settings.discardConfirm'))) return;
    setIsCreating(false);
    dispatch(selectDictionary(id));
  };

  const handleNew = () => {
    if (dirty && !window.confirm(t('settings.discardConfirm'))) return;
    setIsCreating(true);
    setName('');
    setTerms(EMPTY_TERMS());
    setDirty(false);
  };

  const handleCancelNew = () => {
    setIsCreating(false);
    setDirty(false);
  };

  const handleDelete = async () => {
    if (!selected) return;
    try {
      await dispatch(deleteDictionary(selected.dictionary_id)).unwrap();
      setConfirmDelete(false);
      setIsCreating(false);
    } catch {
      // error surfaces via Redux state
    }
  };

  const handleNameChange = (value: string) => {
    setName(value.slice(0, DICT_NAME_MAX_LENGTH));
    setDirty(true);
  };

  const updateTerm = (localId: string, patch: Partial<DictionaryTerm>) => {
    setTerms(prev => prev.map(term => term._localId === localId ? { ...term, ...patch } : term));
    setDirty(true);
  };

  const addTermRow = () => {
    setTerms(prev => [...prev, { _localId: makeLocalId(), wrong_term: '', correct_term: '' }]);
    setDirty(true);
  };

  const removeTerm = (localId: string) => {
    setTerms(prev => prev.filter(term => term._localId !== localId));
    setDirty(true);
  };

  const handleSave = async () => {
    const cleanedTerms = terms
      .map(stripLocal)
      .filter(term => term.wrong_term.trim() && term.correct_term.trim())
      .map(term => ({ wrong_term: term.wrong_term.trim(), correct_term: term.correct_term.trim() }));

    const cleanedName = name.trim().slice(0, DICT_NAME_MAX_LENGTH);

    if (!cleanedName) { alert(t('settings.nameRequired')); return; }
    if (cleanedTerms.length === 0) { alert(t('settings.termRequired')); return; }

    const payload = { dictionary_name: cleanedName, dictionary_terms: cleanedTerms };
    try {
      if (isCreating) {
        await dispatch(createDictionary(payload)).unwrap();
        setIsCreating(false);
      } else if (selected) {
        await dispatch(updateDictionary({ id: selected.dictionary_id, payload })).unwrap();
      }
      setDirty(false);
      setSavedFlash(true);
      setTimeout(() => setSavedFlash(false), 2000);
    } catch { /* error surfaces via Redux state */ }
  };

  const filteredTerms = useMemo(() => {
    if (!search.trim()) return terms;
    const q = search.toLowerCase();
    return terms.filter(term =>
      term.wrong_term.toLowerCase().includes(q) ||
      term.correct_term.toLowerCase().includes(q),
    );
  }, [terms, search]);

  const showEditor = isCreating || !!selected;
  const nameAtLimit = name.length >= DICT_NAME_MAX_LENGTH;

  return (
    <div className={styles.dictLayout}>
      <ListPanel
        loading={loading}
        items={items}
        selectedId={selectedId}
        isCreating={isCreating}
        onSelect={handleSelect}
        onNew={handleNew}
      />

      <section className={styles.dictDetail}>
        {!showEditor ? (
          <div className={styles.dictDetailEmpty}>
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
              <path d="M4 4h12a3 3 0 013 3v13H7a3 3 0 01-3-3V4z" />
              <path d="M4 4v13a3 3 0 003 3" />
              <path d="M8 8h7M8 12h7M8 16h4" />
            </svg>
            <div>{t('settings.selectOrCreate')}</div>
          </div>
        ) : (
          <>
            {/* ── Detail header ── */}
            <div className={styles.dictDetailHead}>
              <div className={styles.dictDetailHeadMeta}>
                <div className={styles.dictNameField}>
                  <div className={styles.dictNameLabelRow}>
                    <label className={styles.dictNameLabel}>{t('settings.dictNameLabel')}</label>
                    <span className={`${styles.dictNameCount} ${nameAtLimit ? styles.dictNameCountMax : ''}`}>
                      {name.length}/{DICT_NAME_MAX_LENGTH}
                    </span>
                  </div>
                  <input
                    className={styles.dictNameInput}
                    value={name}
                    maxLength={DICT_NAME_MAX_LENGTH}
                    onChange={e => handleNameChange(e.target.value)}
                    placeholder={t('settings.dictNamePlaceholder')}
                    data-tour="dict-name-input"
                  />
                </div>
                {selected && !isCreating && (
                  <div className={styles.dictDetailDates}>
                    {t('settings.updated', { date: new Date(selected.updated_at).toLocaleString() })}
                  </div>
                )}
              </div>
              <div className={styles.dictDetailActions}>
                {/* ── Delete flow (existing dicts only) ── */}
                {!isCreating && selected && (
                  confirmDelete ? (
                    <div className={styles.dictDeleteConfirm}>
                      <span className={styles.dictDeleteWarn}>
                        {t('settings.deleteWarning')}
                      </span>
                      <button
                        className={`${styles.btn} ${styles.btnDanger}`}
                        onClick={handleDelete}
                        disabled={deleting}
                      >
                        {deleting ? t('settings.deleting') : t('settings.confirmDelete')}
                      </button>
                      <button
                        className={styles.btn}
                        onClick={() => setConfirmDelete(false)}
                        disabled={deleting}
                      >
                        {t('settings.cancel')}
                      </button>
                    </div>
                  ) : (
                    <button
                      className={`${styles.btn} ${styles.btnDangerGhost}`}
                      onClick={() => setConfirmDelete(true)}
                      disabled={saving}
                      title={t('settings.deleteDict')}
                    >
                      <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                        <path d="M2 4h12M5 4V2.5h6V4M6 7l.5 6M10 7l-.5 6M3.5 4l.7 9.5a1 1 0 001 .9h5.6a1 1 0 001-.9L12.5 4" />
                      </svg>
                      {t('settings.deleteDict')}
                    </button>
                  )
                )}

                {/* ── Cancel / Save ── */}
                {!confirmDelete && (
                  <>
                    {isCreating && (
                      <button className={styles.btn} onClick={handleCancelNew} disabled={saving}>
                        {t('settings.cancel')}
                      </button>
                    )}
                    <button
                      className={`${styles.btn} ${styles.btnP} ${savedFlash ? styles.btnSaved : ''}`}
                      onClick={handleSave}
                      disabled={saving || (!dirty && !isCreating)}
                      data-tour="dict-save-btn"
                    >
                      {saving ? t('settings.saving') : savedFlash ? (
                        <><svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round"><path d="M3 8l3.5 3.5L13 5" /></svg>{t('settings.saved')}</>
                      ) : isCreating ? t('settings.create') : t('settings.saveChanges')}
                    </button>
                  </>
                )}
              </div>
            </div>

            {error && <div className={styles.dictErrorBox}>{error}</div>}

            {/* ── Search / count ── */}
            <div className={styles.dictDetailToolbar}>
              <div className={styles.dictSearch}>
                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round">
                  <circle cx="6.5" cy="6.5" r="4.5" /><path d="M10 10l3 3" />
                </svg>
                <input
                  className={styles.dictSearchInput}
                  value={search}
                  onChange={e => setSearch(e.target.value)}
                  placeholder={t('settings.searchPlaceholder')}
                />
                {search && (
                  <button className={styles.dictSearchClear} onClick={() => setSearch('')}>✕</button>
                )}
              </div>
              <div className={styles.dictMeta}>
                <span className={styles.dictCount}>{terms.length}</span>
                <span className={styles.dictTotal}>{t('settings.term', { count: terms.length })}</span>
              </div>
            </div>

            {/* ── Add-term toolbar ── */}
            <div className={styles.dictTermsToolbar}>
              <button className={styles.dictAddRowBtn} onClick={addTermRow} data-tour="dict-add-term-btn">
                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                  <path d="M8 3v10M3 8h10" />
                </svg>
                {t('settings.addTerm')}
              </button>
            </div>

            {/* ── Terms table ── */}
            <div className={styles.dictTermsWrap}>
              <div className={styles.dictTable} data-tour="dict-terms-table">
                <div className={styles.dictTermsHead}>
                  <span>{t('settings.wrongTerm')}</span>
                  <span />
                  <span>{t('settings.correctTerm')}</span>
                  <span />
                </div>

                {filteredTerms.length === 0 ? (
                  <div className={styles.dictEmpty}>
                    {search ? t('settings.noSearchMatch') : t('settings.noTermsYet')}
                  </div>
                ) : filteredTerms.map(term => (
                  <div key={term._localId} className={styles.dictTermsRow}>
                    <input
                      className={styles.dictInput}
                      value={term.wrong_term}
                      onChange={e => updateTerm(term._localId, { wrong_term: e.target.value })}
                      placeholder={t('settings.wrongPlaceholder')}
                    />
                    <span className={styles.dictArrow}>→</span>
                    <input
                      className={styles.dictInput}
                      value={term.correct_term}
                      onChange={e => updateTerm(term._localId, { correct_term: e.target.value })}
                      placeholder={t('settings.correctPlaceholder')}
                    />
                    <button
                      className={`${styles.dictActionBtn} ${styles.dictActionDel}`}
                      onClick={() => removeTerm(term._localId)}
                      title={t('settings.removeTerm')}
                    >
                      <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                        <path d="M2 3.5h10M5.5 3.5V2.5h3v1M5 6l.5 5M9 6l-.5 5" />
                      </svg>
                    </button>
                  </div>
                ))}
              </div>
            </div>

            <div className={styles.dictInfo}>
              <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeWidth="1.3" strokeLinecap="round">
                <circle cx="7" cy="7" r="5.5" /><path d="M7 4.5v3M7 9v.5" />
              </svg>
              {t('settings.dictInfo')}
            </div>
          </>
        )}
      </section>
    </div>
  );
};

// ─────────────────────────────────────────────
// Settings root
// ─────────────────────────────────────────────
const Settings: React.FC = () => {
  const { t } = useTranslation();
  const [tourActive, setTourActive] = useState(false);

  return (
    <div className={styles.page}>
      <div className={styles.ph}>
        <div className={styles.phRow}>
          <div>
            <div className={styles.phTitle}>{t('settings.pageTitle')}</div>
            <div className={styles.phSub}>{t('settings.pageSub')}</div>
          </div>
          <button
            type="button"
            className={styles.tourTriggerBtn}
            onClick={() => setTourActive(true)}
          >
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
              <circle cx="8" cy="8" r="6.25" />
              <path d="M6.1 6.2a1.9 1.9 0 013.6.7c0 1.3-1.7 1.5-1.7 2.7M8 11.4v.1" />
            </svg>
            {t('uploadInfer.tour.takeTour')}
          </button>
        </div>
      </div>

      <div className={styles.body}>
        <UserDictionary />
      </div>

      <TourGuide
        steps={getDictTourSteps(t)}
        active={tourActive}
        onFinish={() => setTourActive(false)}
      />
    </div>
  );
};

export default Settings;














// ═══════════════════════════════════════════════
// store/dictionarySlice.ts
// Content Analytics · Dictionary state + async thunks
// ═══════════════════════════════════════════════
import { createSlice, createAsyncThunk, type PayloadAction } from '@reduxjs/toolkit';
import {
  dictionaryApi,
  type Dictionary,
  type DictionaryPayload,
} from '../services/dictionaryApi';

// ── State shape ──────────────────────────────────
interface DictionaryState {
  items:        Dictionary[];
  selectedId:   number | null;
  loading:      boolean;
  saving:       boolean;
  deleting:     boolean;
  error:        string | null;
}

const initialState: DictionaryState = {
  items:      [],
  selectedId: null,
  loading:    false,
  saving:     false,
  deleting:   false,
  error:      null,
};

// ── Async thunks ─────────────────────────────────
export const fetchDictionaries = createAsyncThunk<Dictionary[]>(
  'dictionary/fetchAll',
  async () => dictionaryApi.list(),
);

export const fetchDictionaryById = createAsyncThunk<Dictionary, number>(
  'dictionary/fetchOne',
  async (id) => dictionaryApi.get(id),
);

export const createDictionary = createAsyncThunk<Dictionary, DictionaryPayload>(
  'dictionary/create',
  async (payload) => dictionaryApi.create(payload),
);

export const updateDictionary = createAsyncThunk<
  Dictionary,
  { id: number; payload: DictionaryPayload }
>(
  'dictionary/update',
  async ({ id, payload }) => dictionaryApi.update(id, payload),
);

export const deleteDictionary = createAsyncThunk<number, number>(
  'dictionary/delete',
  async (id) => {
    await dictionaryApi.delete(id);
    return id; // return the id so the reducer can remove it from items
  },
);

// ── Slice ────────────────────────────────────────
const dictionarySlice = createSlice({
  name: 'dictionary',
  initialState,
  reducers: {
    selectDictionary(state, action: PayloadAction<number | null>) {
      state.selectedId = action.payload;
    },
    clearError(state) {
      state.error = null;
    },
  },
  extraReducers: (builder) => {
    builder
      // ── fetch list ──────────────────────────
      .addCase(fetchDictionaries.pending, (state) => {
        state.loading = true;
        state.error   = null;
      })
      .addCase(fetchDictionaries.fulfilled, (state, action) => {
        state.loading = false;
        state.items   = action.payload;
        // Auto-select first dictionary if none selected
        if (state.selectedId === null && action.payload.length > 0) {
          state.selectedId = action.payload[0].dictionary_id;
        }
      })
      .addCase(fetchDictionaries.rejected, (state, action) => {
        state.loading = false;
        state.error   = action.error.message ?? 'Failed to load dictionaries';
      })
      // ── fetch by id (refresh single) ────────
      .addCase(fetchDictionaryById.fulfilled, (state, action) => {
        const idx = state.items.findIndex(
          (d) => d.dictionary_id === action.payload.dictionary_id,
        );
        if (idx >= 0) state.items[idx] = action.payload;
        else          state.items.push(action.payload);
      })
      // ── create ──────────────────────────────
      .addCase(createDictionary.pending, (state) => {
        state.saving = true;
        state.error  = null;
      })
      .addCase(createDictionary.fulfilled, (state, action) => {
        state.saving      = false;
        state.items.push(action.payload);
        state.selectedId  = action.payload.dictionary_id;
      })
      .addCase(createDictionary.rejected, (state, action) => {
        state.saving = false;
        state.error  = action.error.message ?? 'Failed to create dictionary';
      })
      // ── update ──────────────────────────────
      .addCase(updateDictionary.pending, (state) => {
        state.saving = true;
        state.error  = null;
      })
      .addCase(updateDictionary.fulfilled, (state, action) => {
        state.saving = false;
        const idx = state.items.findIndex(
          (d) => d.dictionary_id === action.payload.dictionary_id,
        );
        if (idx >= 0) state.items[idx] = action.payload;
      })
      .addCase(updateDictionary.rejected, (state, action) => {
        state.saving = false;
        state.error  = action.error.message ?? 'Failed to update dictionary';
      })
      // ── delete ──────────────────────────────
      .addCase(deleteDictionary.pending, (state) => {
        state.deleting = true;
        state.error    = null;
      })
      .addCase(deleteDictionary.fulfilled, (state, action) => {
        state.deleting  = false;
        state.items     = state.items.filter(d => d.dictionary_id !== action.payload);
        // Clear selection if the deleted dictionary was selected; fall back
        // to the first remaining item so the panel doesn't go blank unexpectedly.
        if (state.selectedId === action.payload) {
          state.selectedId = state.items.length > 0 ? state.items[0].dictionary_id : null;
        }
      })
      .addCase(deleteDictionary.rejected, (state, action) => {
        state.deleting = false;
        state.error    = action.error.message ?? 'Failed to delete dictionary';
      });
  },
});

export const { selectDictionary, clearError } = dictionarySlice.actions;
export default dictionarySlice.reducer;
















// ═══════════════════════════════════════════════
// services/dictionaryApi.ts
// Content Analytics · Dictionary CRUD endpoints
// ═══════════════════════════════════════════════
import api from './api';

// ── Types ───────────────────────────────────────
export interface DictionaryTerm {
  wrong_term:   string;
  correct_term: string;
}

export interface Dictionary {
  dictionary_id:    number;
  dictionary_name:  string;
  dictionary_terms: DictionaryTerm[];
  created_at:       string;
  updated_at:       string;
}

export interface DictionaryPayload {
  dictionary_name:  string;
  dictionary_terms: DictionaryTerm[];
}

interface ApiResponse<T> {
  status: 'success' | 'error';
  data:   T;
}

interface DeleteResponse {
  status:  'success' | 'error';
  message: string;
}

// ── Endpoints ───────────────────────────────────
export const dictionaryApi = {
  /** GET /dictionary/list — fetch all dictionaries */
  list: async (): Promise<Dictionary[]> => {
    const { data } = await api.get<ApiResponse<Dictionary[]>>('/dictionary/list');
    return data.data;
  },

  /** GET /dictionary/:id — fetch single dictionary */
  get: async (id: number): Promise<Dictionary> => {
    const { data } = await api.get<ApiResponse<Dictionary>>(`/dictionary/${id}`);
    return data.data;
  },

  /** POST /dictionary/create — create a new dictionary */
  create: async (payload: DictionaryPayload): Promise<Dictionary> => {
    const { data } = await api.post<ApiResponse<Dictionary>>('/dictionary/create', payload);
    return data.data;
  },

  /** PUT /dictionary/:id — update an existing dictionary */
  update: async (id: number, payload: DictionaryPayload): Promise<Dictionary> => {
    const { data } = await api.put<ApiResponse<Dictionary>>(`/dictionary/${id}`, payload);
    return data.data;
  },

  /** POST /dictionary/delete — delete a dictionary by id */
  delete: async (id: number): Promise<string> => {
    const { data } = await api.post<DeleteResponse>('/dictionary/delete', { dictionary_id: id });
    return data.message;
  },
};


















// ═══════════════════════════════════════════════
// Settings.module.scss
// Content Analytics · User Dictionary page
// ═══════════════════════════════════════════════
@use '../../styles/mixins' as m;

// ── Page shell ──────────────────────────────────
.page {
  display: flex;
  flex-direction: column;
  height: 100%;
  overflow: hidden;
}

// ── Page header ─────────────────────────────────
.ph {
  background: var(--bg1);
  border-bottom: 1px solid var(--bdr);
  flex-shrink: 0;
}

.phRow {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
  padding: 14px 24px;
}

.phTitle {
  font-size: 18px;
  font-weight: 600;
  color: var(--t0);
  letter-spacing: -0.3px;
  font-family: var(--font-display);
}

.phSub {
  font-size: 11px;
  color: var(--t2);
  margin-top: 3px;
  @include m.mono;
}

.tourTriggerBtn {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 4px 10px;
  border-radius: 99px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  font-family: var(--font-ui);
  font-size: 12px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.12s;
  flex-shrink: 0;

  svg { width: 13px; height: 13px; }

  &:hover {
    background: var(--bg3);
    color: var(--t1);
    border-color: var(--bdr3);
  }
}

// ── Scrollable body ──────────────────────────────
.body {
  flex: 1;
  display: flex;
  flex-direction: column;
  min-height: 0;
  overflow: hidden;
}

// ══════════════════════════════════════════
// SHARED — meta count, search, info box
// ══════════════════════════════════════════
.dictMeta {
  display: flex;
  align-items: baseline;
  gap: 6px;
}

.dictCount {
  font-size: 14px;
  font-weight: 700;
  color: var(--blue);
  @include m.mono;
}

.dictTotal {
  font-size: 11px;
  color: var(--t2);
  @include m.mono;
}

.dictInput {
  padding: 8px 12px;
  background: var(--bg0);
  border: 1px solid var(--bdr3);
  border-radius: var(--r);
  color: var(--t0);
  font-family: var(--font-ui);
  font-size: 12px;
  outline: none;
  appearance: none;
  width: 100%;
  transition: border-color 0.12s, box-shadow 0.12s;

  &:focus { border-color: var(--blue); box-shadow: 0 0 0 2px var(--blue-dim); }
  &::placeholder { color: var(--t2); }
}

// ── Search box ─────────────────────────────────────
.dictSearch {
  display: flex;
  align-items: center;
  gap: 7px;
  flex: 1;
  min-width: 180px;
  background: var(--bg1);
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  padding: 6px 10px;
  transition: border-color 0.12s;

  &:focus-within { border-color: var(--blue); box-shadow: 0 0 0 2px var(--blue-dim); }

  svg { width: 13px; height: 13px; color: var(--t2); flex-shrink: 0; }
}

.dictSearchInput {
  flex: 1;
  background: transparent;
  border: none;
  outline: none;
  font-size: 12px;
  color: var(--t0);
  font-family: var(--font-ui);
  &::placeholder { color: var(--t2); }
}

.dictSearchClear {
  background: none;
  border: none;
  color: var(--t2);
  cursor: pointer;
  font-size: 11px;
  padding: 0 2px;
  &:hover { color: var(--red); }
}

// ── Empty state inside table ──────────────────────
.dictEmpty {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 10px;
  padding: 36px 20px;
  font-size: 12px;
  color: var(--t2);
  @include m.mono;
  text-align: center;
}

.dictTable {
  background: var(--bg1);
  border: 1px solid var(--bdr);
  border-radius: var(--rl);
  overflow: hidden;
}

.dictArrow {
  font-size: 13px;
  color: var(--t2);
  text-align: center;
}

// ── Row action button ─────────────────────────────
.dictActionBtn {
  width: 26px;
  height: 26px;
  border-radius: 6px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  transition: all 0.12s;
  svg { width: 12px; height: 12px; }

  &:hover { background: var(--blue-dim); border-color: var(--blue-bdr); color: var(--blue); }
}

.dictActionDel {
  &:hover { background: var(--red-dim); border-color: var(--red-bdr); color: var(--red); }
}

// ── Info box ──────────────────────────────────────
.dictInfo {
  display: flex;
  align-items: flex-start;
  gap: 8px;
  margin: 0 24px 16px;
  padding: 11px 14px;
  background: var(--blue-dim);
  border: 1px solid var(--blue-bdr);
  border-radius: var(--r);
  font-size: 11px;
  color: var(--blue);
  line-height: 1.6;
  @include m.mono;
  flex-shrink: 0;

  svg { width: 13px; height: 13px; flex-shrink: 0; margin-top: 1px; }
}

// ── Buttons ───────────────────────────────────────
.btn {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 8px 16px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t1);
  font-family: var(--font-ui);
  font-size: 12px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.12s;
  white-space: nowrap;
  svg { width: 13px; height: 13px; }
  &:hover { background: var(--bg3); color: var(--t0); border-color: var(--bdr3); }
  &:disabled { opacity: 0.4; cursor: not-allowed; pointer-events: none; }
}

.btnP {
  background: var(--blue);
  color: #fff;
  border-color: var(--blue);
  font-weight: 600;
  &:hover { background: #a78bfa; border-color: #a78bfa; color: #fff; }
}

.btnSaved {
  background: var(--green);
  border-color: var(--green);
  &:hover { background: var(--green); border-color: var(--green); }
}

.btnDangerGhost {
  border-color: var(--bdr2);
  color: var(--t2);
  &:hover {
    background: var(--red-dim);
    border-color: var(--red-bdr);
    color: var(--red);
  }
}

.btnDanger {
  background: var(--red-dim);
  border-color: var(--red-bdr);
  color: var(--red);
  font-weight: 600;
  &:hover { background: var(--red); border-color: var(--red); color: #fff; }
}

// ── Inline delete confirmation ─────────────────
.dictDeleteConfirm {
  display: flex;
  align-items: center;
  gap: 8px;
  flex-wrap: wrap;
}

.dictDeleteWarn {
  font-size: 11px;
  color: var(--amber);
  @include m.mono;
  white-space: nowrap;
}

// ══════════════════════════════════════════
// USER DICTIONARY · Master-detail layout
// ══════════════════════════════════════════

.dictLayout {
  flex: 1;
  display: flex;
  min-height: 0;
  overflow: hidden;
}

// ── Left: list panel ──────────────────────────────
.dictListPanel {
  width: 280px;
  flex-shrink: 0;
  background: var(--bg1);
  border-right: 1px solid var(--bdr);
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.dictListHead {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 14px 18px 10px;
  border-bottom: 1px solid var(--bdr);
  flex-shrink: 0;
}

.dictListTitle {
  font-size: 11px;
  font-weight: 600;
  color: var(--t2);
  text-transform: uppercase;
  letter-spacing: 0.1em;
  @include m.mono;
}

.dictListAdd {
  width: 26px;
  height: 26px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t1);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.12s;

  svg { width: 13px; height: 13px; }

  &:hover {
    background: var(--blue-dim);
    border-color: var(--blue-bdr);
    color: var(--blue);
  }
}

.dictList {
  flex: 1;
  overflow-y: auto;
  padding: 8px 8px 16px;
  display: flex;
  flex-direction: column;
  gap: 2px;
  @include m.scrollbar;
}

.dictListLoading,
.dictListEmpty {
  padding: 24px 16px;
  font-size: 11px;
  color: var(--t2);
  text-align: center;
  line-height: 1.6;
  @include m.mono;
}

.dictListItem {
  text-align: left;
  background: transparent;
  border: 1px solid transparent;
  border-radius: var(--r);
  padding: 9px 11px;
  cursor: pointer;
  font-family: var(--font-ui);
  color: var(--t1);
  transition: all 0.12s;
  display: flex;
  flex-direction: column;
  gap: 2px;

  &:hover {
    background: var(--bg2);
    color: var(--t0);
  }

  &.dictListItemActive {
    background: var(--blue-dim);
    border-color: var(--blue-bdr);
    color: var(--blue);
  }
}

.dictListItemName {
  font-size: 13px;
  font-weight: 500;
  @include m.truncate;
}

.dictListItemMeta {
  font-size: 10px;
  color: var(--t2);
  @include m.mono;

  .dictListItemActive & { color: var(--blue); opacity: 0.75; }
}

// ── Right: detail panel ───────────────────────────
.dictDetail {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  background: var(--bg0);
}

.dictDetailEmpty {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 14px;
  color: var(--t2);
  font-size: 12px;
  @include m.mono;
  padding: 40px;
  text-align: center;

  svg { width: 36px; height: 36px; opacity: 0.35; }
}

.dictDetailHead {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
  padding: 16px 24px 14px;
  background: var(--bg1);
  border-bottom: 1px solid var(--bdr);
  flex-shrink: 0;
}

.dictDetailHeadMeta {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.dictNameField {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.dictNameLabelRow {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
}

.dictNameLabel {
  font-size: 10px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.1em;
  color: var(--t2);
  @include m.mono;
}

.dictNameCount {
  font-size: 10px;
  color: var(--t2);
  @include m.mono;

  &.dictNameCountMax { color: var(--amber); }
}

.dictNameInput {
  width: 100%;
  background: var(--bg0);
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  padding: 9px 12px;
  font-family: var(--font-display);
  font-size: 16px;
  font-weight: 600;
  color: var(--t0);
  letter-spacing: -0.2px;
  outline: none;
  transition: border-color 0.12s, box-shadow 0.12s;

  &:hover  { border-color: var(--bdr3); }
  &:focus  { border-color: var(--blue); box-shadow: 0 0 0 3px var(--blue-dim); }
  &::placeholder { color: var(--t2); font-weight: 400; }
}

.dictDetailDates {
  font-size: 10px;
  color: var(--t2);
  @include m.mono;
}

.dictDetailActions {
  display: flex;
  gap: 8px;
  flex-shrink: 0;
}

.dictErrorBox {
  margin: 12px 24px 0;
  padding: 9px 12px;
  background: var(--red-dim);
  border: 1px solid var(--red-bdr);
  border-radius: var(--r);
  font-size: 11px;
  color: var(--red);
  @include m.mono;
}

.dictDetailToolbar {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 14px 24px 10px;
  flex-shrink: 0;
}

// ── Toolbar row (Add term, right-aligned) ─────────
.dictTermsToolbar {
  display: flex;
  justify-content: flex-end;
  align-items: center;
  padding: 0 24px 10px;
  flex-shrink: 0;
}

// ── Terms editor (scroll container) ───────────────
.dictTermsWrap {
  flex: 1;
  overflow-y: auto;
  padding: 0 24px 24px;
  @include m.scrollbar;
}

.dictTermsHead {
  display: grid;
  grid-template-columns: 1fr 24px 1fr 32px;
  gap: 10px;
  padding: 9px 14px;
  background: var(--bg2);
  border-bottom: 1px solid var(--bdr);
  font-size: 10px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.08em;
  color: var(--t2);
  position: sticky;
  top: 0;
  z-index: 1;
  @include m.mono;
}

.dictTermsRow {
  display: grid;
  grid-template-columns: 1fr 24px 1fr 32px;
  align-items: center;
  gap: 10px;
  padding: 9px 14px;
  border-bottom: 1px solid var(--bdr);
  transition: background 0.1s;

  &:last-child { border-bottom: none; }
  &:hover { background: var(--bg2); }
}

.dictAddRowBtn {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 7px 12px;
  background: transparent;
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  color: var(--t1);
  font-family: var(--font-ui);
  font-size: 12px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.12s;

  svg { width: 12px; height: 12px; }

  &:hover {
    background: var(--blue-dim);
    border-color: var(--blue-bdr);
    color: var(--blue);
  }
}




















{
  "_comment": "Add these keys inside the existing 'settings' object in en.json (alongside the 'tour' block)",
  "deleteDict":     "Delete dictionary",
  "deleteWarning":  "Files will be disassociated.",
  "confirmDelete":  "Delete",
  "deleting":       "Deleting…"
}













{
  "_comment": "Add these keys inside the existing 'settings' object in ko.json (alongside the 'tour' block)",
  "deleteDict":     "사전 삭제",
  "deleteWarning":  "연결된 파일이 해제됩니다.",
  "confirmDelete":  "삭제",
  "deleting":       "삭제 중…"
}
