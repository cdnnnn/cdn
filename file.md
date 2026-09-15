import type { TicketStatus, TicketPriority, Ticket, TicketUser } from '../../types/tickets';

// ─────────────────────────────────────────────────────────────────────────
// Adapter from the app's existing SsoLoginResult (state.auth.user, from
// authSlice.ts) to this feature's minimal TicketUser shape ({ id, name }).
//
// SsoLoginResult has no `id` field — `username` is the stable per-user
// identifier (used for the owner check, avatar color hashing, etc.) and
// `profileName` is the display name shown throughout the UI.
// ─────────────────────────────────────────────────────────────────────────
export function toTicketUser(
  sso: { username: string; profileName: string } | null | undefined
): TicketUser | null {
  if (!sso) return null;
  return { id: sso.username, name: sso.profileName };
}

export interface ColumnMeta {
  status: TicketStatus;
  label: string;
  /** Accent hex used for the column dot + card left-border. */
  accent: string;
}

// Order here is the left-to-right order on the board.
export const COLUMNS: ColumnMeta[] = [
  { status: 'todo', label: 'To Do', accent: '#8A909B' },
  { status: 'in_progress', label: 'In Progress', accent: '#2B2BF5' },
  { status: 'in_review', label: 'In Review', accent: '#E08600' },
  { status: 'done', label: 'Done / Discard', accent: '#0FA968' },
];

export const PRIORITY_META: Record<TicketPriority, { label: string; accent: string }> = {
  low: { label: 'Low', accent: '#8A909B' },
  medium: { label: 'Medium', accent: '#0369A1' },
  high: { label: 'High', accent: '#E08600' },
  urgent: { label: 'Urgent', accent: '#DC2626' },
};

// ─────────────────────────────────────────────────────────────────────────
// Permission model.
//
// Requirement: only the requester (ticket owner) may move a ticket into the
// terminal `done` column. Any user may move it among todo / in_progress /
// in_review. `done` covers both "completed" and "discarded" resolutions —
// both are owner-only since both close the ticket.
//
// This is a UX gate only. The /tickets/:id/status endpoint MUST re-check
// ownership server-side; never rely on the disabled button alone.
// ─────────────────────────────────────────────────────────────────────────

export const isOwner = (ticket: Ticket, currentUserId: string) =>
  ticket.owner?.id === currentUserId;

/** Can `currentUserId` move `ticket` into `target`? (owner rule only — see
 *  `canDropTicket` for the combined owner + sequence check used everywhere
 *  a move is actually attempted.) */
export const canTransition = (
  ticket: Ticket,
  target: TicketStatus,
  currentUserId: string
): boolean => {
  if (target === 'done') return isOwner(ticket, currentUserId);
  return true;
};

export const OWNER_ONLY_HINT = 'Only the requester can close this ticket.';

// ─────────────────────────────────────────────────────────────────────────
// Sequence rule: a ticket may only advance one column at a time — a
// forward move (e.g. To Do → In Review, or In Progress → Done) that skips
// over an intermediate column is not allowed. Moving *backward* to any
// earlier column, from anywhere, is always allowed — e.g. Done → To Do,
// In Review → To Do, In Progress → To Do are all fine.
// ─────────────────────────────────────────────────────────────────────────

const COLUMN_ORDER: TicketStatus[] = ['todo', 'in_progress', 'in_review', 'done'];

export const isSequentialMove = (from: TicketStatus, to: TicketStatus): boolean => {
  const fromIndex = COLUMN_ORDER.indexOf(from);
  const toIndex = COLUMN_ORDER.indexOf(to);
  if (toIndex <= fromIndex) return true; // backward (or no-op) — always fine
  return toIndex === fromIndex + 1; // forward — only one step at a time
};

export const SEQUENCE_HINT = "Move one step at a time — you can't skip a column.";

export interface DropCheck {
  ok: boolean;
  reason?: string;
}

/** The single source of truth for "can this ticket move to this column right
 *  now" — combines the sequence rule and the owner-only-close rule. Use this
 *  (not `canTransition`/`isSequentialMove` individually) at every point a
 *  move is attempted or a drop target's valid/locked state is computed. */
export const canDropTicket = (
  ticket: Ticket,
  target: TicketStatus,
  currentUserId: string
): DropCheck => {
  if (!isSequentialMove(ticket.status, target)) {
    return { ok: false, reason: SEQUENCE_HINT };
  }
  if (!canTransition(ticket, target, currentUserId)) {
    return { ok: false, reason: OWNER_ONLY_HINT };
  }
  return { ok: true };
};

/** Two-letter initials for an avatar chip. */
export const initials = (user?: TicketUser | null) => {
  if (!user?.name) return '?';
  const parts = user.name.trim().split(/\s+/);
  return (parts[0][0] + (parts[1]?.[0] ?? '')).toUpperCase();
};

/** Deterministic accent for an avatar, derived from the user id. */
export const avatarAccent = (user?: TicketUser | null) => {
  const palette = ['#2B2BF5', '#0FA968', '#E08600', '#DC2626', '#0369A1', '#DB2777'];
  if (!user?.id) return palette[0];
  let h = 0;
  for (let i = 0; i < user.id.length; i++) h = (h * 31 + user.id.charCodeAt(i)) >>> 0;
  return palette[h % palette.length];
};
