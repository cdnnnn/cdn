//Ticketmeta.ts
import DOMPurify from 'dompurify';
import { Flame, ArrowUp, Minus, ArrowDown } from 'lucide-react';
import type { TicketStatus, TicketPriority, Ticket, TicketUser } from '../../types/tickets';

// ─────────────────────────────────────────────────────────────────────────
// Sanitizes rich-text HTML (description / comment text) before it's ever
// injected via dangerouslySetInnerHTML.
//
// `ALLOWED_URI_REGEXP` is load-bearing, not decorative: every single image
// in this feature is a base64 `data:` URI — there is no hosted-URL path at
// all (see TicketDescriptionEditor.tsx's insertImage). If a sanitizer's
// default allowed-URI-scheme list doesn't happen to include `data:`, every
// image's `src` gets stripped (or the whole tag does) while the rest of the
// content — text, formatting, structure — renders completely normally.
// That's exactly the "images don't show in the read-only view, but the live
// editor (which never runs through this function) is fine" symptom this
// was written to fix: the live editor's images are plain React-rendered
// DOM, entirely unaffected by whatever this function strips; the read-only
// view's images only exist after surviving this sanitize call. Rather than
// trust whatever DOMPurify's current default scheme allowlist happens to
// permit, this explicitly adds `data:` to it alongside the standard safe
// schemes DOMPurify ships with by default.
//
// `ADD_ATTR: ['style']` is now just cheap insurance — images no longer
// carry any size info via style (see tiptapGridImage.tsx: fixed-size
// thumbnails, nothing per-image to persist) — but costs nothing to keep in
// case inline styles are ever used for something else in this content later.
const ALLOWED_URI_REGEXP =
  /^(?:(?:(?:f|ht)tps?|mailto|tel|callto|sms|cid|xmpp|data):|[^a-z]|[a-z+.\-]+(?:[^a-z+.\-:]|$))/i;

export const sanitizeTicketHtml = (html: string | null | undefined): string =>
  DOMPurify.sanitize(html ?? '', {
    ADD_ATTR: ['style'],
    ALLOWED_URI_REGEXP,
  });

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

// One small icon per level instead of relying on color alone to convey
// urgency — also reads faster at a glance than text alone on a small card.
export const PRIORITY_ICON: Record<TicketPriority, typeof Flame> = {
  low: ArrowDown,
  medium: Minus,
  high: ArrowUp,
  urgent: Flame,
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
export const OWNER_ONLY_DELETE_HINT = 'Only the requester can delete this ticket.';

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

/** The description field now stores rich-text HTML (from the description
 *  editor). An "empty" editor still outputs something like `<p></p>`, so a
 *  plain falsy/blank check isn't enough — strip tags and check what's left. */
export const isEmptyHtml = (html?: string | null): boolean => {
  if (!html) return true;
  return html.replace(/<[^>]*>/g, '').trim().length === 0;
};
