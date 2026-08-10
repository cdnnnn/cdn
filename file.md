//Datasets.tsx
import { useEffect, useMemo, useState } from 'react';
import { RefreshCw, Search, ExternalLink, Layers, AlertTriangle } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchBenchmarks } from '../../store/slices/benchmarksSlice';
import type { Benchmark } from '../../types';
import styles from './Datasets.module.scss';

// Deterministic color hash so the same capability always gets the same pill
// color across cards/renders, without a hardcoded lookup table.
const PILL_COLORS = [
  { bg: '#E9EBF8', fg: '#1428A0' }, { bg: '#FFFBEB', fg: '#D97706' }, { bg: '#ECFDF5', fg: '#059669' },
  { bg: '#FFF1F2', fg: '#F43F5E' }, { bg: '#F0F9FF', fg: '#0EA5E9' }, { bg: '#FEF2F2', fg: '#EF4444' },
];
function hashColor(label: string) {
  const sum = [...label].reduce((acc, ch) => acc + ch.charCodeAt(0), 0);
  return PILL_COLORS[sum % PILL_COLORS.length];
}

export default function Datasets() {
  const dispatch = useAppDispatch();
  const { items, status, error } = useAppSelector((s) => s.benchmarks);
  const [search, setSearch] = useState('');
  const [typeFilter, setTypeFilter] = useState('All');
  const [subgroupsFor, setSubgroupsFor] = useState<Benchmark | null>(null);

  useEffect(() => {
    dispatch(fetchBenchmarks());
  }, [dispatch]);

  const types = useMemo(() => ['All', ...new Set(items.map((b) => b.type))], [items]);
  const filtered = items.filter((b) => {
    if (typeFilter !== 'All' && b.type !== typeFilter) return false;
    const q = search.toLowerCase();
    return !q || b.name.toLowerCase().includes(q) || b.description.toLowerCase().includes(q);
  });

  return (
    <div className="page-enter pg-shell">
      <div className={styles.header}>
        <div className={styles.header__eyebrow}>Datasets</div>
        <h1 className={styles.header__title}>Test Suite Library</h1>
        <p className={styles.header__subtitle}>Browse every benchmark available for evaluations, independent of any single wizard run.</p>
        <div className={styles.header__row}>
          <span className={styles.header__count}>{items.length} suites available</span>
          <button className="btn btn-ghost btn-sm" onClick={() => dispatch(fetchBenchmarks())}><RefreshCw size={14} /> Refresh</button>
        </div>
      </div>

      <div className="pg-toolbar" style={{ paddingTop: 0 }}>
        <div className="toolbar">
          <div className="search-box"><Search size={16} color="#9CA3AF" /><input placeholder="Search datasets…" value={search} onChange={(e) => setSearch(e.target.value)} /></div>
          <div className="pills">{types.map((t) => <button key={t} className={`pill ${typeFilter === t ? 'on' : ''}`} onClick={() => setTypeFilter(t)}>{t}</button>)}</div>
        </div>
      </div>

      <div className="pg-body" style={{ paddingTop: 0 }}>

        {status === 'loading' && (
          <div className={styles.state}><Layers size={28} /><div>Loading datasets…</div></div>
        )}

        {status === 'failed' && (
          <div className={styles.state}>
            <AlertTriangle size={28} color="#EF4444" />
            <div>{error || 'Failed to load datasets.'}</div>
            <button className="btn btn-ind btn-sm" onClick={() => dispatch(fetchBenchmarks())}>Retry</button>
          </div>
        )}

        {status === 'succeeded' && filtered.length === 0 && (
          <div className={styles.state}><Layers size={28} /><div>No datasets match your search.</div></div>
        )}

        <div className="cards-grid">
          {status !== 'loading' && filtered.map((b) => {
            // Defensive per spec §3.2 / §5 — tasks & required_capabilities are
            // normalized to [] in benchmarksApi.list(), but reads still fall
            // back defensively here in case a consumer bypasses that layer.
            const tasks = b.tasks ?? [];
            const caps = b.required_capabilities ?? [];
            return (
              <div className="card" key={b.name}>
                <div className="card-hdr">
                  <div>
                    <div className="card-title">{b.name}</div>
                    <div style={{ marginTop: 6 }}><span className="tag tag-amb">{b.type}</span></div>
                  </div>
                  <div style={{ textAlign: 'right' }}>
                    <div style={{ fontFamily: "'Segoe UI', Roboto, Arial, sans-serif", fontSize: 24, fontWeight: 700 }}>{b.task_count.toLocaleString()}</div>
                    <div style={{ fontSize: 11, color: '#9CA3AF', fontWeight: 600 }}>tasks</div>
                  </div>
                </div>
                <div className="card-desc">{b.description}</div>

                <div className={styles.statRow}>
                  <span>{b.task_count.toLocaleString()} Tasks</span>
                  <span>{caps.length} Capabilities</span>
                  <span>{b.huggingface_dataset.split('/')[0]}</span>
                </div>

                <div className={styles.pillRow}>
                  {caps.map((c) => {
                    const color = hashColor(c);
                    return <span key={c} className={styles.pill} style={{ background: color.bg, color: color.fg }}>{c}</span>;
                  })}
                </div>

                <div className="card-foot">
                  {tasks.length > 0 ? (
                    <button className="btn btn-sm btn-ghost" onClick={() => setSubgroupsFor(b)}>View subgroups</button>
                  ) : <span />}
                  <a
                    className={styles.sourceLink}
                    href={`https://huggingface.co/datasets/${b.huggingface_dataset}`}
                    target="_blank" rel="noreferrer"
                  >
                    Source <ExternalLink size={12} />
                  </a>
                </div>
              </div>
            );
          })}
        </div>
      </div>

      {subgroupsFor && (
        <div className={styles.modalOverlay} onClick={() => setSubgroupsFor(null)}>
          <div className={styles.modal} onClick={(e) => e.stopPropagation()}>
            <div className={styles.modal__hdr}>
              <span>{subgroupsFor.name} — subgroups</span>
              <button className="btn btn-sm btn-ghost" onClick={() => setSubgroupsFor(null)}>Close</button>
            </div>
            <div className={styles.modal__body}>
              {(subgroupsFor.tasks ?? []).map((t) => (
                <div key={t.value} className={styles.modal__row}><span>{t.name}</span><code>{t.value}</code></div>
              ))}
            </div>
          </div>
        </div>
      )}
    </div>
  );
}
