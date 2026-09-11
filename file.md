//Tickets.ts
// ─────────────────────────────────────────────────────────────────────────
// Backend-backed implementation, wired up by default — see
// docs/API-SPEC.md for the full request/response contract for every method
// here. `mock/ticketsApi.mock.ts` has the same method names/signatures and
// can be swapped in for offline/local dev (see README "Offline / local dev
// mode") without touching anything outside `store/slices/ticketsSlice.ts`'s
// one import line.
// ─────────────────────────────────────────────────────────────────────────
import api from '../axiosInstance';
import type {
  Ticket,
  CreateTicketRequest,
  UpdateTicketRequest,
  MoveTicketRequest,
  AddCommentRequest,
} from '../../types/tickets';

export interface DeleteTicketResponse {
  status: string;
  id: string;
}

// Grouped the same way as modelsApi: one exported object, each method a thin
// wrapper that unwraps the axios response to just the payload the callers care
// about.
export const ticketsApi = {
  // GET /tickets — every ticket for the board
  list: () =>
    api.get<{ tickets: Ticket[] }>('/tickets').then((r) => r.data.tickets ?? []),

  // POST /tickets — returns the created ticket (with server key + timestamps)
  create: (payload: CreateTicketRequest) =>
    api.post<Ticket>('/tickets', payload).then((r) => r.data),

  // PATCH /tickets/:id — edit metadata (title/description/priority/labels/assignee)
  update: ({ id, ...rest }: UpdateTicketRequest) =>
    api.patch<Ticket>(`/tickets/${id}`, rest).then((r) => r.data),

  // PATCH /tickets/:id/status — dedicated transition endpoint. The backend MUST
  // re-check that the caller owns the ticket when `status === 'done'`; the UI
  // gate is a convenience, not a security boundary.
  move: ({ id, status, resolution }: MoveTicketRequest) =>
    api
      .patch<Ticket>(`/tickets/${id}/status`, { status, resolution })
      .then((r) => r.data),

  // DELETE /tickets/:id
  remove: (id: string) =>
    api.delete<DeleteTicketResponse>(`/tickets/${id}`).then((r) => r.data),

  // POST /tickets/:id/comments — appends a comment, returns the updated ticket
  addComment: ({ ticket_id, text }: AddCommentRequest) =>
    api.post<Ticket>(`/tickets/${ticket_id}/comments`, { text }).then((r) => r.data),
};



















//Users.ts
import api from '../axiosInstance';
import type { TicketUser } from '../../types/tickets';

// GET /users — id + name for every assignable teammate (the roster used by
// the Assignee dropdown). If the app already has a users/team endpoint
// under a different path, point `list` at that instead.
export const usersApi = {
  list: () => api.get<{ users: TicketUser[] }>('/users').then((r) => r.data.users ?? []),
};















//Uploads.ts
import api from '../axiosInstance';
import type { TicketAttachment } from '../../types/tickets';

// ─────────────────────────────────────────────────────────────────────────
// Image uploads for ticket attachments. Separate from the tickets endpoints
// because the upload itself is multipart/form-data while everything else in
// this feature is plain JSON — keeping it in its own file/module avoids
// mixing content-types inside ticketsApi.
// ─────────────────────────────────────────────────────────────────────────

export const uploadsApi = {
  // POST /uploads/images — multipart upload, returns hosted attachment
  // metadata (id + URL) the frontend then attaches to a ticket via
  // create/update. Kept as its own step (rather than bundling the file into
  // the ticket payload) so an image can start uploading the instant it's
  // picked, before the rest of the form is filled in or submitted.
  uploadImage: (file: File, onProgress?: (pct: number) => void) => {
    const form = new FormData();
    form.append('file', file);
    return api
      .post<TicketAttachment>('/uploads/images', form, {
        headers: { 'Content-Type': 'multipart/form-data' },
        onUploadProgress: (e) => {
          if (onProgress && e.total) onProgress(Math.round((e.loaded / e.total) * 100));
        },
      })
      .then((r) => r.data);
  },
};




















//Auth.ts
import api from '../axiosInstance';
import type { TicketUser } from '../../types/tickets';

// ─────────────────────────────────────────────────────────────────────────
// If your app already has a real auth/session slice, delete this file and
// point `store/slices/authSlice.ts`'s thunk at your existing "current user"
// selector or endpoint instead — this exists purely as the minimal
// current-user source the owner-only "move to Done" gate needs.
// ─────────────────────────────────────────────────────────────────────────

export const authApi = {
  // GET /auth/me — the signed-in user. Assumes session/JWT auth is already
  // handled by the shared axios instance (cookie or Authorization header
  // attached via interceptor) — this endpoint just resolves who that is.
  me: () => api.get<TicketUser>('/auth/me').then((r) => r.data),
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

export const createTicket = createAsyncThunk(
  'tickets/create',
  (payload: CreateTicketRequest) => ticketsApi.create(payload)
);

export const updateTicket = createAsyncThunk(
  'tickets/update',
  (payload: UpdateTicketRequest) => ticketsApi.update(payload)
);

// The board moves the card the instant you drop it (see `pending` below) and
// only reconciles with the server response afterward, so drag-and-drop feels
// immediate. A rejection snaps it back.
export const moveTicket = createAsyncThunk(
  'tickets/move',
  (payload: MoveArg) => ticketsApi.move(payload)
);

export const deleteTicket = createAsyncThunk(
  'tickets/delete',
  async (id: string) => {
    const res = await ticketsApi.remove(id);
    return { id: res.id || id };
  }
);

export const addTicketComment = createAsyncThunk(
  'tickets/addComment',
  (payload: AddCommentRequest) => ticketsApi.addComment(payload)
);

const upsert = (list: Ticket[], t: Ticket) => {
  const i = list.findIndex((x) => x.id === t.id);
  if (i === -1) return [t, ...list];
  const next = list.slice();
  next[i] = t;
  return next;
};

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
      .addCase(createTicket.fulfilled, (state, action) => {
        state.creating = false;
        state.items = upsert(state.items, action.payload);
      })
      .addCase(createTicket.rejected, (state, action) => {
        state.creating = false;
        state.error = action.error.message || 'Failed to create ticket';
      })

      // ---- update (metadata) ---------------------------------------------
      .addCase(updateTicket.pending, (state, action) => {
        state.updatingId = action.meta.arg.id;
      })
      .addCase(updateTicket.fulfilled, (state, action) => {
        state.updatingId = null;
        state.items = upsert(state.items, action.payload);
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
        state.items = upsert(state.items, action.payload); // trust the server copy
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
      .addCase(addTicketComment.fulfilled, (state, action) => {
        state.commentingId = null;
        state.items = upsert(state.items, action.payload);
      })
      .addCase(addTicketComment.rejected, (state, action) => {
        state.commentingId = null;
        state.error = action.error.message || 'Failed to post comment';
      });
  },
});

