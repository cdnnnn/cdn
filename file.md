// ═══════════════════════════════════════════════
// src/environment.ts
// SemcoEval · Environment configuration
//
// For local dev  → copy .env.example → .env and fill values
// For production → set as CI/CD env vars
//
// Vite convention: all vars must start with VITE_ and are read via
// import.meta.env — see src/vite-env.d.ts for the typed shape.
// ═══════════════════════════════════════════════

const environment = {
  // ── REST API base URL ──────────────────────────
  // e.g. http://localhost:8000  or  https://api.semcoeval.io
  API_BASE_URL: import.meta.env.VITE_API_BASE_URL || '/api',

  // ── Mock authentication (dev only) ─────────────
  // Set VITE_MOCK_AUTH=true in .env to bypass the Knox SSO WebSocket
  // entirely and log in instantly with a fake user. Independent of
  // VITE_ENABLE_MOCKS, which only mocks REST calls via MSW.
  MOCK_AUTH: import.meta.env.VITE_MOCK_AUTH === 'true',

  // ── SSO WebSocket ──────────────────────────────
  SSO: {
    // WebSocket endpoint for Knox SSO handshake
    // e.g. ws://sso.knox.edu/ws  or  wss://sso.knox.edu/ws
    WS_URL: import.meta.env.VITE_SSO_WS_URL || 'ws://localhost:9000/sso',

    // rqtype value sent in the opening SSO message
    RQTYPE: 'SSO_AUTH',

    // data payload sent in the opening SSO message
    DATA: 'semcoeval',
  },
} as const;

export default environment;






















// ═══════════════════════════════════════════════
// hooks/useSsoAuth.ts
// SemcoEval · Knox SSO handshake
//
// Runs whenever auth.status === 'idle' (fresh load, or after resetAuth()).
// Two paths:
//   MOCK_AUTH=true → skip the WebSocket, log in instantly with a fake user
//   MOCK_AUTH=false → open the SSO WebSocket, wait for the identity
//                     provider's handshake token, then exchange it via
//                     POST /sso_login (authApi.ssoLogin) for a real session
// ═══════════════════════════════════════════════
import { useEffect, useRef } from 'react';
import { useAppDispatch, useAppSelector } from './redux';
import { authStart, authSuccess, authError } from '../store/slices/authSlice';
import type { AuthLanguage, AuthUser } from '../store/slices/authSlice';
import { authApi } from '../api/endpoints/auth';
import environment from '../environment';

function toAuthUser(result: { username: string; email: string; language: string; profile_name: string }): AuthUser {
  return {
    username: result.username,
    email: result.email,
    language: (result.language as AuthLanguage) || 'en',
    profileName: result.profile_name,
  };
}

export function useSsoAuth(): void {
  const dispatch = useAppDispatch();
  const status = useAppSelector((s) => s.auth.status);
  const socketRef = useRef<WebSocket | null>(null);

  useEffect(() => {
    if (status !== 'idle') return undefined;

    let cancelled = false;
    dispatch(authStart());

    // ---- Mock path: skip the WebSocket entirely ----
    if (environment.MOCK_AUTH) {
      authApi
        .ssoLogin({ token: 'mock-handshake-token', data: environment.SSO.DATA })
        .then((res) => {
          if (cancelled) return;
          dispatch(authSuccess({ token: res.result.token, user: toAuthUser(res.result) }));
        })
        .catch((err) => {
          if (cancelled) return;
          dispatch(authError(err?.message || 'Mock SSO exchange failed'));
        });
      return () => {
        cancelled = true;
      };
    }

    // ---- Real path: Knox SSO WebSocket handshake ----
    let socket: WebSocket;
    try {
      socket = new WebSocket(environment.SSO.WS_URL);
    } catch {
      dispatch(authError('Unable to reach the SSO service'));
      return undefined;
    }
    socketRef.current = socket;

    socket.onopen = () => {
      socket.send(JSON.stringify({ rqtype: environment.SSO.RQTYPE, data: environment.SSO.DATA }));
    };

    socket.onmessage = async (event) => {
      if (cancelled) return;
      try {
        const payload = JSON.parse(event.data) as { token?: string; error?: string };
        if (payload.error || !payload.token) {
          dispatch(authError(payload.error || 'SSO handshake did not return a token'));
          socket.close();
          return;
        }
        // Exchange the handshake token for a real session via the REST API.
        const res = await authApi.ssoLogin({ token: payload.token, data: environment.SSO.DATA });
        if (cancelled) return;
        dispatch(authSuccess({ token: res.result.token, user: toAuthUser(res.result) }));
      } catch (err) {
        if (!cancelled) {
          dispatch(authError(err instanceof Error ? err.message : 'SSO exchange failed'));
        }
      } finally {
        socket.close();
      }
    };

    socket.onerror = () => {
      if (!cancelled) dispatch(authError('SSO WebSocket connection failed'));
    };

    socket.onclose = (event) => {
      if (!cancelled && !event.wasClean) {
        dispatch(authError('SSO connection closed unexpectedly'));
      }
    };

    return () => {
      cancelled = true;
      socket.close();
      socketRef.current = null;
    };
  }, [status, dispatch]);
}






















