//Sidebar.tsx
import { NavLink } from 'react-router-dom';
import { Home, Link2, Cpu, BookOpen, Play, FlaskConical, GitCompare, FileText, LogOut } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { logout } from '../../store/slices/authSlice';
import styles from './Sidebar.module.scss';

const navItems = [
  { to: '/app/dashboard', icon: <Home size={18} />, label: 'Dashboard' },
  { to: '/app/providers', icon: <Link2 size={18} />, label: 'Providers' },
  { to: '/app/models', icon: <Cpu size={18} />, label: 'Models' },
  { to: '/app/suites', icon: <BookOpen size={18} />, label: 'Test Suites' },
];

const workflowItems = [
  { to: '/app/new-eval', icon: <Play size={18} />, label: 'New Evaluation' },
  { to: '/app/evaluations', icon: <FlaskConical size={18} />, label: 'Evaluations' },
  { to: '/app/comparison', icon: <GitCompare size={18} />, label: 'Comparison' },
  { to: '/app/reports', icon: <FileText size={18} />, label: 'Reports' },
];

export default function Sidebar() {
  const dispatch = useAppDispatch();
  const user = useAppSelector((s) => s.auth.user);

  const navLinkClass = ({ isActive }: { isActive: boolean }) =>
    `${styles['nav-item']} ${isActive ? styles.active : ''}`;

  return (
    <div className={styles.sidebar}>
      <div className={styles['sidebar__logo']}>
        <div className={styles['sidebar__mark']}>&#9670;</div>
        SemcoEval
      </div>
      <nav className={styles['sidebar__nav']}>
        {navItems.map((item) => (
          <NavLink key={item.to} to={item.to} className={navLinkClass}>
            {item.icon}
            {item.label}
          </NavLink>
        ))}
        <div className={styles['sidebar__section']}>Workflow</div>
        {workflowItems.map((item) => (
          <NavLink key={item.to} to={item.to} className={navLinkClass}>
            {item.icon}
            {item.label}
          </NavLink>
        ))}
      </nav>
      <div className={styles['sidebar__foot']}>
        <div className={styles['sidebar__user']}>
          <div className={styles['sidebar__avatar']}>{(user?.username || 'U').slice(0, 2).toUpperCase()}</div>
          <div className={styles['sidebar__user-info']}>
            <div className={styles['sidebar__user-name']}>{user?.profile_name || user?.username || 'Guest'}</div>
            <div className={styles['sidebar__user-email']}>{user?.email || ''}</div>
          </div>
          <LogOut size={16} style={{ color: '#9CA3AF', cursor: 'pointer' }} onClick={() => dispatch(logout())} />
        </div>
      </div>
    </div>
  );
}

















//Reports.ts
import api from '../axiosInstance';

export type ReportStatus = 'running' | 'completed' | 'failed';

export interface ReportListItem {
  id: string;
  title: string;
  evaluation_id: string;
  status: ReportStatus;
  file_path: string | null;
  top_model: string | null;
  top_score: number | null;
  eval_type: string;
  eval_name: string;
  metrics_tested: string[];
  created_at: string;
}

export interface ReportModelResult {
  model_id: string;
  rank: number;
  score: number;
  passed: number;
  failed: number;
  total: number;
  metrics: Record<string, unknown>;
}

export interface ReportDetail {
  id: string;
  title: string;
  evaluation_id: string;
  date: string;
  summary: string;
  topModel: string | null;
  verdict: string | null;
  metricsTested: string[];
  downloadSize: number | null;
  status: ReportStatus | 'pending';
  file_path: string | null;
  eval_type: string;
  eval_name: string;
  top_score: number | null;
  total_questions: number;
  passed_tests: number;
  failed_tests: number;
  models: ReportModelResult[];
  created_at: string;
}

export type ReportDownloadFormat = 'json' | 'csv' | 'csv_detailed' | 'pdf';

const DOWNLOAD_MIME: Record<ReportDownloadFormat, string> = {
  json: 'application/json',
  csv: 'text/csv',
  csv_detailed: 'text/csv',
  pdf: 'application/pdf',
};

const DOWNLOAD_EXT: Record<ReportDownloadFormat, string> = {
  json: 'json',
  csv: 'csv',
  csv_detailed: 'csv',
  pdf: 'pdf',
};

