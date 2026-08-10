//Providers.tsx
import { useEffect, useState } from 'react';
import { Search, Check, Plus, Settings, Unlink, Loader2, Cable } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchProviders, connectProvider, disconnectProvider } from '../../store/slices/providersSlice';
import styles from './Providers.module.scss';

type Filter = 'all' | 'connected' | 'available';

export default function Providers() {
  const dispatch = useAppDispatch();
  const { items, status, mutatingId } = useAppSelector((s) => s.providers);
  const [search, setSearch] = useState('');
  const [filter, setFilter] = useState<Filter>('all');
  const [keyPromptFor, setKeyPromptFor] = useState<string | null>(null);
  const [apiKeyInput, setApiKeyInput] = useState('');

  useEffect(() => {
    dispatch(fetchProviders());
  }, [dispatch]);

  const connectedCount = items.filter((p) => p.status === 'connected').length;

  const filtered = items.filter((p) => {
    if (filter === 'connected' && p.status !== 'connected') return false;
    if (filter === 'available' && p.status === 'connected') return false;
    return !search || p.name.toLowerCase().includes(search.toLowerCase());
  });

  const submitConnect = (providerId: string) => {
    if (!apiKeyInput.trim()) return;
    dispatch(connectProvider({ providerId, payload: { api_key: apiKeyInput } }));
    setKeyPromptFor(null);
    setApiKeyInput('');
  };

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
      <div className="pg-toolbar">
        <div className="toolbar">
          <div className="search-box">
            <Search size={16} color="#9CA3AF" />
            <input placeholder="Search providers…" value={search} onChange={(e) => setSearch(e.target.value)} />
          </div>
          <div className="pills">
            {(['all', 'connected', 'available'] as Filter[]).map((f) => (
              <button key={f} className={`pill ${filter === f ? 'on' : ''}`} onClick={() => setFilter(f)}>
                {f[0].toUpperCase() + f.slice(1)}
              </button>
            ))}
          </div>
        </div>
      </div>
      <div className="pg-body">
        {status === 'loading' && <div className={styles['providers__loading']}><Loader2 size={18} style={{ animation: 'spin 1.5s linear infinite' }} /> Loading providers…</div>}

        <div className="cards-grid">
          {status !== 'loading' && filtered.map((p) => (
            <div className="card" key={p.id}>
              <div className="card-hdr">
                <div style={{ display: 'flex', alignItems: 'center', gap: 14 }}>
                  <div className={`card-icon ${styles['providers__icon']}`}>{p.logo_url ? <img src={p.logo_url} alt={p.name} /> : p.name[0]}</div>
                  <div>
                    <div className="card-title">{p.name}</div>
                    <div style={{ fontSize: 12, color: '#6B7280', marginTop: 2 }}>{p.model_count} models</div>
                  </div>
                </div>
                <span className={`badge ${p.status === 'connected' ? 'badge-green' : 'badge-gray'}`}>
                  {p.status === 'connected' ? <><Check size={11} /> Connected</> : 'Not connected'}
                </span>
              </div>
              <div className="card-desc">{p.description}</div>

              {keyPromptFor === p.id ? (
                <div className={styles['providers__key-form']}>
                  <input
                    className="fi"
                    type="password"
                    placeholder="Paste API key…"
                    value={apiKeyInput}
                    onChange={(e) => setApiKeyInput(e.target.value)}
                    autoFocus
                  />
                  <div className={styles['providers__key-actions']}>
                    <button className="btn btn-sm btn-ind" onClick={() => submitConnect(p.id)}>Save</button>
                    <button className="btn btn-sm btn-ghost" onClick={() => setKeyPromptFor(null)}>Cancel</button>
                  </div>
                </div>
              ) : (
                <div className="card-foot">
                  <button
                    className={`btn btn-sm ${p.status === 'connected' ? 'btn-ghost' : 'btn-ind'}`}
                    disabled={mutatingId === p.id}
                    onClick={() => setKeyPromptFor(p.id)}
                  >
                    {mutatingId === p.id ? (
                      <Loader2 size={14} style={{ animation: 'spin 1.5s linear infinite' }} />
                    ) : p.status === 'connected' ? (
                      <><Settings size={14} /> Configure</>
                    ) : (
                      <><Plus size={14} /> Connect</>
                    )}
                  </button>
                  {p.status === 'connected' && (
                    <button className="btn btn-sm btn-danger" disabled={mutatingId === p.id} onClick={() => dispatch(disconnectProvider(p.id))}>
                      <Unlink size={14} /> Disconnect
                    </button>
                  )}
                </div>
              )}
            </div>
          ))}
        </div>
      </div>
    </div>
  );
}
















//Providers.module.scss
@use '../../styles/_variables' as *;

.providers {
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
  &__icon {
    background: $indigo-pale; color: $indigo; font-size: 18px; font-weight: 700;
    img { width: 24px; height: 24px; object-fit: contain; }
  }
  &__key-form { display: flex; gap: 8px; margin-top: 4px; }
  &__key-actions { display: flex; gap: 6px; }
}
