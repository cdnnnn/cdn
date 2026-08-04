//Authslice.ts
import { createSlice, type PayloadAction } from '@reduxjs/toolkit';

export type AuthStatus = 'idle' | 'authenticating' | 'authenticated' | 'error' | 'logged_out';

export interface AuthState {
  status: AuthStatus;
  error: string | null;
}

const initialState: AuthState = {
  status: 'idle',
  error: null,
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
    // SSO confirmed the session
    authSuccess: (state) => {
      state.status = 'authenticated';
      state.error = null;
    },
    // SSO failed, or the WebSocket dropped before confirming
    authError: (state, action: PayloadAction<string>) => {
      state.status = 'error';
      state.error = action.payload;
    },
    // User explicitly signed out
    logout: (state) => {
      state.status = 'logged_out';
      state.error = null;
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




















//index.ts
import { configureStore } from '@reduxjs/toolkit';
import uiReducer from './slices/uiSlice';
import userReducer from './slices/userSlice';
import authReducer from './slices/authSlice';

export const store = configureStore({
  reducer: {
    ui: uiReducer,
    user: userReducer,
    auth: authReducer,
  },
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;

export default store;












//usessoauth.ts
import { useEffect, useRef } from 'react';
import { useAppDispatch, useAppSelector } from '../store/hooks';
import { authStart, authSuccess, authError } from '../store/slices/authSlice';

// TODO: point this at the real Knox/SSO WebSocket gateway. Falls back to a
// placeholder so the app doesn't crash on missing env config during setup.
const SSO_WS_URL = import.meta.env.VITE_SSO_WS_URL ?? 'wss://sso.example.com/auth/ws';

/**
 * Drives the SSO handshake. Whenever auth.status is 'idle' (fresh load, or
 * after resetAuth() from the sign-in button), opens a WebSocket to the SSO
 * gateway and waits for it to confirm or reject the session.
 *
 * Expected server messages (adjust to match your gateway's actual contract):
 *   { "type": "authenticated" }
 *   { "type": "error", "message": "..." }
 */
export function useSsoAuth(): void {
  const dispatch = useAppDispatch();
  const status = useAppSelector((s) => s.auth.status);
  const socketRef = useRef<WebSocket | null>(null);

  useEffect(() => {
    if (status !== 'idle') return;

    dispatch(authStart());

    let socket: WebSocket;
    try {
      socket = new WebSocket(SSO_WS_URL);
    } catch {
      dispatch(authError('Unable to reach the SSO service.'));
      return;
    }
    socketRef.current = socket;

    socket.onmessage = (event) => {
      try {
        const payload = JSON.parse(event.data);
        if (payload.type === 'authenticated') {
          dispatch(authSuccess());
        } else if (payload.type === 'error') {
          dispatch(authError(payload.message ?? 'Authentication failed.'));
        }
      } catch {
        dispatch(authError('Received an invalid response from the SSO service.'));
      }
    };

    socket.onerror = () => {
      dispatch(authError('Could not connect to the SSO service.'));
    };

    return () => {
      socket.close();
      socketRef.current = null;
    };
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [status]);
}









//Authguard.tsx
// ═══════════════════════════════════════════════
// components/AuthGuard/AuthGuard.tsx
// SemcoEval · Global auth gate
//
// Used as a layout route in App.tsx — wraps every route except
// /sso-login, so the whole app requires SSO auth.
//
// Renders:
//   idle / authenticating → <AuthSpinner>  (SSO WebSocket running)
//   error / logged_out    → <Navigate to="/sso-login">
//   authenticated         → <Outlet />     (render the matched child route)
// ═══════════════════════════════════════════════
import type { FC } from 'react';
import { Navigate, Outlet, useLocation } from 'react-router-dom';
import { useAppSelector } from '../../store/hooks';
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












//Authspinner.tsx
// ═══════════════════════════════════════════════
// components/AuthSpinner/AuthSpinner.tsx
// SemcoEval · Full-screen auth overlay
// ═══════════════════════════════════════════════
import type { FC } from 'react';
import { Gauge } from 'lucide-react';
import './AuthSpinner.scss';

const AuthSpinner: FC = () => (
  <div className="auth-spinner" role="status" aria-live="polite">
    <div className="auth-spinner__card">
      <div className="auth-spinner__mark">
        <Gauge size={22} strokeWidth={2.25} />
      </div>

      <div className="auth-spinner__ring">
        <div className="auth-spinner__arc" />
      </div>

      <div className="auth-spinner__label">Authenticating…</div>
      <div className="auth-spinner__sub">Connecting to Knox SSO</div>
    </div>
  </div>
);

export default AuthSpinner;


















//Authspinner.scss
@use '../../styles/variables' as *;

.auth-spinner {
  position: fixed;
  inset: 0;
  z-index: 1000;
  display: grid;
  place-items: center;
  background: $bg-page;

  &__card {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 1.125rem;
    padding: 2.5rem 3rem;
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 1.25rem;
    box-shadow: $shadow-lg;
  }

  &__mark {
    width: 52px;
    height: 52px;
    border-radius: 1rem;
    display: grid;
    place-items: center;
    color: $on-primary;
    background: linear-gradient(155deg, $primary 0%, $primary-hover 100%);
    box-shadow: $shadow-md, inset 0 0 0 0.0625rem rgba(255, 255, 255, 0.14);
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
    border: 3px solid $border-default;
    border-top-color: $primary;
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
    color: $text-tertiary;
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
    box-shadow: $shadow-md, inset 0 0 0 0.0625rem rgba(255, 255, 255, 0.14);
  }
  50% {
    transform: scale(1.05);
    box-shadow: $shadow-lg, inset 0 0 0 0.0625rem rgba(255, 255, 255, 0.18);
  }
}























//ssologin.tsx
// ═══════════════════════════════════════════════
// pages/SsoLogin/SsoLogin.tsx
// SemcoEval · SSO Login / re-auth page
//
// Shown when:
//   • User explicitly logs out   (status === 'logged_out')
//   • SSO authentication fails   (status === 'error')
//
// "Sign in" button calls resetAuth(), which sets status back to
// 'idle', causing useSsoAuth() to re-run the WebSocket flow.
// ═══════════════════════════════════════════════
import type { FC } from 'react';
import { Navigate, useLocation } from 'react-router-dom';
import { Gauge, AlertCircle, CheckCircle2, ShieldCheck } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import { resetAuth } from '../../store/slices/authSlice';
import { useSsoAuth } from '../../hooks/useSsoAuth';
import AuthSpinner from '../../components/AuthSpinner/AuthSpinner';
import './SsoLogin.scss';

interface LocationState {
  from?: { pathname: string };
  errorMessage?: string;
}

const SsoLogin: FC = () => {
  const dispatch = useAppDispatch();
  const status = useAppSelector((s) => s.auth.status);
  const location = useLocation();
  const state = (location.state ?? {}) as LocationState;

  // Runs the SSO hook — fires the WebSocket flow once status becomes 'idle'
  useSsoAuth();

  // Already authenticated → go to the page the user originally wanted
  if (status === 'authenticated') {
    return <Navigate to={state.from?.pathname ?? '/'} replace />;
  }

  // SSO in progress → show the same full-screen spinner as the guard
  if (status === 'authenticating') {
    return <AuthSpinner />;
  }

  const isError = status === 'error';
  const isLoggedOut = status === 'logged_out';
  const errorMessage = state.errorMessage ?? 'Authentication failed. Please try again.';

  const handleSignIn = () => {
    dispatch(resetAuth());
  };

  return (
    <div className="sso-login">
      <div className="sso-login__grid" aria-hidden="true" />

      <div className="sso-login__card">
        <div className="sso-login__mark">
          <Gauge size={22} strokeWidth={2.25} />
        </div>

        <h1 className="sso-login__title">
          Semco<span>Eval</span>
        </h1>
        <p className="sso-login__sub">Model Evaluation Platform</p>

        {isError && (
          <div className="sso-login__banner sso-login__banner--error" role="alert">
            <AlertCircle size={15} strokeWidth={2} />
            {errorMessage}
          </div>
        )}

        {isLoggedOut && !isError && (
          <div className="sso-login__banner sso-login__banner--info">
            <CheckCircle2 size={15} strokeWidth={2} />
            You've been signed out successfully.
          </div>
        )}

        <button type="button" className="sso-login__signin-btn" onClick={handleSignIn}>
          <ShieldCheck size={16} strokeWidth={2} />
          Sign in with Knox SSO
        </button>

        <p className="sso-login__hint">
          Authentication is handled automatically via your institution's Knox SSO service.
        </p>
      </div>
    </div>
  );
};

export default SsoLogin;

























//ssologin.scss
@use '../../styles/variables' as *;

.sso-login {
  position: relative;
  min-height: 100vh;
  display: grid;
  place-items: center;
  padding: 2rem;
  background: $bg-page;
  overflow: hidden;

  &__grid {
    position: absolute;
    inset: 0;
    background-image: linear-gradient(to right, $border-subtle 0.0625rem, transparent 0.0625rem),
      linear-gradient(to bottom, $border-subtle 0.0625rem, transparent 0.0625rem);
    background-size: 3.375rem 3.375rem;
    mask-image: radial-gradient(60% 55% at 50% 40%, #000 22%, transparent 72%);
    -webkit-mask-image: radial-gradient(60% 55% at 50% 40%, #000 22%, transparent 72%);
    pointer-events: none;
  }

  &__card {
    position: relative;
    width: 100%;
    max-width: 380px;
    display: flex;
    flex-direction: column;
    align-items: center;
    text-align: center;
    gap: 0.25rem;
    padding: 2.75rem 2.25rem 2.25rem;
    background: $bg-main;
    border: 1px solid $border-subtle;
    border-radius: 1.25rem;
    box-shadow: $shadow-lg;
  }

  &__mark {
    width: 52px;
    height: 52px;
    border-radius: 1rem;
    display: grid;
    place-items: center;
    color: $on-primary;
    background: linear-gradient(155deg, $primary 0%, $primary-hover 100%);
    box-shadow: $shadow-md, inset 0 0 0 0.0625rem rgba(255, 255, 255, 0.14);
    margin-bottom: 1.25rem;
  }

  &__title {
    font-family: $font-display;
    font-size: 1.375rem;
    font-weight: 800;
    letter-spacing: -0.02em;
    color: $text-primary;

    span {
      color: $primary;
    }
  }

  &__sub {
    margin-top: 0.25rem;
    font-size: 0.84375rem;
    color: $text-tertiary;
  }

  &__banner {
    width: 100%;
    display: flex;
    align-items: flex-start;
    gap: 0.5rem;
    margin-top: 1.5rem;
    padding: 0.6875rem 0.8125rem;
    border-radius: 0.625rem;
    font-size: 0.8125rem;
    line-height: 1.45;
    text-align: left;

    svg {
      flex-shrink: 0;
      margin-top: 0.0625rem;
    }

    &--error {
      background: $danger-subtle;
      color: $danger;
    }

    &--info {
      background: $success-subtle;
      color: $success;
    }
  }

  &__signin-btn {
    width: 100%;
    margin-top: 1.75rem;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 0.5625rem;
    font-family: $font-body;
    font-size: 0.90625rem;
    font-weight: 600;
    color: $on-primary;
    background: $primary;
    border: 1px solid $primary;
    border-radius: 0.625rem;
    padding: 0.75rem 1rem;
    cursor: pointer;
    transition: background 0.14s ease, border-color 0.14s ease;

    &:hover {
      background: $primary-hover;
      border-color: $primary-hover;
    }

    &:focus-visible {
      outline: none;
      box-shadow: 0 0 0 0.1875rem $primary-subtle;
    }
  }

  &__hint {
    margin-top: 1.25rem;
    font-size: 0.75rem;
    line-height: 1.5;
    color: $text-tertiary;
  }
}

















//app.tsx
import { useEffect } from 'react';
import { Routes, Route } from 'react-router-dom';
import type { FC } from 'react';
import MainLayout from './layouts/MainLayout';
import WorkspaceLayout from './layouts/WorkspaceLayout';
import Landing from './pages/Landing/Landing';
import SsoLogin from './pages/SsoLogin/SsoLogin';
import Dashboard from './pages/Workspace/Dashboard/Dashboard';
import History from './pages/Workspace/History/History';
import Models from './pages/Workspace/Models/Models';
import Providers from './pages/Workspace/Providers/Providers';
import Datasets from './pages/Workspace/Datasets/Datasets';
import Reports from './pages/Workspace/Reports/Reports';
import Settings from './pages/Workspace/Settings/Settings';
import RunEvaluation from './pages/Workspace/RunEvaluation/RunEvaluation';
import AuthGuard from './components/AuthGuard/AuthGuard';
import { useAppSelector } from './store/hooks';

const THEME_STORAGE_KEY = 'semcoeval-theme';

const App: FC = () => {
  const theme = useAppSelector((s) => s.ui.theme);

  useEffect(() => {
    document.documentElement.setAttribute('data-theme', theme);
    window.localStorage.setItem(THEME_STORAGE_KEY, theme);
  }, [theme]);

  return (
    <Routes>
      {/* Public — SSO login/re-auth page, outside the auth gate */}
      <Route path="/sso-login" element={<SsoLogin />} />

      {/* Everything else requires an authenticated session */}
      <Route element={<AuthGuard />}>
        <Route element={<MainLayout />}>
          <Route path="/" element={<Landing />} />

          <Route path="/app" element={<WorkspaceLayout />}>
            <Route index element={<Dashboard />} />
            <Route path="run-evaluation" element={<RunEvaluation />} />
            <Route path="history" element={<History />} />
            <Route path="models" element={<Models />} />
            <Route path="providers" element={<Providers />} />
            <Route path="datasets" element={<Datasets />} />
            <Route path="reports" element={<Reports />} />
            <Route path="settings" element={<Settings />} />
          </Route>
        </Route>
      </Route>
    </Routes>
  );
};

export default App;