// ═══════════════════════════════════════════════
// components/AuthGuard/AuthGuard.tsx
// SemcoEval · Global auth gate
//
// Used as a layout route in AppRoutes.tsx — wraps the /app/* subtree,
// so that section of the app requires SSO auth.
//
// Renders:
//   idle / authenticating → <AuthSpinner>  (SSO WebSocket running)
//   error / logged_out    → <Navigate to="/sso-login">
//   authenticated         → <Outlet />     (render the matched child route)
// ═══════════════════════════════════════════════
import type { FC } from 'react';
import { Navigate, Outlet, useLocation } from 'react-router-dom';
import { useAppSelector } from '../../hooks/redux';
import { useSsoAuth } from '../../hooks/useSsoAuth';
import AuthSpinner from '../AuthSpinner/AuthSpinner';

const AuthGuard: FC = () => {
  // Triggers the SSO WebSocket flow whenever status === 'idle'
  useSsoAuth();

  const status = useAppSelector((s) => s.auth.status);
  const error = useAppSelector((s) => s.auth.error);
  const location = useLocation();

  switch (status) {
    // SSO in progress (or very first render before authStart fires)
    case 'idle':
    case 'authenticating':
      return <AuthSpinner />;

    // Auth failed or user logged out → go to the SSO login page
    case 'error':
    case 'logged_out':
      return <Navigate to="/sso-login" replace state={{ from: location, errorMessage: error }} />;

    // All good — render whichever child route matched
    case 'authenticated':
      return <Outlet />;

    default:
      return <AuthSpinner />;
  }
};

export default AuthGuard;























// ═══════════════════════════════════════════════
// components/AuthSpinner/AuthSpinner.tsx
// SemcoEval · Full-screen auth overlay
// ═══════════════════════════════════════════════
import type { FC } from 'react';
import { Gauge } from 'lucide-react';
import styles from './AuthSpinner.module.scss';

const AuthSpinner: FC = () => (
  <div className={styles['auth-spinner']} role="status" aria-live="polite">
    <div className={styles['auth-spinner__card']}>
      <div className={styles['auth-spinner__mark']}>
        <Gauge size={22} strokeWidth={2.25} />
      </div>

      <div className={styles['auth-spinner__ring']}>
        <div className={styles['auth-spinner__arc']} />
      </div>

      <div className={styles['auth-spinner__label']}>Authenticating…</div>
      <div className={styles['auth-spinner__sub']}>Connecting to Knox SSO</div>
    </div>
  </div>
);

export default AuthSpinner;














//Authspinner.module.scss
@use '../../styles/_variables' as *;

// Token mapping from the reference design system to ours:
//   $bg-page        -> $bg
//   $bg-main        -> $surface
//   $border-subtle  -> $border-light
//   $border-default -> $border
//   $shadow-lg      -> $shadow-4
//   $shadow-md      -> $shadow-3
//   $primary        -> $indigo
//   $primary-hover  -> $indigo-dark
//   $on-primary     -> #fff
//   $text-tertiary  -> $text-muted

.auth-spinner {
  position: fixed;
  inset: 0;
  z-index: 1000;
  display: grid;
  place-items: center;
  background: $bg;

  &__card {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 1.125rem;
    padding: 2.5rem 3rem;
    background: $surface;
    border: 1px solid $border-light;
    border-radius: 1.25rem;
    box-shadow: $shadow-4;
  }

  &__mark {
    width: 52px;
    height: 52px;
    border-radius: 1rem;
    display: grid;
    place-items: center;
    color: #fff;
    background: linear-gradient(155deg, $indigo 0%, $indigo-dark 100%);
    box-shadow: $shadow-3, inset 0 0 0 0.0625rem rgba(255, 255, 255, 0.14);
    animation: auth-spinner-pulse 2.2s ease-in-out infinite;
  }

  &__ring {
    position: relative;
    width: 32px;
    height: 32px;
  }

  &__arc {
    position: absolute;
    inset: 0;
    border-radius: 50%;
    border: 3px solid $border;
    border-top-color: $indigo;
    animation: auth-spinner-spin 0.8s linear infinite;
  }

  &__label {
    font-family: $font-display;
    font-size: 0.9375rem;
    font-weight: 700;
    color: $text-primary;
    letter-spacing: -0.01em;
  }

  &__sub {
    margin-top: -0.75rem;
    font-size: 0.8125rem;
    color: $text-muted;
  }

  /* ---------- ultra-wide: nudge text sizes up a touch ---------- */
  @media (min-width: 1800px) {
    &__label {
      font-size: 1.03125rem;
    }

    &__sub {
      font-size: 0.90625rem;
    }
  }
}

