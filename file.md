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
  selectDictionary,
} from '../../store/dictionarySlice';
import type { DictionaryTerm } from '../../services/dictionaryApi';
import TourGuide, { type TourStep } from '../UploadInfer/TourGuide';
import styles from './Settings.module.scss';

const DICT_NAME_MAX_LENGTH = 255;

// ─────────────────────────────────────────────
// Tour steps for the User Dictionary feature.
// onEnter callbacks use DOM clicks so they work
// without lifting state up into the parent.
// ─────────────────────────────────────────────
const ensureEditorOpen = () => {
  // If the name input isn't in the DOM yet the editor panel is closed —
  // click the + button to open a new-dictionary form first.
  if (!document.querySelector('[data-tour="dict-name-input"]')) {
    document.querySelector<HTMLElement>('[data-tour="dict-new-btn"]')?.click();
  }
};

export const DICT_TOUR_STEPS: TourStep[] = [
  {
    target: 'dict-list-panel',
    placement: 'right',
    title: 'Your dictionaries',
    content:
      'All your saved dictionaries live here. Select one to view and edit it, or create a new one with the + button at the top.',
  },
  {
    target: 'dict-new-btn',
    placement: 'right',
    title: 'Create a dictionary',
    content:
      'Click + to start a new dictionary. You can have multiple dictionaries — for example one per course or subject area.',
  },
  {
    target: 'dict-name-input',
    placement: 'bottom',
    title: 'Name your dictionary',
    content:
      'Give it a clear, descriptive name (up to 255 characters). The name appears in the list panel and in any pipeline that uses this dictionary.',
    onEnter: ensureEditorOpen,
  },
  {
    target: 'dict-add-term-btn',
    placement: 'bottom',
    title: 'Add correction terms',
    content:
      'Click "Add term" to insert a new row. Each row maps one wrong form to its correct replacement — you can add as many as you need.',
    onEnter: ensureEditorOpen,
  },
  {
    target: 'dict-terms-table',
    placement: 'top',
    title: 'Wrong → Correct pairs',
    content:
      'Type the misspelled or incorrect form in the left column and the replacement in the right. The delete button removes a row. Use the search bar above to filter when the list is long.',
    onEnter: ensureEditorOpen,
  },
  {
    target: 'dict-save-btn',
    placement: 'left',
    title: 'Save your dictionary',
    content:
      'Hit "Create" (new) or "Save changes" (existing) to persist your work. The dictionary is applied automatically during STT transcription and inference runs.',
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
  const { items, selectedId, loading, saving, error } = useAppSelector(s => s.dictionary);

  const [isCreating, setIsCreating] = useState(false);
  const [name, setName] = useState('');
  const [terms, setTerms] = useState<LocalTerm[]>([]);
  const [search, setSearch] = useState('');
  const [dirty, setDirty] = useState(false);
  const [savedFlash, setSavedFlash] = useState(false);

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
    } else {
      setName('');
      setTerms([]);
      setDirty(false);
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
            className={styles.tourTriggerBtn}
            onClick={() => setTourActive(true)}
            title="Take a tour of User Dictionary"
          >
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
              <circle cx="8" cy="8" r="6.5" />
              <path d="M6.5 6.2C6.7 5.3 7.3 4.8 8 4.8s1.5.6 1.5 1.4C9.5 7.5 8 8 8 9.2" />
              <circle cx="8" cy="11.2" r=".6" fill="currentColor" stroke="none" />
            </svg>
            Take a tour
          </button>
        </div>
      </div>

      <div className={styles.body}>
        <UserDictionary />
      </div>

      <TourGuide
        steps={DICT_TOUR_STEPS}
        active={tourActive}
        onFinish={() => setTourActive(false)}
      />
    </div>
  );
};

export default Settings;






















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
  padding: 7px 14px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t1);
  font-family: var(--font-ui);
  font-size: 12px;
  font-weight: 500;
  cursor: pointer;
  white-space: nowrap;
  flex-shrink: 0;
  transition: all 0.12s;

  svg { width: 14px; height: 14px; }

  &:hover {
    background: var(--blue-dim);
    border-color: var(--blue-bdr);
    color: var(--blue);
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
