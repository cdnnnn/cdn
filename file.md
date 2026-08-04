// ═══════════════════════════════════════════════
// src/environment.ts
// SemcoEval · Environment configuration
//
// For local dev  → copy .env.local.example → .env.local and fill values
// For production → set as CI/CD env vars
//
// CRA convention: all vars must start with REACT_APP_
// ═══════════════════════════════════════════════

const environment = {
  // ── REST API base URL ──────────────────────────
  // e.g. http://localhost:8000  or  https://api.semcoeval.io
  API_BASE_URL: 'http://localhost:8000',

  // ── Mock authentication (dev only) ─────────────
  // Set REACT_APP_MOCK_AUTH=true in .env.local to bypass the Knox SSO
  // WebSocket entirely and log in instantly with a fake user.
  MOCK_AUTH: true,

  // ── SSO WebSocket ──────────────────────────────
  SSO: {
    // WebSocket endpoint for Knox SSO handshake
    // e.g. ws://sso.knox.edu/ws  or  wss://sso.knox.edu/ws
    WS_URL: 'ws://localhost:9000/sso',

    // rqtype value sent in the opening SSO message
    RQTYPE: 'SSO_AUTH',

    // data payload sent in the opening SSO message
    DATA: 'semcoeval',
  },
} as const;

export default environment;


































//usessoauth.ts
import { useEffect, useRef } from 'react';
import { useAppDispatch, useAppSelector } from '../store/hooks';
import { authStart, authSuccess, authError } from '../store/slices/authSlice';
import environment from '../environment';

/**
 * Drives the SSO handshake. Whenever auth.status is 'idle' (fresh load, or
 * after resetAuth() from the sign-in button):
 *
 *   - if environment.MOCK_AUTH is true, skips the real handshake and logs
 *     in instantly — handy for local dev without a live Knox SSO gateway.
 *   - otherwise opens a WebSocket to environment.SSO.WS_URL, sends the
 *     opening { rqtype, data } message, and waits for the gateway to
 *     confirm or reject the session.
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

    // Dev shortcut — bypass the real SSO handshake entirely.
    if (environment.MOCK_AUTH) {
      const timer = setTimeout(() => dispatch(authSuccess()), 400);
      return () => clearTimeout(timer);
    }

    let socket: WebSocket;
    try {
      socket = new WebSocket(environment.SSO.WS_URL);
    } catch {
      dispatch(authError('Unable to reach the SSO service.'));
      return;
    }
    socketRef.current = socket;

    socket.onopen = () => {
      socket.send(
        JSON.stringify({
          rqtype: environment.SSO.RQTYPE,
          data: environment.SSO.DATA,
        })
      );
    };

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