@keyframes auth-spinner-spin {
  to {
    transform: rotate(360deg);
  }
}

@keyframes auth-spinner-pulse {
  0%,
  100% {
    transform: scale(1);
    box-shadow: $shadow-3, inset 0 0 0 0.0625rem rgba(255, 255, 255, 0.14);
  }
  50% {
    transform: scale(1.05);
    box-shadow: $shadow-4, inset 0 0 0 0.0625rem rgba(255, 255, 255, 0.18);
  }
}

























// ═══════════════════════════════════════════════
// components/auth/SsoLogin.tsx
// SemcoEval · SSO error / signed-out landing
//
// AuthGuard redirects here on status === 'error' | 'logged_out', passing
// { from, errorMessage } via router state. "Retry" dispatches resetAuth(),
// which flips status back to 'idle' so useSsoAuth() re-runs the handshake
// the next time AuthGuard mounts.
// ═══════════════════════════════════════════════
import type { FC } from 'react';
import { useLocation, useNavigate } from 'react-router-dom';
import { ShieldAlert, RotateCw } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { resetAuth } from '../../store/slices/authSlice';
import styles from './SsoLogin.module.scss';

interface LocationState {
  from?: { pathname: string };
  errorMessage?: string | null;
}

const SsoLogin: FC = () => {
  const dispatch = useAppDispatch();
  const navigate = useNavigate();
  const location = useLocation();
  const status = useAppSelector((s) => s.auth.status);

  const state = (location.state as LocationState) || {};
  const message =
    state.errorMessage ||
    (status === 'logged_out' ? "You've been signed out." : 'We could not verify your session.');

  const retry = () => {
    dispatch(resetAuth());
    navigate(state.from?.pathname || '/app/dashboard', { replace: true });
  };

  return (
    <div className={styles['sso-login']}>
      <div className={styles['sso-login__card']}>
        <div className={styles['sso-login__icon']}>
          <ShieldAlert size={22} strokeWidth={2.25} />
        </div>
        <h1 className={styles['sso-login__title']}>Sign-in required</h1>
        <p className={styles['sso-login__message']}>{message}</p>
        <button className="btn btn-ind" onClick={retry}>
          <RotateCw size={14} /> Retry sign-in
        </button>
      </div>
    </div>
  );
};

export default SsoLogin;
















//Ssologin.module.scss
@use '../../styles/_variables' as *;

.sso-login {
  position: fixed;
  inset: 0;
  display: grid;
  place-items: center;
  background: $bg;
  padding: 24px;

  &__card {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 14px;
    max-width: 380px;
    text-align: center;
    padding: 40px 36px;
    background: $surface;
    border: 1px solid $border-light;
    border-radius: 20px;
    box-shadow: $shadow-4;
  }

  &__icon {
    width: 52px;
    height: 52px;
    border-radius: 16px;
    display: grid;
    place-items: center;
    color: $red;
    background: $red-pale;
  }

  &__title {
    font-family: $font-display;
    font-size: 18px;
    font-weight: 700;
    color: $text-primary;
  }

  &__message {
    font-size: 14px;
    color: $text-secondary;
    line-height: 1.6;
    margin-bottom: 6px;
  }
}




















//Authslice.ts
import { createSlice } from '@reduxjs/toolkit';
import type { PayloadAction } from '@reduxjs/toolkit';

export type AuthStatus = 'idle' | 'authenticating' | 'authenticated' | 'error' | 'logged_out';

export type AuthLanguage = 'en' | 'ko';

export interface AuthUser {
  username: string;
  email: string;
  language: AuthLanguage;
  profileName: string;
}

export interface AuthState {
  status: AuthStatus;
  error: string | null;
  token: string | null;
  user: AuthUser | null;
}

const initialState: AuthState = {
  status: 'idle',
  error: null,
  token: null,
  user: null,
};

