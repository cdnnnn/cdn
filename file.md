//Authslice.ts
import { createSlice, type PayloadAction } from '@reduxjs/toolkit';

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
















//Usessoauth.ts
// ═══════════════════════════════════════════════
// hooks/useSsoAuth.ts
// SemcoEval · SSO authentication hook
//
// Two modes controlled by environment.MOCK_AUTH:
//
//   MOCK_AUTH=false (production default)
//     1. status === 'idle'  → dispatch authStart
//     2. Open WebSocket to SSO endpoint
//     3. onopen  → send { rqtype, token:'', data }
//     4. onmessage → rpcode === 'RETURN_SUCCESS'
//                      → POST /sso_login
//                      → response.status === 'success'
//                          → dispatch authSuccess
//                      → else dispatch authError
//     5. onerror / abnormal close → dispatch authError
//
//   MOCK_AUTH=true  (local dev / demo)
//     Skips WebSocket and REST call entirely.
//     After a short simulated delay, dispatches authSuccess
//     with a hard-coded mock user and token.
//
// To enable mock mode add this line to .env.local:
//   REACT_APP_MOCK_AUTH=true
// ═══════════════════════════════════════════════
import { useEffect, useRef } from 'react';
import { useAppDispatch, useAppSelector } from '../store/hooks';
import { authStart, authSuccess, authError } from '../store/slices/authSlice';
import api from '../services/api';
import environment from '../environment';

// ── Mock user (only used when MOCK_AUTH=true) ──
const MOCK_USER = {
  token: 'mock-token-dev-only',
  username: 'kim.jiyeon',
  email: 'kim.jiyeon@example.com',
  language: 'en' as const,
  profileName: 'Prof. Kim Jiyeon',
};

interface SsoPayload {
  userInfo: unknown;
  key: string;
}

interface WsMessage {
  rpcode: string;
  data: string;
}

// Shape returned by POST /sso_login
interface SsoLoginResponse {
  status: 'success' | string;
  message: string | null;
  result?: {
    token: string;
    username: string;
    email: string;
    language: 'en' | 'ko';
    profile_name: string;
  };
}

