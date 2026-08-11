//Providers.tsx
import { useEffect, useState } from 'react';
import { Search, Check, Plus, Settings, Unlink, Loader2, Cable, Trash2, RefreshCw, Eye, ListPlus, ListFilter } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import {
  fetchProviders,
  createProvider,
  deleteProvider,
  connectProvider,
  disconnectProvider,
  syncModels,
} from '../../store/slices/providersSlice';
import { fetchModelsByProvider, createCustomModel } from '../../store/slices/modelsSlice';
import AddProviderDrawer from './AddProviderDrawer';
import AddCustomModelDrawer from './AddCustomModelDrawer';
import ProviderModelsSidebar from './ProviderModelsSidebar';
import { SkeletonCards } from '../common/Skeleton';
import styles from './Providers.module.scss';
import type { Provider } from '../../types';

type Filter = 'all' | 'connected' | 'available';

export default function Providers() {
  const dispatch = useAppDispatch();
  const { items, status, mutatingId, creating, syncingId } = useAppSelector((s) => s.providers);
  const modelsByProvider = useAppSelector((s) => s.models.byProvider);
  const modelsByProviderStatus = useAppSelector((s) => s.models.byProviderStatus);
  const customModelCreating = useAppSelector((s) => s.models.creating);
  const [search, setSearch] = useState('');
  const [filter, setFilter] = useState<Filter>('all');
  const [keyPromptFor, setKeyPromptFor] = useState<string | null>(null);
  const [apiKeyInput, setApiKeyInput] = useState('');
  const [drawerOpen, setDrawerOpen] = useState(false);
  const [addModelOpen, setAddModelOpen] = useState(false);
  const [viewModelsProvider, setViewModelsProvider] = useState<Provider | null>(null);

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
      <div className="pg-toolbar">
        <div className="toolbar">
          <div className="search-box">
            <Search size={16} color="var(--text-muted)" />
            <input placeholder="Search providers…" value={search} onChange={(e) => setSearch(e.target.value)} />
          </div>
          <div style={{ display: 'flex', gap: 14, alignItems: 'center' }}>
            <div className={styles['providers__filter-group']}>
              <span className={styles['providers__toolbar-label']}>
                <ListFilter size={11} /> Status
              </span>
              <div className="pills">
                {(['all', 'connected', 'available'] as Filter[]).map((f) => (
                  <button key={f} className={`pill ${filter === f ? 'on' : ''}`} onClick={() => setFilter(f)}>
                    {f[0].toUpperCase() + f.slice(1)}
                  </button>
                ))}
              </div>
            </div>
            <span className={styles['providers__toolbar-divider']} />
            <button className="btn btn-ind btn-sm" onClick={() => setDrawerOpen(true)}><Plus size={14} /> Add Provider</button>
          </div>
        </div>
      </div>
      <div className="pg-body">
        <div className="cards-grid">
          {status === 'loading' && <SkeletonCards count={6} />}
          {status !== 'loading' && filtered.map((p) => {
            const isCustom = p.name === 'Custom';
            return (
              <div className="card" key={p.id} style={{ display: 'flex', flexDirection: 'column', height: '100%' }}>
                <div className="card-hdr">
                  <div style={{ display: 'flex', alignItems: 'center', gap: 14 }}>
                    <div className={`card-icon ${styles['providers__icon']}`}>{p.logo_url ? <img src={p.logo_url} alt={p.name} /> : p.name[0]}</div>
                    <div>
                      <div className="card-title">{p.name}</div>
                      <div style={{ fontSize: 12, color: 'var(--text-secondary)', marginTop: 2 }}>{p.model_count} models</div>
                    </div>
                  </div>
                  <div style={{ display: 'flex', alignItems: 'center', gap: 6 }}>
                    <button
                      className={`btn btn-sm btn-ghost ${styles['providers__icon-btn']}`}
                      onClick={() => openModelsSidebar(p)}
                      title="View models"
                      aria-label={`View models for ${p.name}`}
                    >
                      <Eye size={14} />
                    </button>
                    {p.status === 'connected' ? (
                      <span className="badge badge-green"><Check size={11} /> Connected</span>
                    ) : (
                      <span className={styles['providers__badge-idle']}>Not connected</span>
                    )}
                  </div>
                </div>
                <div className="card-desc" style={{ flex: 1 }}>{p.description}</div>

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
                  <div className={styles['providers__foot-actions']}>
                    {isCustom && (
                      <button
                        className={`btn btn-sm btn-ind ${styles['providers__foot-btn']}`}
                        onClick={() => setAddModelOpen(true)}
                      >
                        <ListPlus size={13} /> Add Model
                      </button>
                    )}
                    <button
                      className={`btn btn-sm ${p.status === 'connected' ? 'btn-ghost' : 'btn-ind'} ${styles['providers__foot-btn']}`}
                      disabled={mutatingId === p.id}
                      onClick={() => setKeyPromptFor(p.id)}
                    >
                      {mutatingId === p.id ? (
                        <Loader2 size={13} style={{ animation: 'spin 1.5s linear infinite' }} />
                      ) : p.status === 'connected' ? (
                        <><Settings size={13} /> Configure</>
                      ) : (
                        <><Plus size={13} /> Connect</>
                      )}
                    </button>
                    {p.status === 'connected' && (
                      <>
                        <button
                          className={`btn btn-sm btn-ghost ${styles['providers__foot-btn']}`}
                          disabled={syncingId === p.id}
                          onClick={() => dispatch(syncModels(p.id))}
                        >
                          {syncingId === p.id ? (
                            <Loader2 size={13} style={{ animation: 'spin 1.5s linear infinite' }} />
                          ) : (
                            <><RefreshCw size={13} /> Sync</>
                          )}
                        </button>
                        <button
                          className={`btn btn-sm btn-danger ${styles['providers__foot-btn']}`}
                          disabled={mutatingId === p.id}
                          onClick={() => dispatch(disconnectProvider(p.id))}
                        >
                          <Unlink size={13} /> Disconnect
                        </button>
                      </>
                    )}
                    {p.status !== 'connected' && (
                      <button
                        className={`btn btn-sm btn-danger ${styles['providers__foot-btn']}`}
                        disabled={mutatingId === p.id}
                        onClick={() => {
                          if (window.confirm(`Delete ${p.name}? This cannot be undone.`)) {
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
          submitting={customModelCreating}
          onClose={() => setAddModelOpen(false)}
          onSubmit={(payload) => {
            dispatch(createCustomModel(payload)).then(() => {
              setAddModelOpen(false);
              dispatch(fetchProviders());
              dispatch(fetchModelsByProvider(customProvider.id));
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
        />
      )}
    </div>
  );
}





















//providers.module.scss
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

  &__filter-group {
    display: inline-flex;
    align-items: center;
    gap: 10px;
    padding: 5px 12px 5px 5px;
    background: $surface-alt;
    border: 1px solid $border-light;
    border-radius: 999px;
  }

  &__toolbar-label {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 5px 10px;
    border-radius: 999px;
    background: $indigo-pale;
    color: $indigo;
    font-size: 0.6875rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    white-space: nowrap;
  }

  &__toolbar-divider {
    flex-shrink: 0;
    width: 1px;
    height: 28px;
    background: linear-gradient(to bottom, transparent, $border-light 15%, $border-light 85%, transparent);
  }

  &__badge-idle {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 4px 11px;
    border-radius: 999px;
    font-size: 0.6875rem;
    font-weight: 600;
    color: $text-muted;
    background: transparent;
    border: 1px dashed $border-light;
    white-space: nowrap;

    &::before {
      content: '';
      flex-shrink: 0;
      width: 6px;
      height: 6px;
      border-radius: 50%;
      background: $text-muted;
      opacity: 0.6;
    }
  }

  &__icon-btn {
    padding: 6px !important;
    min-width: auto;
    border-radius: 8px;
  }

  // Action row at the bottom of each provider card. Wraps so buttons never
  // overflow the card, with tight, even spacing between them.
  &__foot-actions {
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 6px;
    margin-top: 12px;
  }

  &__foot-btn {
    flex: 0 0 auto;
    padding: 6px 10px !important;
    font-size: 12.5px !important;
    line-height: 1.2;
    gap: 4px !important;
    white-space: nowrap;
  }

  // --- Provider models sidebar ---
  &__sidebar-overlay {
    position: fixed;
    inset: 0;
    background: rgba(15, 18, 26, 0.45);
    z-index: 40;
  }

  &__sidebar {
    position: fixed;
    top: 0;
    right: 0;
    bottom: 0;
    width: min(420px, 100vw);
    background: var(--surface, #fff);
    background: var(--surface, #fff);
    border-left: 1px solid $border-light;
    box-shadow: -12px 0 32px rgba(15, 18, 26, 0.12);
    z-index: 41;
    display: flex;
    flex-direction: column;
    animation: providers-sidebar-in 0.2s ease-out;
  }

  &__sidebar-header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 12px;
    padding: 20px 20px 16px;
    border-bottom: 1px solid $border-light;
  }

  &__sidebar-title {
    font-family: $font-display;
    font-size: 1.05rem;
    font-weight: 800;
    color: $text-primary;
  }

  &__sidebar-subtitle {
    margin-top: 3px;
    font-size: 0.75rem;
    color: $text-secondary;
  }

  &__sidebar-body {
    flex: 1;
    overflow-y: auto;
    padding: 16px 20px 24px;
    display: flex;
    flex-direction: column;
    gap: 12px;
  }

  &__sidebar-empty {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 8px;
    padding: 48px 12px;
    color: $text-secondary;
    font-size: 13px;
    text-align: center;
  }

  &__model-row {
    border: 1px solid $border-light;
    border-radius: 12px;
    padding: 14px;
    background: $surface-alt;
  }

  &__model-row-head {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
  }

  &__model-row-name {
    font-weight: 700;
    font-size: 0.875rem;
    color: $text-primary;
  }

  &__model-row-tags {
    margin-top: 8px;
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
  }

  &__model-row-meta {
    margin-top: 10px;
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 8px 12px;

    > div {
      display: flex;
      flex-direction: column;
      gap: 2px;
      font-size: 0.8125rem;
      color: $text-primary;
    }
  }

  &__model-row-meta-label {
    font-size: 0.6875rem;
    text-transform: uppercase;
    letter-spacing: 0.06em;
    color: $text-secondary;
  }

  &__model-row-url {
    margin-top: 10px;
    padding-top: 10px;
    border-top: 1px dashed $border-light;
    font-family: $font-mono;
    font-size: 0.75rem;
    color: $text-secondary;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }
}

@keyframes providers-sidebar-in {
  from { transform: translateX(100%); }
  to { transform: translateX(0); }
}



















//Modelcatalog.tsx
import { useEffect, useMemo, useState } from 'react';
import { Search, Boxes, ChevronUp, ChevronDown, ChevronsUpDown, ChevronLeft, ChevronRight, ChevronsLeft, ChevronsRight, ListFilter } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchModels } from '../../store/slices/modelsSlice';
import { fetchProviders } from '../../store/slices/providersSlice';
import { SkeletonTableRows } from '../common/Skeleton';
import type { Model } from '../../types';
import styles from './ModelCatalog.module.scss';

type SortKey = 'name' | 'provider' | 'context_window' | 'price' | 'accuracy' | 'status';
type SortDir = 'asc' | 'desc';

const PAGE_SIZE_OPTIONS = [10, 25, 50, 100];

// Builds a compact page-number list with ellipses, e.g. [1, '…', 4, 5, 6, '…', 12]
function buildPageList(current: number, total: number): (number | '…')[] {
  if (total <= 7) return Array.from({ length: total }, (_, i) => i + 1);
  const pages = new Set<number>([1, total, current, current - 1, current + 1]);
  const sorted = [...pages].filter((p) => p >= 1 && p <= total).sort((a, b) => a - b);
  const result: (number | '…')[] = [];
  let prev = 0;
  for (const p of sorted) {
    if (prev && p - prev > 1) result.push('…');
    result.push(p);
    prev = p;
  }
  return result;
}

interface SortableThProps {
  label: string;
  sortKey: SortKey;
  activeKey: SortKey;
  dir: SortDir;
  onSort: (key: SortKey) => void;
}

function SortableTh({ label, sortKey, activeKey, dir, onSort }: SortableThProps) {
  const active = activeKey === sortKey;
  return (
    <th className={styles['model-catalog__sortable-th']}>
      <button
        type="button"
        className={`${styles['model-catalog__sort-btn']} ${active ? styles['model-catalog__sort-btn--active'] : ''}`}
        onClick={() => onSort(sortKey)}
      >
        {label}
        {active ? (
          dir === 'asc' ? <ChevronUp size={13} /> : <ChevronDown size={13} />
        ) : (
          <ChevronsUpDown size={13} className={styles['model-catalog__sort-icon-idle']} />
        )}
      </button>
    </th>
  );
}

export default function ModelCatalog() {
  const dispatch = useAppDispatch();
  const { items, status } = useAppSelector((s) => s.models);
  const providers = useAppSelector((s) => s.providers.items);
  const [search, setSearch] = useState('');
  const [capFilter, setCapFilter] = useState('All');
  const [sortKey, setSortKey] = useState<SortKey>('name');
  const [sortDir, setSortDir] = useState<SortDir>('asc');
  const [page, setPage] = useState(1);
  const [pageSize, setPageSize] = useState(10);

  useEffect(() => {
    dispatch(fetchModels());
    dispatch(fetchProviders());
  }, [dispatch]);

  const caps = useMemo(() => ['All', ...new Set(items.flatMap((m) => m.capabilities))], [items]);
  const providerName = (id: string) => providers.find((p) => p.id === id)?.name || id;

  const filtered = useMemo(() => {
    return items.filter((m) => {
      if (capFilter !== 'All' && !m.capabilities.includes(capFilter)) return false;
      const q = search.toLowerCase();
      return !q || m.name.toLowerCase().includes(q) || providerName(m.provider_id).toLowerCase().includes(q);
    });
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [items, providers, search, capFilter]);

  const sorted = useMemo(() => {
    const dir = sortDir === 'asc' ? 1 : -1;
    const compare = (a: Model, b: Model): number => {
      switch (sortKey) {
        case 'name':
          return a.name.localeCompare(b.name) * dir;
        case 'provider':
          return providerName(a.provider_id).localeCompare(providerName(b.provider_id)) * dir;
        case 'context_window':
          return (a.context_window - b.context_window) * dir;
        case 'price':
          return ((a.input_price ?? -1) - (b.input_price ?? -1)) * dir;
        case 'accuracy':
          return ((a.accuracy_score ?? -1) - (b.accuracy_score ?? -1)) * dir;
        case 'status':
          return (Number(a.is_active) - Number(b.is_active)) * dir;
        default:
          return 0;
      }
    };
    return [...filtered].sort(compare);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [filtered, sortKey, sortDir, providers]);

  const total = sorted.length;
  const totalPages = Math.max(1, Math.ceil(total / pageSize));
  const safePage = Math.min(page, totalPages);
  const startIdx = (safePage - 1) * pageSize;
  const pageItems = sorted.slice(startIdx, startIdx + pageSize);
  const pageList = useMemo(() => buildPageList(safePage, totalPages), [safePage, totalPages]);

  useEffect(() => {
    setPage(1);
  }, [search, capFilter, pageSize]);

  const toggleSort = (key: SortKey) => {
    if (sortKey === key) {
      setSortDir((d) => (d === 'asc' ? 'desc' : 'asc'));
    } else {
      setSortKey(key);
      setSortDir('asc');
    }
  };

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
            <Search size={16} color="var(--text-muted)" />
            <input placeholder="Search models or providers…" value={search} onChange={(e) => setSearch(e.target.value)} />
          </div>
          <div style={{ display: 'flex', gap: 8, alignItems: 'center' }}>
            <div className={styles['model-catalog__filter-group']}>
              <span className={styles['model-catalog__toolbar-label']}>
                <ListFilter size={11} /> Capability
              </span>
              <div className="pills">{caps.map((c) => <button key={c} className={`pill ${capFilter === c ? 'on' : ''}`} onClick={() => setCapFilter(c)}>{c}</button>)}</div>
            </div>
          </div>
        </div>
      </div>
      <div className="pg-body">
        <div className="tw">
          <table className="tbl">
            <thead>
              <tr>
                <SortableTh label="Model" sortKey="name" activeKey={sortKey} dir={sortDir} onSort={toggleSort} />
                <SortableTh label="Provider" sortKey="provider" activeKey={sortKey} dir={sortDir} onSort={toggleSort} />
                <th>Capabilities</th>
                <SortableTh label="Context" sortKey="context_window" activeKey={sortKey} dir={sortDir} onSort={toggleSort} />
                <SortableTh label="Price (in/out)" sortKey="price" activeKey={sortKey} dir={sortDir} onSort={toggleSort} />
                <SortableTh label="Accuracy" sortKey="accuracy" activeKey={sortKey} dir={sortDir} onSort={toggleSort} />
                <SortableTh label="Status" sortKey="status" activeKey={sortKey} dir={sortDir} onSort={toggleSort} />
              </tr>
            </thead>
            <tbody>
              {status === 'loading' && <SkeletonTableRows columns={7} rows={6} />}
              {status !== 'loading' && pageItems.map((m) => (
                <tr key={m.id}>
                  <td style={{ fontWeight: 700 }}>{m.name}</td>
                  <td style={{ color: 'var(--text-secondary)' }}>{providerName(m.provider_id)}</td>
                  <td>{m.capabilities.map((c) => <span key={c} className="tag tag-ind">{c}</span>)}</td>
                  <td style={{ fontFamily: "'Segoe UI', Roboto, Arial, sans-serif", fontSize: 13 }}>{m.context_window.toLocaleString()}</td>
                  <td style={{ fontFamily: "'Segoe UI', Roboto, Arial, sans-serif", fontSize: 13, color: 'var(--text-secondary)' }}>
                    {m.input_price != null ? `$${m.input_price.toFixed(2)}` : '—'} / {m.output_price != null ? `$${m.output_price.toFixed(2)}` : '—'}
                  </td>
                  <td>
                    <span style={{ fontFamily: "'Segoe UI', Roboto, Arial, sans-serif", fontWeight: 700, color: (m.accuracy_score || 0) >= 90 ? '#10B981' : 'var(--text-primary)' }}>
                      {m.accuracy_score != null ? `${m.accuracy_score}%` : '—'}
                    </span>
                  </td>
                  <td><span className={`badge ${m.is_active ? 'badge-green' : 'badge-gray'}`}>{m.is_active ? 'Active' : 'Inactive'}</span></td>
                </tr>
              ))}
              {status !== 'loading' && pageItems.length === 0 && (
                <tr>
                  <td colSpan={7} className={styles['model-catalog__empty']}>No models match your filters.</td>
                </tr>
              )}
            </tbody>
          </table>

          {status !== 'loading' && total > 0 && (
            <div className={styles['model-catalog__pagination']}>
              <div className={styles['model-catalog__pagination-info']}>
                <span>
                  Showing <strong>{startIdx + 1}–{Math.min(startIdx + pageSize, total)}</strong> of <strong>{total}</strong> model{total === 1 ? '' : 's'}
                </span>
                <div className={styles['model-catalog__page-size']}>
                  <label htmlFor="model-catalog-page-size">Rows per page</label>
                  <select
                    id="model-catalog-page-size"
                    value={pageSize}
                    onChange={(e) => setPageSize(Number(e.target.value))}
                  >
                    {PAGE_SIZE_OPTIONS.map((n) => (
                      <option key={n} value={n}>{n}</option>
                    ))}
                  </select>
                </div>
              </div>

              <div className={styles['model-catalog__pager']}>
                <button
                  className={styles['model-catalog__page-btn']}
                  disabled={safePage === 1}
                  onClick={() => setPage(1)}
                  aria-label="First page"
                >
                  <ChevronsLeft size={14} />
                </button>
                <button
                  className={styles['model-catalog__page-btn']}
                  disabled={safePage === 1}
                  onClick={() => setPage((p) => Math.max(1, p - 1))}
                  aria-label="Previous page"
                >
                  <ChevronLeft size={14} />
                </button>

                {pageList.map((p, i) =>
                  p === '…' ? (
                    <span key={`dots-${i}`} className={styles['model-catalog__page-dots']}>…</span>
                  ) : (
                    <button
                      key={p}
                      className={`${styles['model-catalog__page-btn']} ${styles['model-catalog__page-btn--num']} ${p === safePage ? styles['model-catalog__page-btn--active'] : ''}`}
                      onClick={() => setPage(p)}
                      aria-current={p === safePage ? 'page' : undefined}
                    >
                      {p}
                    </button>
                  )
                )}

                <button
                  className={styles['model-catalog__page-btn']}
                  disabled={safePage === totalPages}
                  onClick={() => setPage((p) => Math.min(totalPages, p + 1))}
                  aria-label="Next page"
                >
                  <ChevronRight size={14} />
                </button>
                <button
                  className={styles['model-catalog__page-btn']}
                  disabled={safePage === totalPages}
                  onClick={() => setPage(totalPages)}
                  aria-label="Last page"
                >
                  <ChevronsRight size={14} />
                </button>
              </div>
            </div>
          )}
        </div>
      </div>
    </div>
  );
}























//Modelcatalog.module.scss
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

  &__filter-group {
    display: inline-flex;
    align-items: center;
    gap: 10px;
    padding: 5px 12px 5px 5px;
    background: $surface-alt;
    border: 1px solid $border-light;
    border-radius: 999px;
  }

  &__toolbar-label {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 5px 10px;
    border-radius: 999px;
    background: $indigo-pale;
    color: $indigo;
    font-size: 0.6875rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    white-space: nowrap;
  }

  // --- Sortable column headers ---
  &__sortable-th {
    padding: 0 !important;
  }

  &__sort-btn {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    width: 100%;
    padding: 12px 16px;
    background: none;
    border: none;
    cursor: pointer;
    font: inherit;
    font-weight: 700;
    font-size: inherit;
    color: inherit;
    text-align: left;

    &:hover {
      color: $indigo;

      .model-catalog__sort-icon-idle { opacity: 0.7; }
    }

    &--active {
      color: $indigo;
    }
  }

  &__sort-icon-idle {
    opacity: 0.28;
    transition: opacity 0.15s ease;
  }

  &__empty {
    text-align: center;
    padding: 40px 16px !important;
    color: $text-secondary;
    font-size: 0.875rem;
  }

  // --- Pagination bar ---
  &__pagination {
    display: flex;
    align-items: center;
    justify-content: space-between;
    flex-wrap: wrap;
    gap: 12px;
    padding: 14px 20px;
    border-top: 1px solid $border-light;
    background: $surface-alt;
  }

  &__pagination-info {
    display: flex;
    align-items: center;
    flex-wrap: wrap;
    gap: 18px;
    font-size: 0.8125rem;
    color: $text-secondary;

    strong { color: $text-primary; font-weight: 700; }
  }

  &__page-size {
    display: flex;
    align-items: center;
    gap: 8px;

    label {
      font-size: 0.75rem;
      color: $text-secondary;
      white-space: nowrap;
    }

    select {
      appearance: none;
      -webkit-appearance: none;
      font: inherit;
      font-size: 0.8125rem;
      font-weight: 600;
      color: $text-primary;
      background: $surface;
      border: 1px solid $border-light;
      border-radius: 8px;
      padding: 5px 26px 5px 10px;
      cursor: pointer;
      background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='10' height='6' viewBox='0 0 10 6' fill='none'%3E%3Cpath d='M1 1L5 5L9 1' stroke='%236B7280' stroke-width='1.5' stroke-linecap='round' stroke-linejoin='round'/%3E%3C/svg%3E");
      background-repeat: no-repeat;
      background-position: right 10px center;

      &:hover { border-color: $indigo; }
      &:focus { outline: none; border-color: $indigo; box-shadow: 0 0 0 3px $indigo-pale; }
    }
  }

  &__pager {
    display: flex;
    align-items: center;
    gap: 4px;
  }

  &__page-btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    min-width: 30px;
    height: 30px;
    padding: 0 6px;
    border-radius: 8px;
    border: 1px solid transparent;
    background: transparent;
    color: $text-secondary;
    font-size: 0.8125rem;
    font-weight: 600;
    cursor: pointer;
    transition: background 0.12s ease, color 0.12s ease, border-color 0.12s ease;

    &:hover:not(:disabled) {
      background: $indigo-pale;
      color: $indigo;
    }

    &:disabled {
      opacity: 0.35;
      cursor: not-allowed;
    }

    &--num {
      min-width: 30px;
    }

    &--active {
      background: $indigo;
      color: #fff;

      &:hover:not(:disabled) {
        background: $indigo;
        color: #fff;
      }
    }
  }

  &__page-dots {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    min-width: 20px;
    height: 30px;
    color: $text-muted;
    font-size: 0.8125rem;
  }
}