export default ticketsSlice.reducer;


















//Usersslice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { usersApi } from '../../api/endpoints/users';
import type { TicketUser } from '../../types/tickets';

// ─────────────────────────────────────────────────────────────────────────
// Assignee roster, backed by the real API. If the app already has a
// users/team/members slice, delete this file and point TicketBoard.tsx's
// `fetchTeamMembers` import and `state.users` selector at that one instead
// — don't run two competing sources of the same roster.
// ─────────────────────────────────────────────────────────────────────────

type FetchStatus = 'idle' | 'loading' | 'succeeded' | 'failed';

interface UsersState {
  items: TicketUser[];
  status: FetchStatus;
  error: string | null;
}

const initialState: UsersState = {
  items: [],
  status: 'idle',
  error: null,
};

// GET /users — id + name for every assignable teammate
export const fetchTeamMembers = createAsyncThunk('users/fetchAll', () => usersApi.list());

const usersSlice = createSlice({
  name: 'users',
  initialState,
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(fetchTeamMembers.pending, (state) => {
        state.status = 'loading';
      })
      .addCase(fetchTeamMembers.fulfilled, (state, action) => {
        state.status = 'succeeded';
        state.items = action.payload ?? [];
      })
      .addCase(fetchTeamMembers.rejected, (state, action) => {
        state.status = 'failed';
        state.error = action.error.message || 'Failed to load team members';
      });
  },
});

export default usersSlice.reducer;




















//Authslice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { authApi } from '../../api/endpoints/auth';
import type { TicketUser } from '../../types/tickets';

// ─────────────────────────────────────────────────────────────────────────
// Minimal current-user slice for the owner-only gate in TicketBoard. If the
// app already has a real auth/session slice, delete this file and repoint
// TicketBoard.tsx's `useAppSelector((s) => s.auth.user)` at the existing one
// — don't run two competing sources of "who's signed in".
// ─────────────────────────────────────────────────────────────────────────

type FetchStatus = 'idle' | 'loading' | 'succeeded' | 'failed';

interface AuthState {
  user: TicketUser | null;
  status: FetchStatus;
  error: string | null;
}

const initialState: AuthState = {
  user: null,
  status: 'idle',
  error: null,
};

export const fetchCurrentUser = createAsyncThunk('auth/fetchCurrentUser', () => authApi.me());

const authSlice = createSlice({
  name: 'auth',
  initialState,
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(fetchCurrentUser.pending, (state) => {
        state.status = 'loading';
      })
      .addCase(fetchCurrentUser.fulfilled, (state, action) => {
        state.status = 'succeeded';
        state.user = action.payload;
      })
      .addCase(fetchCurrentUser.rejected, (state, action) => {
        state.status = 'failed';
        state.error = action.error.message || 'Failed to load current user';
      });
  },
});

export default authSlice.reducer;





















//Store.ts
import { configureStore } from '@reduxjs/toolkit';

// ── existing slices — keep whatever you already have here ──────────────
import modelsReducer from './slices/modelsSlice';
// import providersReducer from './slices/providersSlice';
// import ...other existing slices

// ── new for the requirements/tickets feature ────────────────────────────
import ticketsReducer from './slices/ticketsSlice';
import usersReducer from './slices/usersSlice';
// authSlice is only needed if the app doesn't already have a real
// auth/session slice — see its file header for how to swap it out.
import authReducer from './slices/authSlice';

