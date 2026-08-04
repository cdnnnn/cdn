//usessoauth.ts
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
//                      → dispatch authSuccess
//                else → dispatch authError
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
  userName: 'Prof. Kim Jiyeon',
  isAdmin: true,
};

interface SsoPayload {
  userInfo: unknown;
  key: string;
}

interface WsMessage {
  rpcode: string;
  data: string;
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
          userName: MOCK_USER.userName,
          isAdmin: MOCK_USER.isAdmin,
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

        const res = await api.post('/sso_login', {
          userInfo: ssoInfo?.userInfo,
          aesKey: ssoInfo?.key,
        });

        const { token, is_admin, user_name } = (res.data as any)?.result ?? {};

        if (!token) {
          resolvedRef.current = true;
          dispatch(authError('SSO login succeeded but no token was returned.'));
          return;
        }

        resolvedRef.current = true;
        dispatch(authSuccess({
          token,
          user: {
            userName: user_name ?? 'Unknown User',
            isAdmin: Boolean(is_admin),
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

















//authslice.ts
import { createSlice, type PayloadAction } from '@reduxjs/toolkit';

export type AuthStatus = 'idle' | 'authenticating' | 'authenticated' | 'error' | 'logged_out';

export interface AuthUser {
  userName: string;
  isAdmin: boolean;
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





















//api.ts
// ═══════════════════════════════════════════════
// services/api.ts
// SemcoEval · Shared axios instance
//
// Base URL comes from environment.ts. Requests automatically pick up
// the current auth token from the store (set by authSuccess), so
// callers never need to attach it manually.
// ═══════════════════════════════════════════════
import axios from 'axios';
import environment from '../environment';
import { store } from '../store';

const api = axios.create({
  baseURL: environment.API_BASE_URL,
});

api.interceptors.request.use((config) => {
  const token = store.getState().auth.token;
  if (token) {
    config.headers = config.headers ?? {};
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export default api;