export function useSsoAuth() {
  const dispatch = useAppDispatch();
  const status = useAppSelector(s => s.auth.status);
  const resolvedRef = useRef(false);

  useEffect(() => {
    if (status !== 'idle') return;

    resolvedRef.current = false;
    dispatch(authStart());

    // ════════════════════════════════════════════
    // MOCK MODE — instant login, no network calls
    // ════════════════════════════════════════════
    if (environment.MOCK_AUTH) {
      // const timer = setTimeout(() => {
      resolvedRef.current = true;
      dispatch(authSuccess({
        token: MOCK_USER.token,
        user: {
          username: MOCK_USER.username,
          email: MOCK_USER.email,
          language: MOCK_USER.language,
          profileName: MOCK_USER.profileName,
        },
      }));
      // }, 800); // short delay so the spinner is visible

      // return () => clearTimeout(timer);
    }

    // ════════════════════════════════════════════
    // PRODUCTION MODE — real Knox SSO WebSocket
    // ════════════════════════════════════════════
    const ws = new WebSocket(environment.SSO.WS_URL);

    // ── Step 1: Send handshake ───────────────────────
    ws.onopen = () => {
      ws.send(JSON.stringify({
        rqtype: environment.SSO.RQTYPE,
        token: '',
        data: environment.SSO.DATA,
      }));
    };

    // ── Step 2: Handle SSO response ─────────────────
    ws.onmessage = async (event: MessageEvent) => {
      try {
        const message: WsMessage = JSON.parse(event.data as string);

        if (message?.rpcode !== 'RETURN_SUCCESS') {
          resolvedRef.current = true;
          dispatch(authError(
            `SSO returned unexpected code: ${message?.rpcode ?? 'unknown'}`
          ));
          ws.close();
          return;
        }

        const ssoInfo: SsoPayload = JSON.parse(message.data);

        const res = await api.post<SsoLoginResponse>('/sso_login', {
          userInfo: ssoInfo?.userInfo,
          aesKey: ssoInfo?.key,
        });

        const { status: loginStatus, message: loginMessage, result } = res.data;

        if (loginStatus !== 'success' || !result?.token) {
          resolvedRef.current = true;
          dispatch(authError(loginMessage ?? 'SSO login succeeded but no token was returned.'));
          return;
        }

        const { token, username, email, language, profile_name } = result;

        resolvedRef.current = true;
        dispatch(authSuccess({
          token,
          user: {
            username,
            email,
            language,
            profileName: profile_name,
          },
        }));
      } catch (err) {
        resolvedRef.current = true;
        dispatch(authError(
          err instanceof Error ? err.message : 'SSO authentication failed.'
        ));
      } finally {
        ws.close();
      }
    };

    // ── Step 3: WebSocket error ──────────────────────
    ws.onerror = () => {
      if (!resolvedRef.current) {
        resolvedRef.current = true;
        dispatch(authError('Could not connect to the SSO server. Please try again.'));
      }
    };

    // ── Step 4: Abnormal close before resolution ─────
    ws.onclose = (event: CloseEvent) => {
      if (!resolvedRef.current && !event.wasClean) {
        resolvedRef.current = true;
        dispatch(authError(
          `SSO connection closed unexpectedly (code ${event.code}).`
        ));
      }
    };

    return () => {
      if (
        ws.readyState === WebSocket.OPEN ||
        ws.readyState === WebSocket.CONNECTING
      ) {
        ws.close();
      }
    };
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [status]);
}
















//Header.tsx
import { Link } from 'react-router-dom';
import { useEffect, useRef, useState, type FC } from 'react';
import { Gauge, LogOut, Sun, Moon } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import { toggleTheme } from '../../store/slices/uiSlice';
import { logout } from '../../store/slices/authSlice';
import './Header.scss';

const Header: FC = () => {
  const dispatch = useAppDispatch();
  const user = useAppSelector((s) => s.auth.user);
  const theme = useAppSelector((s) => s.ui.theme);
  const [menuOpen, setMenuOpen] = useState(false);
  const menuRef = useRef<HTMLDivElement>(null);
  const isDark = theme === 'dark';

  // Avatar shows just the first letter of the profile name (e.g. "Prof. Kim Jiyeon" → "P")
  const initial = user?.profileName?.trim().charAt(0).toUpperCase() || '?';

  useEffect(() => {
    const handler = (e: MouseEvent) => {
      if (menuRef.current && !menuRef.current.contains(e.target as Node)) {
        setMenuOpen(false);
      }
    };
    document.addEventListener('mousedown', handler);
    return () => document.removeEventListener('mousedown', handler);
  }, []);

  return (
    <header className="app-header">
      <Link className="app-header__brand" to="/">
        <span className="app-header__brand-mark">
          <Gauge size={18} strokeWidth={2.25} />
        </span>
        <span className="app-header__brand-name">SemcoEval</span>
      </Link>

      <div className="app-header__right">
        <button
          type="button"
          className="app-header__theme-toggle"
          onClick={() => dispatch(toggleTheme())}
          aria-label={isDark ? 'Switch to light mode' : 'Switch to dark mode'}
          title={isDark ? 'Switch to light mode' : 'Switch to dark mode'}
        >
          <Sun size={12} strokeWidth={2.25} className="app-header__theme-static app-header__theme-static--sun" />
          <Moon size={12} strokeWidth={2.25} className="app-header__theme-static app-header__theme-static--moon" />

          <span className={`app-header__theme-knob${isDark ? ' app-header__theme-knob--dark' : ' app-header__theme-knob--light'}`}>
            {isDark ? <Moon size={13} strokeWidth={2.5} /> : <Sun size={13} strokeWidth={2.5} />}
          </span>
        </button>

        <div className="app-header__user" ref={menuRef}>
          <button
            type="button"
            className={`app-header__avatar${menuOpen ? ' app-header__avatar--open' : ''}`}
            onClick={() => setMenuOpen((v) => !v)}
            aria-label="User menu"
            aria-expanded={menuOpen}
            title={user?.profileName ?? 'Account'}
          >
            {initial}
          </button>

          {menuOpen && (
            <div className="app-header__dropdown">
              <div className="app-header__drop-user">
                <div className="app-header__drop-avatar">{initial}</div>
                <div className="app-header__drop-info">
                  <div className="app-header__drop-name">{user?.profileName ?? 'Unknown user'}</div>
                  <div className="app-header__drop-role">{user?.email ?? '—'}</div>
                </div>
              </div>
              <div className="app-header__drop-divider" />
              <button type="button" className="app-header__drop-item" onClick={() => dispatch(logout())}>
                <LogOut size={15} strokeWidth={2} />
                Log out
              </button>
            </div>
          )}
        </div>
      </div>
    </header>
  );
};

export default Header;
