//package.json
{
  "name": "poc",
  "private": true,
  "version": "0.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "dev:mock": "cross-env VITE_ENABLE_MOCKS=true vite",
    "build": "tsc -b && vite build",
    "lint": "eslint .",
    "preview": "vite preview",
    "mocks:setup": "msw init public --save",
    "postinstall": "npm run mocks:setup"
  },
  "dependencies": {
    "@reduxjs/toolkit": "^2.12.0",
    "axios": "^1.19.0",
    "lucide-react": "^1.30.0",
    "react": "^19.2.8",
    "react-dom": "^19.2.8",
    "react-redux": "^9.3.0",
    "react-router-dom": "^7.18.2"
  },
  "devDependencies": {
    "@babel/core": "^7.29.7",
    "@eslint/js": "^10.0.1",
    "@rolldown/plugin-babel": "^0.2.3",
    "@types/babel__core": "^7.20.5",
    "@types/node": "^24.13.3",
    "@types/react": "^19.2.17",
    "@types/react-dom": "^19.2.3",
    "@vitejs/plugin-react": "^6.0.4",
    "babel-plugin-react-compiler": "^1.0.0",
    "cross-env": "^7.0.3",
    "eslint": "^10.8.0",
    "eslint-plugin-react-hooks": "^7.1.1",
    "eslint-plugin-react-refresh": "^0.5.3",
    "globals": "^17.7.0",
    "msw": "^2.15.0",
    "sass": "^1.102.0",
    "typescript": "~6.0.2",
    "typescript-eslint": "^8.65.0",
    "vite": "^8.2.0"
  }
}


















//App.tsx
import { BrowserRouter } from 'react-router-dom';
import { Provider } from 'react-redux';
import { store } from './store/store';
import AppRoutes from './routes/AppRoutes';
import './styles/global.scss';

export default function App() {
  return (
    <Provider store={store}>
      <BrowserRouter>
        <AppRoutes />
      </BrowserRouter>
    </Provider>
  );
}














//main.tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';
import { store } from './store/store';
import { hydrateMockSession } from './store/slices/authSlice';
import { tokenStorage } from './api/axiosInstance';

const MOCKS_ENABLED = import.meta.env.VITE_ENABLE_MOCKS === 'true';

async function enableMocking() {
  if (!MOCKS_ENABLED) return;
  const { worker } = await import('./mocks/browser');
  await worker.start({
    onUnhandledRequest: 'bypass',
    serviceWorker: { url: '/mockServiceWorker.js' },
  });
}

// Bypass SSO entirely in mock mode: seed a fake session/token before the app
// renders so ProtectedRoute never redirects to the landing page and every
// axios call already carries a bearer token. Never runs against a real backend.
function bypassAuthForMocks() {
  if (!MOCKS_ENABLED) return;
  const mockUser = {
    token: 'mock-session-token',
    username: 'jane.doe',
    email: 'jane@semco.ai',
    language: 'en',
    profile_name: 'Jane Doe',
  };
  tokenStorage.set(mockUser.token);
  store.dispatch(hydrateMockSession(mockUser));
}

bypassAuthForMocks();

enableMocking()
  .catch((err) => {
    // Never let a broken/missing mock service worker leave the app on a
    // blank screen — log it and continue rendering. Common cause: run
    // `npm run mocks:setup` (or `npx msw init public --save`) to
    // (re)generate public/mockServiceWorker.js after a fresh install.
    console.error(
      '[mocks] Failed to start MSW — continuing without request mocking. ' +
        'Run `npm run mocks:setup` to regenerate public/mockServiceWorker.js.',
      err
    );
  })
  .finally(() => {
    ReactDOM.createRoot(document.getElementById('root') as HTMLElement).render(
      <React.StrictMode>
        <App />
      </React.StrictMode>
    );
  });
























//axiosInstance.ts
import axios from 'axios';
import type { AxiosError, InternalAxiosRequestConfig } from 'axios';
import { store } from '../store/store';
import { logout } from '../store/slices/authSlice';

const TOKEN_STORAGE_KEY = 'semcoeval_token';

export const tokenStorage = {
  get: (): string | null => localStorage.getItem(TOKEN_STORAGE_KEY),
  set: (token: string): void => localStorage.setItem(TOKEN_STORAGE_KEY, token),
  clear: (): void => localStorage.removeItem(TOKEN_STORAGE_KEY),
};

export const api = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL || '/api',
  timeout: 20000,
  headers: {
    'Content-Type': 'application/json',
  },
});