export const store = configureStore({
  reducer: {
    models: modelsReducer,
    // providers: providersReducer,
    // ...other existing reducers

    tickets: ticketsReducer,
    users: usersReducer,
    auth: authReducer,
  },
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;





















//Ticketboard.tsx
import { useEffect, useMemo, useState } from 'react';
import {
  DndContext,
  DragOverlay,
  PointerSensor,
  useSensor,
  useSensors,
  closestCenter,
  type DragEndEvent,
  type DragStartEvent,
} from '@dnd-kit/core';
import { Plus, Search, Loader2, Layers, ListFilter } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import {
  fetchTickets,
  createTicket,
  updateTicket,
  moveTicket,
  deleteTicket,
} from '../../store/slices/ticketsSlice';
import { fetchTeamMembers } from '../../store/slices/usersSlice';
import { fetchCurrentUser } from '../../store/slices/authSlice';
import type { Ticket, TicketStatus, TicketResolution, TicketPriority } from '../../types/tickets';
import { useToast } from '../common/Toast';
import { COLUMNS, PRIORITY_META, canTransition, OWNER_ONLY_HINT } from './ticketMeta';
import TicketCard from './TicketCard';
import TicketColumn from './TicketColumn';
import CreateTicketDrawer, { type TicketSubmitPayload } from './CreateTicketDrawer';
import TicketDetailSidebar from './TicketDetailSidebar';
import TicketCloseConfirm from './TicketCloseConfirm';
import styles from './TicketBoard.module.scss';

// ─────────────────────────────────────────────────────────────────────────
// Self-contained: no props. Fetches its own tickets, its own assignee
// roster, AND its own current-user identity (via authSlice / GET /auth/me)
// — the owner-only "move to Done" gate depends on knowing who's signed in.
//
// If the app already has a real auth/session slice, delete
// `store/slices/authSlice.ts` and swap the `useAppSelector((s) => s.auth.user)`
// line below for whatever selector/hook your existing auth already exposes
// — nothing else in this component needs to change.
// ─────────────────────────────────────────────────────────────────────────

export default function TicketBoard() {
  const dispatch = useAppDispatch();
  const toast = useToast();
  const { items, status, creating, updatingId, movingIds, deletingId } = useAppSelector(
    (s) => s.tickets
  );
  const currentUser = useAppSelector((s) => s.auth.user);
  const authStatus = useAppSelector((s) => s.auth.status);
  const members = useAppSelector((s) => s.users.items);
  const membersStatus = useAppSelector((s) => s.users.status);

  const [drawerOpen, setDrawerOpen] = useState(false);
  const [editing, setEditing] = useState<Ticket | null>(null);
  const [detail, setDetail] = useState<Ticket | null>(null);

  const [query, setQuery] = useState('');
  const [priorityFilter, setPriorityFilter] = useState<TicketPriority | 'all'>('all');

  // The ticket id currently being dragged, driving both the floating
  // DragOverlay clone and each column's locked/over highlighting.
  const [activeId, setActiveId] = useState<string | null>(null);

  // Set whenever a move would land a ticket in the terminal column; renders
  // TicketCloseConfirm instead of moving immediately. Cleared on choose/cancel.
  const [pendingClose, setPendingClose] = useState<{ id: string; key: string; title: string } | null>(
    null
  );

  // A short activation distance keeps ordinary clicks (open card, menu
  // buttons) working normally — a drag only "activates" once the pointer
  // has moved a few pixels past its starting point.
  const sensors = useSensors(useSensor(PointerSensor, { activationConstraint: { distance: 6 } }));

  useEffect(() => {
    if (authStatus === 'idle') dispatch(fetchCurrentUser());
  }, [authStatus, dispatch]);

  useEffect(() => {
    if (status === 'idle') dispatch(fetchTickets());
  }, [status, dispatch]);

  useEffect(() => {
    if (membersStatus === 'idle') dispatch(fetchTeamMembers());
  }, [membersStatus, dispatch]);

  // Keep the open detail panel in sync with the store (e.g. after a move).
  useEffect(() => {
    if (!detail) return;
    const fresh = items.find((t) => t.id === detail.id);
    if (fresh && fresh !== detail) setDetail(fresh);
    if (!fresh) setDetail(null);
  }, [items, detail]);

  const activeTicket = activeId ? items.find((t) => t.id === activeId) ?? null : null;

  const filtered = useMemo(() => {
    const q = query.trim().toLowerCase();
    return items.filter((t) => {
      if (priorityFilter !== 'all' && t.priority !== priorityFilter) return false;
      if (!q) return true;
      return (
        t.title.toLowerCase().includes(q) ||
        t.key.toLowerCase().includes(q) ||
        (t.labels ?? []).some((l) => l.toLowerCase().includes(q))
      );
    });
  }, [items, query, priorityFilter]);

  const byStatus = useMemo(() => {
    const map: Record<TicketStatus, Ticket[]> = {
      todo: [],
      in_progress: [],
      in_review: [],
      done: [],
    };
    for (const t of filtered) map[t.status].push(t);
    return map;
  }, [filtered]);

  // currentUser drives the owner-only "move to Done" gate everywhere below,
  // so don't render the interactive board until it's resolved. Placed after
  // every hook above (rules of hooks) — TypeScript also narrows `currentUser`
  // from `TicketUser | null` to `TicketUser` for the rest of this function
  // once this guard has passed.
  if (!currentUser) {
    return (
      <div className={`page-enter pg-shell ${styles['ticket-board']}`}>
        <div className={styles['ticket-board__loading']}>
          <Loader2 size={20} className={styles['ticket-board__spin']} />
          <span>
            {authStatus === 'failed'
              ? "Couldn't load your account — refresh to retry."
              : 'Loading…'}
          </span>
        </div>
      </div>
    );
  }

  const runMove = (id: string, to: TicketStatus, resolution?: TicketResolution | null) => {
    const ticket = items.find((t) => t.id === id);
    if (!ticket || ticket.status === to && !resolution) return;
    if (!canTransition(ticket, to, currentUser.id)) {
      toast.warning(OWNER_ONLY_HINT);
      return;
    }
    dispatch(moveTicket({ id, status: to, resolution }))
      .unwrap()
      .then(() => {
        if (to === 'done') {
          toast.success(resolution === 'discarded' ? 'Ticket discarded' : 'Ticket marked done');
        }
      })
      .catch((e) => toast.error(typeof e === 'string' ? e : 'Could not move ticket'));
  };

  // Entry point for every move request (drag, card menu, detail rail).
  // Moves into the terminal column always pause for an explicit Done/Discard
  // choice instead of executing immediately — the caller's `resolution`
  // guess (e.g. a drag onto Done) is discarded in favor of what the user
  // actually confirms.
  const requestMove = (id: string, to: TicketStatus) => {
    if (to !== 'done') {
      runMove(id, to);
      return;
    }
    const ticket = items.find((t) => t.id === id);
    if (!ticket) return;
    if (!canTransition(ticket, 'done', currentUser.id)) {
      toast.warning(OWNER_ONLY_HINT);
      return;
    }
    setPendingClose({ id, key: ticket.key, title: ticket.title });
  };

  const handleDragStart = (event: DragStartEvent) => {
    setActiveId(String(event.active.id));
  };

  const handleDragEnd = (event: DragEndEvent) => {
    const { active, over } = event;
    setActiveId(null);
    if (!over) return;
    const to = over.id as TicketStatus;
    const ticket = items.find((t) => t.id === active.id);
    if (!ticket || ticket.status === to) return;
    if (!canTransition(ticket, to, currentUser.id)) {
      toast.warning(OWNER_ONLY_HINT);
      return;
    }
    requestMove(String(active.id), to);
  };

  const handleSubmit = (payload: TicketSubmitPayload) => {
    const action = editing
      ? updateTicket({ id: editing.id, ...payload })
      : createTicket(payload);
    return dispatch(action)
      .unwrap()
      .then(() => {
        toast.success(editing ? 'Ticket updated' : 'Requirement created');
        setDrawerOpen(false);
        setEditing(null);
      })
      .catch((e) => {
        toast.error(typeof e === 'string' ? e : 'Could not save ticket');
        throw e; // keep the drawer open
      });
  };

  const handleDelete = (id: string) => {
    dispatch(deleteTicket(id))
      .unwrap()
      .then(() => {
        toast.success('Ticket deleted');
        setDetail(null);
      })
      .catch((e) => toast.error(typeof e === 'string' ? e : 'Could not delete ticket'));
  };

  return (
    <div className={`page-enter pg-shell ${styles['ticket-board']}`}>
      <div className={styles['ticket-board__header']}>
        <div>
          <p className={styles['ticket-board__header-eyebrow']}>Delivery</p>
          <h1>Requirements</h1>
          <p className={styles['ticket-board__header-sub']}>
            Drag a card between columns to change its status
          </p>
        </div>
        <div className={styles['ticket-board__header-meta']}>
          <Layers size={13} />
          {items.length} ticket{items.length === 1 ? '' : 's'}
        </div>
      </div>

      <div className={styles['ticket-board__toolbar']}>
        <div className={styles['ticket-board__search']}>
          <Search size={16} />
          <input
            value={query}
            onChange={(e) => setQuery(e.target.value)}
            placeholder="Search key, title, label…"
            aria-label="Search tickets"
          />
        </div>

        <div className={styles['ticket-board__toolbar-right']}>
          <div className={styles['ticket-board__filter-group']}>
            <span className={styles['ticket-board__toolbar-label']}>
              <ListFilter size={11} /> Priority
            </span>
            {(['all', ...Object.keys(PRIORITY_META)] as (TicketPriority | 'all')[]).map((p) => (
              <button
                key={p}
                className={`${styles['ticket-board__filter-pill']} ${
                  priorityFilter === p ? styles['ticket-board__filter-pill--on'] : ''
                }`}
                onClick={() => setPriorityFilter(p)}
              >
                {p === 'all' ? 'All' : PRIORITY_META[p].label}
              </button>
            ))}
          </div>

          <div className={styles['ticket-board__toolbar-divider']} />

          <button
            className={styles['ticket-board__add-btn']}
            onClick={() => {
              setEditing(null);
              setDrawerOpen(true);
            }}
          >
            <Plus size={16} />
            New Requirement
          </button>
        </div>
      </div>

      {status === 'loading' && items.length === 0 ? (
        <div className={styles['ticket-board__loading']}>
          <Loader2 size={20} className={styles['ticket-board__spin']} />
          <span>Loading board…</span>
        </div>
      ) : (
        <DndContext
          sensors={sensors}
          collisionDetection={closestCenter}
          onDragStart={handleDragStart}
          onDragEnd={handleDragEnd}
          onDragCancel={() => setActiveId(null)}
        >
          <div className={styles['ticket-board__columns']}>
            {COLUMNS.map((col) => (
              <TicketColumn
                key={col.status}
                col={col}
                tickets={byStatus[col.status]}
                currentUser={currentUser}
                movingIds={movingIds}
                activeTicket={activeTicket}
                onOpen={setDetail}
                onMove={requestMove}
              />
            ))}
          </div>

          {/* The floating clone that actually follows the cursor — this is
              what makes the drag read as "the card itself is moving"
              instead of the browser's native drag snapshot. */}
          <DragOverlay dropAnimation={{ duration: 180, easing: 'cubic-bezier(0.18, 0.67, 0.6, 1.22)' }}>
            {activeTicket ? (
              <div className={styles['ticket-board__drag-overlay']}>
                <TicketCard ticket={activeTicket} currentUser={currentUser} onOpen={() => {}} onMove={() => {}} overlay />
              </div>
            ) : null}
          </DragOverlay>
        </DndContext>
      )}

      {drawerOpen && (
        <CreateTicketDrawer
          mode={editing ? 'edit' : 'create'}
          initialTicket={editing ?? undefined}
          members={members}
          submitting={editing ? updatingId === editing.id : creating}
          onClose={() => {
            setDrawerOpen(false);
            setEditing(null);
          }}
          onSubmit={handleSubmit}
        />
      )}

      {detail && (
        <TicketDetailSidebar
          ticket={detail}
          currentUser={currentUser}
          moving={movingIds.includes(detail.id)}
          deleting={deletingId === detail.id}
          onClose={() => setDetail(null)}
          onMove={requestMove}
          onEdit={(t) => {
            setDetail(null);
            setEditing(t);
            setDrawerOpen(true);
          }}
          onDelete={handleDelete}
        />
      )}

      {pendingClose && (
        <TicketCloseConfirm
          ticketKey={pendingClose.key}
          title={pendingClose.title}
          onCancel={() => setPendingClose(null)}
          onConfirm={(resolution) => {
            runMove(pendingClose.id, 'done', resolution);
            setPendingClose(null);
          }}
        />
      )}
    </div>
  );
}




















//Createticketdrawer.tsx
import { useEffect, useMemo, useRef, useState } from 'react';
import { X, Loader2, ImagePlus, FileText, RotateCw } from 'lucide-react';
import type { Ticket, TicketAttachment, TicketPriority, TicketUser } from '../../types/tickets';
import { uploadsApi } from '../../api/endpoints/uploads';
import { PRIORITY_META, initials, avatarAccent } from './ticketMeta';
import TicketSelect from './TicketSelect';
import styles from './CreateTicketDrawer.module.scss';

/** What the drawer hands back on submit — no id/status, the board adds those. */
export interface TicketSubmitPayload {
  title: string;
  description?: string;
  priority: TicketPriority;
  labels?: string[];
  assignee_id?: string | null;
  attachments?: TicketAttachment[];
}

interface CreateTicketDrawerProps {
  mode: 'create' | 'edit';
  initialTicket?: Ticket;
  members?: TicketUser[];
  submitting?: boolean;
  onClose: () => void;
  /** Return a Promise to keep the modal open on failure and close it on success. */
  onSubmit: (payload: TicketSubmitPayload) => Promise<unknown> | void;
}

// A local attachment that's mid-upload, uploaded, or failed. Only 'done'
// entries have a real, server-hosted `url` and are included in the submit
// payload — 'uploading'/'failed' rows are drawn from `file`/`localUrl`
// (an object URL) purely for the in-progress thumbnail.
interface DraftAttachment extends TicketAttachment {
  status: 'uploading' | 'done' | 'failed';
  progress?: number;
  errorMessage?: string;
  localUrl?: string;
  file?: File;
}

const MAX_ATTACHMENT_BYTES = 5 * 1024 * 1024; // 5MB per image

const formatSize = (bytes: number) =>
  bytes < 1024 * 1024 ? `${Math.round(bytes / 1024)} KB` : `${(bytes / (1024 * 1024)).toFixed(1)} MB`;

// Centered modal, same shell as the rest of the app's dialogs (see
// .modal-overlay / .modal in this file's SCSS module). Rendered inline —
// no portal.
export default function CreateTicketDrawer({
  mode,
  initialTicket,
  members = [],
  submitting = false,
  onClose,
  onSubmit,
}: CreateTicketDrawerProps) {
  const [title, setTitle] = useState(initialTicket?.title ?? '');
  const [description, setDescription] = useState(initialTicket?.description ?? '');
  const [priority, setPriority] = useState<TicketPriority>(initialTicket?.priority ?? 'medium');
  const [labels, setLabels] = useState<string[]>(initialTicket?.labels ?? []);
  const [labelDraft, setLabelDraft] = useState('');
  const [assigneeId, setAssigneeId] = useState<string>(initialTicket?.assignee?.id ?? '');
  const [attachments, setAttachments] = useState<DraftAttachment[]>(
    (initialTicket?.attachments ?? []).map((a) => ({ ...a, status: 'done' as const }))
  );
  const [attachError, setAttachError] = useState('');
  const [touched, setTouched] = useState(false);

  const firstFieldRef = useRef<HTMLInputElement | null>(null);
  const fileInputRef = useRef<HTMLInputElement | null>(null);

  useEffect(() => {
    firstFieldRef.current?.focus();
  }, []);

  useEffect(() => {
    const onKey = (e: KeyboardEvent) => {
      if (e.key === 'Escape' && !submitting) onClose();
    };
    document.addEventListener('keydown', onKey);
    return () => document.removeEventListener('keydown', onKey);
  }, [onClose, submitting]);

  // Revoke any object URLs created for local previews on unmount, so we
  // don't leak blob: URLs once the modal closes.
  useEffect(() => {
    return () => {
      attachments.forEach((a) => {
        if (a.localUrl) URL.revokeObjectURL(a.localUrl);
      });
    };
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  const titleError = touched && !title.trim() ? 'Title is required' : '';
  const anyUploading = attachments.some((a) => a.status === 'uploading');
  const valid = title.trim().length > 0 && !anyUploading;

  const addLabel = () => {
    const v = labelDraft.trim();
    if (!v) return;
    if (!labels.includes(v)) setLabels((prev) => [...prev, v]);
    setLabelDraft('');
  };

  // Kicks off the real upload for one already-added draft row, identified
  // by its temporary local id, and swaps that row for the server's response
  // (or marks it failed) once the request settles.
  const runUpload = (localId: string, file: File) => {
    uploadsApi
      .uploadImage(file, (pct) => {
        setAttachments((prev) => prev.map((a) => (a.id === localId ? { ...a, progress: pct } : a)));
      })
      .then((uploaded) => {
        setAttachments((prev) =>
          prev.map((a) => {
            if (a.id !== localId) return a;
            if (a.localUrl) URL.revokeObjectURL(a.localUrl);
            return { ...uploaded, status: 'done' };
          })
        );
      })
      .catch((e) => {
        setAttachments((prev) =>
          prev.map((a) =>
            a.id === localId
              ? { ...a, status: 'failed', errorMessage: e instanceof Error ? e.message : 'Upload failed' }
              : a
          )
        );
      });
  };

  const handleFiles = (fileList: FileList | null) => {
    if (!fileList || fileList.length === 0) return;
    setAttachError('');
    const files = Array.from(fileList);
    const oversized = files.find((f) => f.size > MAX_ATTACHMENT_BYTES);
    if (oversized) {
      setAttachError(`"${oversized.name}" is over 5MB — pick a smaller image.`);
      return;
    }
    const nonImage = files.find((f) => !f.type.startsWith('image/'));
    if (nonImage) {
      setAttachError('Only image files can be attached.');
      return;
    }

    const drafts: DraftAttachment[] = files.map((f) => ({
      id: `local-${Date.now()}-${Math.random().toString(36).slice(2, 7)}`,
      name: f.name,
      url: '',
      size: f.size,
      status: 'uploading',
      progress: 0,
      localUrl: URL.createObjectURL(f),
      file: f,
    }));
    setAttachments((prev) => [...prev, ...drafts]);
    drafts.forEach((d) => runUpload(d.id, d.file!));
  };

  const retryUpload = (a: DraftAttachment) => {
    if (!a.file) return;
    setAttachments((prev) =>
      prev.map((x) => (x.id === a.id ? { ...x, status: 'uploading', progress: 0, errorMessage: undefined } : x))
    );
    runUpload(a.id, a.file);
  };

  const removeAttachment = (id: string) => {
    setAttachments((prev) => {
      const target = prev.find((a) => a.id === id);
      if (target?.localUrl) URL.revokeObjectURL(target.localUrl);
      return prev.filter((a) => a.id !== id);
    });
  };

  const submit = () => {
    setTouched(true);
    if (!valid) return;
    const readyAttachments: TicketAttachment[] = attachments
      .filter((a) => a.status === 'done')
      .map(({ id, name, url, size }) => ({ id, name, url, size }));
    onSubmit({
      title: title.trim(),
      description: description.trim() || undefined,
      priority,
      labels: labels.length ? labels : undefined,
      assignee_id: assigneeId || null,
      attachments: readyAttachments.length ? readyAttachments : undefined,
    });
  };

  const heading = mode === 'edit' ? 'Edit requirement' : 'New requirement';
  const cta = mode === 'edit' ? 'Save changes' : 'Create requirement';

  const priorityOptions = useMemo(
    () =>
      (Object.keys(PRIORITY_META) as TicketPriority[]).map((p) => ({
        value: p,
        label: PRIORITY_META[p].label,
        accent: PRIORITY_META[p].accent,
      })),
    []
  );

  const assigneeOptions = useMemo(
    () => [
      { value: '', label: 'Unassigned' },
      ...members.map((m) => ({
        value: m.id,
        label: m.name,
        accent: avatarAccent(m),
        glyph: initials(m),
      })),
    ],
    [members]
  );

  return (
    <div className={styles['modal-overlay']} onClick={() => !submitting && onClose()}>
      <div
        className={styles['modal']}
        role="dialog"
        aria-modal="true"
        aria-label={heading}
        onClick={(e) => e.stopPropagation()}
      >
        <header className={styles['modal-hdr']}>
          <div className={styles['modal-header-text']}>
            <span className={styles['modal-eyebrow']}>{mode === 'edit' ? 'Editing' : 'New'}</span>
            <span className={styles['modal-title']}>{heading}</span>
          </div>
          <button
            className={styles['modal-close']}
            onClick={onClose}
            disabled={submitting}
            aria-label="Close"
          >
            <X size={16} />
          </button>
        </header>

        <div className={styles['modal-body']}>
          {/* ---- main column: title, description, attachments ---- */}
          <div className={styles['modal-main']}>
            <label className={styles['field']}>
              <span className={styles['field-label']}>
                Title <span className={styles['field-req']}>*</span>
              </span>
              <input
                ref={firstFieldRef}
                className={`${styles['field-input']} ${styles['field-input--lg']} ${titleError ? styles['field-input--error'] : ''}`}
                value={title}
                onChange={(e) => setTitle(e.target.value)}
                onBlur={() => setTouched(true)}
                placeholder="Short summary of the requirement"
              />
              {titleError && <span className={styles['field-error']}>{titleError}</span>}
            </label>

            <label className={styles['field']}>
              <span className={styles['field-label']}>Description</span>
              <textarea
                className={styles['field-textarea']}
                value={description}
                onChange={(e) => setDescription(e.target.value)}
                rows={7}
                placeholder="Context, acceptance criteria, links…"
              />
            </label>

            <div className={styles['field']}>
              <span className={styles['field-label']}>Attachments</span>
              <input
                ref={fileInputRef}
                type="file"
                accept="image/*"
                multiple
                hidden
                onChange={(e) => {
                  handleFiles(e.target.files);
                  e.target.value = '';
                }}
              />
              <div className={styles['attach-zone']}>
                {attachments.map((a) => (
                  <div
                    key={a.id}
                    className={`${styles['attach-thumb']} ${a.status === 'failed' ? styles['attach-thumb--failed'] : ''}`}
                  >
                    <img src={a.status === 'done' ? a.url : a.localUrl} alt={a.name} />

                    {a.status === 'uploading' && (
                      <div className={styles['attach-progress']}>
                        <Loader2 size={16} className={styles['spin']} />
                        {typeof a.progress === 'number' && (
                          <span className={styles['attach-progress-pct']}>{a.progress}%</span>
                        )}
                      </div>
                    )}

                    {a.status === 'failed' && (
                      <button
                        type="button"
                        className={styles['attach-retry']}
                        onClick={() => retryUpload(a)}
                        title={a.errorMessage || 'Upload failed — retry'}
                      >
                        <RotateCw size={14} />
                        Retry
                      </button>
                    )}

                    <button
                      type="button"
                      className={styles['attach-remove']}
                      onClick={() => removeAttachment(a.id)}
                      aria-label={`Remove ${a.name}`}
                    >
                      <X size={11} />
                    </button>
                    {a.status === 'done' && (
                      <span className={styles['attach-meta']}>{formatSize(a.size)}</span>
                    )}
                  </div>
                ))}
                <button
                  type="button"
                  className={styles['attach-add']}
                  onClick={() => fileInputRef.current?.click()}
                >
                  <ImagePlus size={18} />
                  Add image
                </button>
              </div>
              {attachError && <span className={styles['field-error']}>{attachError}</span>}
              {anyUploading && (
                <span className={styles['field-note']} style={{ marginTop: 0, borderTop: 'none', paddingTop: 0 }}>
                  Uploading — you can keep filling out the form, just wait before saving.
                </span>
              )}
            </div>
          </div>

          {/* ---- meta column: priority, assignee, labels ---- */}
          <div className={styles['modal-rail']}>
            <div className={styles['field']}>
              <span className={styles['field-label']}>Priority</span>
              <TicketSelect
                value={priority}
                options={priorityOptions}
                onChange={(v) => setPriority(v as TicketPriority)}
                aria-label="Priority"
              />
            </div>

            <div className={styles['field']}>
              <span className={styles['field-label']}>Assignee</span>
              <TicketSelect
                value={assigneeId}
                options={assigneeOptions}
                placeholder="Unassigned"
                onChange={setAssigneeId}
                disabled={members.length === 0}
                aria-label="Assignee"
              />
            </div>

            <div className={styles['field']}>
              <span className={styles['field-label']}>Labels</span>
              <div className={styles['chip-input']}>
                {labels.map((l) => (
                  <span key={l} className={styles['chip']}>
                    {l}
                    <button
                      type="button"
                      onClick={() => setLabels((prev) => prev.filter((x) => x !== l))}
                      aria-label={`Remove ${l}`}
                    >
                      <X size={12} />
                    </button>
                  </span>
                ))}
                <input
                  className={styles['chip-field']}
                  value={labelDraft}
                  onChange={(e) => setLabelDraft(e.target.value)}
                  onKeyDown={(e) => {
                    if (e.key === 'Enter' || e.key === ',') {
                      e.preventDefault();
                      addLabel();
                    } else if (e.key === 'Backspace' && !labelDraft && labels.length) {
                      setLabels((prev) => prev.slice(0, -1));
                    }
                  }}
                  placeholder={labels.length ? 'Add another…' : 'Type, press Enter'}
                />
              </div>
            </div>

            {mode === 'edit' && initialTicket && (
              <div className={styles['field-note']}>
                <FileText size={12} />
                {initialTicket.key} · created{' '}
                {new Date(initialTicket.created_at).toLocaleDateString()}
              </div>
            )}
          </div>
        </div>

        <footer className={styles['modal-foot']}>
          <button className={styles['modal-cancel']} onClick={onClose} disabled={submitting}>
            Cancel
          </button>
          <button className={styles['modal-submit']} onClick={submit} disabled={submitting || !valid}>
            {submitting ? (
              <>
                <Loader2 size={15} className={styles['spin']} />
                Saving…
              </>
            ) : (
              cta
            )}
          </button>
        </footer>
      </div>
    </div>
  );
}


















//Createticketdrawer.module.scss
@use '../../styles/_variables' as *;

// Centered modal — same shell convention used across the app (see
// Datasets.module.scss's __modal-overlay/__modal, and TicketBoard's
// .modal-overlay/.modal). Rendered inline, no portal: a full-viewport
// fixed overlay that flex-centers its panel.

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;

@keyframes ticket-fade-in {
  from { opacity: 0; }
  to { opacity: 1; }
}
@keyframes ticket-modal-in {
  from { opacity: 0; transform: translateY(8px) scale(0.98); }
  to { opacity: 1; transform: none; }
}
@keyframes ticket-spin {
  to { transform: rotate(360deg); }
}

.modal-overlay {
  position: fixed;
  inset: 0;
  z-index: 200;
  background: rgba(20, 22, 27, 0.45);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 24px;
  animation: ticket-fade-in 0.15s ease;
}
.modal {
  width: min(780px, 100%);
  max-height: 88vh;
  display: flex;
  flex-direction: column;
  background: $card;
  border: 1px solid $line;
  border-radius: 18px;
  box-shadow: 0 24px 60px -20px rgba(20, 22, 27, 0.4);
  overflow: hidden;
  animation: ticket-modal-in 0.18s cubic-bezier(0.22, 1, 0.36, 1);
  // Own base size, a touch larger than the page base at very wide
  // viewports — a focused modal reads better slightly bigger than the
  // dense board sitting behind it.
  font-size: 0.8125rem;
  @media (min-width: 1800px) {
    font-size: 1.0625rem;
  }
}

// ---- header -----------------------------------------------------------
.modal-hdr {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1.1em 1.25em;
  border-bottom: 1px solid $line;
}
.modal-header-text {
  display: flex;
  flex-direction: column;
  gap: 0.15em;
}
.modal-eyebrow {
  font-family: $mono;
  font-size: 0.68em;
  font-weight: 700;
  letter-spacing: 0.1em;
  text-transform: uppercase;
  color: $signal;
}
.modal-title {
  font-family: $display;
  font-size: 1.2em;
  font-weight: 700;
  color: $ink;
}
.modal-close {
  flex-shrink: 0;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 28px;
  height: 28px;
  border-radius: 8px;
  border: 1px solid $line;
  background: $paper;
  color: $ink-2;
  cursor: pointer;
  transition: border-color 0.15s ease, color 0.15s ease;
  &:hover:not(:disabled) { border-color: $ink-3; color: $ink; }
  &:disabled { opacity: 0.4; cursor: not-allowed; }
}

// ---- two-column body ----------------------------------------------------
.modal-body {
  flex: 1;
  min-height: 0;
  display: flex;
  overflow: hidden;
}
.modal-main {
  flex: 1;
  min-width: 0;
  overflow-y: auto;
  padding: 1.25em;
  display: flex;
  flex-direction: column;
  gap: 1.1em;
}
.modal-rail {
  flex: none;
  width: 260px;
  border-left: 1px solid $line;
  background: $paper;
  overflow-y: auto;
  padding: 1.25em;
  display: flex;
  flex-direction: column;
  gap: 1.1em;
}

.field {
  display: flex;
  flex-direction: column;
  gap: 0.4em;
}
.field-label {
  font-size: 0.78em;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.04em;
  color: $ink-2;
}
.field-req {
  color: $danger;
}
.field-input,
.field-textarea {
  width: 100%;
  border: 1px solid $line;
  border-radius: 8px;
  background: $card;
  color: $ink;
  font-size: 0.92em;
  font-family: inherit;
  padding: 0.65em 0.7em;
  outline: 0;
  transition: border-color 0.15s, box-shadow 0.15s;
  &:focus {
    border-color: $signal;
    box-shadow: 0 0 0 3px $wash;
  }
  &::placeholder {
    color: $ink-3;
  }
}
.field-input--lg {
  font-size: 1.15em;
  font-weight: 600;
  padding: 0.6em 0.7em;
}
.field-textarea {
  resize: vertical;
  line-height: 1.55;
  flex: 1;
}
.field-input--error {
  border-color: $danger;
  &:focus {
    box-shadow: 0 0 0 3px $danger-wash;
  }
}
.field-error {
  font-size: 0.78em;
  color: $danger;
}
.field-note {
  margin-top: auto;
  display: flex;
  align-items: center;
  gap: 0.4em;
  font-size: 0.76em;
  color: $ink-3;
  padding-top: 0.8em;
  border-top: 1px dashed $line;
}

// ---- attachments ----------------------------------------------------------
.attach-zone {
  display: flex;
  flex-wrap: wrap;
  gap: 0.6em;
}
.attach-thumb {
  position: relative;
  width: 84px;
  height: 84px;
  border-radius: 8px;
  overflow: hidden;
  border: 1px solid $line;
  background: $paper;

  img {
    width: 100%;
    height: 100%;
    object-fit: cover;
    display: block;
  }
}
.attach-thumb--failed {
  border-color: $danger;

  img {
    opacity: 0.4;
  }
}
.attach-progress {
  position: absolute;
  inset: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 0.3em;
  background: rgba(17, 24, 39, 0.45);
  color: #fff;
}
.attach-progress-pct {
  font-size: 0.65em;
  font-weight: 700;
}
.attach-retry {
  position: absolute;
  inset: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 0.25em;
  border: 0;
  background: rgba(220, 38, 38, 0.88);
  color: #fff;
  font-size: 0.68em;
  font-weight: 700;
  cursor: pointer;
  &:hover {
    background: $danger;
  }
}
.attach-remove {
  position: absolute;
  top: 4px;
  right: 4px;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  border: 0;
  background: rgba(17, 24, 39, 0.65);
  color: #fff;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  &:hover {
    background: $danger;
  }
}
.attach-meta {
  position: absolute;
  left: 0;
  right: 0;
  bottom: 0;
  padding: 0.15em 0.35em;
  font-size: 0.6em;
  color: #fff;
  background: rgba(17, 24, 39, 0.55);
  text-align: center;
}
.attach-add {
  width: 84px;
  height: 84px;
  border-radius: 8px;
  border: 1.5px dashed $line;
  background: $paper;
  color: $ink-3;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 0.3em;
  font-size: 0.72em;
  font-weight: 600;
  cursor: pointer;
  transition: border-color 0.15s, color 0.15s, background 0.15s;
  &:hover {
    border-color: $signal;
    color: $signal;
    background: $wash;
  }
}

// ---- chip input -----------------------------------------------------------
.chip-input {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 0.4em;
  border: 1px solid $line;
  border-radius: 8px;
  background: $card;
  padding: 0.45em 0.5em;
  min-height: 2.6em;
  &:focus-within {
    border-color: $signal;
    box-shadow: 0 0 0 3px $wash;
  }
}
.chip {
  display: inline-flex;
  align-items: center;
  gap: 0.3em;
  font-size: 0.8em;
  padding: 0.25em 0.3em 0.25em 0.55em;
  border-radius: 6px;
  background: $ink-wash;
  color: $ink-2;
  button {
    border: 0;
    background: transparent;
    color: $ink-3;
    display: inline-flex;
    cursor: pointer;
    padding: 0.1em;
    border-radius: 4px;
    &:hover {
      color: $danger;
      background: $danger-wash;
    }
  }
}
.chip-field {
  flex: 1;
  min-width: 6em;
  border: 0;
  outline: 0;
  background: transparent;
  color: $ink;
  font-size: 0.9em;
  &::placeholder {
    color: $ink-3;
  }
}

// ---- footer -----------------------------------------------------------
.modal-foot {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 0.6em;
  padding: 1em 1.25em;
  border-top: 1px solid $line;
}
.modal-cancel {
  padding: 9px 16px;
  border: 1px solid $line;
  border-radius: 10px;
  background: $card;
  color: $ink-2;
  font-family: $sans;
  font-size: 0.9em;
  font-weight: 650;
  cursor: pointer;
  transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;
  &:hover:not(:disabled) { border-color: $ink-3; color: $ink; background: $paper; }
  &:disabled { opacity: 0.5; cursor: not-allowed; }
}
.modal-submit {
  display: inline-flex;
  align-items: center;
  gap: 7px;
  padding: 9px 18px;
  border: 1px solid transparent;
  border-radius: 10px;
  background: $signal;
  color: #fff;
  font-family: $sans;
  font-size: 0.9em;
  font-weight: 650;
  cursor: pointer;
  transition: background 0.15s ease;
  &:hover:not(:disabled) { background: $signal-2; }
  &:disabled { opacity: 0.6; cursor: not-allowed; }
}
.spin {
  animation: ticket-spin 0.8s linear infinite;
}

@media (max-width: 768px) {
  .modal-overlay { padding: 12px; }
  .modal { width: 100%; max-height: 94vh; border-radius: 14px; }
  .modal-body { flex-direction: column; overflow-y: auto; }
  .modal-rail { width: auto; border-left: 0; border-top: 1px solid $line; }
}

@media (prefers-reduced-motion: reduce) {
  .modal-overlay,
  .modal,
  .spin {
    animation: none;
  }
}

// ===========================================================================
// Custom select — replaces the native <select> with themed dropdown chrome
// (used for Priority / Assignee in the rail).
// ===========================================================================
.select {
  position: relative;
}
.select-trigger {
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.5em;
  border: 1px solid $line;
  border-radius: 8px;
  background: $card;
  color: $ink;
  font-size: 0.92em;
  font-family: inherit;
  padding: 0.62em 0.7em;
  cursor: pointer;
  transition: border-color 0.15s, box-shadow 0.15s, background 0.15s;

  &:hover:not(:disabled) {
    border-color: $ink-3;
  }
  &:disabled {
    opacity: 0.55;
    cursor: not-allowed;
  }
}
.select-trigger--open {
  border-color: $signal;
  box-shadow: 0 0 0 3px $wash;
}
.select-value {
  display: flex;
  align-items: center;
  gap: 0.5em;
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  font-weight: 500;
}
.select-placeholder {
  color: $ink-3;
  font-weight: 400;
}
.select-caret {
  flex-shrink: 0;
  color: $ink-3;
  transition: transform 0.15s;
  .select-trigger--open & {
    transform: rotate(180deg);
  }
}

.select-menu {
  position: absolute;
  top: calc(100% + 6px);
  left: 0;
  right: 0;
  z-index: 20;
  max-height: 15em;
  overflow-y: auto;
  background: $card;
  border: 1px solid $line;
  border-radius: 10px;
  box-shadow: 0 14px 32px -12px rgba(20, 22, 27, 0.35);
  padding: 0.35em;
  display: flex;
  flex-direction: column;
  gap: 0.1em;
  animation: ticket-modal-in 0.12s ease both;
}
.select-empty {
  padding: 0.6em 0.7em;
  font-size: 0.86em;
  color: $ink-3;
  text-align: center;
}
.select-option {
  display: flex;
  align-items: center;
  gap: 0.55em;
  width: 100%;
  border: 0;
  background: transparent;
  color: $ink;
  font-size: 0.9em;
  font-family: inherit;
  text-align: left;
  padding: 0.55em 0.6em;
  border-radius: 7px;
  cursor: pointer;
  transition: background 0.12s;

  &:hover {
    background: $paper;
  }
}
.select-option--active {
  background: $wash;
  font-weight: 600;
  color: $signal;
}
.select-option--highlight {
  background: $paper;
}
.select-option--active.select-option--highlight {
  background: $wash;
  box-shadow: inset 0 0 0 1px rgba(43, 43, 245, 0.25);
}
.select-option-label {
  flex: 1;
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.select-check {
  flex-shrink: 0;
  color: $signal;
}
.select-dot {
  flex-shrink: 0;
  width: 1.5em;
  height: 1.5em;
  border-radius: 50%;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  font-size: 0.62em;
  font-weight: 700;
  color: #fff;
  // used both as a plain priority-color dot (no glyph) and as a tiny
  // avatar-style badge (with initials as glyph) for the assignee list
}
