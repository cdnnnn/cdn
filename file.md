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
 *  plain falsy/blank check isn't enough — strip tags and check what's left.
 *
 *  Stripping tags to check for leftover text would also strip <img> tags
 *  themselves, though — a description that's *only* images (no text at
 *  all) would have nothing left after that and get misclassified as
 *  "empty", hiding the images entirely instead of rendering them. An image
 *  is content, so check for one before falling back to the text-only check. */
export const isEmptyHtml = (html?: string | null): boolean => {
  if (!html) return true;
  if (/<img[\s>]/i.test(html)) return false;
  return html.replace(/<[^>]*>/g, '').trim().length === 0;

};













//Ticketsslice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { ticketsApi } from '../../api/endpoints/tickets';
import type {
  Ticket,
  TicketStatus,
  TicketResolution,
  CreateTicketRequest,
  UpdateTicketRequest,
  AddCommentRequest,
} from '../../types/tickets';

type FetchStatus = 'idle' | 'loading' | 'succeeded' | 'failed';

interface MoveArg {
  id: string;
  status: TicketStatus;
  resolution?: TicketResolution | null;
}

interface TicketsState {
  items: Ticket[];
  status: FetchStatus;
  error: string | null;
  creating: boolean;
  updatingId: string | null;
  deletingId: string | null;
  commentingId: string | null;
  // Ids currently mid-move: optimistically applied in `pending`, confirmed in
  // `fulfilled`, rolled back in `rejected`.
  movingIds: string[];
  // Snapshot of {status, resolution} captured at move-start, keyed by id, so a
  // failed transition can be reverted to exactly where the card came from.
  rollback: Record<string, { status: TicketStatus; resolution?: TicketResolution | null }>;
}

const initialState: TicketsState = {
  items: [],
  status: 'idle',
  error: null,
  creating: false,
  updatingId: null,
  deletingId: null,
  commentingId: null,
  movingIds: [],
  rollback: {},
};

export const fetchTickets = createAsyncThunk('tickets/fetchAll', () => ticketsApi.list());

// Same "don't trust the mutation endpoint's response body, refetch instead"
// reasoning as updateTicket/moveTicket/addTicketComment below — this thunk
// had been left trusting `ticketsApi.create()`'s response directly, which
// is exactly the kind of gap that leaves a freshly-created ticket's content
// (e.g. an image just added while creating it) invisible until something
// else refetches.
export const createTicket = createAsyncThunk(
  'tickets/create',
  async (payload: CreateTicketRequest, { dispatch }) => {
    await ticketsApi.create(payload);
    await dispatch(fetchTickets());
  }
);

// Same "don't trust the mutation endpoint's response body, refetch instead"
// reasoning as addTicketComment below — this is also where the backend is
// expected to append an assignee-change history entry (docs/API-SPEC.md §3),
// so trusting a response that might not include the updated `history` array
// would silently leave that entry invisible until something else refetched.
export const updateTicket = createAsyncThunk(
  'tickets/update',
  async (payload: UpdateTicketRequest, { dispatch }) => {
    await ticketsApi.update(payload);
    await dispatch(fetchTickets());
  }
);

// The board moves the card the instant you drop it (see `pending` below) and
// only reconciles with the server response afterward, so drag-and-drop feels
// immediate. A rejection snaps it back. Like updateTicket/addTicketComment,
// this refetches rather than trusting the move endpoint's response body —
// that response is exactly where the backend appends a new status-history
// entry (docs/API-SPEC.md §4), and trusting a response shape that might not
// include it was the actual cause of "history doesn't show until I refresh".
export const moveTicket = createAsyncThunk(
  'tickets/move',
  async (payload: MoveArg, { dispatch }) => {
    await ticketsApi.move(payload);
    await dispatch(fetchTickets());
  }
);

export const deleteTicket = createAsyncThunk(
  'tickets/delete',
  async (id: string) => {
    const res = await ticketsApi.remove(id);
    return { id: res.id || id };
  }
);

// The comment endpoint's response shape can vary across backends (full
// updated ticket vs. just the created comment vs. some other envelope) —
// rather than trust it and upsert `action.payload` directly (which silently
// does nothing if that assumption is wrong, leaving the new comment
// invisible until something else refetches), refetch the authoritative
// list once the post succeeds. Same "mutate, then refetch" pattern already
// used by createCustomModel elsewhere in this app.
export const addTicketComment = createAsyncThunk(
  'tickets/addComment',
  async (payload: AddCommentRequest, { dispatch }) => {
    await ticketsApi.addComment(payload);
    await dispatch(fetchTickets());
  }
);

