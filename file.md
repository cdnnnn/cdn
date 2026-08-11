import { useEffect, useMemo, useRef, useState } from 'react';
import { useSearchParams } from 'react-router-dom';
import {
  Search, FileText, Loader2, Download, Award, ListChecks, Clock,
  FileBarChart, SlidersHorizontal, X,
} from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchReports, fetchReportDetail, downloadReport } from '../../store/slices/reportsSlice';
import type { ReportDownloadFormat } from '../../api/endpoints/reports';
import styles from './Reports.module.scss';

const DOWNLOAD_OPTIONS: { format: ReportDownloadFormat; label: string }[] = [
  { format: 'json', label: 'JSON' },
  { format: 'csv', label: 'CSV' },
  { format: 'csv_detailed', label: 'CSV (Detailed)' },
  { format: 'pdf', label: 'PDF' },
];

// Maps a raw status to the module status-pill variant suffix (mirrors History).
function statusVariant(status: string): string {
  switch (status) {
    case 'completed': return 'completed';
    case 'running': return 'running';
    case 'pending': return 'pending';
    case 'failed': return 'failed';
    default: return 'pending';
  }
}

export default function Reports() {
  const dispatch = useAppDispatch();
  const [searchParams, setSearchParams] = useSearchParams();
  const selectedId = searchParams.get('id');

  const { list, listStatus, listError, detailById, detailStatusById, detailErrorById, downloadingId } =
    useAppSelector((s) => s.reports);

  const [search, setSearch] = useState('');
  const [typeFilter, setTypeFilter] = useState('All');
  const [activeFilter, setActiveFilter] = useState<'search' | 'type' | null>(null);
  const searchInputRef = useRef<HTMLInputElement>(null);

  const toggleFilter = (key: 'search' | 'type') => {
    setActiveFilter((prev) => (prev === key ? null : key));
  };

  useEffect(() => {
    if (activeFilter === 'search') searchInputRef.current?.focus();
  }, [activeFilter]);

  useEffect(() => {
    dispatch(fetchReports());
  }, [dispatch]);

  const types = useMemo(() => ['All', ...new Set(list.map((r) => r.eval_type))], [list]);

  const filtered = useMemo(() => {
    return list.filter((r) => {
      if (search && !r.title.toLowerCase().includes(search.toLowerCase())) return false;
      if (typeFilter !== 'All' && r.eval_type !== typeFilter) return false;
      return true;
    });
  }, [list, search, typeFilter]);

  const selected = list.find((r) => r.id === selectedId) || filtered[0] || null;

  useEffect(() => {
    if (selected && !detailById[selected.id] && detailStatusById[selected.id] !== 'loading') {
      dispatch(fetchReportDetail(selected.id));
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [selected?.id]);

  const selectRow = (id: string) => setSearchParams({ id });

  const detail = selected ? detailById[selected.id] : undefined;
  const detailStatus = selected ? detailStatusById[selected.id] : undefined;
  const detailError = selected ? detailErrorById[selected.id] : undefined;

  const StatusBadge = ({ status }: { status: string }) => (
    <span className={`${styles.status} ${styles[`status--${statusVariant(status)}`]}`}>
      {status === 'running' && <span className={styles['live-dot']} />}
      {status}
    </span>
  );

  return (
    <div className="page-enter pg-shell">
      <div className={styles['reports__header']}>
        <div>
          <p className={styles['reports__header-eyebrow']}>Output</p>
          <h1>Reports</h1>
          <p className={styles['reports__header-sub']}>Generated reports for completed and in-progress evaluations</p>
        </div>
        <div className={styles['reports__header-meta']}>
          <FileBarChart size={13} />
          {list.length} report{list.length === 1 ? '' : 's'} generated
        </div>
      </div>

      <div className={`pg-body ${styles['pg-body-fixed']}`}>
        <div className={styles.shell}>
          {/* ---------- Sidebar list ---------- */}
          <div className={styles.sidebar}>
            <div className={styles.filters}>
              <div className={styles['filter-toolbar']}>
                <span className={styles['filter-toolbar__label']}>Filters</span>
                <div className={styles['filter-toolbar__divider']} />
                <button
                  type="button"
                  className={`${styles['filter-toolbar__btn']} ${activeFilter === 'search' ? styles.on : ''}`}
                  onClick={() => toggleFilter('search')}
                  title="Search"
                >
                  <Search size={15} />
                  {search && <span className={styles['filter-toolbar__dot']} />}
                </button>
                <button
                  type="button"
                  className={`${styles['filter-toolbar__btn']} ${activeFilter === 'type' ? styles.on : ''}`}
                  onClick={() => toggleFilter('type')}
                  title="Filter by type"
                >
                  <SlidersHorizontal size={15} />
                  {typeFilter !== 'All' && <span className={styles['filter-toolbar__dot']} />}
                </button>

                <div className={styles['filter-toolbar__summary']}>
                  {search && (
                    <span className={styles['filter-chip']}>
                      <span>“{search}”</span>
                      <X size={11} onClick={() => setSearch('')} />
                    </span>
                  )}
                  {typeFilter !== 'All' && (
                    <span className={styles['filter-chip']}>
                      <span>{typeFilter}</span>
                      <X size={11} onClick={() => setTypeFilter('All')} />
                    </span>
                  )}
                </div>
              </div>

              <div className={`${styles['filter-panel']} ${activeFilter ? styles['filter-panel--open'] : ''}`}>
                {activeFilter === 'search' && (
                  <div>
                    <div className={styles['panel-search']}>
                      <Search size={16} />
                      <input
                        ref={searchInputRef}
                        placeholder="Search reports…"
                        value={search}
                        onChange={(e) => setSearch(e.target.value)}
                      />
                    </div>
                  </div>
                )}
                {activeFilter === 'type' && (
                  <div>
                    <div className={styles['panel-pills']}>
                      {types.map((t) => (
                        <button
                          key={t}
                          className={`${styles['panel-pill']} ${typeFilter === t ? styles.on : ''}`}
                          onClick={() => {
                            setTypeFilter(t);
                            setActiveFilter(null);
                          }}
                        >
                          {t}
                        </button>
                      ))}
                    </div>
                  </div>
                )}
              </div>

              {listStatus === 'loading' && list.length === 0 && (
                <div className={styles.empty}><Loader2 size={16} className={styles.spin} /> Loading reports…</div>
              )}
              {listStatus === 'failed' && list.length === 0 && (
                <div className={styles.empty}>{listError || 'Failed to load reports.'}</div>
              )}
              {listStatus !== 'loading' && filtered.length === 0 && list.length > 0 && (
                <div className={styles.empty}>No reports match your filters.</div>
              )}
            </div>

            <div className={styles.rows}>
              {filtered.map((r) => {
                const isSelected = selected?.id === r.id;
                return (
                  <div
                    key={r.id}
                    className={`${styles.row} ${isSelected ? styles.selected : ''}`}
                    onClick={() => selectRow(r.id)}
                  >
                    <div className={styles.row__top}>
                      <div className={styles.row__icon}><FileText size={16} /></div>
                      <div className={styles.row__name}>{r.title}</div>
                    </div>
                    <div className={styles.row__badges}>
                      <span className={styles['type-tag']}>{r.eval_type}</span>
                      <StatusBadge status={r.status} />
                    </div>
                    <div className={styles.row__meta}>{new Date(r.created_at).toLocaleDateString()}</div>
                    <div className={styles.row__stats}>
                      <span>{r.top_model ? `🏆 ${r.top_model}` : '—'}</span>
                      <span>{r.top_score != null ? `${Math.round(r.top_score * 100)}%` : '—'}</span>
                    </div>
                  </div>
                );
              })}
            </div>
          </div>

          {/* ---------- Detail panel ---------- */}
          <div className={styles.detail}>
            {!selected ? (
              <div className={styles['detail-empty']}>Select a report to see its details.</div>
            ) : (
              <>
                <div className={styles['detail-hdr']}>
                  <div>
                    <div className={styles['detail-hdr__badges']}>
                      <span className={styles['type-tag']}>{selected.eval_type}</span>
                      <StatusBadge status={selected.status} />
                    </div>
                    <h2 className={styles['detail-hdr__name']}>{selected.title}</h2>
                    <div className={styles['detail-hdr__date']}>Created {new Date(selected.created_at).toLocaleString()}</div>
                  </div>
                  <div className={styles['detail-hdr__actions']}>
                    {DOWNLOAD_OPTIONS.map((opt) => (
                      <button
                        key={opt.format}
                        className={styles['dl-btn']}
                        disabled={selected.status !== 'completed' || downloadingId === selected.id}
                        onClick={() => dispatch(downloadReport({ reportId: selected.id, format: opt.format, filenameHint: selected.title }))}
                      >
                        {downloadingId === selected.id ? <Loader2 size={12} className={styles.spin} /> : <Download size={12} />}
                        {opt.label}
                      </button>
                    ))}
                  </div>
                </div>

                {detailStatus === 'loading' && (
                  <div className={styles.empty}><Loader2 size={16} className={styles.spin} /> Loading report…</div>
                )}
                {detailStatus === 'failed' && <div className={styles.empty}>{detailError}</div>}

                {detail && (
                  <>
                    <div className={styles['summary-cards']}>
                      <div className={styles['summary-card']}>
                        <span className={`${styles['summary-card__icon']} ${styles['summary-card__icon--win']}`}>
                          <Award size={16} />
                        </span>
                        <div>
                          <div className={styles['summary-card__label']}>Top Model</div>
                          <div className={styles['summary-card__val']}>
                            {detail.topModel || '—'}
                            {detail.top_score != null ? ` · ${Math.round(detail.top_score * 100)}%` : ''}
                          </div>
                        </div>
                      </div>
                      <div className={styles['summary-card']}>
                        <span className={`${styles['summary-card__icon']} ${styles['summary-card__icon--info']}`}>
                          <ListChecks size={16} />
                        </span>
                        <div>
                          <div className={styles['summary-card__label']}>Questions</div>
                          <div className={styles['summary-card__val']}>
                            {detail.total_questions.toLocaleString()} &middot; {detail.passed_tests} passed &middot; {detail.failed_tests} failed
                          </div>
                        </div>
                      </div>
                      <div className={styles['summary-card']}>
                        <span className={`${styles['summary-card__icon']} ${styles['summary-card__icon--status']}`}>
                          <Clock size={16} />
                        </span>
                        <div>
                          <div className={styles['summary-card__label']}>Status</div>
                          <div className={styles['summary-card__val']}>
                            {detail.status}{detail.date ? ` · ${detail.date}` : ''}
                          </div>
                        </div>
                      </div>
                    </div>

                    {detail.summary && <div className={styles['summary-text']}>{detail.summary}</div>}

                    {detail.models.length > 0 ? (
                      <div className={styles.results}>
                        <table className={styles['results-table']}>
                          <thead>
                            <tr>
                              <th>Rank</th>
                              <th>Model</th>
                              <th>Score</th>
                              <th>Passed</th>
                              <th>Failed</th>
                              <th>Total</th>
                            </tr>
                          </thead>
                          <tbody>
                            {detail.models.map((m) => (
                              <tr key={m.model_id} className={m.rank === 1 ? styles.winner : ''}>
                                <td className={styles['cell-rank']}>{m.rank === 1 ? '🏆 ' : ''}{m.rank}</td>
                                <td className={styles['cell-model']}>{m.model_id}</td>
                                <td className={styles['cell-num']}>{Math.round(m.score * 100)}%</td>
                                <td className={styles['cell-pass']}>{m.passed}</td>
                                <td className={styles['cell-fail']}>{m.failed}</td>
                                <td className={`${styles['cell-num']} ${styles['cell-num--muted']}`}>{m.total}</td>
                              </tr>
                            ))}
                          </tbody>
                        </table>
                      </div>
                    ) : (
                      <div className={styles['status-message']}>
                        {detail.status === 'pending' && 'This report hasn\u2019t been generated yet.'}
                        {detail.status === 'running' && 'This report is still being generated.'}
                        {detail.status === 'failed' && 'Report generation failed.'}
                      </div>
                    )}
                  </>
                )}
              </>
            )}
          </div>
        </div>
      </div>
    </div>
  );
}













@use '../../styles/_variables' as *;

// ===========================================================================
// Reports — mirrors History.module.scss's design system exactly: ink/paper
// palette, ultramarine signal accent, mono instrument labels, hover-lift,
// mono numerals, self-contained master–detail shell.
// ===========================================================================

$ink:      #14161B;
$ink-2:    #565B66;
$ink-3:    #8A909B;
$paper:    #F5F6F8;
$card:     #FFFFFF;
$line:     #E6E8EC;
$line-2:   #EEF0F3;
$signal:   #2B2BF5;
$signal-2: #1C1CC7;
$wash:     #ECEDFF;
$ok:       #0FA968;
$ok-wash:  #E7F7EF;
$amber:    #E08600;
$amber-wash: #FDF3E3;
$danger:   #DC2626;
$danger-wash: #FDECEC;
$ink-wash: #EEF0F2;

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;

$soft: 0 1px 2px rgba(20, 22, 27, 0.05);
$lift: 0 14px 30px -14px rgba(20, 22, 27, 0.22);

%micro {
  font-family: $mono;
  font-size: 0.6875rem;
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}

.reports {
  &__header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    gap: 1rem;
    padding: 24px 32px 20px;
    border-bottom: 1px solid $line;
    background: $card;

    h1 {
      font-family: $display;
      font-size: 1.5rem;
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
    font-size: 0.84375rem;
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
    font-size: 0.71875rem;
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: $ink-2;
    white-space: nowrap;
    margin-bottom: 3px;
  }
}

@keyframes reports-live-pulse {
  0%, 100% { opacity: 1; transform: scale(1); }
  50% { opacity: 0.5; transform: scale(1.3); }
}
@keyframes reports-spin { to { transform: rotate(360deg); } }

.pg-body-fixed {
  overflow: hidden;
  display: flex;
  flex-direction: column;
  padding: 20px 32px 24px;
}

// ---- self-contained split shell -------------------------------------------
.shell {
  flex: 1;
  min-height: 0;
  display: flex;
  gap: 16px;
}

.sidebar {
  flex-shrink: 0;
  width: 380px;
  display: flex;
  flex-direction: column;
  min-height: 0;
  padding: 16px;
  background: $card;
  border: 1px solid $line;
  border-radius: 16px;
  box-shadow: $soft;
}

.detail {
  flex: 1;
  min-width: 0;
  min-height: 0;
  overflow-y: auto;
  padding: 24px;
  background: $card;
  border: 1px solid $line;
  border-radius: 16px;
  box-shadow: $soft;
}

.filters { flex-shrink: 0; }

// ---- filter toolbar --------------------------------------------------------
.filter-toolbar {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 6px 8px;
  margin-bottom: 8px;
  background: $paper;
  border: 1px solid $line;
  border-radius: 12px;
}

.filter-toolbar__label {
  flex-shrink: 0;
  @extend %micro;
  font-size: 0.625rem;
  color: $ink-3;
  padding-left: 4px;
}

.filter-toolbar__divider {
  flex-shrink: 0;
  width: 1px;
  height: 16px;
  background: $line;
}

.filter-toolbar__btn {
  position: relative;
  flex-shrink: 0;
  width: 30px;
  height: 30px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 8px;
  border: 1px solid transparent;
  background: transparent;
  color: $ink-2;
  cursor: pointer;
  transition: background 0.15s ease, color 0.15s ease, border-color 0.15s ease, box-shadow 0.15s ease;

  &:hover { background: $card; color: $signal; box-shadow: $soft; }

  &.on {
    border-color: $signal;
    background: $card;
    color: $signal;
    box-shadow: 0 0 0 3px $wash;
  }
}

.filter-toolbar__dot {
  position: absolute;
  top: -2px;
  right: -2px;
  width: 7px;
  height: 7px;
  border-radius: 50%;
  background: $signal;
  border: 1.5px solid $paper;
}

.filter-toolbar__summary {
  display: flex;
  align-items: center;
  gap: 6px;
  flex-wrap: wrap;
  min-width: 0;
  overflow: hidden;
}

.filter-chip {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  font-family: $mono;
  font-size: 0.65625rem;
  font-weight: 700;
  color: $signal;
  background: $wash;
  border: 1px solid rgba($signal, 0.18);
  border-radius: 999px;
  padding: 4px 8px;
  white-space: nowrap;
  max-width: 140px;

  span { overflow: hidden; text-overflow: ellipsis; }

  svg {
    cursor: pointer;
    flex-shrink: 0;
    opacity: 0.6;
    transition: opacity 0.15s ease;
    &:hover { opacity: 1; }
  }
}

// ---- collapsible filter panel ----------------------------------------------
.filter-panel {
  display: grid;
  grid-template-rows: 0fr;
  opacity: 0;
  transition: grid-template-rows 0.18s ease, opacity 0.15s ease, margin-bottom 0.18s ease;

  > * {
    overflow: hidden;
    min-height: 0;
    background: $paper;
    border: 1px solid $line;
    border-radius: 12px;
    padding: 10px;
  }

  &--open {
    grid-template-rows: 1fr;
    opacity: 1;
    margin-bottom: 8px;
  }
}

.panel-search {
  position: relative;

  svg {
    position: absolute;
    top: 50%;
    left: 12px;
    transform: translateY(-50%);
    color: $ink-3;
    pointer-events: none;
  }

  input {
    width: 100%;
    border: 1.5px solid $line;
    border-radius: 9px;
    padding: 8px 11px 8px 36px;
    font-size: 0.8125rem;
    font-family: $sans;
    color: $ink;
    background: $card;

    &::placeholder { color: $ink-3; }
    &:focus { outline: none; border-color: $signal; box-shadow: 0 0 0 3px $wash; }
  }
}

.panel-pills {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}

.panel-pill {
  padding: 6px 12px;
  border: 1px solid $line;
  border-radius: 999px;
  background: $card;
  color: $ink-2;
  font-size: 0.75rem;
  font-weight: 650;
  cursor: pointer;
  transition: all 0.14s ease;

  &:hover { border-color: $ink-3; color: $ink; }

  &.on {
    border-color: $signal;
    background: $signal;
    color: #fff;
  }
}

.empty {
  padding: 24px;
  text-align: center;
  color: $ink-3;
  font-size: 0.8125rem;
  display: flex;
  align-items: center;
  gap: 8px;
  justify-content: center;
}

// ---- report list rows ------------------------------------------------------
.rows {
  flex: 1;
  min-height: 0;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 10px;
  margin-top: 14px;
  padding-right: 2px;
}

.row {
  position: relative;
  border: 1.5px solid $line;
  border-radius: 14px;
  padding: 14px;
  cursor: pointer;
  background: $card;
  transition: border-color 0.15s ease, box-shadow 0.15s ease, transform 0.15s ease, background 0.15s ease;
}
.row:hover { border-color: $ink-3; box-shadow: $soft; transform: translateY(-1px); }
.row.selected { border-color: $signal; background: $wash; box-shadow: 0 0 0 1px $signal inset; }

.row__top { display: flex; align-items: center; gap: 10px; margin-bottom: 8px; }
.row__icon {
  width: 30px;
  height: 30px;
  border-radius: 9px;
  background: $wash;
  color: $signal;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}
.row__name {
  font-family: $display;
  font-weight: 700;
  font-size: 0.875rem;
  color: $ink;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.row__badges { display: flex; align-items: center; gap: 6px; margin-bottom: 6px; flex-wrap: wrap; }
.row__meta { font-family: $mono; font-size: 0.6875rem; color: $ink-3; margin-bottom: 8px; }
.row__stats {
  display: flex;
  gap: 12px;
  font-family: $mono;
  font-size: 0.6875rem;
  color: $ink-2;
  flex-wrap: wrap;
}

// ---- type tag + status badge (shared visual grammar) -----------------------
.type-tag {
  display: inline-flex;
  align-items: center;
  font-family: $mono;
  font-size: 0.625rem;
  font-weight: 700;
  letter-spacing: 0.04em;
  text-transform: uppercase;
  color: $signal;
  background: $wash;
  border-radius: 6px;
  padding: 3px 8px;
  white-space: nowrap;
}

.status {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 3px 10px 3px 9px;
  border-radius: 999px;
  font-family: $mono;
  font-size: 0.625rem;
  font-weight: 700;
  letter-spacing: 0.06em;
  text-transform: uppercase;

  &::before { content: ''; width: 5px; height: 5px; border-radius: 50%; }

  &--completed { color: $ok; background: $ok-wash; &::before { background: $ok; } }
  &--running   { color: $signal; background: $wash; &::before { display: none; } }
  &--pending   { color: $amber; background: $amber-wash; &::before { background: $amber; } }
  &--failed    { color: $ink-3; background: $ink-wash; &::before { background: $ink-3; } }
}

.live-dot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: currentColor;
  display: inline-block;
  animation: reports-live-pulse 1.4s ease-in-out infinite;
}

// ---- detail panel ----------------------------------------------------------
.detail-empty {
  padding: 80px 24px;
  text-align: center;
  color: $ink-3;
  font-size: 0.875rem;
}

.detail-hdr {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 24px;
  flex-wrap: wrap;
  gap: 16px;
}
.detail-hdr__badges { display: flex; gap: 8px; margin-bottom: 12px; }
.detail-hdr__name {
  font-family: $display;
  font-size: 1.375rem;
  font-weight: 800;
  letter-spacing: -0.02em;
  color: $ink;
}
.detail-hdr__date { font-family: $mono; font-size: 0.71875rem; color: $ink-3; margin-top: 6px; }
.detail-hdr__actions {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}

.dl-btn {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 7px 12px;
  border-radius: 9px;
  border: 1px solid $line;
  background: $paper;
  color: $ink-2;
  font-family: $mono;
  font-size: 0.6875rem;
  font-weight: 700;
  letter-spacing: 0.03em;
  cursor: pointer;
  transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

  &:hover:not(:disabled) { border-color: $signal; color: $signal; background: $wash; }

  &:disabled {
    opacity: 0.45;
    cursor: not-allowed;
  }
}

.summary-cards {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 12px;
  margin-bottom: 24px;
}
.summary-card {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 15px 16px;
  background: $paper;
  border: 1px solid $line;
  border-radius: 14px;
}
.summary-card__icon {
  flex-shrink: 0;
  width: 34px;
  height: 34px;
  border-radius: 10px;
  display: grid;
  place-items: center;
  background: $card;
  border: 1px solid $line;

  &--win { color: $amber; }
  &--info { color: $signal; }
  &--status { color: $ok; }
}
.summary-card__label {
  @extend %micro;
  font-size: 0.5625rem;
  color: $ink-3;
}
.summary-card__val {
  font-size: 0.8125rem;
  font-weight: 700;
  color: $ink;
  margin-top: 3px;
}

.summary-text {
  padding: 14px 16px;
  margin-bottom: 20px;
  background: $paper;
  border: 1px solid $line;
  border-radius: 12px;
  font-size: 0.84375rem;
  line-height: 1.55;
  color: $ink-2;
}

// ---- results table ---------------------------------------------------------
.results {
  border: 1px solid $line;
  border-radius: 14px;
  overflow: hidden;
}

.results-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.84375rem;

  thead th {
    text-align: left;
    background: $paper;
    border-bottom: 1px solid $line;
    @extend %micro;
    font-size: 0.5625rem;
    color: $ink-3;
    padding: 11px 14px;
    white-space: nowrap;
  }

  tbody tr {
    border-bottom: 1px solid $line-2;
    transition: background 0.13s ease;

    &:last-child { border-bottom: 0; }
    &:hover { background: $paper; }
  }

  tbody tr.winner {
    background: rgba($amber, 0.05);
    &:hover { background: rgba($amber, 0.09); }
  }

  tbody td {
    padding: 12px 14px;
    color: $ink;
  }
}

.cell-rank { font-family: $mono; font-weight: 700; color: $ink; }
.cell-model { font-family: $display; font-weight: 700; color: $ink; }
.cell-num { font-family: $mono; font-size: 0.8125rem; font-weight: 700; color: $ink; }
.cell-num--muted { font-weight: 500; color: $ink-2; }
.cell-pass { font-family: $mono; font-size: 0.8125rem; font-weight: 700; color: $ok; }
.cell-fail { font-family: $mono; font-size: 0.8125rem; font-weight: 700; color: $danger; }

.status-message {
  padding: 40px;
  text-align: center;
  background: $paper;
  border: 1px dashed $line;
  border-radius: 14px;
  color: $ink-2;
  font-size: 0.875rem;
}

.spin { animation: reports-spin 0.8s linear infinite; }

@media (max-width: 900px) {
  .shell { flex-direction: column; }
  .sidebar { width: 100%; }
  .summary-cards { grid-template-columns: 1fr; }
  .pg-body-fixed { overflow-y: auto; }
  .sidebar, .detail { overflow-y: visible; min-height: 0; }
  .rows { overflow-y: visible; }
}

@media (max-width: 640px) {
  .reports__header { padding: 20px 18px 16px; flex-direction: column; align-items: flex-start; gap: 10px; }
  .pg-body-fixed { padding: 16px 18px 22px; }
}