const authSlice = createSlice({
  name: 'auth',
  initialState,
  reducers: {
    // SSO flow has kicked off (WebSocket opened, waiting on the identity provider)
    authStart: (state) => {
      state.status = 'authenticating';
      state.error = null;
    },
    // SSO (or the /sso_login exchange) confirmed the session
    authSuccess: (state, action: PayloadAction<{ token: string; user: AuthUser }>) => {
      state.status = 'authenticated';
      state.error = null;
      state.token = action.payload.token;
      state.user = action.payload.user;
    },
    // SSO failed, the WebSocket dropped, or /sso_login rejected the exchange
    authError: (state, action: PayloadAction<string>) => {
      state.status = 'error';
      state.error = action.payload;
      state.token = null;
      state.user = null;
    },
    // User explicitly signed out
    logout: (state) => {
      state.status = 'logged_out';
      state.error = null;
      state.token = null;
      state.user = null;
    },
    // Back to 'idle' — useSsoAuth() picks this up and re-runs the flow
    resetAuth: (state) => {
      state.status = 'idle';
      state.error = null;
    },
  },
});

export const { authStart, authSuccess, authError, logout, resetAuth } = authSlice.actions;
export default authSlice.reducer;






















//Axiosinstance.ts
import axios from 'axios';
import type { AxiosError, InternalAxiosRequestConfig } from 'axios';
import { store } from '../store/store';
import { logout } from '../store/slices/authSlice';
import environment from '../environment';

export const api = axios.create({
  baseURL: environment.API_BASE_URL,
  timeout: 20000,
  headers: {
    'Content-Type': 'application/json',
  },
});

// ---- Request interceptor: attach bearer token from Redux state ----
// No localStorage — the SSO WebSocket (useSsoAuth) re-authenticates on every
// load, so the token only ever needs to live in memory for the session.
api.interceptors.request.use(
  (config: InternalAxiosRequestConfig) => {
    const token = store.getState().auth.token;
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





















//Approutes.tsx
import { Routes, Route, Navigate } from 'react-router-dom';
import Landing from '../components/landing/Landing';
import SsoLogin from '../components/auth/SsoLogin';
import AuthGuard from '../components/AuthGuard/AuthGuard';
import AppShell from '../components/layout/AppShell';
import Dashboard from '../components/dashboard/Dashboard';
import Providers from '../components/providers/Providers';
import ModelCatalog from '../components/models/ModelCatalog';
import TestSuites from '../components/suites/TestSuites';
import NewEvaluation from '../components/evaluations/NewEvaluation';
import Evaluations from '../components/evaluations/Evaluations';
import EvaluationDetail from '../components/evaluations/EvaluationDetail';
import Comparison from '../components/comparison/Comparison';

export default function AppRoutes() {
  return (
    <Routes>
      <Route path="/" element={<Landing />} />
      {/* AuthGuard redirects here with { from, errorMessage } on error/logged_out. */}
      <Route path="/sso-login" element={<SsoLogin />} />

      {/* AuthGuard triggers the SSO WebSocket handshake (useSsoAuth) as soon as
          this layout route mounts, and shows AuthSpinner until authenticated. */}
      <Route element={<AuthGuard />}>
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























//main.tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';

const MOCKS_ENABLED = import.meta.env.VITE_ENABLE_MOCKS === 'true';

async function enableMocking() {
  if (!MOCKS_ENABLED) return;
  const { worker } = await import('./mocks/browser');
  await worker.start({
    onUnhandledRequest: 'bypass',
    serviceWorker: { url: '/mockServiceWorker.js' },
  });
}

// Auth is no longer bypassed here — AuthGuard + useSsoAuth() handle that
// natively via VITE_MOCK_AUTH (see src/environment.ts), so the auth state
// machine runs the same code path in mock and real modes alike.
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






















  //Landing.tsx
  import { useEffect, useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { ArrowRight, Play, Award, Link2, Cpu, FlaskConical, BarChart3, GitCompare, Shield } from 'lucide-react';
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
  const navigate = useNavigate();

  useEffect(() => {
    const t = setTimeout(() => setAnimated(true), 200);
    return () => clearTimeout(t);
  }, []);

  // AuthGuard (wrapping /app) triggers the SSO handshake automatically as
  // soon as it mounts and shows AuthSpinner while it's in flight — Landing
  // only needs to navigate there.
  const handleSignIn = () => navigate('/app/dashboard');

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
          <button className={styles['btn-primary']} onClick={handleSignIn} style={{ padding: '10px 22px', fontSize: 14 }}>
            Sign In
          </button>
        </div>
      </nav>

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
            <div className={styles['sidebar__user-name']}>{user?.profileName || user?.username || 'Guest'}</div>
            <div className={styles['sidebar__user-email']}>{user?.email || ''}</div>
          </div>
          <LogOut size={16} style={{ color: '#9CA3AF', cursor: 'pointer' }} onClick={() => dispatch(logout())} />
        </div>
      </div>
    </div>
  );
}



















/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_BASE_URL: string;
  readonly VITE_ENABLE_MOCKS: string;
  readonly VITE_MOCK_AUTH: string;
  readonly VITE_SSO_WS_URL: string;
}
interface ImportMeta {
  readonly env: ImportMetaEnv;
}