export const reportsApi = {
  list: () => api.get<{ reports: ReportListItem[] }>('/reports').then((r) => r.data.reports),

  getById: (reportId: string) =>
    api.get<ReportDetail>(`/reports/${reportId}`).then((r) => r.data),

  // Downloads go through the shared axios instance (so the auth interceptor
  // still attaches the bearer token) as a blob, then get saved client-side
  // via an object URL — a plain <a href> to the API wouldn't carry auth.
  download: async (
    reportId: string,
    format: ReportDownloadFormat,
    filenameHint = reportId
  ) => {
    const response = await api.get(`/reports/${reportId}/download`, {
      params: { format },
      responseType: 'blob',
    });
    const blob = new Blob([response.data], { type: DOWNLOAD_MIME[format] });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = `${filenameHint}.${DOWNLOAD_EXT[format]}`;
    document.body.appendChild(a);
    a.click();
    a.remove();
    URL.revokeObjectURL(url);
  },
};














//Reportsslice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { reportsApi } from '../../api/endpoints/reports';
import type { ReportListItem, ReportDetail, ReportDownloadFormat } from '../../api/endpoints/reports';

type Status = 'idle' | 'loading' | 'succeeded' | 'failed';

interface ReportsState {
  list: ReportListItem[];
  listStatus: Status;
  listError: string | null;

  detailById: Record<string, ReportDetail>;
  detailStatusById: Record<string, Status>;
  detailErrorById: Record<string, string | null>;

  downloadingId: string | null;
}

const initialState: ReportsState = {
  list: [],
  listStatus: 'idle',
  listError: null,

  detailById: {},
  detailStatusById: {},
  detailErrorById: {},

  downloadingId: null,
};

export const fetchReports = createAsyncThunk('reports/fetchAll', () => reportsApi.list());

export const fetchReportDetail = createAsyncThunk('reports/fetchDetail', (reportId: string) =>
  reportsApi.getById(reportId)
);

export const downloadReport = createAsyncThunk(
  'reports/download',
  async ({
    reportId,
    format,
    filenameHint,
  }: {
    reportId: string;
    format: ReportDownloadFormat;
    filenameHint?: string;
  }) => {
    await reportsApi.download(reportId, format, filenameHint);
    return reportId;
  }
);

const reportsSlice = createSlice({
  name: 'reports',
  initialState,
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(fetchReports.pending, (state) => {
        state.listStatus = 'loading';
      })
      .addCase(fetchReports.fulfilled, (state, action) => {
        state.listStatus = 'succeeded';
        state.list = action.payload;
      })
      .addCase(fetchReports.rejected, (state, action) => {
        state.listStatus = 'failed';
        state.listError = action.error.message || 'Failed to load reports';
      })

      .addCase(fetchReportDetail.pending, (state, action) => {
        state.detailStatusById[action.meta.arg] = 'loading';
        state.detailErrorById[action.meta.arg] = null;
      })
      .addCase(fetchReportDetail.fulfilled, (state, action) => {
        state.detailStatusById[action.meta.arg] = 'succeeded';
        state.detailById[action.meta.arg] = action.payload;
      })
      .addCase(fetchReportDetail.rejected, (state, action) => {
        state.detailStatusById[action.meta.arg] = 'failed';
        state.detailErrorById[action.meta.arg] = action.error.message || 'Failed to load report';
      })

      .addCase(downloadReport.pending, (state, action) => {
        state.downloadingId = action.meta.arg.reportId;
      })
      .addCase(downloadReport.fulfilled, (state) => {
        state.downloadingId = null;
      })
      .addCase(downloadReport.rejected, (state) => {
        state.downloadingId = null;
      });
  },
});

export default reportsSlice.reducer;


















//store.ts
import { configureStore } from '@reduxjs/toolkit';
import authReducer from './slices/authSlice';
import providersReducer from './slices/providersSlice';
import modelsReducer from './slices/modelsSlice';
import benchmarksReducer from './slices/benchmarksSlice';
import metricsReducer from './slices/metricsSlice';
import evaluationsReducer from './slices/evaluationsSlice';
import reportsReducer from './slices/reportsSlice';