// ---- Request interceptor: attach bearer token ----
api.interceptors.request.use(
  (config: InternalAxiosRequestConfig) => {
    const token = tokenStorage.get();
    if (token) {
      config.headers = config.headers ?? {};
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error: AxiosError) => Promise.reject(error)
);

// ---- Response interceptor: normalize errors + handle 401 ----
let isLoggingOut = false;

api.interceptors.response.use(
  (response) => response,
  (error: AxiosError<{ message?: string }>) => {
    const status = error.response?.status;

    if (status === 401 && !isLoggingOut) {
      isLoggingOut = true;
      tokenStorage.clear();
      store.dispatch(logout());
      isLoggingOut = false;
    }

    const message =
      error.response?.data?.message ||
      error.message ||
      'Something went wrong while talking to the server.';

    return Promise.reject({ ...error, message });
  }
);

export default api;




















//auth.ts
import api from '../axiosInstance';
import type { SsoLoginRequest, SsoLoginResponse } from '../../types';

export const authApi = {
  ssoLogin: (payload: SsoLoginRequest) =>
    api.post<SsoLoginResponse>('/sso_login', payload).then((r) => r.data),
};

















//benchmarks.ts
import api from '../axiosInstance';
import type { BenchmarksResponse } from '../../types';

export const benchmarksApi = {
  list: () => api.get<BenchmarksResponse>('/benchmarks').then((r) => r.data),
};













//evaluations.ts
import api from '../axiosInstance';
import type {
  CreateEvaluationRequest,
  CreateEvaluationResponse,
  EvaluationStatusResponse,
} from '../../types';

export const evaluationsApi = {
  create: (payload: CreateEvaluationRequest) =>
    api.post<CreateEvaluationResponse>('/evaluations', payload).then((r) => r.data),

  start: (evaluationId: string) =>
    api.post<void>(`/evaluations/${evaluationId}/start`).then(() => undefined),

  status: (evaluationId: string) =>
    api
      .get<EvaluationStatusResponse>(`/evaluations/${evaluationId}/status`)
      .then((r) => r.data),

  // Convenience helper used by the wizard: create then immediately start.
  createAndStart: async (payload: CreateEvaluationRequest) => {
    const created = await evaluationsApi.create(payload);
    const id = created.id || created.evaluation_id;
    if (!id) {
      throw new Error('Evaluation was created but no id was returned by the server.');
    }
    await evaluationsApi.start(id);
    return id;
  },
};

// Polling helper described in the API reference notes.
export function pollEvaluationStatus(
  evaluationId: string,
  onUpdate: (status: EvaluationStatusResponse) => void,
  intervalMs = 3000
): () => void {
  let cancelled = false;

  const tick = async () => {
    if (cancelled) return;
    try {
      const status = await evaluationsApi.status(evaluationId);
      onUpdate(status);
      if (status.status === 'completed' || status.status === 'failed' || status.status === 'canceled') {
        return;
      }
    } catch {
      // swallow transient polling errors; interceptor already normalizes them
    }
    if (!cancelled) setTimeout(tick, intervalMs);
  };

  tick();
  return () => {
    cancelled = true;
  };
}











//metrics.ts
import api from '../axiosInstance';
import type { MetricsResponse } from '../../types';

export const metricsApi = {
  list: () => api.get<MetricsResponse>('/metrics').then((r) => r.data),
};


















//models.ts
import api from '../axiosInstance';
import type { Model, CustomModelRequest } from '../../types';

export const modelsApi = {
  list: () => api.get<{ models: Model[] }>('/models').then((r) => r.data.models),

  createCustom: (payload: CustomModelRequest) =>
    api.post<void>('/models/custom', payload).then(() => undefined),
};










//providers.ts
import api from '../axiosInstance';
import type {
  Provider,
  ConnectProviderRequest,
  ConnectProviderResponse,
  DisconnectProviderResponse,
} from '../../types';

export const providersApi = {
  list: () => api.get<{ providers: Provider[] }>('/providers').then((r) => r.data.providers),

  connect: (providerId: string, payload: ConnectProviderRequest) =>
    api
      .post<ConnectProviderResponse>(`/providers/${providerId}/connect`, payload)
      .then((r) => r.data),

  disconnect: (providerId: string) =>
    api
      .delete<DisconnectProviderResponse>(`/providers/${providerId}/disconnect`)
      .then((r) => r.data),
};













//HBar.tsx
import { useEffect, useState } from 'react';

interface HBarProps {
  value: number;
  max?: number;
  color?: string;
  label: string;
  sublabel?: string;
}

export default function HBar({ value, max = 100, color = '#6366F1', label, sublabel }: HBarProps) {
  const [width, setWidth] = useState(0);
  useEffect(() => {
    const t = setTimeout(() => setWidth(value), 50);
    return () => clearTimeout(t);
  }, [value]);

  return (
    <div style={{ marginBottom: 14 }}>
      <div style={{ display: 'flex', justifyContent: 'space-between', marginBottom: 5 }}>
        <span style={{ fontSize: 13, fontWeight: 600 }}>{label}</span>
        <span style={{ fontFamily: "'JetBrains Mono', monospace", fontSize: 13, fontWeight: 700, color }}>{value}%</span>
      </div>
      {sublabel && <div style={{ fontSize: 11, color: '#9CA3AF', marginBottom: 4 }}>{sublabel}</div>}
      <div style={{ height: 8, background: '#F1F4F9', borderRadius: 4, overflow: 'hidden' }}>
        <div
          style={{
            height: '100%',
            width: `${(width / max) * 100}%`,
            background: color,
            borderRadius: 4,
            transition: 'width 1s cubic-bezier(.16,1,.3,1)',
          }}
        />
      </div>
    </div>
  );
}











//RadarChart.tsx
import { useState } from 'react';

export interface RadarModel {
  name: string;
  values: number[]; // 0..1, one per metric, same order as `metrics`
}

interface RadarChartProps {
  models: RadarModel[];
  metrics?: string[];
  size?: number;
  colors?: string[];
}

const DEFAULT_METRICS = ['Accuracy', 'Speed', 'Cost Eff.', 'Context', 'Capability'];
const DEFAULT_COLORS = ['#6366F1', '#F59E0B', '#10B981'];

export default function RadarChart({ models, metrics = DEFAULT_METRICS, size = 300, colors = DEFAULT_COLORS }: RadarChartProps) {
  const [hovered, setHovered] = useState<number | null>(null);
  const cx = size / 2;
  const cy = size / 2;
  const r = size / 2 - 44;
  const angleStep = (2 * Math.PI) / metrics.length;
  const pt = (angle: number, radius: number) => ({ x: cx + radius * Math.sin(angle), y: cy - radius * Math.cos(angle) });

  return (
    <svg width={size} height={size} viewBox={`0 0 ${size} ${size}`} style={{ overflow: 'visible' }}>
      {[0.2, 0.4, 0.6, 0.8, 1].map((l, i) => (
        <polygon
          key={i}
          points={metrics.map((_, mi) => { const p = pt(mi * angleStep, r * l); return `${p.x},${p.y}`; }).join(' ')}
          fill={i < 4 ? '#F1F4F9' : 'none'}
          stroke="#E5E7EB"
          strokeWidth={1}
          opacity={0.6}
        />
      ))}
      {metrics.map((m, i) => {
        const label = pt(i * angleStep, r + 24);
        const lineEnd = pt(i * angleStep, r);
        return (
          <g key={i}>
            <line x1={cx} y1={cy} x2={lineEnd.x} y2={lineEnd.y} stroke="#E5E7EB" strokeWidth={1} strokeDasharray="3,3" />
            <text x={label.x} y={label.y} textAnchor="middle" dominantBaseline="middle" fill="#6B7280" fontSize={11} fontWeight={600} fontFamily="Space Grotesk, sans-serif">
              {m}
            </text>
          </g>
        );
      })}
      {models.map((model, mi) => (
        <g key={mi} onMouseEnter={() => setHovered(mi)} onMouseLeave={() => setHovered(null)} style={{ cursor: 'pointer' }}>
          <polygon
            points={model.values.map((v, vi) => { const p = pt(vi * angleStep, r * v); return `${p.x},${p.y}`; }).join(' ')}
            fill={colors[mi]}
            fillOpacity={hovered === mi ? 0.18 : 0.08}
            stroke={colors[mi]}
            strokeWidth={hovered === mi ? 3 : 2}
            strokeLinejoin="round"
            style={{ transition: 'all .25s' }}
          />
          {model.values.map((v, vi) => {
            const p = pt(vi * angleStep, r * v);
            return <circle key={vi} cx={p.x} cy={p.y} r={hovered === mi ? 5 : 3.5} fill="#FFFFFF" stroke={colors[mi]} strokeWidth={2} style={{ transition: 'r .2s' }} />;
          })}
        </g>
      ))}
    </svg>
  );
}















//ScoreRing.module.scss
.score-ring {
  position: relative;
  flex-shrink: 0;

  &__progress { transition: stroke-dashoffset 1.2s cubic-bezier(.16, 1, .3, 1); }
  &__center {
    position: absolute; inset: 0; display: flex; flex-direction: column;
    align-items: center; justify-content: center;
  }
  &__value { font-family: 'JetBrains Mono', monospace; font-weight: 700; color: #111827; line-height: 1; }
  &__label { font-size: 9px; color: #9CA3AF; font-weight: 600; margin-top: 1px; }
}












//ScoreRing.tsx
import { useEffect, useState } from 'react';
import styles from './ScoreRing.module.scss';

interface ScoreRingProps {
  score: number;
  size?: number;
  stroke?: number;
  color?: string;
  label?: string;
}

export default function ScoreRing({ score, size = 72, stroke = 6, color = '#6366F1', label }: ScoreRingProps) {
  const [mounted, setMounted] = useState(false);
  useEffect(() => {
    const t = setTimeout(() => setMounted(true), 100);
    return () => clearTimeout(t);
  }, []);

  const r = (size - stroke) / 2;
  const c = 2 * Math.PI * r;
  const offset = c - (c * (mounted ? score : 0)) / 100;

  return (
    <div className={styles["score-ring"]} style={{ width: size, height: size }}>
      <svg width={size} height={size} style={{ transform: 'rotate(-90deg)' }}>
        <circle cx={size / 2} cy={size / 2} r={r} fill="none" stroke="#F3F4F6" strokeWidth={stroke} />
        <circle
          cx={size / 2}
          cy={size / 2}
          r={r}
          fill="none"
          stroke={color}
          strokeWidth={stroke}
          strokeDasharray={c}
          strokeDashoffset={offset}
          strokeLinecap="round"
          className={styles["score-ring__progress"]}
        />
      </svg>
      <div className={styles["score-ring__center"]}>
        <span className={styles["score-ring__value"]} style={{ fontSize: size * 0.22 }}>
          {mounted ? score : 0}
        </span>
        {label && <span className={styles["score-ring__label"]}>{label}</span>}
      </div>
    </div>
  );
}











//Sparkline.tsx
interface SparklineProps {
  data: number[];
  color?: string;
  width?: number;
  height?: number;
}

export default function Sparkline({ data, color = '#6366F1', width = 80, height = 28 }: SparklineProps) {
  const max = Math.max(...data);
  const min = Math.min(...data);
  const range = max - min || 1;
  const points = data
    .map((v, i) => `${(i / (data.length - 1)) * width},${height - ((v - min) / range) * height}`)
    .join(' ');
  const gradientId = 'sparkline-grad-' + color.replace('#', '');

  return (
    <svg width={width} height={height} style={{ overflow: 'visible' }}>
      <defs>
        <linearGradient id={gradientId} x1="0" y1="0" x2="0" y2="1">
          <stop offset="0" stopColor={color} stopOpacity=".2" />
          <stop offset="1" stopColor={color} stopOpacity="0" />
        </linearGradient>
      </defs>
      <polygon points={`0,${height} ${points} ${width},${height}`} fill={`url(#${gradientId})`} />
      <polyline points={points} fill="none" stroke={color} strokeWidth={2} strokeLinecap="round" strokeLinejoin="round" />
    </svg>
  );
}















//useCounter.ts
import { useEffect, useState } from 'react';

export function useCounter(end: number, duration = 1200, active = true): number {
  const [value, setValue] = useState(0);
  useEffect(() => {
    if (!active) return;
    const t0 = performance.now();
    let raf: number;
    const step = (now: number) => {
      const p = Math.min((now - t0) / duration, 1);
      setValue(Math.round(end * (1 - Math.pow(1 - p, 3))));
      if (p < 1) raf = requestAnimationFrame(step);
    };
    raf = requestAnimationFrame(step);
    return () => cancelAnimationFrame(raf);
  }, [end, duration, active]);
  return value;
}
















//Comparison.module.scss
@use '../../styles/_variables' as *;

.comparison {
  &__controls { display: flex; gap: 14px; margin-bottom: 24px; align-items: center; flex-wrap: wrap; }
  &__label { font-size: 13px; font-weight: 700; color: $text-secondary; }
  &__select { width: auto; padding: 8px 14px; cursor: pointer; border-radius: 10px; }
  &__grid { display: grid; grid-template-columns: 1fr 1fr; gap: 16px; margin-bottom: 24px; }
  &__panel-title { font-family: $font-display; font-weight: 700; font-size: 15px; }
  &__panel-sub { font-size: 13px; color: $text-secondary; margin-bottom: 8px; }
  &__legend { display: flex; justify-content: center; gap: 20px; margin-top: 4px; flex-wrap: wrap; }
  &__legend span { display: flex; align-items: center; gap: 6px; font-size: 12px; font-weight: 600; color: $text-secondary; }
  &__dot { width: 8px; height: 8px; border-radius: 50%; display: inline-block; }
  &__scores { display: grid; grid-template-columns: repeat(3, 1fr); gap: 32px; }
  &__score-item { display: flex; flex-direction: column; align-items: center; gap: 12px; }
}

.model-chip {
  display: inline-flex; align-items: center; gap: 6px; padding: 7px 16px; border-radius: 100px;
  font-size: 13px; font-weight: 700; border: 2px solid; cursor: pointer; transition: all .2s;
}
.model-chip:hover { transform: translateY(-1px); }
.model-chip__dot { width: 8px; height: 8px; border-radius: 50%; }

@media (max-width: 768px) {
  .comparison__grid { grid-template-columns: 1fr; }
  .comparison__scores { grid-template-columns: 1fr; }
}












//Comparison.tsx
import { useEffect, useMemo, useState } from 'react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchModels } from '../../store/slices/modelsSlice';
import { fetchBenchmarks } from '../../store/slices/benchmarksSlice';
import RadarChart from '../common/RadarChart';
import ScoreRing from '../common/ScoreRing';
import styles from './Comparison.module.scss';

const COLORS = ['#6366F1', '#F59E0B', '#10B981'];

export default function Comparison() {
  const dispatch = useAppDispatch();
  const models = useAppSelector((s) => s.models.items);
  const benchmarks = useAppSelector((s) => s.benchmarks.items);

  // Explicit user selections; null/empty means "no override yet", in which
  // case the memoized defaults below take over. This avoids writing state
  // from inside an effect just to seed a default.
  const [selSuiteOverride, setSelSuiteOverride] = useState<string | null>(null);
  // No model-picker UI yet — reserved for when users can swap which models
  // are compared; falls back to the first three models in the catalog.
  const [selModelIdsOverride] = useState<string[] | null>(null);

  useEffect(() => {
    dispatch(fetchModels());
    dispatch(fetchBenchmarks());
  }, [dispatch]);

  const selSuite = selSuiteOverride ?? benchmarks[0]?.name ?? '';
  const defaultModelIds = useMemo(() => models.slice(0, 3).map((m) => m.id), [models]);
  const selModelIds = selModelIdsOverride ?? defaultModelIds;

  const comp = selModelIds
    .map((id) => models.find((m) => m.id === id))
    .filter((m): m is NonNullable<typeof m> => Boolean(m))
    .map((m) => ({
      name: m.name,
      accuracy: m.accuracy_score ?? 0,
      speed: m.input_price ?? 0,
      cost: m.output_price ?? 0,
      ctx: m.context_window,
      values: [
        (m.accuracy_score || 0) / 100,
        0.75,
        m.input_price ? Math.max(0, 1 - m.input_price / 10) : 0.5,
        Math.min(1, m.context_window / 200000),
        (m.agent_score || m.accuracy_score || 0) / 100,
      ],
    }));

  const mets = [
    { k: 'accuracy' as const, l: 'Accuracy' },
    { k: 'ctx' as const, l: 'Context Window' },
    { k: 'speed' as const, l: 'Input Price' },
    { k: 'cost' as const, l: 'Output Price' },
  ];

  return (
    <div className="page-enter">
      <div className="pg-hdr"><h1>Model Comparison</h1><p>Side-by-side performance across a shared benchmark</p></div>
      <div className="pg-body">
        <div className={styles['comparison__controls']}>
          <span className={styles['comparison__label']}>Test Suite:</span>
          <select className={`fi ${styles['comparison__select']}`} value={selSuite} onChange={(e) => setSelSuiteOverride(e.target.value)}>
            {benchmarks.map((b) => <option key={b.name} value={b.name}>{b.name}</option>)}
          </select>
          <span className={styles['comparison__label']}>Comparing:</span>
          {comp.map((m, i) => (
            <span key={m.name} className={styles['model-chip']} style={{ borderColor: COLORS[i], color: COLORS[i], background: `${COLORS[i]}14` }}>
              <span className={styles['model-chip__dot']} style={{ background: COLORS[i] }} /> {m.name}
            </span>
          ))}
        </div>

        <div className={styles['comparison__grid']}>
          <div className="card">
            <div className={styles['comparison__panel-title']}>Strength Profile</div>
            <div className={styles['comparison__panel-sub']}>Multi-dimensional performance comparison</div>
            <div className="radar-wrap"><RadarChart models={comp} size={280} colors={COLORS} /></div>
            <div className={styles['comparison__legend']}>
              {comp.map((m, i) => <span key={i}><span className={styles['comparison__dot']} style={{ background: COLORS[i] }} /> {m.name}</span>)}
            </div>
          </div>

          <div className="card" style={{ padding: 0 }}>
            <div className={styles['comparison__panel-title']} style={{ padding: '20px 24px', borderBottom: '1px solid #F3F4F6' }}>Metric Breakdown</div>
            <table className="tbl">
              <thead><tr><th>Metric</th>{comp.map((m, i) => <th key={i} style={{ color: COLORS[i] }}>{m.name}</th>)}</tr></thead>
              <tbody>
                {mets.map((met) => (
                  <tr key={met.k}>
                    <td style={{ fontWeight: 700, fontSize: 13 }}>{met.l}</td>
                    {comp.map((m, i) => <td key={i} style={{ fontFamily: "'JetBrains Mono',monospace", fontWeight: 700 }}>{m[met.k]}</td>)}
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        </div>

        <div className="card">
          <div className={styles['comparison__panel-title']} style={{ marginBottom: 20 }}>Score Comparison</div>
          <div className={styles['comparison__scores']}>
            {comp.map((m, i) => (
              <div key={i} className={styles['comparison__score-item']}>
                <ScoreRing score={Math.round(m.accuracy)} size={100} stroke={7} color={COLORS[i]} label="ACCURACY" />
                <div style={{ fontWeight: 700, fontSize: 14, textAlign: 'center' }}>{m.name}</div>
                <div style={{ fontSize: 12, color: '#6B7280' }}>{m.ctx.toLocaleString()} ctx</div>
              </div>
            ))}
          </div>
        </div>
      </div>
    </div>
  );
}


















//Dashboard.module.scss
@use '../../styles/_variables' as *;

.d-stats { display: grid; grid-template-columns: repeat(3, 1fr); gap: 16px; margin-bottom: 24px; }
.d-stat {
  background: $surface; border: 1px solid $border; border-radius: 16px; padding: 22px 24px; transition: all .2s;
}
.d-stat:hover { box-shadow: $shadow-2; transform: translateY(-2px); }
.d-stat-top { display: flex; justify-content: space-between; align-items: flex-start; }
.d-stat-label { font-size: 12px; color: $text-secondary; font-weight: 600; text-transform: uppercase; letter-spacing: .5px; }
.d-stat-val { font-family: $font-mono; font-size: 34px; font-weight: 700; letter-spacing: -1px; line-height: 1; margin-top: 8px; }
.d-stat-change { font-size: 12px; color: $emerald; font-weight: 600; margin-top: 6px; display: flex; align-items: center; gap: 4px; }

.dash {
  &__grid { display: grid; grid-template-columns: 1.2fr 1fr; gap: 16px; margin-bottom: 24px; }
  &__panel-hdr {
    padding: 20px 24px; border-bottom: 1px solid $border-light; display: flex; justify-content: space-between;
    align-items: center; font-family: $font-display; font-weight: 700; font-size: 15px;
  }
  &__run-row {
    padding: 16px 24px; border-bottom: 1px solid $border-light; display: flex; justify-content: space-between;
    align-items: center; transition: background .15s; cursor: pointer;
  }
  &__run-row:hover { background: $surface-hover; }
  &__spinner {
    width: 44px; height: 44px; border-radius: 50%; background: $indigo-pale;
    display: flex; align-items: center; justify-content: center;
  }
  &__empty { padding: 32px 24px; color: $text-secondary; font-size: 13px; }
  &__legend { display: flex; justify-content: center; gap: 20px; margin-top: 4px; flex-wrap: wrap; }
  &__legend span { display: flex; align-items: center; gap: 6px; font-size: 12px; font-weight: 600; color: $text-secondary; }
  &__dot { width: 8px; height: 8px; border-radius: 50%; display: inline-block; }
  &__actions { display: grid; grid-template-columns: repeat(4, 1fr); gap: 14px; }
}

// Note: .radar-wrap moved to src/styles/global.scss — it's shared with Comparison.

.qa-btn {
  display: flex; flex-direction: column; align-items: center; gap: 12px; padding: 24px 20px;
  border: 1px solid $border; border-radius: 16px; background: $surface; cursor: pointer; transition: all .25s;
  text-align: center;
}
.qa-btn:hover { transform: translateY(-4px); box-shadow: $shadow-3; border-color: transparent; }
.qa-btn__icon { width: 48px; height: 48px; border-radius: 14px; display: flex; align-items: center; justify-content: center; }
.qa-btn--ind .qa-btn__icon { background: $indigo-pale; color: $indigo; }
.qa-btn--em .qa-btn__icon { background: $emerald-pale; color: $emerald; }
.qa-btn--amb .qa-btn__icon { background: $amber-pale; color: $amber; }
.qa-btn--sky .qa-btn__icon { background: $sky-pale; color: $sky; }
.qa-btn__label { font-size: 14px; font-weight: 700; font-family: $font-display; }
.qa-btn__desc { font-size: 12px; color: $text-secondary; margin-top: 2px; }

@media (max-width: 768px) {
  .d-stats { grid-template-columns: repeat(2, 1fr); }
  .dash__grid { grid-template-columns: 1fr; }
  .dash__actions { grid-template-columns: 1fr 1fr; }
}



















//Dashboard.tsx
import { useEffect } from 'react';
import { useNavigate } from 'react-router-dom';
import { Loader2, TrendingUp, Play, Plus, GitCompare, BookOpen, ChevronRight } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchProviders } from '../../store/slices/providersSlice';
import { fetchModels } from '../../store/slices/modelsSlice';
import ScoreRing from '../common/ScoreRing';
import Sparkline from '../common/Sparkline';
import RadarChart from '../common/RadarChart';
import { useCounter } from '../common/useCounter';
import styles from './Dashboard.module.scss';

export default function Dashboard() {
  const dispatch = useAppDispatch();
  const navigate = useNavigate();

  const providers = useAppSelector((s) => s.providers.items);
  const models = useAppSelector((s) => s.models.items);
  const runs = useAppSelector((s) => s.evaluations.runs);

  useEffect(() => {
    dispatch(fetchProviders());
    dispatch(fetchModels());
  }, [dispatch]);

  const connectedCount = providers.filter((p) => p.status === 'connected').length;
  const totalEvals = useCounter(runs.length, 900);
  const connectedAnim = useCounter(connectedCount, 900);
  const avgAccuracy = models.length
    ? (models.reduce((sum, m) => sum + (m.accuracy_score || 0), 0) / models.length).toFixed(1)
    : '0.0';

  const stats = [
    { label: 'Total Evaluations', value: totalEvals, change: `${runs.length} tracked locally`, spark: [2, 4, 3, 5, 6, 5, runs.length || 1] },
    { label: 'Models Connected', value: connectedAnim, change: `${models.length} in catalog`, spark: [1, 2, 2, 3, 3, 4, connectedCount || 1] },
    { label: 'Avg. Accuracy', value: `${avgAccuracy}%`, change: 'Across active models', spark: [85, 86, 87, 88, 89, 90, Number(avgAccuracy) || 90] },
  ];

  const radarModels = models.slice(0, 3).map((m) => ({
    name: m.name,
    values: [
      (m.accuracy_score || 0) / 100,
      0.8,
      m.input_price ? Math.max(0, 1 - m.input_price / 10) : 0.5,
      Math.min(1, m.context_window / 200000),
      (m.agent_score || m.accuracy_score || 0) / 100,
    ],
  }));

  return (
    <div className="page-enter">
      <div className="pg-hdr"><h1>Dashboard</h1><p>Your evaluation activity at a glance</p></div>
      <div className="pg-body">
        <div className={styles['d-stats']}>
          {stats.map((s, i) => (
            <div className={styles['d-stat']} key={i}>
              <div className={styles['d-stat-top']}>
                <div>
                  <div className={styles['d-stat-label']}>{s.label}</div>
                  <div className={styles['d-stat-val']}>{s.value}</div>
                  <div className={styles['d-stat-change']}><TrendingUp size={12} /> {s.change}</div>
                </div>
                <Sparkline data={s.spark} color="#6366F1" width={72} height={32} />
              </div>
            </div>
          ))}
        </div>

        <div className={styles['dash__grid']}>
          <div className="card" style={{ padding: 0 }}>
            <div className={styles['dash__panel-hdr']}>
              <span>Recent Evaluations</span>
              <button className="btn btn-ghost btn-sm" onClick={() => navigate('/app/evaluations')}>View All <ChevronRight size={14} /></button>
            </div>
            {runs.length === 0 && (
              <div className={styles['dash__empty']}>No evaluations yet — launch one from Quick Actions below.</div>
            )}
            {runs.slice(0, 4).map((run) => (
              <div key={run.id} className={styles['dash__run-row']} onClick={() => navigate(`/app/evaluations/${run.id}`)}>
                <div style={{ display: 'flex', alignItems: 'center', gap: 14 }}>
                  {run.status === 'completed' ? (
                    <ScoreRing score={Math.round((run.progress / (run.total || 1)) * 100)} size={44} stroke={4} color="#6366F1" />
                  ) : (
                    <div className={styles['dash__spinner']}><Loader2 size={18} color="#6366F1" style={{ animation: 'spin 1.5s linear infinite' }} /></div>
                  )}
                  <div>
                    <div style={{ fontWeight: 600, fontSize: 14 }}>{run.name}</div>
                    <div style={{ fontSize: 12, color: '#6B7280' }}>{run.benchmark || '—'} &middot; {new Date(run.created_at).toLocaleDateString()}</div>
                  </div>
                </div>
                <span className={`badge ${run.status === 'completed' ? 'badge-green' : 'badge-run'}`}>{run.status}</span>
              </div>
            ))}
          </div>

          <div className="card">
            <div style={{ fontFamily: "'Space Grotesk',sans-serif", fontWeight: 700, fontSize: 15, marginBottom: 4 }}>Top Models</div>
            <div style={{ fontSize: 13, color: '#6B7280', marginBottom: 12 }}>Strength comparison across 5 dimensions</div>
            {radarModels.length > 0 ? (
              <>
                <div className="radar-wrap"><RadarChart models={radarModels} size={260} /></div>
                <div className={styles['dash__legend']}>
                  {radarModels.map((m, i) => (
                    <span key={i}><span className={styles['dash__dot']} style={{ background: ['#6366F1', '#F59E0B', '#10B981'][i] }} /> {m.name}</span>
                  ))}
                </div>
              </>
            ) : (
              <div className={styles['dash__empty']}>Connect a provider to see model comparisons.</div>
            )}
          </div>
        </div>

        <div className="card" style={{ padding: 24 }}>
          <div style={{ fontFamily: "'Space Grotesk',sans-serif", fontWeight: 700, fontSize: 15, marginBottom: 20 }}>Quick Actions</div>
          <div className={styles['dash__actions']}>
            {[
              { icon: <Play size={20} />, label: 'New Evaluation', desc: 'Start a benchmark run', to: '/app/new-eval', cls: 'ind' },
              { icon: <Plus size={20} />, label: 'Add Provider', desc: 'Connect an API', to: '/app/providers', cls: 'em' },
              { icon: <GitCompare size={20} />, label: 'Compare Models', desc: 'Side-by-side analysis', to: '/app/comparison', cls: 'amb' },
              { icon: <BookOpen size={20} />, label: 'Browse Suites', desc: 'Explore benchmarks', to: '/app/suites', cls: 'sky' },
            ].map((a, i) => (
              <button key={i} onClick={() => navigate(a.to)} className={`${styles['qa-btn']} ${styles[`qa-btn--${a.cls}`]}`}>
                <div className={styles['qa-btn__icon']}>{a.icon}</div>
                <div><div className={styles['qa-btn__label']}>{a.label}</div><div className={styles['qa-btn__desc']}>{a.desc}</div></div>
              </button>
            ))}
          </div>
        </div>
      </div>
    </div>
  );
}
















//EvaluationDetail.module.scss
@use '../../styles/_variables' as *;

.eval-detail__card {
  display: flex; gap: 32px; align-items: center; background: $surface; border: 1px solid $border;
  border-radius: 16px; padding: 28px;
}
.eval-detail__progress-ring { position: relative; width: 96px; height: 96px; flex-shrink: 0; }
.eval-detail__progress-val {
  position: absolute; inset: 0; display: flex; align-items: center; justify-content: center;
  font-family: $font-mono; font-weight: 700; font-size: 18px;
}
.eval-detail__meta { flex: 1; }
.eval-detail__meta-row {
  display: flex; justify-content: space-between; padding: 10px 0; border-bottom: 1px solid $border-light;
  font-size: 13px; color: $text-secondary;
  strong { color: $text-primary; font-weight: 600; }
}
.eval-detail__meta-row:last-child { border-bottom: none; }
.eval-detail__error { margin-top: 12px; padding: 10px 14px; background: $red-pale; color: $red; border-radius: 10px; font-size: 13px; }















//EvaluationDetail.tsx
import { useEffect } from 'react';
import { useParams, useNavigate } from 'react-router-dom';
import { ChevronLeft, Loader2 } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { pollEvaluationStatus } from '../../api/endpoints/evaluations';
import { updateRunStatus } from '../../store/slices/evaluationsSlice';
import styles from './EvaluationDetail.module.scss';

export default function EvaluationDetail() {
  const { id } = useParams<{ id: string }>();
  const navigate = useNavigate();
  const dispatch = useAppDispatch();
  const run = useAppSelector((s) => s.evaluations.runs.find((r) => r.id === id));
  const models = useAppSelector((s) => s.models.items);

  useEffect(() => {
    if (!id || !run) return;
    if (run.status === 'completed' || run.status === 'failed' || run.status === 'canceled') return;
    const stop = pollEvaluationStatus(id, (status) => dispatch(updateRunStatus({ id, status })));
    return stop;
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [id]);

  if (!run) {
    return (
      <div className="page-enter">
        <div className="pg-hdr"><h1>Evaluation not found</h1></div>
        <div className="pg-body"><button className="btn btn-ghost" onClick={() => navigate('/app/evaluations')}><ChevronLeft size={16} /> Back</button></div>
      </div>
    );
  }

  const pct = run.total > 0 ? Math.round((run.progress / run.total) * 100) : run.status === 'completed' ? 100 : 0;

  return (
    <div className="page-enter">
      <div className="pg-hdr">
        <div style={{ display: 'flex', alignItems: 'center', gap: 12 }}>
          <button className="btn btn-ghost btn-sm" onClick={() => navigate('/app/evaluations')}><ChevronLeft size={16} /> Back</button>
          <h1>{run.name}</h1>
          <span className={`badge ${run.status === 'completed' ? 'badge-green' : run.status === 'failed' ? 'badge-gray' : 'badge-run'}`}>
            {run.status !== 'completed' && run.status !== 'failed' && <Loader2 size={11} style={{ animation: 'spin 1.5s linear infinite' }} />} {run.status}
          </span>
        </div>
      </div>
      <div className="pg-body">
        <div className={styles['eval-detail__card']}>
          <div className={styles['eval-detail__progress-ring']}>
            <svg width="96" height="96" style={{ transform: 'rotate(-90deg)' }}>
              <circle cx="48" cy="48" r="42" fill="none" stroke="#F3F4F6" strokeWidth="7" />
              <circle cx="48" cy="48" r="42" fill="none" stroke="#6366F1" strokeWidth="7" strokeDasharray={2 * Math.PI * 42} strokeDashoffset={2 * Math.PI * 42 * (1 - pct / 100)} strokeLinecap="round" style={{ transition: 'stroke-dashoffset 1s ease' }} />
            </svg>
            <div className={styles['eval-detail__progress-val']}>{pct}%</div>
          </div>
          <div className={styles['eval-detail__meta']}>
            {[{ l: 'Type', v: run.eval_type }, { l: 'Benchmark', v: run.benchmark || '—' },
              { l: 'Models', v: run.model_ids.map((mid) => models.find((m) => m.id === mid)?.name || mid).join(', ') || '—' },
              { l: 'Progress', v: run.total ? `${run.progress} / ${run.total} tasks` : 'Awaiting first update…' }].map((d, i) => (
              <div key={i} className={styles['eval-detail__meta-row']}><span>{d.l}</span><strong>{d.v}</strong></div>
            ))}
            {run.error_message && <div className={styles['eval-detail__error']}>{run.error_message}</div>}
          </div>
        </div>
      </div>
    </div>
  );
}

















//Evaluations.module.scss
.evals__empty {
  padding: 64px; text-align: center; color: #6B7280; background: #FFFFFF;
  border: 1px solid #E5E7EB; border-radius: 16px; font-size: 14px;
}















//Evaluations.tsx
import { useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { Search, Plus, ChevronRight, Loader2 } from 'lucide-react';
import { useAppSelector } from '../../hooks/redux';
import styles from './Evaluations.module.scss';

export default function Evaluations() {
  const navigate = useNavigate();
  const runs = useAppSelector((s) => s.evaluations.runs);
  const [search, setSearch] = useState('');

  const filtered = runs.filter((r) => !search || r.name.toLowerCase().includes(search.toLowerCase()));

  return (
    <div className="page-enter">
      <div className="pg-hdr"><h1>Evaluations</h1><p>History of all evaluation runs launched from this session</p></div>
      <div className="pg-body">
        <div className="toolbar">
          <div className="search-box"><Search size={16} color="#9CA3AF" /><input placeholder="Search evaluations…" value={search} onChange={(e) => setSearch(e.target.value)} /></div>
          <button className="btn btn-ind" onClick={() => navigate('/app/new-eval')}><Plus size={14} /> New Evaluation</button>
        </div>

        {filtered.length === 0 && <div className={styles['evals__empty']}>No evaluations launched yet.</div>}

        {filtered.length > 0 && (
          <div className="tw">
            <table className="tbl">
              <thead><tr><th>Name</th><th>Type</th><th>Started</th><th>Status</th><th>Progress</th><th>Models</th><th></th></tr></thead>
              <tbody>
                {filtered.map((run) => (
                  <tr key={run.id} style={{ cursor: 'pointer' }} onClick={() => navigate(`/app/evaluations/${run.id}`)}>
                    <td style={{ fontWeight: 700 }}>{run.name}</td>
                    <td><span className="tag tag-ind">{run.eval_type}</span></td>
                    <td style={{ color: '#6B7280' }}>{new Date(run.created_at).toLocaleString()}</td>
                    <td>
                      <span className={`badge ${run.status === 'completed' ? 'badge-green' : run.status === 'failed' ? 'badge-gray' : 'badge-run'}`}>
                        {run.status === 'running' && <Loader2 size={11} style={{ animation: 'spin 1.5s linear infinite' }} />} {run.status}
                      </span>
                    </td>
                    <td style={{ fontFamily: "'JetBrains Mono',monospace" }}>{run.total ? `${run.progress}/${run.total}` : '—'}</td>
                    <td style={{ fontFamily: "'JetBrains Mono',monospace" }}>{run.model_ids.length}</td>
                    <td><ChevronRight size={16} color="#9CA3AF" /></td>
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        )}
      </div>
    </div>
  );
}












//NewEvaluation.module.scss
@use '../../styles/_variables' as *;

.wiz {
  background: $surface; border: 1px solid $border; border-radius: 20px; overflow: hidden; box-shadow: $shadow-2;
}
.wiz-progress { display: flex; padding: 24px 32px; background: $surface-alt; border-bottom: 1px solid $border; gap: 4px; }
.wiz-s { display: flex; align-items: center; gap: 8px; flex: 1; }
.wiz-num {
  width: 30px; height: 30px; border-radius: 50%; display: flex; align-items: center; justify-content: center;
  font-size: 12px; font-weight: 700; font-family: $font-mono; transition: all .3s;
}
.wiz-num.done { background: $grad-primary; color: #fff; box-shadow: 0 2px 6px rgba(99, 102, 241, .25); }
.wiz-num.cur { background: $grad-primary; color: #fff; box-shadow: 0 0 0 4px rgba(99, 102, 241, .15), 0 2px 6px rgba(99, 102, 241, .25); }
.wiz-num.pen { background: $surface; color: $text-muted; border: 2px solid $border; }
.wiz-lbl { font-size: 12px; font-weight: 600; color: $text-secondary; white-space: nowrap; }
.wiz-lbl.cur { color: $indigo; }
.wiz-line { flex: 1; height: 2px; background: $border; margin: 0 8px; border-radius: 1px; }
.wiz-line.done { background: $indigo; }
.wiz-body { padding: 36px; min-height: 380px; }
.wiz-body h2 { font-size: 24px; font-weight: 700; margin-bottom: 8px; letter-spacing: -.4px; }
.wiz-body .sub { font-size: 14px; color: $text-secondary; margin-bottom: 28px; line-height: 1.5; }
.wiz-foot { display: flex; justify-content: space-between; padding: 20px 36px; border-top: 1px solid $border; background: $surface-alt; }
.wiz-launch { background: $emerald; box-shadow: 0 4px 14px rgba(16, 185, 129, .3); }
.wiz-empty { padding: 48px; text-align: center; color: $text-secondary; background: $surface-alt; border-radius: 14px; grid-column: 1 / -1; }
.wiz-error { margin-top: 16px; padding: 12px 16px; background: $red-pale; color: $red; border-radius: 10px; font-size: 13px; font-weight: 600; }

.wiz-review-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 12px; }
.wiz-review-cell { padding: 16px; background: $surface-alt; border-radius: 12px; border: 1px solid $border-light; }
.wiz-review-label { font-size: 11px; color: $text-muted; font-weight: 700; text-transform: uppercase; letter-spacing: 1px; margin-bottom: 4px; }
.wiz-review-val { font-size: 14px; font-weight: 600; }

.wiz-summary {
  display: flex; gap: 32px; margin-top: 24px; padding: 24px; background: $indigo-pale;
  border-radius: 14px; border: 1px solid rgba(99, 102, 241, .12);
}
.wiz-summary-label { font-size: 11px; color: $text-secondary; font-weight: 700; letter-spacing: 1px; }
.wiz-summary-val { font-family: $font-mono; font-size: 28px; font-weight: 700; margin-top: 4px; }

.toast__icon { width: 36px; height: 36px; border-radius: 10px; background: $emerald-pale; display: flex; align-items: center; justify-content: center; }

@media (max-width: 768px) {
  .wiz-lbl { display: none; }
  .wiz-review-grid { grid-template-columns: 1fr; }
}




















//NewEvaluation.tsx
import { useEffect, useMemo, useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { ChevronLeft, ChevronRight, Check, Cpu, Bot, Database, Play } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchProviders } from '../../store/slices/providersSlice';
import { fetchModels } from '../../store/slices/modelsSlice';
import { fetchBenchmarks } from '../../store/slices/benchmarksSlice';
import { fetchMetrics } from '../../store/slices/metricsSlice';
import { launchEvaluation, setDraft } from '../../store/slices/evaluationsSlice';
import type { CreateEvaluationRequest } from '../../types';
import styles from './NewEvaluation.module.scss';

const STEPS = ['Name & Type', 'Providers', 'Models', 'Test Suite', 'Metrics', 'Review'];

export default function NewEvaluation() {
  const dispatch = useAppDispatch();
  const navigate = useNavigate();
  const [step, setStep] = useState(0);
  const [toast, setToast] = useState(false);

  const draft = useAppSelector((s) => s.evaluations.draft);
  const launching = useAppSelector((s) => s.evaluations.launching);
  const launchError = useAppSelector((s) => s.evaluations.launchError);

  const providers = useAppSelector((s) => s.providers.items);
  const models = useAppSelector((s) => s.models.items);
  const benchmarks = useAppSelector((s) => s.benchmarks.items);
  const metrics = useAppSelector((s) => s.metrics);

  useEffect(() => {
    dispatch(fetchProviders());
    dispatch(fetchModels());
    dispatch(fetchBenchmarks());
    dispatch(fetchMetrics());
  }, [dispatch]);

  const connectedProviders = providers.filter((p) => p.status === 'connected');
  const availableModels = useMemo(
    () => models.filter((m) => draft.selProviders.includes(m.provider_id)),
    [models, draft.selProviders]
  );
  const activeMetricsList = draft.eval_type === 'agent'
    ? [...metrics.allMetrics, ...metrics.customAgentMetrics]
    : metrics.allMetrics;

  const toggle = (list: string[], value: string) => (list.includes(value) ? list.filter((v) => v !== value) : [...list, value]);

  const canGo = () => {
    if (step === 0) return draft.name.trim() && draft.eval_type;
    if (step === 1) return draft.selProviders.length > 0;
    if (step === 2) return draft.selModels.length > 0;
    if (step === 3) return Boolean(draft.selBenchmark);
    return true;
  };

  const launch = async () => {
    const benchmark = benchmarks.find((b) => b.name === draft.selBenchmark);
    const payload: CreateEvaluationRequest = {
      name: draft.name,
      eval_type: draft.eval_type.toLowerCase(),
      dataset_id: benchmark?.huggingface_dataset || '',
      benchmark: draft.selBenchmark || undefined,
      model_ids: draft.selModels,
      selected_metrics: draft.selMetrics,
      selected_category: benchmark ? [benchmark.type] : undefined,
      ...(draft.judgeModelId
        ? { judge_config: { model_id: draft.judgeModelId, base_url: '', api_key: '' } }
        : {}),
    };

    const result = await dispatch(launchEvaluation(payload));
    if (launchEvaluation.fulfilled.match(result)) {
      setToast(true);
      setTimeout(() => {
        setToast(false);
        navigate('/app/evaluations');
      }, 2000);
    }
  };

  return (
    <div className="page-enter">
      <div className="pg-hdr"><h1>New Evaluation</h1><p>Set up and launch a structured model evaluation</p></div>
      <div className="pg-body">
        <div className={styles.wiz}>
          <div className={styles['wiz-progress']}>
            {STEPS.map((s, i) => (
              <div key={i} className={styles['wiz-s']} style={{ flex: i < STEPS.length - 1 ? 1 : 0 }}>
                <div className={`${styles['wiz-num']} ${i < step ? styles.done : i === step ? styles.cur : styles.pen}`}>{i < step ? <Check size={13} /> : i + 1}</div>
                <span className={`${styles['wiz-lbl']} ${i === step ? styles.cur : ''}`}>{s}</span>
                {i < STEPS.length - 1 && <div className={`${styles['wiz-line']} ${i < step ? styles.done : ''}`} />}
              </div>
            ))}
          </div>

          <div className={styles['wiz-body']}>
            {step === 0 && (
              <>
                <h2>Name your evaluation</h2>
                <p className={styles.sub}>Give it a recognizable name and choose what you're evaluating.</p>
                <div className="fg"><label className="fl">Evaluation Name</label>
                  <input className="fi" placeholder="e.g. Q3 Model Selection" value={draft.name} onChange={(e) => dispatch(setDraft({ name: e.target.value }))} />
                </div>
                <div className="fg"><label className="fl">Evaluation Type</label>
                  <div className="sel-grid">
                    {[{ v: 'Model', i: <Cpu size={18} />, s: 'General-purpose model' }, { v: 'Agent', i: <Bot size={18} />, s: 'Autonomous agent' }, { v: 'RAG', i: <Database size={18} />, s: 'Retrieval-augmented' }].map((o) => (
                      <div key={o.v} className={`sel-opt ${draft.eval_type === o.v ? 'on' : ''}`} onClick={() => dispatch(setDraft({ eval_type: o.v }))}>
                        <div className="sel-chk">{draft.eval_type === o.v && <Check size={13} />}</div>
                        <div><div className="sel-txt" style={{ display: 'flex', alignItems: 'center', gap: 6 }}>{o.i} {o.v}</div><div className="sel-sub">{o.s}</div></div>
                      </div>
                    ))}
                  </div>
                </div>
              </>
            )}

            {step === 1 && (
              <>
                <h2>Select providers</h2>
                <p className={styles.sub}>Choose which connected providers to draw models from.</p>
                <div className="sel-grid">
                  {connectedProviders.map((p) => (
                    <div key={p.id} className={`sel-opt ${draft.selProviders.includes(p.id) ? 'on' : ''}`} onClick={() => dispatch(setDraft({ selProviders: toggle(draft.selProviders, p.id) }))}>
                      <div className="sel-chk">{draft.selProviders.includes(p.id) && <Check size={13} />}</div>
                      <div><div className="sel-txt">{p.name}</div><div className="sel-sub">{p.model_count} models available</div></div>
                    </div>
                  ))}
                  {connectedProviders.length === 0 && <div className={styles['wiz-empty']}>No connected providers yet — connect one from the Providers page first.</div>}
                </div>
              </>
            )}

            {step === 2 && (
              <>
                <h2>Choose models</h2>
                <p className={styles.sub}>Pick which models to include in this evaluation.</p>
                {availableModels.length > 0 ? (
                  <div className="sel-grid">
                    {availableModels.map((m) => (
                      <div key={m.id} className={`sel-opt ${draft.selModels.includes(m.id) ? 'on' : ''}`} onClick={() => dispatch(setDraft({ selModels: toggle(draft.selModels, m.id) }))}>
                        <div className="sel-chk">{draft.selModels.includes(m.id) && <Check size={13} />}</div>
                        <div><div className="sel-txt">{m.name}</div><div className="sel-sub">{m.context_window.toLocaleString()} ctx</div></div>
                      </div>
                    ))}
                  </div>
                ) : (
                  <div className={styles['wiz-empty']}>Select providers first to see available models.</div>
                )}
              </>
            )}

            {step === 3 && (
              <>
                <h2>Pick a test suite</h2>
                <p className={styles.sub}>Select the benchmark to evaluate against.</p>
                <div className="sel-grid" style={{ gridTemplateColumns: 'repeat(auto-fill,minmax(280px,1fr))' }}>
                  {benchmarks.map((b) => (
                    <div key={b.name} className={`sel-opt ${draft.selBenchmark === b.name ? 'on' : ''}`} onClick={() => dispatch(setDraft({ selBenchmark: b.name }))} style={{ flexDirection: 'column', alignItems: 'flex-start', gap: 8 }}>
                      <div style={{ display: 'flex', width: '100%', justifyContent: 'space-between', alignItems: 'center' }}>
                        <div className="sel-txt">{b.name}</div><div className="sel-chk">{draft.selBenchmark === b.name && <Check size={13} />}</div>
                      </div>
                      <div className="sel-sub">{b.description}</div>
                      <div><span className="tag tag-amb">{b.type}</span><span style={{ fontSize: 11, color: '#9CA3AF', marginLeft: 8 }}>{b.task_count.toLocaleString()} tasks</span></div>
                    </div>
                  ))}
                </div>
              </>
            )}

            {step === 4 && (
              <>
                <h2>Configure metrics</h2>
                <p className={styles.sub}>Choose which metrics to measure.</p>
                <div style={{ marginBottom: 16, display: 'flex', gap: 8 }}>
                  <button className="btn btn-sm btn-ghost" onClick={() => dispatch(setDraft({ selMetrics: [...activeMetricsList] }))}>Select All</button>
                  <button className="btn btn-sm btn-ghost" onClick={() => dispatch(setDraft({ selMetrics: [] }))}>Clear All</button>
                </div>
                <div className="sel-grid" style={{ gridTemplateColumns: 'repeat(auto-fill,minmax(160px,1fr))' }}>
                  {activeMetricsList.map((m) => (
                    <div key={m} className={`sel-opt ${draft.selMetrics.includes(m) ? 'on' : ''}`} onClick={() => dispatch(setDraft({ selMetrics: toggle(draft.selMetrics, m) }))}>
                      <div className="sel-chk">{draft.selMetrics.includes(m) && <Check size={13} />}</div><div className="sel-txt">{m}</div>
                    </div>
                  ))}
                </div>
                <div className="fg" style={{ marginTop: 24 }}>
                  <label className="fl">Judge Model <span className="opt">(optional — grades other models)</span></label>
                  <select className="fi" value={draft.judgeModelId || ''} onChange={(e) => dispatch(setDraft({ judgeModelId: e.target.value || undefined }))}>
                    <option value="">None</option>
                    {models.filter((m) => m.is_active).map((m) => <option key={m.id} value={m.id}>{m.name}</option>)}
                  </select>
                </div>
              </>
            )}

            {step === 5 && (() => {
              const suite = benchmarks.find((b) => b.name === draft.selBenchmark);
              const modelNames = draft.selModels.map((id) => models.find((m) => m.id === id)?.name || id).join(', ');
              const providerNames = draft.selProviders.map((id) => providers.find((p) => p.id === id)?.name || id).join(', ');
              return (
                <>
                  <h2>Review & Launch</h2>
                  <p className={styles.sub}>Confirm your evaluation setup.</p>
                  <div className={styles['wiz-review-grid']}>
                    {[
                      { l: 'Name', v: draft.name }, { l: 'Type', v: draft.eval_type },
                      { l: 'Providers', v: providerNames || '—' }, { l: 'Models', v: modelNames || '—' },
                      { l: 'Test Suite', v: suite?.name || '—' }, { l: 'Metrics', v: draft.selMetrics.join(', ') || '—' },
                    ].map((r, i) => (
                      <div key={i} className={styles['wiz-review-cell']}><div className={styles['wiz-review-label']}>{r.l}</div><div className={styles['wiz-review-val']}>{r.v}</div></div>
                    ))}
                  </div>
                  <div className={styles['wiz-summary']}>
                    {[{ l: 'TOTAL TASKS', v: suite?.task_count.toLocaleString() || '—' }, { l: 'MODELS', v: String(draft.selModels.length) }, { l: 'METRICS', v: String(draft.selMetrics.length) }].map((d, i) => (
                      <div key={i}><div className={styles['wiz-summary-label']}>{d.l}</div><div className={styles['wiz-summary-val']}>{d.v}</div></div>
                    ))}
                  </div>
                  {launchError && <div className={styles['wiz-error']}>{launchError}</div>}
                </>
              );
            })()}
          </div>

          <div className={styles['wiz-foot']}>
            <button className="btn btn-ghost" onClick={() => (step > 0 ? setStep(step - 1) : navigate('/app/dashboard'))}>
              <ChevronLeft size={16} /> {step === 0 ? 'Cancel' : 'Back'}
            </button>
            {step < 5 ? (
              <button className="btn btn-ind" onClick={() => setStep(step + 1)} disabled={!canGo()}>Continue <ChevronRight size={16} /></button>
            ) : (
              <button className={`btn btn-ind ${styles['wiz-launch']}`} onClick={launch} disabled={launching}>
                <Play size={16} /> {launching ? 'Launching…' : 'Launch Evaluation'}
              </button>
            )}
          </div>
        </div>
      </div>

      {toast && (
        <div className="toast">
          <div className={styles['toast__icon']}><Check size={18} color="#10B981" /></div>
          <div><div style={{ fontWeight: 700, fontSize: 14 }}>Evaluation launched</div><div style={{ fontSize: 12, color: '#6B7280' }}>You'll find it in Evaluations once it completes.</div></div>
        </div>
      )}
    </div>
  );
}























//Landing.module.scss
@use '../../styles/_variables' as *;

.landing { min-height: 100vh; background: $surface; overflow-x: hidden; }

.l-error {
  max-width: 1440px; margin: 0 auto; padding: 0 48px;
  color: $red; font-size: 13px; font-weight: 600;
}

.l-nav {
  display: flex; align-items: center; justify-content: space-between; padding: 18px 48px;
  max-width: 1440px; margin: 0 auto; position: relative; z-index: 10;
}
.l-logo {
  font-family: $font-display; font-size: 22px; font-weight: 700; letter-spacing: -.5px;
  display: flex; align-items: center; gap: 10px; color: $text-primary;

  .mark {
    width: 34px; height: 34px; background: $grad-primary; border-radius: 10px;
    display: flex; align-items: center; justify-content: center; color: #fff; font-size: 18px;
    box-shadow: 0 2px 8px rgba(99, 102, 241, .35);
  }
}
.l-links {
  display: flex; gap: 32px; align-items: center;
  a { color: $text-secondary; text-decoration: none; font-size: 14px; font-weight: 500; transition: color .2s; cursor: pointer; }
  a:hover { color: $text-primary; }
}

.btn-primary {
  display: inline-flex; align-items: center; gap: 8px; background: $grad-primary; color: #fff; border: none;
  padding: 14px 28px; border-radius: 14px; font-size: 15px; font-weight: 600; cursor: pointer; transition: all .25s;
  font-family: $font-display; box-shadow: 0 4px 14px rgba(99, 102, 241, .3);
}
.btn-primary:hover { transform: translateY(-2px); box-shadow: 0 8px 24px rgba(99, 102, 241, .35); }
.btn-primary:disabled { opacity: .6; cursor: default; transform: none; }

.btn-secondary {
  display: inline-flex; align-items: center; gap: 8px; background: $surface; color: $text-primary;
  border: 1px solid $border; padding: 14px 28px; border-radius: 14px; font-size: 15px; font-weight: 600;
  cursor: pointer; transition: all .25s; font-family: $font-display;
}
.btn-secondary:hover { border-color: $indigo; color: $indigo; background: $indigo-pale; box-shadow: $shadow-2; }

.hero-section {
  position: relative; max-width: 1440px; margin: 0 auto; padding: 64px 48px 80px;
  display: grid; grid-template-columns: 1fr 1fr; gap: 80px; align-items: center;
}
.hero-bg-dots {
  position: absolute; inset: 0; background-image: radial-gradient(circle, $border 1px, transparent 1px);
  background-size: 24px 24px; opacity: .5; pointer-events: none; animation: dotPulse 4s ease infinite;
}
.hero-content { position: relative; z-index: 2; }
.hero-badge {
  display: inline-flex; align-items: center; gap: 8px; background: $indigo-pale;
  border: 1px solid rgba(99, 102, 241, .15); border-radius: 100px; padding: 6px 16px 6px 8px;
  font-size: 13px; color: $indigo; margin-bottom: 28px; font-weight: 600;

  .badge-dot { width: 8px; height: 8px; border-radius: 50%; background: $indigo; animation: pulse 2s infinite; }
}
.hero-content h1 {
  font-size: 54px; font-weight: 700; line-height: 1.08; letter-spacing: -2.5px; margin-bottom: 24px;
  .grad { background: $grad-primary; -webkit-background-clip: text; -webkit-text-fill-color: transparent; }
}
.hero-content > p { font-size: 17px; color: $text-secondary; line-height: 1.7; margin-bottom: 40px; max-width: 480px; }
.hero-actions { display: flex; gap: 14px; }

.hero-visual { position: relative; z-index: 2; }
.hero-card { background: $surface; border: 1px solid $border; border-radius: 20px; padding: 28px; box-shadow: $shadow-4; }
.hero-card-hdr { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; }
.hero-card-title { font-family: $font-display; font-size: 13px; font-weight: 700; color: $text-muted; text-transform: uppercase; letter-spacing: 1.5px; }
.hero-bar { display: flex; align-items: center; gap: 12px; margin-bottom: 12px; }
.hero-bar:last-child { margin-bottom: 0; }
.hero-bar-label { font-size: 13px; color: $text-secondary; width: 120px; flex-shrink: 0; font-weight: 600; }
.hero-bar-track { flex: 1; height: 34px; background: $surface-alt; border-radius: 10px; overflow: hidden; }
.hero-bar-fill {
  height: 100%; border-radius: 10px; display: flex; align-items: center; padding: 0 14px;
  font-family: $font-mono; font-size: 12px; font-weight: 700; color: #fff; transition: width 1.8s cubic-bezier(.16, 1, .3, 1);

  &--primary { background: $grad-primary; }
  &--warm { background: $grad-warm; }
  &--cool { background: $grad-cool; }
  &--gray { background: linear-gradient(135deg, #6B7280, #9CA3AF); }
}

.float-badge {
  position: absolute; background: $surface; border: 1px solid $border; border-radius: 14px;
  padding: 12px 18px; display: flex; align-items: center; gap: 10px; box-shadow: $shadow-3;
  z-index: 3; animation: float 4s ease infinite; font-size: 13px; color: $text-secondary;
  strong { color: $text-primary; }
}
.float-badge.tr { top: -24px; right: -16px; animation-delay: .5s; }
.float-badge.bl { bottom: -20px; left: -16px; animation-delay: 1.5s; }
.pulse-dot { width: 8px; height: 8px; border-radius: 50%; background: $emerald; animation: pulse 2s infinite; }

.features { max-width: 1440px; margin: 0 auto; padding: 96px 48px; background: $bg; }
.feat-header { text-align: center; margin-bottom: 72px; }
.feat-header h2 { font-size: 38px; font-weight: 700; letter-spacing: -1.5px; margin-bottom: 14px; }
.feat-header p { color: $text-secondary; font-size: 16px; max-width: 460px; margin: 0 auto; line-height: 1.6; }
.feat-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 20px; }
.feat-card {
  background: $surface; border: 1px solid $border; border-radius: 18px; padding: 32px;
  transition: all .3s; cursor: default; position: relative; overflow: hidden;

  &::before { content: ''; position: absolute; top: 0; left: 0; right: 0; height: 3px; background: $grad-primary; opacity: 0; transition: opacity .3s; }
  &:hover { border-color: transparent; box-shadow: $shadow-3; transform: translateY(-6px); }
  &:hover::before { opacity: 1; }
  h3 { font-size: 17px; font-weight: 700; margin-bottom: 8px; letter-spacing: -.2px; }
  p { font-size: 14px; color: $text-secondary; line-height: 1.65; }
}
.feat-icon {
  width: 50px; height: 50px; border-radius: 14px; display: flex; align-items: center; justify-content: center; margin-bottom: 22px;
  &--ind { background: $indigo-pale; color: $indigo; }
  &--amb { background: $amber-pale; color: $amber; }
  &--em { background: $emerald-pale; color: $emerald; }
  &--sky { background: $sky-pale; color: $sky; }
  &--rose { background: $rose-pale; color: $rose; }
}

.stats-section {
  max-width: 1440px; margin: 0 auto; padding: 64px 48px; display: grid; grid-template-columns: repeat(4, 1fr);
  gap: 24px; background: $surface; border-top: 1px solid $border; border-bottom: 1px solid $border;
}
.stat-box { text-align: center; padding: 16px; }
.stat-val { font-family: $font-mono; font-size: 44px; font-weight: 700; letter-spacing: -2px; background: $grad-primary; -webkit-background-clip: text; -webkit-text-fill-color: transparent; }
.stat-lbl { font-size: 14px; color: $text-secondary; margin-top: 4px; font-weight: 500; }

.cta-section { max-width: 1440px; margin: 0 auto; padding: 96px 48px; text-align: center; background: $bg; }
.cta-box {
  background: $surface; border: 1px solid $border; border-radius: 28px; padding: 72px;
  position: relative; overflow: hidden; box-shadow: $shadow-2;
  &::before { content: ''; position: absolute; inset: -2px; background: $grad-primary; border-radius: 30px; z-index: -1; opacity: .15; }
  h2 { font-size: 38px; font-weight: 700; letter-spacing: -1.5px; margin-bottom: 14px; }
  p { color: $text-secondary; font-size: 16px; margin-bottom: 36px; line-height: 1.6; }
}

.l-footer { text-align: center; padding: 32px 48px; color: $text-muted; font-size: 13px; border-top: 1px solid $border; background: $surface; }

@media (max-width: 768px) {
  .hero-section { grid-template-columns: 1fr; padding: 48px 24px; gap: 40px; }
  .hero-content h1 { font-size: 36px; }
  .feat-grid { grid-template-columns: 1fr; }
  .stats-section { grid-template-columns: repeat(2, 1fr); }
}















//Landing.tsx
import { useEffect, useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { ArrowRight, Play, Award, Link2, Cpu, FlaskConical, BarChart3, GitCompare, Shield } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { ssoLogin } from '../../store/slices/authSlice';
import { tokenStorage } from '../../api/axiosInstance';
import styles from './Landing.module.scss';

const features = [
  { icon: <Link2 size={22} />, cls: 'ind', title: 'Provider Hub', desc: 'Connect any AI provider with API keys. Manage credentials, monitor status, and see available models instantly.' },
  { icon: <Cpu size={22} />, cls: 'amb', title: 'Model Catalog', desc: 'Browse all models across providers. Filter by capability, compare pricing, and register custom endpoints.' },
  { icon: <FlaskConical size={22} />, cls: 'em', title: 'Guided Evaluations', desc: 'A step-by-step wizard for model selection, test suite choice, and metric configuration.' },
  { icon: <BarChart3 size={22} />, cls: 'sky', title: 'Results & History', desc: 'Every evaluation stored with full breakdowns. Duplicate past runs, track trends, export findings.' },
  { icon: <GitCompare size={22} />, cls: 'rose', title: 'Visual Comparison', desc: 'Radar charts and metric tables make it obvious where each model excels or falls short.' },
  { icon: <Shield size={22} />, cls: 'ind', title: 'SSO & Security', desc: 'Enterprise-grade sign-in. API keys encrypted and isolated to your environment.' },
];

export default function Landing() {
  const [animated, setAnimated] = useState(false);
  const dispatch = useAppDispatch();
  const navigate = useNavigate();
  const authStatus = useAppSelector((s) => s.auth.status);
  const authError = useAppSelector((s) => s.auth.error);

  useEffect(() => {
    const t = setTimeout(() => setAnimated(true), 200);
    return () => clearTimeout(t);
  }, []);

  useEffect(() => {
    if (tokenStorage.get()) navigate('/app/dashboard');
  }, [navigate]);

  const handleSignIn = async () => {
    // In production this token comes from the SSO WebSocket handshake.
    const result = await dispatch(ssoLogin({ token: 'handshake-demo-token', data: 'semcoeval' }));
    if (ssoLogin.fulfilled.match(result)) {
      navigate('/app/dashboard');
    }
  };

  const bars = [
    { label: 'Claude Sonnet 4', pct: animated ? 94 : 0, cls: 'primary' },
    { label: 'GPT-4o', pct: animated ? 91 : 0, cls: 'warm' },
    { label: 'Gemini 2.5 Pro', pct: animated ? 89 : 0, cls: 'cool' },
    { label: 'Mistral Large', pct: animated ? 85 : 0, cls: 'gray' },
  ];

  return (
    <div className={styles.landing}>
      <nav className={styles['l-nav']}>
        <div className={styles['l-logo']}><div className={styles.mark}>&#9670;</div>SemcoEval</div>
        <div className={styles['l-links']}>
          <a>Features</a><a>Docs</a><a>Pricing</a>
          <button className={styles['btn-primary']} onClick={handleSignIn} style={{ padding: '10px 22px', fontSize: 14 }} disabled={authStatus === 'loading'}>
            {authStatus === 'loading' ? 'Signing in…' : 'Sign In'}
          </button>
        </div>
      </nav>

      {authError && <div className={styles['l-error']}>{authError}</div>}

      <section className={styles['hero-section']}>
        <div className={styles['hero-bg-dots']} />
        <div className={styles['hero-content']}>
          <div className={styles['hero-badge']}><div className={styles['badge-dot']} /> Now supporting 40+ models</div>
          <h1>Evaluate AI models<br />with <span className={styles.grad}>measured evidence</span></h1>
          <p>Stop guessing which model fits your use case. Run structured benchmarks, compare results side-by-side, and make selection decisions backed by real data.</p>
          <div className={styles['hero-actions']}>
            <button className={styles['btn-primary']} onClick={handleSignIn}>Open Dashboard <ArrowRight size={16} /></button>
            <button className={styles['btn-secondary']}><Play size={16} /> Watch Demo</button>
          </div>
        </div>
        <div className={styles['hero-visual']}>
          <div className={styles['hero-card']}>
            <div className={styles['hero-card-hdr']}>
              <span className={styles['hero-card-title']}>Live Benchmark</span>
              <span className="badge badge-green"><div className={styles['pulse-dot']} /> Running</span>
            </div>
            {bars.map((b, i) => (
              <div className={styles['hero-bar']} key={i}>
                <span className={styles['hero-bar-label']}>{b.label}</span>
                <div className={styles['hero-bar-track']}>
                  <div className={`${styles['hero-bar-fill']} ${styles[`hero-bar-fill--${b.cls}`]}`} style={{ width: `${b.pct}%`, transitionDelay: `${i * 250}ms` }}>
                    {b.pct > 0 && <span>{b.pct}%</span>}
                  </div>
                </div>
              </div>
            ))}
          </div>
          <div className={`${styles['float-badge']} ${styles.tr}`}><Award size={16} style={{ color: '#F59E0B' }} /><span>Winner: <strong>Claude Sonnet 4</strong></span></div>
          <div className={`${styles['float-badge']} ${styles.bl}`}><div className={styles['pulse-dot']} /><span>3 evaluations running</span></div>
        </div>
      </section>

      <section className={styles.features}>
        <div className={styles['feat-header']}><h2>Everything you need to decide</h2><p>From connecting providers to comparing results — a complete evaluation workflow</p></div>
        <div className={styles['feat-grid']}>
          {features.map((f, i) => (
            <div className={styles['feat-card']} key={i}>
              <div className={`${styles['feat-icon']} ${styles[`feat-icon--${f.cls}`]}`}>{f.icon}</div>
              <h3>{f.title}</h3>
              <p>{f.desc}</p>
            </div>
          ))}
        </div>
      </section>

      <section className={styles['stats-section']}>
        {[{ v: '40+', l: 'Models supported' }, { v: '6', l: 'Benchmark suites' }, { v: '12K+', l: 'Evaluation tasks' }, { v: '<5min', l: 'Average eval time' }].map((s, i) => (
          <div className={styles['stat-box']} key={i}><div className={styles['stat-val']}>{s.v}</div><div className={styles['stat-lbl']}>{s.l}</div></div>
        ))}
      </section>

      <section className={styles['cta-section']}>
        <div className={styles['cta-box']}>
          <h2>Ready to evaluate with confidence?</h2>
          <p>Connect your first provider and run a benchmark in under five minutes.</p>
          <button className={styles['btn-primary']} onClick={handleSignIn}>Get Started Free <ArrowRight size={16} /></button>
        </div>
      </section>

      <footer className={styles['l-footer']}>&copy; 2026 SemcoEval &middot; Privacy &middot; Terms &middot; Documentation</footer>
    </div>
  );
}













//AppShell.module.scss
@use '../../styles/_variables' as *;

.app-shell {
  display: flex;
  height: 100vh;
  background: $bg;

  &__main {
    flex: 1;
    overflow-y: auto;
    background: $bg;
  }
}
















//AppShell.tsx
import { Outlet } from 'react-router-dom';
import Sidebar from './Sidebar';
import styles from './AppShell.module.scss';

export default function AppShell() {
  return (
    <div className={styles['app-shell']}>
      <Sidebar />
      <main className={styles['app-shell__main']}>
        <Outlet />
      </main>
    </div>
  );
}















//Sidebar.module.scss
@use '../../styles/_variables' as *;

.sidebar {
  width: 256px; background: $surface; border-right: 1px solid $border;
  display: flex; flex-direction: column; flex-shrink: 0;

  &__logo {
    padding: 24px; display: flex; align-items: center; gap: 10px;
    font-family: $font-display; font-size: 18px; font-weight: 700;
    color: $text-primary; border-bottom: 1px solid $border-light;
  }
  &__mark {
    width: 30px; height: 30px; background: $grad-primary; border-radius: 9px;
    display: flex; align-items: center; justify-content: center; color: #fff;
    font-size: 14px; box-shadow: 0 2px 8px rgba(99, 102, 241, .25);
  }
  &__nav { flex: 1; padding: 14px 12px; display: flex; flex-direction: column; gap: 2px; }
  &__section {
    font-size: 11px; font-weight: 700; color: $text-muted; text-transform: uppercase;
    letter-spacing: 1.5px; padding: 20px 14px 8px; font-family: $font-display;
  }
  &__foot { padding: 16px; border-top: 1px solid $border-light; }
  &__user {
    display: flex; align-items: center; gap: 10px; padding: 10px; border-radius: 12px;
    transition: background .15s; cursor: pointer;
  }
  &__user:hover { background: $surface-alt; }
  &__avatar {
    width: 36px; height: 36px; border-radius: 10px; background: $grad-primary;
    display: flex; align-items: center; justify-content: center; font-size: 14px; font-weight: 700; color: #fff;
  }
  &__user-info { flex: 1; }
  &__user-name { font-size: 13px; font-weight: 600; color: $text-primary; }
  &__user-email { font-size: 11px; color: $text-muted; }
}

.nav-item {
  display: flex; align-items: center; gap: 12px; padding: 11px 14px; border-radius: 12px;
  font-size: 14px; font-weight: 500; color: $text-secondary; cursor: pointer; transition: all .2s;
  border: none; background: none; width: 100%; text-align: left; text-decoration: none;
}
.nav-item:hover { color: $text-primary; background: $surface-alt; }
.nav-item.active {
  color: $indigo; background: $indigo-pale; font-weight: 600; box-shadow: inset 3px 0 0 $indigo;
  svg { color: $indigo; }
}

@media (max-width: 768px) {
  .sidebar { display: none; }
}



















//Sidebar.tsx
import { NavLink } from 'react-router-dom';
import { Home, Link2, Cpu, BookOpen, Play, FlaskConical, GitCompare, LogOut } from 'lucide-react';
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














//AddCustomModelDrawer.module.scss
@use '../../styles/_variables' as *;

.drawer-overlay {
  position: fixed; inset: 0; background: rgba(17, 24, 39, .4); z-index: 100;
  display: flex; justify-content: flex-end;
}
.drawer {
  width: 420px; max-width: 100%; height: 100%; background: $surface; box-shadow: $shadow-4;
  display: flex; flex-direction: column; animation: drawerIn .25s ease both;
}
@keyframes drawerIn { from { transform: translateX(24px); opacity: 0; } to { transform: translateX(0); opacity: 1; } }
.drawer__hdr {
  display: flex; justify-content: space-between; align-items: center; padding: 20px 24px; border-bottom: 1px solid $border-light;
  h2 { font-size: 18px; font-weight: 700; }
}
.drawer__close { background: none; border: none; cursor: pointer; color: $text-muted; }
.drawer__body { flex: 1; overflow-y: auto; padding: 24px; }
.drawer__foot { display: flex; justify-content: flex-end; gap: 10px; padding: 16px 24px; border-top: 1px solid $border-light; }
















//AddCustomModelDrawer.tsx
import { useState } from 'react';
import { X } from 'lucide-react';
import type { CustomModelRequest } from '../../types';
import styles from './AddCustomModelDrawer.module.scss';

interface Props {
  onClose: () => void;
  onSubmit: (payload: CustomModelRequest) => void;
  submitting: boolean;
}

const initial: CustomModelRequest = {
  base_url: '',
  category: 'General',
  api_key: '',
  model_id: '',
  name: '',
  context_window: 128000,
  description: '',
};

export default function AddCustomModelDrawer({ onClose, onSubmit, submitting }: Props) {
  const [form, setForm] = useState<CustomModelRequest>(initial);

  const set = <K extends keyof CustomModelRequest>(key: K, value: CustomModelRequest[K]) =>
    setForm((f) => ({ ...f, [key]: value }));

  const valid = form.name.trim() && form.model_id.trim() && form.base_url.trim() && form.api_key.trim();

  return (
    <div className={styles['drawer-overlay']} onClick={onClose}>
      <div className={styles.drawer} onClick={(e) => e.stopPropagation()}>
        <div className={styles['drawer__hdr']}>
          <h2>Register Custom Model</h2>
          <button className={styles['drawer__close']} onClick={onClose}><X size={18} /></button>
        </div>
        <div className={styles['drawer__body']}>
          <div className="fg"><label className="fl">Display Name</label><input className="fi" value={form.name} onChange={(e) => set('name', e.target.value)} placeholder="e.g. Internal Llama Endpoint" /></div>
          <div className="fg"><label className="fl">Model ID</label><input className="fi" value={form.model_id} onChange={(e) => set('model_id', e.target.value)} placeholder="e.g. llama-3.1-405b-instruct" /></div>
          <div className="fg"><label className="fl">Base URL</label><input className="fi" value={form.base_url} onChange={(e) => set('base_url', e.target.value)} placeholder="https://…" /></div>
          <div className="fg"><label className="fl">API Key</label><input className="fi" type="password" value={form.api_key} onChange={(e) => set('api_key', e.target.value)} placeholder="sk-…" /></div>
          <div className="fg"><label className="fl">Category</label>
            <select className="fi" value={form.category} onChange={(e) => set('category', e.target.value)}>
              {['General', 'Efficient', 'Open Weight', 'RAG'].map((c) => <option key={c}>{c}</option>)}
            </select>
          </div>
          <div className="fg"><label className="fl">Context Window</label><input className="fi" type="number" value={form.context_window} onChange={(e) => set('context_window', Number(e.target.value))} /></div>
          <div className="fg"><label className="fl">Description <span className="opt">(optional)</span></label><textarea className="fi" rows={3} value={form.description} onChange={(e) => set('description', e.target.value)} /></div>
        </div>
        <div className={styles['drawer__foot']}>
          <button className="btn btn-ghost" onClick={onClose}>Cancel</button>
          <button className="btn btn-ind" disabled={!valid || submitting} onClick={() => onSubmit(form)}>{submitting ? 'Registering…' : 'Register Model'}</button>
        </div>
      </div>
    </div>
  );
}

















//ModelCatalog.module.scss
.model-catalog__loading { color: #6B7280; font-size: 13px; margin-bottom: 16px; }








//ModelCatalog.tsx
import { useEffect, useMemo, useState } from 'react';
import { Search, Plus } from 'lucide-react';
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
    <div className="page-enter">
      <div className="pg-hdr"><h1>Model Catalog</h1><p>All models across connected providers</p></div>
      <div className="pg-body">
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

        {status === 'loading' && <div className={styles['model-catalog__loading']}>Loading models…</div>}

        <div className="tw">
          <table className="tbl">
            <thead>
              <tr><th>Model</th><th>Provider</th><th>Capabilities</th><th>Context</th><th>Price (in/out)</th><th>Accuracy</th><th>Status</th></tr>
            </thead>
            <tbody>
              {filtered.map((m) => (
                <tr key={m.id}>
                  <td style={{ fontWeight: 700 }}>{m.name}</td>
                  <td style={{ color: '#6B7280' }}>{providerName(m.provider_id)}</td>
                  <td>{m.capabilities.map((c) => <span key={c} className="tag tag-ind">{c}</span>)}</td>
                  <td style={{ fontFamily: "'JetBrains Mono',monospace", fontSize: 13 }}>{m.context_window.toLocaleString()}</td>
                  <td style={{ fontFamily: "'JetBrains Mono',monospace", fontSize: 13, color: '#6B7280' }}>
                    {m.input_price != null ? `$${m.input_price.toFixed(2)}` : '—'} / {m.output_price != null ? `$${m.output_price.toFixed(2)}` : '—'}
                  </td>
                  <td>
                    <span style={{ fontFamily: "'JetBrains Mono',monospace", fontWeight: 700, color: (m.accuracy_score || 0) >= 90 ? '#10B981' : '#111827' }}>
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

















//Providers.module.scss
@use '../../styles/_variables' as *;

.providers {
  &__loading { display: flex; align-items: center; gap: 8px; color: $text-secondary; font-size: 13px; margin-bottom: 16px; }
  &__icon {
    background: $indigo-pale; color: $indigo; font-size: 18px; font-weight: 700;
    img { width: 24px; height: 24px; object-fit: contain; }
  }
  &__key-form { display: flex; gap: 8px; margin-top: 4px; }
  &__key-actions { display: flex; gap: 6px; }
}













//Providers.tsx
import { useEffect, useState } from 'react';
import { Search, Check, Plus, Settings, Unlink, Loader2 } from 'lucide-react';
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
    <div className="page-enter">
      <div className="pg-hdr"><h1>Providers</h1><p>Manage your AI provider connections</p></div>
      <div className="pg-body">
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

        {status === 'loading' && <div className={styles['providers__loading']}><Loader2 size={18} style={{ animation: 'spin 1.5s linear infinite' }} /> Loading providers…</div>}

        <div className="cards-grid">
          {filtered.map((p) => (
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




















//TestSuites.module.scss
.suites__loading { color: #6B7280; font-size: 13px; margin-bottom: 16px; }












//TestSuites.tsx
import { useEffect, useMemo, useState } from 'react';
import { Search, Eye } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { fetchBenchmarks } from '../../store/slices/benchmarksSlice';
import styles from './TestSuites.module.scss';

export default function TestSuites() {
  const dispatch = useAppDispatch();
  const { items, status } = useAppSelector((s) => s.benchmarks);
  const [search, setSearch] = useState('');
  const [typeFilter, setTypeFilter] = useState('All');

  useEffect(() => {
    dispatch(fetchBenchmarks());
  }, [dispatch]);

  const types = useMemo(() => ['All', ...new Set(items.map((s) => s.type))], [items]);
  const filtered = items.filter((s) => {
    if (typeFilter !== 'All' && s.type !== typeFilter) return false;
    return !search || s.name.toLowerCase().includes(search.toLowerCase());
  });

  return (
    <div className="page-enter">
      <div className="pg-hdr"><h1>Test Suites</h1><p>Benchmark suites for structured evaluation</p></div>
      <div className="pg-body">
        <div className="toolbar">
          <div className="search-box"><Search size={16} color="#9CA3AF" /><input placeholder="Search suites…" value={search} onChange={(e) => setSearch(e.target.value)} /></div>
          <div className="pills">{types.map((t) => <button key={t} className={`pill ${typeFilter === t ? 'on' : ''}`} onClick={() => setTypeFilter(t)}>{t}</button>)}</div>
        </div>

        {status === 'loading' && <div className={styles['suites__loading']}>Loading test suites…</div>}

        <div className="cards-grid">
          {filtered.map((s) => (
            <div className="card" key={s.name}>
              <div className="card-hdr">
                <div>
                  <div className="card-title">{s.name}</div>
                  <div style={{ marginTop: 6 }}><span className="tag tag-amb">{s.type}</span></div>
                </div>
                <div style={{ textAlign: 'right' }}>
                  <div style={{ fontFamily: "'JetBrains Mono',monospace", fontSize: 26, fontWeight: 700 }}>{s.task_count.toLocaleString()}</div>
                  <div style={{ fontSize: 11, color: '#9CA3AF', fontWeight: 600 }}>tasks</div>
                </div>
              </div>
              <div className="card-desc">{s.description}</div>
              <div style={{ marginBottom: 4 }}>{s.required_capabilities.map((c) => <span key={c} className="tag tag-ind">{c}</span>)}</div>
              <div className="card-foot">
                <span style={{ fontSize: 12, color: '#9CA3AF' }}>{s.huggingface_dataset}</span>
                <button className="btn btn-sm btn-ghost"><Eye size={14} /> View Tasks</button>
              </div>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
}
















//redux.ts
import { useDispatch, useSelector } from 'react-redux';
import type { TypedUseSelectorHook } from 'react-redux';
import type { RootState, AppDispatch } from '../store/store';

export const useAppDispatch = () => useDispatch<AppDispatch>();
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;













//browser.ts
import { setupWorker } from 'msw/browser';
import { handlers } from './handlers';

export const worker = setupWorker(...handlers);















//data.ts
// Seed data for the mock API layer — shaped to exactly match the response
// contracts in the API reference doc, so swapping MSW off for a real
// backend later is a no-op for every component/slice/type.
import type {
  Provider, Model, Benchmark, EvaluationStatusResponse,
} from '../types';

export const mockProviders: Provider[] = [
  { id: 'openai', name: 'OpenAI', description: 'GPT family — industry-leading multimodal models', logo_url: null, base_url: 'https://api.openai.com/v1', url_template: null, model_count: 8, status: 'connected' },
  { id: 'anthropic', name: 'Anthropic', description: 'Claude family — safety-focused, high-capability models', logo_url: null, base_url: 'https://api.anthropic.com/v1', url_template: null, model_count: 5, status: 'connected' },
  { id: 'google-deepmind', name: 'Google DeepMind', description: 'Gemini & PaLM — massive context, multimodal', logo_url: null, base_url: 'https://generativelanguage.googleapis.com/v1', url_template: null, model_count: 6, status: 'not_connected' },
  { id: 'meta-ai', name: 'Meta AI', description: 'Llama — open-weight models for self-hosting', logo_url: null, base_url: null, url_template: null, model_count: 4, status: 'not_connected' },
  { id: 'mistral-ai', name: 'Mistral AI', description: 'Mistral & Mixtral — efficient European models', logo_url: null, base_url: 'https://api.mistral.ai/v1', url_template: null, model_count: 3, status: 'connected' },
  { id: 'cohere', name: 'Cohere', description: 'Command & Embed — enterprise RAG specialists', logo_url: null, base_url: 'https://api.cohere.ai/v1', url_template: null, model_count: 3, status: 'not_connected' },
];

export const mockModels: Model[] = [
  { id: 'gpt-4o', name: 'GPT-4o', provider_id: 'openai', category: 'General', capabilities: ['Text', 'Vision', 'Code'], context_window: 128000, input_price: 2.5, output_price: 10, accuracy_score: 92.4, agent_score: 88, is_active: true, base_url: null },
  { id: 'claude-sonnet-4', name: 'Claude Sonnet 4', provider_id: 'anthropic', category: 'General', capabilities: ['Text', 'Vision', 'Code', 'Analysis'], context_window: 200000, input_price: 3, output_price: 15, accuracy_score: 94.1, agent_score: 92, is_active: true, base_url: null },
  { id: 'gemini-2-5-pro', name: 'Gemini 2.5 Pro', provider_id: 'google-deepmind', category: 'General', capabilities: ['Text', 'Vision', 'Code'], context_window: 1000000, input_price: 1.25, output_price: 5, accuracy_score: 91.8, agent_score: 85, is_active: false, base_url: null },
  { id: 'gpt-4o-mini', name: 'GPT-4o mini', provider_id: 'openai', category: 'Efficient', capabilities: ['Text', 'Code'], context_window: 128000, input_price: 0.15, output_price: 0.6, accuracy_score: 87.2, agent_score: 79, is_active: true, base_url: null },
  { id: 'claude-haiku-4', name: 'Claude Haiku 4', provider_id: 'anthropic', category: 'Efficient', capabilities: ['Text', 'Code'], context_window: 200000, input_price: 0.25, output_price: 1.25, accuracy_score: 86.9, agent_score: 77, is_active: true, base_url: null },
  { id: 'mistral-large', name: 'Mistral Large', provider_id: 'mistral-ai', category: 'General', capabilities: ['Text', 'Code', 'Analysis'], context_window: 128000, input_price: 2, output_price: 6, accuracy_score: 89.7, agent_score: 81, is_active: true, base_url: null },
  { id: 'llama-3-1-405b', name: 'Llama 3.1 405B', provider_id: 'meta-ai', category: 'Open Weight', capabilities: ['Text', 'Code'], context_window: 128000, input_price: 0, output_price: 0, accuracy_score: 88.5, agent_score: 80, is_active: false, base_url: null },
  { id: 'command-r-plus', name: 'Command R+', provider_id: 'cohere', category: 'RAG', capabilities: ['Text', 'RAG', 'Search'], context_window: 128000, input_price: 3, output_price: 15, accuracy_score: 85.3, agent_score: 75, is_active: false, base_url: null },
];

export const mockBenchmarks: Benchmark[] = [
  { name: 'MMLU Pro', description: 'Massive multitask language understanding with harder questions', tasks: [{ name: 'Sample task 1', value: 'mmlu-pro-1' }], task_count: 12032, required_capabilities: ['Text', 'Reasoning'], huggingface_dataset: 'TIGER-Lab/MMLU-Pro', type: 'Knowledge' },
  { name: 'HumanEval+', description: 'Code generation with extended test cases and edge coverage', tasks: [{ name: 'Sample task 1', value: 'humaneval-plus-1' }], task_count: 820, required_capabilities: ['Code'], huggingface_dataset: 'evalplus/humanevalplus', type: 'Code' },
  { name: 'GPQA Diamond', description: 'Graduate-level science questions verified by domain experts', tasks: [{ name: 'Sample task 1', value: 'gpqa-diamond-1' }], task_count: 448, required_capabilities: ['Reasoning', 'Analysis'], huggingface_dataset: 'Idavidrein/gpqa', type: 'Science' },
  { name: 'MT-Bench', description: 'Multi-turn conversational evaluation across 8 domains', tasks: [{ name: 'Sample task 1', value: 'mt-bench-1' }], task_count: 160, required_capabilities: ['Text', 'Reasoning'], huggingface_dataset: 'lmsys/mt_bench_human_judgments', type: 'Conversation' },
  { name: 'BigCodeBench', description: 'Complex programming tasks requiring deep library knowledge', tasks: [{ name: 'Sample task 1', value: 'bigcodebench-1' }], task_count: 1140, required_capabilities: ['Code', 'Analysis'], huggingface_dataset: 'bigcode/bigcodebench', type: 'Code' },
  { name: 'MATH-500', description: 'Competition-level mathematics problems with step verification', tasks: [{ name: 'Sample task 1', value: 'math-500-1' }], task_count: 500, required_capabilities: ['Reasoning'], huggingface_dataset: 'HuggingFaceH4/MATH-500', type: 'Math' },
];

export const mockAllMetrics = ['Accuracy', 'Speed', 'Cost', 'Consistency', 'Coherence', 'Factuality', 'Creativity', 'Safety'];
export const mockCustomAgentMetrics = ['Tool Call Accuracy', 'Task Completion', 'Step Efficiency'];

// In-memory evaluation run store used only by the mock handlers, so
// POST /evaluations -> POST /evaluations/{id}/start -> GET .../status
// behaves like a real, stateful backend across calls.
interface MockRun {
  id: string;
  progress: number;
  total: number;
  startedAt: number;
}
export const mockRuns = new Map<string, MockRun>();

export function advanceMockRun(id: string): EvaluationStatusResponse {
  const run = mockRuns.get(id);
  if (!run) {
    return { status: 'failed', progress: 0, total: 0, celery_state: 'FAILURE', error_message: 'Evaluation not found' };
  }
  const elapsedMs = Date.now() - run.startedAt;
  // Simulate ~12 seconds to completion, ticking progress deterministically off wall time.
  const pct = Math.min(1, elapsedMs / 12000);
  run.progress = Math.round(pct * run.total);

  if (pct >= 1) {
    return { status: 'completed', progress: run.total, total: run.total, celery_state: 'SUCCESS', error_message: null };
  }
  return { status: 'running', progress: run.progress, total: run.total, celery_state: 'STARTED', error_message: null };
}














//handlers.ts
import { http, HttpResponse, delay } from 'msw';
import {
  mockProviders, mockModels, mockBenchmarks, mockAllMetrics, mockCustomAgentMetrics,
  mockRuns, advanceMockRun,
} from './data';
import type {
  SsoLoginRequest, SsoLoginResponse,
  ConnectProviderRequest, ConnectProviderResponse, DisconnectProviderResponse,
  CustomModelRequest,
  BenchmarksResponse, MetricsResponse,
  CreateEvaluationRequest, CreateEvaluationResponse,
} from '../types';

const API_BASE = import.meta.env.VITE_API_BASE_URL || '/api';
const url = (path: string) => `${API_BASE}${path}`;

let providers = mockProviders.map((p) => ({ ...p }));
let models = mockModels.map((m) => ({ ...m }));
let runIdCounter = 1;

export const handlers = [
  // ---------- Auth ----------
  http.post(url('/sso_login'), async ({ request }) => {
    await delay(400);
    const body = (await request.json()) as SsoLoginRequest;
    const response: SsoLoginResponse = {
      status: 'ok',
      message: 'Signed in',
      result: {
        token: `mock-token-${body.token}-${Date.now()}`,
        username: 'jane.doe',
        email: 'jane@semco.ai',
        language: 'en',
        profile_name: 'Jane Doe',
      },
    };
    return HttpResponse.json(response);
  }),

  // ---------- Providers ----------
  http.get(url('/providers'), async () => {
    await delay(300);
    return HttpResponse.json({ providers });
  }),

  http.post(url('/providers/:providerId/connect'), async ({ params, request }) => {
    await delay(500);
    const { providerId } = params as { providerId: string };
    const body = (await request.json()) as ConnectProviderRequest;
    if (!body.api_key?.trim()) {
      return HttpResponse.json({ message: 'api_key is required' }, { status: 400 });
    }
    providers = providers.map((p) => (p.id === providerId ? { ...p, status: 'connected' } : p));
    const response: ConnectProviderResponse = {
      status: 'connected',
      provider_id: providerId,
      models_synced: providers.find((p) => p.id === providerId)?.model_count ?? 0,
    };
    return HttpResponse.json(response);
  }),

  http.delete(url('/providers/:providerId/disconnect'), async ({ params }) => {
    await delay(300);
    const { providerId } = params as { providerId: string };
    providers = providers.map((p) => (p.id === providerId ? { ...p, status: 'not_connected' } : p));
    const response: DisconnectProviderResponse = { status: 'disconnected', provider_id: providerId };
    return HttpResponse.json(response);
  }),

  // ---------- Models ----------
  http.get(url('/models'), async () => {
    await delay(300);
    return HttpResponse.json({ models });
  }),

  http.post(url('/models/custom'), async ({ request }) => {
    await delay(500);
    const body = (await request.json()) as CustomModelRequest;
    models = [
      ...models,
      {
        id: body.model_id || `custom-${Date.now()}`,
        name: body.name,
        provider_id: 'custom',
        category: body.category,
        capabilities: ['Text'],
        context_window: body.context_window,
        input_price: null,
        output_price: null,
        accuracy_score: null,
        agent_score: null,
        is_active: true,
        base_url: body.base_url,
      },
    ];
    return new HttpResponse(null, { status: 200 });
  }),

  // ---------- Benchmarks ----------
  http.get(url('/benchmarks'), async () => {
    await delay(300);
    const response: BenchmarksResponse = { benchmarks: mockBenchmarks, total: mockBenchmarks.length };
    return HttpResponse.json(response);
  }),

  // ---------- Metrics ----------
  http.get(url('/metrics'), async () => {
    await delay(200);
    const response: MetricsResponse = { all_metrics: mockAllMetrics, custom_agent_metrics: mockCustomAgentMetrics };
    return HttpResponse.json(response);
  }),

  // ---------- Evaluations ----------
  http.post(url('/evaluations'), async ({ request }) => {
    await delay(500);
    const body = (await request.json()) as CreateEvaluationRequest;
    const id = `eval-${runIdCounter++}`;
    const benchmark = mockBenchmarks.find((b) => b.name === body.benchmark);
    mockRuns.set(id, { id, progress: 0, total: benchmark?.task_count ?? 100, startedAt: 0 });
    const response: CreateEvaluationResponse = { id, evaluation_id: id };
    return HttpResponse.json(response);
  }),

  http.post(url('/evaluations/:evaluationId/start'), async ({ params }) => {
    await delay(300);
    const { evaluationId } = params as { evaluationId: string };
    const run = mockRuns.get(evaluationId);
    if (run) run.startedAt = Date.now();
    return new HttpResponse(null, { status: 200 });
  }),

  http.get(url('/evaluations/:evaluationId/status'), async ({ params }) => {
    await delay(150);
    const { evaluationId } = params as { evaluationId: string };
    return HttpResponse.json(advanceMockRun(evaluationId));
  }),
];

















//AppRoutes.tsx
import { Routes, Route, Navigate } from 'react-router-dom';
import Landing from '../components/landing/Landing';
import AppShell from '../components/layout/AppShell';
import ProtectedRoute from './ProtectedRoute';
import Dashboard from '../components/dashboard/Dashboard';
import Providers from '../components/providers/Providers';
import ModelCatalog from '../components/models/ModelCatalog';
import TestSuites from '../components/suites/TestSuites';
import NewEvaluation from '../components/evaluations/NewEvaluation';
import Evaluations from '../components/evaluations/Evaluations';
import EvaluationDetail from '../components/evaluations/EvaluationDetail';
import Comparison from '../components/comparison/Comparison';

const MOCKS_ENABLED = import.meta.env.VITE_ENABLE_MOCKS === 'true';

export default function AppRoutes() {
  return (
    <Routes>
      {/* In mock mode, main.tsx already seeds a session before render, so
          skip the landing/SSO page entirely and land straight in the app. */}
      <Route path="/" element={MOCKS_ENABLED ? <Navigate to="/app/dashboard" replace /> : <Landing />} />

      <Route element={<ProtectedRoute />}>
        <Route path="/app" element={<AppShell />}>
          <Route index element={<Navigate to="dashboard" replace />} />
          <Route path="dashboard" element={<Dashboard />} />
          <Route path="providers" element={<Providers />} />
          <Route path="models" element={<ModelCatalog />} />
          <Route path="suites" element={<TestSuites />} />
          <Route path="new-eval" element={<NewEvaluation />} />
          <Route path="evaluations" element={<Evaluations />} />
          <Route path="evaluations/:id" element={<EvaluationDetail />} />
          <Route path="comparison" element={<Comparison />} />
        </Route>
      </Route>

      <Route path="*" element={<Navigate to="/" replace />} />
    </Routes>
  );
}














//ProtectedRoute.tsx
import { Navigate, Outlet } from 'react-router-dom';
import { tokenStorage } from '../api/axiosInstance';

export default function ProtectedRoute() {
  const hasToken = Boolean(tokenStorage.get());
  if (!hasToken) return <Navigate to="/" replace />;
  return <Outlet />;
}
















//authSlice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import type { PayloadAction } from '@reduxjs/toolkit';
import { authApi } from '../../api/endpoints/auth';
import { tokenStorage } from '../../api/axiosInstance';
import type { SsoLoginRequest, SsoLoginResult } from '../../types';

interface AuthState {
  user: SsoLoginResult | null;
  status: 'idle' | 'loading' | 'succeeded' | 'failed';
  error: string | null;
}

const initialState: AuthState = {
  user: null,
  status: 'idle',
  error: null,
};

export const ssoLogin = createAsyncThunk(
  'auth/ssoLogin',
  async (payload: SsoLoginRequest) => {
    const res = await authApi.ssoLogin(payload);
    return res.result;
  }
);

const authSlice = createSlice({
  name: 'auth',
  initialState,
  reducers: {
    logout(state) {
      tokenStorage.clear();
      state.user = null;
      state.status = 'idle';
      state.error = null;
    },
    // Used only when VITE_ENABLE_MOCKS=true to skip the SSO flow entirely —
    // see main.tsx. Never dispatched against a real backend.
    hydrateMockSession(state, action: PayloadAction<SsoLoginResult>) {
      state.status = 'succeeded';
      state.user = action.payload;
      state.error = null;
      tokenStorage.set(action.payload.token);
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(ssoLogin.pending, (state) => {
        state.status = 'loading';
        state.error = null;
      })
      .addCase(ssoLogin.fulfilled, (state, action: PayloadAction<SsoLoginResult>) => {
        state.status = 'succeeded';
        state.user = action.payload;
        tokenStorage.set(action.payload.token);
      })
      .addCase(ssoLogin.rejected, (state, action) => {
        state.status = 'failed';
        state.error = action.error.message || 'Login failed';
      });
  },
});

export const { logout, hydrateMockSession } = authSlice.actions;
export default authSlice.reducer;



















//benchmarksSlice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { benchmarksApi } from '../../api/endpoints/benchmarks';
import type { Benchmark } from '../../types';

interface BenchmarksState {
  items: Benchmark[];
  total: number;
  status: 'idle' | 'loading' | 'succeeded' | 'failed';
  error: string | null;
}

const initialState: BenchmarksState = {
  items: [],
  total: 0,
  status: 'idle',
  error: null,
};

export const fetchBenchmarks = createAsyncThunk('benchmarks/fetchAll', () => benchmarksApi.list());

const benchmarksSlice = createSlice({
  name: 'benchmarks',
  initialState,
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(fetchBenchmarks.pending, (state) => {
        state.status = 'loading';
      })
      .addCase(fetchBenchmarks.fulfilled, (state, action) => {
        state.status = 'succeeded';
        state.items = action.payload.benchmarks;
        state.total = action.payload.total;
      })
      .addCase(fetchBenchmarks.rejected, (state, action) => {
        state.status = 'failed';
        state.error = action.error.message || 'Failed to load test suites';
      });
  },
});

export default benchmarksSlice.reducer;

















//evaluationsSlice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import type { PayloadAction } from '@reduxjs/toolkit';
import { evaluationsApi, pollEvaluationStatus } from '../../api/endpoints/evaluations';
import type {
  CreateEvaluationRequest,
  EvaluationDraft,
  EvaluationStatusResponse,
} from '../../types';

// The API only exposes create/start/status — there is no GET /evaluations list
// endpoint in the reference, so run history is tracked client-side as each
// evaluation is launched, and enriched via polling.
export interface EvaluationRunRecord {
  id: string;
  name: string;
  eval_type: string;
  benchmark: string | null;
  model_ids: string[];
  created_at: string;
  status: EvaluationStatusResponse['status'];
  progress: number;
  total: number;
  error_message: string | null;
}

interface EvaluationsState {
  draft: EvaluationDraft;
  runs: EvaluationRunRecord[];
  launching: boolean;
  launchError: string | null;
}

const initialDraft: EvaluationDraft = {
  name: '',
  eval_type: '',
  selProviders: [],
  selModels: [],
  selBenchmark: null,
  selMetrics: ['Accuracy', 'Speed', 'Cost'],
};

const initialState: EvaluationsState = {
  draft: initialDraft,
  runs: [],
  launching: false,
  launchError: null,
};

export const launchEvaluation = createAsyncThunk(
  'evaluations/launch',
  async (payload: CreateEvaluationRequest, { dispatch }) => {
    const id = await evaluationsApi.createAndStart(payload);

    // Begin polling; each update is written back into the store.
    pollEvaluationStatus(id, (status) => {
      dispatch(updateRunStatus({ id, status }));
    });

    return {
      id,
      name: payload.name,
      eval_type: payload.eval_type,
      benchmark: payload.benchmark || null,
      model_ids: payload.model_ids,
      created_at: new Date().toISOString(),
      status: 'pending' as const,
      progress: 0,
      total: 0,
      error_message: null,
    };
  }
);

const evaluationsSlice = createSlice({
  name: 'evaluations',
  initialState,
  reducers: {
    setDraft(state, action: PayloadAction<Partial<EvaluationDraft>>) {
      state.draft = { ...state.draft, ...action.payload };
    },
    resetDraft(state) {
      state.draft = initialDraft;
    },
    updateRunStatus(
      state,
      action: PayloadAction<{ id: string; status: EvaluationStatusResponse }>
    ) {
      const run = state.runs.find((r) => r.id === action.payload.id);
      if (run) {
        run.status = action.payload.status.status;
        run.progress = action.payload.status.progress;
        run.total = action.payload.status.total;
        run.error_message = action.payload.status.error_message;
      }
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(launchEvaluation.pending, (state) => {
        state.launching = true;
        state.launchError = null;
      })
      .addCase(launchEvaluation.fulfilled, (state, action) => {
        state.launching = false;
        state.runs.unshift(action.payload);
        state.draft = initialDraft;
      })
      .addCase(launchEvaluation.rejected, (state, action) => {
        state.launching = false;
        state.launchError = action.error.message || 'Failed to launch evaluation';
      });
  },
});

export const { setDraft, resetDraft, updateRunStatus } = evaluationsSlice.actions;
export default evaluationsSlice.reducer;























//metricsSlice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { metricsApi } from '../../api/endpoints/metrics';

interface MetricsState {
  allMetrics: string[];
  customAgentMetrics: string[];
  status: 'idle' | 'loading' | 'succeeded' | 'failed';
  error: string | null;
}

const initialState: MetricsState = {
  allMetrics: [],
  customAgentMetrics: [],
  status: 'idle',
  error: null,
};

export const fetchMetrics = createAsyncThunk('metrics/fetchAll', () => metricsApi.list());

const metricsSlice = createSlice({
  name: 'metrics',
  initialState,
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(fetchMetrics.pending, (state) => {
        state.status = 'loading';
      })
      .addCase(fetchMetrics.fulfilled, (state, action) => {
        state.status = 'succeeded';
        state.allMetrics = action.payload.all_metrics;
        state.customAgentMetrics = action.payload.custom_agent_metrics;
      })
      .addCase(fetchMetrics.rejected, (state, action) => {
        state.status = 'failed';
        state.error = action.error.message || 'Failed to load metrics';
      });
  },
});

export default metricsSlice.reducer;


















//modelsSlice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { modelsApi } from '../../api/endpoints/models';
import type { Model, CustomModelRequest } from '../../types';

interface ModelsState {
  items: Model[];
  status: 'idle' | 'loading' | 'succeeded' | 'failed';
  error: string | null;
  creating: boolean;
}

const initialState: ModelsState = {
  items: [],
  status: 'idle',
  error: null,
  creating: false,
};

export const fetchModels = createAsyncThunk('models/fetchAll', () => modelsApi.list());

export const createCustomModel = createAsyncThunk(
  'models/createCustom',
  async (payload: CustomModelRequest, { dispatch }) => {
    await modelsApi.createCustom(payload);
    // spec: no meaningful body returned, so refetch afterwards
    await dispatch(fetchModels());
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
        state.items = action.payload;
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
      });
  },
});

export default modelsSlice.reducer;



















//providersSlice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { providersApi } from '../../api/endpoints/providers';
import type { Provider, ConnectProviderRequest } from '../../types';

interface ProvidersState {
  items: Provider[];
  status: 'idle' | 'loading' | 'succeeded' | 'failed';
  error: string | null;
  mutatingId: string | null;
}

const initialState: ProvidersState = {
  items: [],
  status: 'idle',
  error: null,
  mutatingId: null,
};

export const fetchProviders = createAsyncThunk('providers/fetchAll', () => providersApi.list());

export const connectProvider = createAsyncThunk(
  'providers/connect',
  async ({ providerId, payload }: { providerId: string; payload: ConnectProviderRequest }) => {
    await providersApi.connect(providerId, payload);
    return providerId;
  }
);

export const disconnectProvider = createAsyncThunk(
  'providers/disconnect',
  async (providerId: string) => {
    await providersApi.disconnect(providerId);
    return providerId;
  }
);

const providersSlice = createSlice({
  name: 'providers',
  initialState,
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(fetchProviders.pending, (state) => {
        state.status = 'loading';
      })
      .addCase(fetchProviders.fulfilled, (state, action) => {
        state.status = 'succeeded';
        state.items = action.payload;
      })
      .addCase(fetchProviders.rejected, (state, action) => {
        state.status = 'failed';
        state.error = action.error.message || 'Failed to load providers';
      })
      .addCase(connectProvider.pending, (state, action) => {
        state.mutatingId = action.meta.arg.providerId;
      })
      .addCase(connectProvider.fulfilled, (state, action) => {
        state.mutatingId = null;
        const p = state.items.find((i) => i.id === action.payload);
        if (p) p.status = 'connected';
      })
      .addCase(connectProvider.rejected, (state) => {
        state.mutatingId = null;
      })
      .addCase(disconnectProvider.pending, (state, action) => {
        state.mutatingId = action.meta.arg;
      })
      .addCase(disconnectProvider.fulfilled, (state, action) => {
        state.mutatingId = null;
        const p = state.items.find((i) => i.id === action.payload);
        if (p) p.status = 'not_connected';
      })
      .addCase(disconnectProvider.rejected, (state) => {
        state.mutatingId = null;
      });
  },
});

export default providersSlice.reducer;




















//store.ts
import { configureStore } from '@reduxjs/toolkit';
import authReducer from './slices/authSlice';
import providersReducer from './slices/providersSlice';
import modelsReducer from './slices/modelsSlice';
import benchmarksReducer from './slices/benchmarksSlice';
import metricsReducer from './slices/metricsSlice';
import evaluationsReducer from './slices/evaluationsSlice';

export const store = configureStore({
  reducer: {
    auth: authReducer,
    providers: providersReducer,
    models: modelsReducer,
    benchmarks: benchmarksReducer,
    metrics: metricsReducer,
    evaluations: evaluationsReducer,
  },
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;


















//_variables.scss
// Design tokens — ported 1:1 from the original theme object (T)
$bg: #F7F8FC;
$surface: #FFFFFF;
$surface-alt: #F1F4F9;
$surface-hover: #F8F9FD;

$indigo: #6366F1;
$indigo-light: #818CF8;
$indigo-dark: #4F46E5;
$violet: #8B5CF6;
$indigo-pale: #EEF2FF;

$amber: #F59E0B;
$amber-dark: #D97706;
$amber-pale: #FFFBEB;

$emerald: #10B981;
$emerald-dark: #059669;
$emerald-pale: #ECFDF5;

$red: #EF4444;
$red-pale: #FEF2F2;
$sky: #0EA5E9;
$sky-pale: #F0F9FF;
$rose: #F43F5E;
$rose-pale: #FFF1F2;

$border: #E5E7EB;
$border-light: #F3F4F6;

$text-primary: #111827;
$text-secondary: #6B7280;
$text-muted: #9CA3AF;

$shadow-2: 0 2px 8px rgba(0, 0, 0, .06), 0 1px 2px rgba(0, 0, 0, .04);
$shadow-3: 0 8px 24px rgba(0, 0, 0, .08), 0 2px 6px rgba(0, 0, 0, .04);
$shadow-4: 0 16px 48px rgba(0, 0, 0, .1), 0 4px 12px rgba(0, 0, 0, .05);

$grad-primary: linear-gradient(135deg, #6366F1, #8B5CF6);
$grad-warm: linear-gradient(135deg, #F59E0B, #F97316);
$grad-cool: linear-gradient(135deg, #10B981, #0EA5E9);

$font-display: 'Space Grotesk', system-ui, sans-serif;
$font-body: 'DM Sans', system-ui, sans-serif;
$font-mono: 'JetBrains Mono', monospace;



















//global.scss
@use './_variables' as *;

@import url('https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@400;500;600;700&family=DM+Sans:ital,wght@0,400;0,500;0,600;0,700;1,400&family=JetBrains+Mono:wght@500;600;700&display=swap');

* { margin: 0; padding: 0; box-sizing: border-box; }

body {
  font-family: $font-body;
  background: $bg;
  color: $text-primary;
  -webkit-font-smoothing: antialiased;
}

h1, h2, h3, h4, h5 { font-family: $font-display; }

.page-enter { animation: pageIn .35s ease both; }
@keyframes pageIn { from { opacity: 0; transform: translateY(12px); } to { opacity: 1; transform: translateY(0); } }
@keyframes spin { to { transform: rotate(360deg); } }
@keyframes pulse { 0%, 100% { opacity: 1; } 50% { opacity: .4; } }
@keyframes float { 0%, 100% { transform: translateY(0); } 50% { transform: translateY(-8px); } }
@keyframes dotPulse { 0%, 100% { opacity: .15; } 50% { opacity: .35; } }
@keyframes toastIn { from { opacity: 0; transform: translateY(20px) scale(.95); } to { opacity: 1; transform: translateY(0) scale(1); } }

// ---- Shared primitives reused across feature components ----
.btn {
  display: inline-flex; align-items: center; gap: 6px;
  padding: 10px 20px; border-radius: 12px; font-size: 14px; font-weight: 600;
  cursor: pointer; transition: all .2s; border: none; font-family: $font-display;
}
.btn-ind { background: $grad-primary; color: #fff; box-shadow: 0 2px 8px rgba(99, 102, 241, .2); }
.btn-ind:hover { box-shadow: 0 4px 16px rgba(99, 102, 241, .3); transform: translateY(-1px); }
.btn-ghost { background: $surface; color: $text-secondary; border: 1px solid $border; }
.btn-ghost:hover { border-color: $indigo; color: $indigo; background: $indigo-pale; }
.btn-sm { padding: 7px 14px; font-size: 13px; border-radius: 10px; }
.btn-danger { background: $red-pale; color: $red; border: 1px solid transparent; }
.btn-danger:hover { background: #FEE2E2; }
.btn:disabled { opacity: .45; cursor: not-allowed; }

.badge {
  display: inline-flex; align-items: center; gap: 6px;
  padding: 5px 12px; border-radius: 100px; font-size: 12px; font-weight: 700; letter-spacing: .2px;
}
.badge-green { background: $emerald-pale; color: $emerald-dark; }
.badge-gray { background: $surface-alt; color: $text-muted; }
.badge-blue { background: $indigo-pale; color: $indigo; }
.badge-amber { background: $amber-pale; color: $amber-dark; }
.badge-run { background: $indigo-pale; color: $indigo; }

.tag { display: inline-block; padding: 3px 9px; border-radius: 7px; font-size: 11px; font-weight: 700; margin-right: 4px; margin-bottom: 4px; }
.tag-ind { background: $indigo-pale; color: $indigo; }
.tag-amb { background: $amber-pale; color: $amber-dark; }
.tag-em { background: $emerald-pale; color: $emerald-dark; }

.search-box {
  display: flex; align-items: center; gap: 8px;
  background: $surface; border: 1px solid $border; border-radius: 12px;
  padding: 10px 16px; min-width: 300px; transition: all .2s;
}
.search-box:focus-within { border-color: $indigo; box-shadow: 0 0 0 4px rgba(99, 102, 241, .08); }
.search-box input { border: none; outline: none; font-size: 14px; flex: 1; color: $text-primary; font-family: $font-body; background: transparent; }
.search-box input::placeholder { color: $text-muted; }

.pills { display: flex; gap: 6px; flex-wrap: wrap; }
.pill {
  padding: 7px 16px; border-radius: 100px; font-size: 13px; font-weight: 600;
  border: 1px solid $border; background: $surface; color: $text-secondary; cursor: pointer; transition: all .2s;
}
.pill:hover { border-color: $indigo; color: $indigo; }
.pill.on { background: $grad-primary; color: #fff; border-color: transparent; box-shadow: 0 2px 8px rgba(99, 102, 241, .25); }

.toolbar { display: flex; align-items: center; justify-content: space-between; margin-bottom: 24px; flex-wrap: wrap; gap: 12px; }

.cards-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(330px, 1fr)); gap: 16px; }
.card {
  background: $surface; border: 1px solid $border; border-radius: 16px; padding: 24px;
  transition: all .25s; position: relative; overflow: hidden;
}
.card:hover { border-color: rgba(99, 102, 241, .2); box-shadow: $shadow-3; transform: translateY(-2px); }
.card-hdr { display: flex; justify-content: space-between; align-items: flex-start; margin-bottom: 14px; }
.card-icon { width: 44px; height: 44px; border-radius: 12px; display: flex; align-items: center; justify-content: center; flex-shrink: 0; }
.card-title { font-family: $font-display; font-size: 16px; font-weight: 700; }
.card-desc { font-size: 13px; color: $text-secondary; line-height: 1.6; margin-bottom: 16px; }
.card-foot { display: flex; justify-content: space-between; align-items: center; margin-top: 16px; padding-top: 16px; border-top: 1px solid $border-light; }

.tw { background: $surface; border: 1px solid $border; border-radius: 16px; overflow: hidden; }
.tbl { width: 100%; border-collapse: collapse; }
.tbl th {
  text-align: left; padding: 14px 20px; font-size: 11px; font-weight: 700; color: $text-muted;
  text-transform: uppercase; letter-spacing: 1px; background: $surface-alt; border-bottom: 1px solid $border;
  font-family: $font-display;
}
.tbl td { padding: 14px 20px; font-size: 14px; border-bottom: 1px solid $border-light; }
.tbl tr:last-child td { border-bottom: none; }
.tbl tr { transition: background .15s; }
.tbl tr:hover td { background: $surface-hover; }
.tbl .winner td { background: $amber-pale; }
.tbl .winner td:first-child { box-shadow: inset 3px 0 0 $amber; }

.fg { margin-bottom: 22px; }
.fl { display: block; font-size: 13px; font-weight: 700; margin-bottom: 8px; }
.fl .opt { color: $text-muted; font-weight: 400; font-size: 12px; }
.fi {
  width: 100%; padding: 12px 16px; border: 1px solid $border; border-radius: 12px;
  font-size: 14px; font-family: $font-body; color: $text-primary; transition: all .2s; background: $surface;
}
.fi:focus { outline: none; border-color: $indigo; box-shadow: 0 0 0 4px rgba(99, 102, 241, .08); }
.fi::placeholder { color: $text-muted; }

.sel-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(200px, 1fr)); gap: 10px; }
.sel-opt {
  display: flex; align-items: center; gap: 12px; padding: 16px;
  border: 1.5px solid $border; border-radius: 14px; cursor: pointer; transition: all .2s; background: $surface;
}
.sel-opt:hover { border-color: $indigo-light; background: rgba(238, 242, 255, .4); }
.sel-opt.on { border-color: $indigo; background: $indigo-pale; }
.sel-chk {
  width: 22px; height: 22px; border: 2px solid $border; border-radius: 7px;
  display: flex; align-items: center; justify-content: center; flex-shrink: 0; transition: all .2s;
}
.sel-opt.on .sel-chk { background: $grad-primary; border-color: $indigo; color: #fff; box-shadow: 0 2px 4px rgba(99, 102, 241, .25); }
.sel-txt { font-size: 14px; font-weight: 600; }
.sel-sub { font-size: 12px; color: $text-secondary; margin-top: 2px; }

.toast {
  position: fixed; bottom: 32px; right: 32px; background: $surface; border: 1px solid $border;
  border-radius: 16px; padding: 18px 24px; display: flex; align-items: center; gap: 12px;
  box-shadow: $shadow-4; z-index: 999; animation: toastIn .4s ease both;
}

// Shared by Dashboard and Comparison — kept global since both need it.
.radar-wrap { display: flex; justify-content: center; align-items: center; padding: 20px; }

.pg-hdr { padding: 32px 40px 0; }
.pg-hdr h1 { font-size: 28px; font-weight: 700; letter-spacing: -.5px; }
.pg-hdr p { color: $text-secondary; font-size: 14px; margin-top: 4px; }
.pg-body { padding: 24px 40px 40px; }

@media (max-width: 768px) {
  .cards-grid { grid-template-columns: 1fr; }
  .pg-hdr, .pg-body { padding-left: 20px; padding-right: 20px; }
}


















//index.ts
// ---------- Auth ----------
export interface SsoLoginRequest {
  token: string;
  data: string;
}
export interface SsoLoginResult {
  token: string;
  username: string;
  email: string;
  language: string;
  profile_name: string;
}
export interface SsoLoginResponse {
  status: string;
  message: string;
  result: SsoLoginResult;
}

// ---------- Providers ----------
export interface Provider {
  id: string;
  name: string;
  description: string;
  logo_url: string | null;
  base_url: string | null;
  url_template: string | null;
  model_count: number;
  status: 'connected' | 'not_connected' | string;
}
export interface ConnectProviderRequest {
  api_key: string;
}
export interface ConnectProviderResponse {
  status: 'connected';
  provider_id: string;
  models_synced: number;
}
export interface DisconnectProviderResponse {
  status: 'disconnected';
  provider_id: string;
}

// ---------- Models ----------
export interface Model {
  id: string;
  name: string;
  provider_id: string;
  category: string;
  capabilities: string[];
  context_window: number;
  input_price: number | null;
  output_price: number | null;
  accuracy_score: number | null;
  agent_score: number | null;
  is_active: boolean;
  base_url: string | null;
}
export interface CustomModelRequest {
  base_url: string;
  category: string;
  api_key: string;
  model_id: string;
  name: string;
  context_window: number;
  description: string;
}

// ---------- Benchmarks ----------
export interface BenchmarkTask {
  name: string;
  value: string;
}
export interface Benchmark {
  name: string;
  description: string;
  tasks: BenchmarkTask[];
  task_count: number;
  required_capabilities: string[];
  huggingface_dataset: string;
  type: string;
}
export interface BenchmarksResponse {
  benchmarks: Benchmark[];
  total: number;
}

// ---------- Metrics ----------
export interface MetricsResponse {
  all_metrics: string[];
  custom_agent_metrics: string[];
}

// ---------- Evaluations ----------
export interface JudgeConfig {
  model_id: string;
  base_url: string;
  api_key: string;
}
export interface CreateEvaluationRequest {
  name: string;
  description?: string;
  eval_type: 'model' | 'agent' | 'rag' | string;
  dataset_id: string;
  benchmark?: string;
  model_ids: string[];
  metrics_config?: Record<string, unknown>;
  selected_metrics: string[];
  dataset_limit?: number;
  selected_category?: string[];
  judge_config?: JudgeConfig;
}
export interface CreateEvaluationResponse {
  id?: string;
  evaluation_id?: string;
  [key: string]: unknown;
}
export type EvaluationStatusValue = 'pending' | 'running' | 'completed' | 'failed' | 'canceled';
export interface EvaluationStatusResponse {
  status: EvaluationStatusValue;
  progress: number;
  total: number;
  celery_state: 'STARTED' | 'SUCCESS' | 'FAILURE' | 'REVOKED' | null;
  error_message: string | null;
}

// UI-only aggregate type used while the wizard builds up a draft
export interface EvaluationDraft {
  name: string;
  eval_type: string;
  selProviders: string[];
  selModels: string[];
  selBenchmark: string | null;
  selMetrics: string[];
  judgeModelId?: string;
}














//App.tsx
import { BrowserRouter } from 'react-router-dom';
import { Provider } from 'react-redux';
import { store } from './store/store';
import AppRoutes from './routes/AppRoutes';
import './styles/global.scss';

export default function App() {
  return (
    <Provider store={store}>
      <BrowserRouter>
        <AppRoutes />
      </BrowserRouter>
    </Provider>
  );
}



















//main.tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';
import { store } from './store/store';
import { hydrateMockSession } from './store/slices/authSlice';
import { tokenStorage } from './api/axiosInstance';

const MOCKS_ENABLED = import.meta.env.VITE_ENABLE_MOCKS === 'true';

async function enableMocking() {
  if (!MOCKS_ENABLED) return;
  const { worker } = await import('./mocks/browser');
  await worker.start({
    onUnhandledRequest: 'bypass',
    serviceWorker: { url: '/mockServiceWorker.js' },
  });
}

// Bypass SSO entirely in mock mode: seed a fake session/token before the app
// renders so ProtectedRoute never redirects to the landing page and every
// axios call already carries a bearer token. Never runs against a real backend.
function bypassAuthForMocks() {
  if (!MOCKS_ENABLED) return;
  const mockUser = {
    token: 'mock-session-token',
    username: 'jane.doe',
    email: 'jane@semco.ai',
    language: 'en',
    profile_name: 'Jane Doe',
  };
  tokenStorage.set(mockUser.token);
  store.dispatch(hydrateMockSession(mockUser));
}

bypassAuthForMocks();

enableMocking()
  .catch((err) => {
    // Never let a broken/missing mock service worker leave the app on a
    // blank screen — log it and continue rendering. Common cause: run
    // `npm run mocks:setup` (or `npx msw init public --save`) to
    // (re)generate public/mockServiceWorker.js after a fresh install.
    console.error(
      '[mocks] Failed to start MSW — continuing without request mocking. ' +
        'Run `npm run mocks:setup` to regenerate public/mockServiceWorker.js.',
      err
    );
  })
  .finally(() => {
    ReactDOM.createRoot(document.getElementById('root') as HTMLElement).render(
      <React.StrictMode>
        <App />
      </React.StrictMode>
    );
  });


















# Base URL axios will call. Leave as /api to use the vite dev proxy in vite.config.ts,
# or point straight at a deployed backend, e.g. https://api.semcoeval.com
VITE_API_BASE_URL=/api

# Set to "true" to intercept every axios call with the MSW mock handlers in
# src/mocks/, so the whole app is usable before a real backend exists.
# Set to "false" (or remove) once the real API is ready.
VITE_ENABLE_MOCKS=true