const ticketsSlice = createSlice({
  name: 'tickets',
  initialState,
  reducers: {},
  extraReducers: (builder) => {
    builder
      // ---- fetch ----------------------------------------------------------
      .addCase(fetchTickets.pending, (state) => {
        state.status = 'loading';
      })
      .addCase(fetchTickets.fulfilled, (state, action) => {
        state.status = 'succeeded';
        state.items = action.payload ?? [];
      })
      .addCase(fetchTickets.rejected, (state, action) => {
        state.status = 'failed';
        state.error = action.error.message || 'Failed to load tickets';
      })

      // ---- create ---------------------------------------------------------
      .addCase(createTicket.pending, (state) => {
        state.creating = true;
      })
      .addCase(createTicket.fulfilled, (state) => {
        state.creating = false;
        // state.items already refreshed by the fetchTickets() the thunk
        // dispatched internally — see the comment on createTicket above.
      })
      .addCase(createTicket.rejected, (state, action) => {
        state.creating = false;
        state.error = action.error.message || 'Failed to create ticket';
      })

      // ---- update (metadata) ---------------------------------------------
      .addCase(updateTicket.pending, (state, action) => {
        state.updatingId = action.meta.arg.id;
      })
      .addCase(updateTicket.fulfilled, (state) => {
        state.updatingId = null;
        // state.items already refreshed by the fetchTickets() the thunk
        // dispatched internally — see the comment on updateTicket above.
      })
      .addCase(updateTicket.rejected, (state, action) => {
        state.updatingId = null;
        state.error = action.error.message || 'Failed to update ticket';
      })

      // ---- move (optimistic) ---------------------------------------------
      .addCase(moveTicket.pending, (state, action) => {
        const { id, status, resolution } = action.meta.arg;
        const t = state.items.find((x) => x.id === id);
        if (!t) return;
        state.rollback[id] = { status: t.status, resolution: t.resolution ?? null };
        t.status = status;
        t.resolution = status === 'done' ? resolution ?? 'completed' : null;
        if (!state.movingIds.includes(id)) state.movingIds.push(id);
      })
      .addCase(moveTicket.fulfilled, (state, action) => {
        const { id } = action.meta.arg;
        state.movingIds = state.movingIds.filter((x) => x !== id);
        delete state.rollback[id];
        // state.items already refreshed by the fetchTickets() the thunk
        // dispatched internally, including the new history entry — see the
        // comment on moveTicket above.
      })
      .addCase(moveTicket.rejected, (state, action) => {
        const { id } = action.meta.arg;
        state.movingIds = state.movingIds.filter((x) => x !== id);
        const snap = state.rollback[id];
        const t = state.items.find((x) => x.id === id);
        if (t && snap) {
          t.status = snap.status;
          t.resolution = snap.resolution ?? null;
        }
        delete state.rollback[id];
        state.error = action.error.message || 'Failed to move ticket';
      })

      // ---- delete ---------------------------------------------------------
      .addCase(deleteTicket.pending, (state, action) => {
        state.deletingId = action.meta.arg;
      })
      .addCase(deleteTicket.fulfilled, (state, action) => {
        state.deletingId = null;
        state.items = state.items.filter((m) => m.id !== action.payload.id);
      })
      .addCase(deleteTicket.rejected, (state, action) => {
        state.deletingId = null;
        state.error = action.error.message || 'Failed to delete ticket';
      })

      // ---- add comment ------------------------------------------------------
      .addCase(addTicketComment.pending, (state, action) => {
        state.commentingId = action.meta.arg.ticket_id;
      })
      .addCase(addTicketComment.fulfilled, (state) => {
        state.commentingId = null;
        // state.items is already up to date — the thunk dispatched
        // fetchTickets() internally, and that action's own .fulfilled
        // reducer (above) already replaced state.items.
      })
      .addCase(addTicketComment.rejected, (state, action) => {
        state.commentingId = null;
        state.error = action.error.message || 'Failed to post comment';
      });
  },
});

export default ticketsSlice.reducer;