export const store = configureStore({
  reducer: {
    auth: authReducer,
    providers: providersReducer,
    models: modelsReducer,
    benchmarks: benchmarksReducer,
    metrics: metricsReducer,
    evaluations: evaluationsReducer,
    reports: reportsReducer,
  },
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
















//Reports.tsx
import { useEffect, useMemo, useState } from 'react';
import { useSearchParams } from 'react-router-dom';
import {
  Search, FileText, Loader2, Download, Award, ListChecks, Clock, FileBarChart,
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

function statusBadgeClass(status: string): string {
  switch (status) {
    case 'completed': return 'badge-green';
    case 'running': return 'badge-run';
    case 'failed': return 'badge-gray';
    default: return 'badge-blue';
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
        <div className={styles.layout}>
          {/* ---------- Sidebar list ---------- */}
          <div className={styles.sidebar}>
            <div className="search-box" style={{ minWidth: 0, marginBottom: 10 }}>
              <Search size={16} color="#9CA3AF" />
              <input placeholder="Search reports…" value={search} onChange={(e) => setSearch(e.target.value)} />
            </div>
            <div className="pills" style={{ marginBottom: 12 }}>
              {types.map((t) => (
                <button key={t} className={`pill ${typeFilter === t ? 'on' : ''}`} onClick={() => setTypeFilter(t)}>{t}</button>
              ))}
            </div>

            {listStatus === 'loading' && list.length === 0 && <div className={styles.empty}><Loader2 size={16} style={{ animation: 'spin 1.5s linear infinite' }} /> Loading reports…</div>}
            {listStatus === 'failed' && list.length === 0 && <div className={styles.empty}>{listError || 'Failed to load reports.'}</div>}
            {listStatus !== 'loading' && filtered.length === 0 && list.length > 0 && <div className={styles.empty}>No reports match your filters.</div>}

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
                      <span className="tag tag-ind">{r.eval_type}</span>
                      <span className={`badge ${statusBadgeClass(r.status)}`}>{r.status}</span>
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
                      <span className="tag tag-ind">{selected.eval_type}</span>
                      <span className={`badge ${statusBadgeClass(selected.status)}`}>{selected.status}</span>
                    </div>
                    <h2 className={styles['detail-hdr__name']}>{selected.title}</h2>
                    <div className={styles['detail-hdr__date']}>Created {new Date(selected.created_at).toLocaleString()}</div>
                  </div>
                  <div className={styles['detail-hdr__actions']}>
                    {DOWNLOAD_OPTIONS.map((opt) => (
                      <button
                        key={opt.format}
                        className="btn btn-ghost btn-sm"
                        disabled={selected.status !== 'completed' || downloadingId === selected.id}
                        onClick={() => dispatch(downloadReport({ reportId: selected.id, format: opt.format, filenameHint: selected.title }))}
                      >
                        {downloadingId === selected.id ? <Loader2 size={12} style={{ animation: 'spin 1.5s linear infinite' }} /> : <Download size={12} />} {opt.label}
                      </button>
                    ))}
                  </div>
                </div>

                {detailStatus === 'loading' && <div className={styles.empty}><Loader2 size={16} style={{ animation: 'spin 1.5s linear infinite' }} /> Loading report…</div>}
                {detailStatus === 'failed' && <div className={styles.empty}>{detailError}</div>}

                {detail && (
                  <>
                    <div className={styles['summary-cards']}>
                      <div className={styles['summary-card']}>
                        <Award size={16} color="#F59E0B" />
                        <div>
                          <div className={styles['summary-card__label']}>Top Model</div>
                          <div className={styles['summary-card__val']}>{detail.topModel || '—'}{detail.top_score != null ? ` · ${Math.round(detail.top_score * 100)}%` : ''}</div>
                        </div>
                      </div>
                      <div className={styles['summary-card']}>
                        <ListChecks size={16} color="#1428A0" />
                        <div>
                          <div className={styles['summary-card__label']}>Questions</div>
                          <div className={styles['summary-card__val']}>{detail.total_questions.toLocaleString()} &middot; {detail.passed_tests} passed &middot; {detail.failed_tests} failed</div>
                        </div>
                      </div>
                      <div className={styles['summary-card']}>
                        <Clock size={16} color="#10B981" />
                        <div>
                          <div className={styles['summary-card__label']}>Status</div>
                          <div className={styles['summary-card__val']}>{detail.status}{detail.date ? ` · ${detail.date}` : ''}</div>
                        </div>
                      </div>
                    </div>

                    {detail.summary && <div className={styles['summary-text']}>{detail.summary}</div>}

                    {detail.models.length > 0 ? (
                      <div className="tw">
                        <table className="tbl">
                          <thead><tr><th>Rank</th><th>Model</th><th>Score</th><th>Passed</th><th>Failed</th><th>Total</th></tr></thead>
                          <tbody>
                            {detail.models.map((m) => (
                              <tr key={m.model_id} className={m.rank === 1 ? 'winner' : ''}>
                                <td style={{ fontWeight: 700 }}>{m.rank === 1 ? '🏆 ' : ''}{m.rank}</td>
                                <td style={{ fontWeight: 700 }}>{m.model_id}</td>
                                <td style={{ fontFamily: "'Segoe UI', Roboto, Arial, sans-serif", fontWeight: 700 }}>{Math.round(m.score * 100)}%</td>
                                <td style={{ fontFamily: "'Segoe UI', Roboto, Arial, sans-serif", color: '#10B981' }}>{m.passed}</td>
                                <td style={{ fontFamily: "'Segoe UI', Roboto, Arial, sans-serif", color: '#EF4444' }}>{m.failed}</td>
                                <td style={{ fontFamily: "'Segoe UI', Roboto, Arial, sans-serif" }}>{m.total}</td>
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


















//Reports.module.scss
@use '../../styles/_variables' as *;

.reports {
  &__header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    gap: 1rem;
    padding: 24px 32px 18px;
    margin-bottom: 24px;
    border-bottom: 1px solid $border-light;

    h1 {
      font-family: $font-display;
      font-size: 1.5rem;
      font-weight: 800;
      letter-spacing: -0.02em;
      color: $text-primary;
      line-height: 1.2;
    }
  }

  &__header-eyebrow {
    display: flex;
    align-items: center;
    gap: 8px;
    font-family: $font-mono;
    font-size: 0.6875rem;
    font-weight: 700;
    letter-spacing: 0.12em;
    text-transform: uppercase;
    color: $indigo;
    margin-bottom: 6px;

    &::before {
      content: '';
      width: 16px;
      height: 2px;
      border-radius: 2px;
      background: $indigo;
    }
  }

  &__header-sub {
    margin-top: 4px;
    font-size: 0.875rem;
    color: $text-secondary;
  }

  &__header-meta {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-size: 0.75rem;
    font-weight: 600;
    color: $text-secondary;
    background: $surface-alt;
    border: 1px solid $border-light;
    border-radius: 999px;
    padding: 7px 13px;
    white-space: nowrap;
    margin-bottom: 3px;
  }
}

.pg-body-fixed {
  height: 100%;
  overflow: hidden;
}

.layout {
  display: grid;
  grid-template-columns: 380px 1fr;
  gap: 20px;
  align-items: start;
}

.sidebar { background: $surface; border: 1px solid $border; border-radius: 20px; padding: 18px; }
.empty { padding: 24px; text-align: center; color: $text-secondary; font-size: 13px; display: flex; align-items: center; gap: 8px; justify-content: center; }

.rows { display: flex; flex-direction: column; gap: 10px; max-height: 720px; overflow-y: auto; }

.row {
  border: 1px solid $border; border-radius: 14px; padding: 14px; cursor: pointer; transition: all .15s;
}
.row:hover { border-color: $indigo-light; }
.row.selected { border-color: $indigo; background: $indigo-pale; }

.row__top { display: flex; align-items: center; gap: 10px; margin-bottom: 8px; }
.row__icon { width: 30px; height: 30px; border-radius: 9px; background: $indigo-pale; color: $indigo; display: flex; align-items: center; justify-content: center; flex-shrink: 0; }
.row__name { font-weight: 700; font-size: 14px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.row__badges { display: flex; align-items: center; gap: 6px; margin-bottom: 6px; flex-wrap: wrap; }
.row__meta { font-size: 11px; color: $text-muted; margin-bottom: 8px; }
.row__stats { display: flex; gap: 12px; font-size: 11px; color: $text-secondary; flex-wrap: wrap; }

.detail { background: $surface; border: 1px solid $border; border-radius: 20px; padding: 24px; min-height: 400px; }
.detail-empty { padding: 80px 24px; text-align: center; color: $text-secondary; font-size: 14px; }

.detail-hdr { display: flex; justify-content: space-between; align-items: flex-start; margin-bottom: 24px; flex-wrap: wrap; gap: 16px; }
.detail-hdr__badges { display: flex; gap: 8px; margin-bottom: 10px; }
.detail-hdr__name { font-size: 22px; font-weight: 700; letter-spacing: -.3px; }
.detail-hdr__date { font-size: 12px; color: $text-muted; margin-top: 4px; }
.detail-hdr__actions { display: flex; gap: 8px; flex-wrap: wrap; }

.summary-cards { display: grid; grid-template-columns: repeat(3,1fr); gap: 14px; margin-bottom: 24px; }
.summary-card { display: flex; align-items: center; gap: 12px; padding: 16px; background: $surface-alt; border-radius: 14px; }
.summary-card__label { font-size: 11px; color: $text-muted; font-weight: 700; text-transform: uppercase; letter-spacing: .5px; }
.summary-card__val { font-size: 13px; font-weight: 700; margin-top: 2px; }

.summary-text { padding: 16px; background: $surface-alt; border-radius: 14px; font-size: 13px; color: $text-secondary; margin-bottom: 24px; }

.status-message { padding: 40px; text-align: center; background: $surface-alt; border-radius: 14px; color: $text-secondary; font-size: 14px; }

.download-row { display: flex; gap: 8px; flex-wrap: wrap; margin-top: 4px; }

@media (max-width: 900px) {
  .layout { grid-template-columns: 1fr; }
  .summary-cards { grid-template-columns: 1fr; }
}
