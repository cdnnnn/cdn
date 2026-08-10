//Modelcatalog.tsx
import { useEffect, useMemo, useState } from 'react';
import { Search, Plus, Loader2, Boxes } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchModels, createCustomModel } from '../../store/slices/modelsSlice';
import { fetchProviders } from '../../store/slices/providersSlice';
import AddCustomModelDrawer from './AddCustomModelDrawer';
import styles from './ModelCatalog.module.scss';

export default function ModelCatalog() {
  const dispatch = useAppDispatch();
  const { items, status, creating } = useAppSelector((s) => s.models);
  const providers = useAppSelector((s) => s.providers.items);
  const [search, setSearch] = useState('');
  const [capFilter, setCapFilter] = useState('All');
  const [drawerOpen, setDrawerOpen] = useState(false);

  useEffect(() => {
    dispatch(fetchModels());
    dispatch(fetchProviders());
  }, [dispatch]);

  const caps = useMemo(() => ['All', ...new Set(items.flatMap((m) => m.capabilities))], [items]);
  const providerName = (id: string) => providers.find((p) => p.id === id)?.name || id;

  const filtered = items.filter((m) => {
    if (capFilter !== 'All' && !m.capabilities.includes(capFilter)) return false;
    const q = search.toLowerCase();
    return !q || m.name.toLowerCase().includes(q) || providerName(m.provider_id).toLowerCase().includes(q);
  });

  return (
    <div className="page-enter pg-shell">
      <div className={styles['model-catalog__header']}>
        <div>
          <p className={styles['model-catalog__header-eyebrow']}>Catalog</p>
          <h1>Model Catalog</h1>
          <p className={styles['model-catalog__header-sub']}>All models across connected providers</p>
        </div>
        <div className={styles['model-catalog__header-meta']}>
          <Boxes size={13} />
          {items.length} model{items.length === 1 ? '' : 's'} listed
        </div>
      </div>
      <div className="pg-toolbar">
        <div className="toolbar">
          <div className="search-box">
            <Search size={16} color="#9CA3AF" />
            <input placeholder="Search models or providers…" value={search} onChange={(e) => setSearch(e.target.value)} />
          </div>
          <div style={{ display: 'flex', gap: 8, alignItems: 'center' }}>
            <div className="pills">{caps.map((c) => <button key={c} className={`pill ${capFilter === c ? 'on' : ''}`} onClick={() => setCapFilter(c)}>{c}</button>)}</div>
            <button className="btn btn-ind btn-sm" onClick={() => setDrawerOpen(true)}><Plus size={14} /> Register Custom</button>
          </div>
        </div>
      </div>
      <div className="pg-body">
        {status === 'loading' && <div className={styles['model-catalog__loading']}><Loader2 size={18} style={{ animation: 'spin 1.5s linear infinite' }} /> Loading models…</div>}

        <div className="tw">
          <table className="tbl">
            <thead>
              <tr><th>Model</th><th>Provider</th><th>Capabilities</th><th>Context</th><th>Price (in/out)</th><th>Accuracy</th><th>Status</th></tr>
            </thead>
            <tbody>
              {status !== 'loading' && filtered.map((m) => (
                <tr key={m.id}>
                  <td style={{ fontWeight: 700 }}>{m.name}</td>
                  <td style={{ color: '#6B7280' }}>{providerName(m.provider_id)}</td>
                  <td>{m.capabilities.map((c) => <span key={c} className="tag tag-ind">{c}</span>)}</td>
                  <td style={{ fontFamily: "'Segoe UI', Roboto, Arial, sans-serif", fontSize: 13 }}>{m.context_window.toLocaleString()}</td>
                  <td style={{ fontFamily: "'Segoe UI', Roboto, Arial, sans-serif", fontSize: 13, color: '#6B7280' }}>
                    {m.input_price != null ? `$${m.input_price.toFixed(2)}` : '—'} / {m.output_price != null ? `$${m.output_price.toFixed(2)}` : '—'}
                  </td>
                  <td>
                    <span style={{ fontFamily: "'Segoe UI', Roboto, Arial, sans-serif", fontWeight: 700, color: (m.accuracy_score || 0) >= 90 ? '#10B981' : '#111827' }}>
                      {m.accuracy_score != null ? `${m.accuracy_score}%` : '—'}
                    </span>
                  </td>
                  <td><span className={`badge ${m.is_active ? 'badge-green' : 'badge-gray'}`}>{m.is_active ? 'Active' : 'Inactive'}</span></td>
                </tr>
              ))}
            </tbody>
          </table>
        </div>
      </div>

      {drawerOpen && (
        <AddCustomModelDrawer
          submitting={creating}
          onClose={() => setDrawerOpen(false)}
          onSubmit={(payload) => {
            dispatch(createCustomModel(payload)).then(() => setDrawerOpen(false));
          }}
        />
      )}
    </div>
  );
}















//Modelcatalog.scss
@use '../../styles/_variables' as *;

.model-catalog {
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

  &__loading { display: flex; align-items: center; gap: 8px; color: $text-secondary; font-size: 13px; margin-bottom: 16px; }
}
