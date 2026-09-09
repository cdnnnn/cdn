//Mockdata.ts
import type { Ticket, TicketUser } from '../types/tickets';

// ─────────────────────────────────────────────────────────────────────────
// Static seed data. Swap `api/tickets.ts` and `api/users.ts` back to real
// axios calls once the backend exists — nothing outside this file and those
// two need to change; the slice, components, and types are unaffected.
// ─────────────────────────────────────────────────────────────────────────

export const TEAM: TicketUser[] = [
  { id: 'u1', name: 'Ava Patel' },
  { id: 'u2', name: 'Marcus Lee' },
  { id: 'u3', name: 'Sofia Nguyen' },
  { id: 'u4', name: 'Jordan Reyes' },
];

/** The signed-in user for this static build. Swap for your real auth source
 *  once it exists — see the note in `store/slices/authSlice.ts`. */
export const CURRENT_USER: TicketUser = TEAM[0];

let seq = 5;
export const nextId = () => `t${seq++}`;
export const nextKey = () => `REQ-${seq}`;

export const SEED_TICKETS: Ticket[] = [
  {
    id: 't1',
    key: 'REQ-1',
    title: 'Add dark-mode toggle to the settings page',
    description: 'Persist the choice per-user and respect prefers-color-scheme on first load.',
    status: 'todo',
    resolution: null,
    priority: 'medium',
    owner: TEAM[0],
    assignee: TEAM[1],
    labels: ['frontend', 'design-system'],
    created_at: '2026-08-20T09:00:00Z',
    updated_at: '2026-08-20T09:00:00Z',
  },
  {
    id: 't2',
    key: 'REQ-2',
    title: 'Custom model discovery times out on slow endpoints',
    description: 'Bump the discover request timeout and surface a retry affordance.',
    status: 'in_progress',
    resolution: null,
    priority: 'high',
    owner: TEAM[1],
    assignee: TEAM[0],
    labels: ['bug', 'models'],
    created_at: '2026-08-18T14:20:00Z',
    updated_at: '2026-09-01T11:00:00Z',
  },
  {
    id: 't3',
    key: 'REQ-3',
    title: 'Export provider usage as CSV',
    status: 'in_review',
    resolution: null,
    priority: 'low',
    owner: TEAM[2],
    assignee: TEAM[2],
    labels: ['reporting'],
    created_at: '2026-08-10T08:00:00Z',
    updated_at: '2026-09-05T16:40:00Z',
  },
  {
    id: 't4',
    key: 'REQ-4',
    title: 'Investigate flaky toast dismissal on Safari',
    description: 'Toasts occasionally fail to auto-dismiss after ~4s on Safari 17.',
    status: 'done',
    resolution: 'completed',
    priority: 'medium',
    owner: TEAM[0],
    assignee: TEAM[3],
    labels: ['bug'],
    created_at: '2026-07-30T10:00:00Z',
    updated_at: '2026-08-02T09:15:00Z',
  },
  {
    id: 't5',
    key: 'REQ-5',
    title: 'Explore moving verify-params to a background job',
    status: 'done',
    resolution: 'discarded',
    priority: 'low',
    owner: TEAM[3],
    assignee: null,
    labels: ['backend', 'exploration'],
    created_at: '2026-07-15T12:00:00Z',
    updated_at: '2026-07-22T17:30:00Z',
  },
];

























//Ticketsapi.mock.ts
import type {
  Ticket,
  CreateTicketRequest,
  UpdateTicketRequest,
  MoveTicketRequest,
} from '../types/tickets';
import { SEED_TICKETS, TEAM, CURRENT_USER, nextId, nextKey } from './mockData';

// ─────────────────────────────────────────────────────────────────────────
// Static, no-backend implementation of the tickets API. Same method names
// and signatures as the real `ticketsApi` (api/endpoints/tickets.ts) so the
// slice doesn't need to change — once the backend exists, delete this file,
// restore the axios-based `ticketsApi`, and repoint the slice's import.
//
// Simulates network latency (250ms) so loading/spinner states are visible,
// and enforces the owner-only "move to done" rule the same way the real
// backend should — reject with an Error, not a silent no-op, so the slice's
// rollback + toast path is exercised too.
// ─────────────────────────────────────────────────────────────────────────

const DELAY = 250;
const wait = <T,>(value: T, delay = DELAY) =>
  new Promise<T>((resolve) => window.setTimeout(() => resolve(value), delay));

let db: Ticket[] = SEED_TICKETS.map((t) => ({ ...t }));

const findOrThrow = (id: string) => {
  const t = db.find((x) => x.id === id);
  if (!t) throw new Error('Ticket not found');
  return t;
};

export interface DeleteTicketResponse {
  status: string;
  id: string;
}

export const ticketsApi = {
  list: () => wait(db.map((t) => ({ ...t }))),

  create: (payload: CreateTicketRequest) => {
    const now = new Date().toISOString();
    const ticket: Ticket = {
      id: nextId(),
      key: nextKey(),
      title: payload.title,
      description: payload.description,
      status: 'todo',
      resolution: null,
      priority: payload.priority,
      owner: CURRENT_USER, // the creator is always the requester/owner
      assignee: TEAM.find((m) => m.id === payload.assignee_id) ?? null,
      labels: payload.labels ?? [],
      created_at: now,
      updated_at: now,
    };
    db = [ticket, ...db];
    return wait({ ...ticket });
  },

  update: ({ id, ...rest }: UpdateTicketRequest) => {
    const t = findOrThrow(id);
    if (rest.title !== undefined) t.title = rest.title;
    if (rest.description !== undefined) t.description = rest.description;
    if (rest.priority !== undefined) t.priority = rest.priority;
    if (rest.labels !== undefined) t.labels = rest.labels;
    if (rest.assignee_id !== undefined) {
      t.assignee = TEAM.find((m) => m.id === rest.assignee_id) ?? null;
    }
    t.updated_at = new Date().toISOString();
    return wait({ ...t });
  },

  // Mirrors the server-side check the real endpoint MUST also perform:
  // only the ticket's owner may transition it into `done`.
  move: ({ id, status, resolution }: MoveTicketRequest) => {
    const t = findOrThrow(id);
    if (status === 'done' && t.owner.id !== CURRENT_USER.id) {
      return wait(undefined, 200).then(() => {
        throw new Error('Only the requester can close this ticket.');
      });
    }
    t.status = status;
    t.resolution = status === 'done' ? resolution ?? 'completed' : null;
    t.updated_at = new Date().toISOString();
    return wait({ ...t });
  },

  remove: (id: string) => {
    findOrThrow(id);
    db = db.filter((t) => t.id !== id);
    return wait<DeleteTicketResponse>({ status: 'ok', id });
  },
};

























//Ticketsslice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
// STATIC BUILD: pointed at the in-memory mock API (no backend yet). Swap this
// one import for '../../api/endpoints/tickets' once the real endpoints exist
// — nothing else in this file needs to change.
import { ticketsApi } from '../../mock/ticketsApi.mock';
import type {
  Ticket,
  TicketStatus,
  TicketResolution,
  CreateTicketRequest,
  UpdateTicketRequest,
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
      });
  },
});

export default ticketsSlice.reducer;


























//Usersslice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import type { TicketUser } from '../../types/tickets';
import { TEAM } from '../../mock/mockData';

// ─────────────────────────────────────────────────────────────────────────
// STATIC BUILD: returns the mock roster instead of hitting `/users`. Swap
// the thunk body for a real `api.get('/users')` call once the backend
// exists — the slice shape (state.users.items / .status) doesn't change,
// so nothing consuming this slice needs to change either.
//
// If the app already has a real users/team slice, delete this file and
// point TicketBoard.tsx's `fetchTeamMembers` import and `state.users`
// selector at that one instead.
// ─────────────────────────────────────────────────────────────────────────

type FetchStatus = 'idle' | 'loading' | 'succeeded' | 'failed';

interface UsersState {
  items: TicketUser[];
  status: FetchStatus;
}

const initialState: UsersState = {
  items: [],
  status: 'idle',
};

export const fetchTeamMembers = createAsyncThunk('users/fetchAll', () =>
  new Promise<TicketUser[]>((resolve) => window.setTimeout(() => resolve(TEAM), 150))
);

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
      .addCase(fetchTeamMembers.rejected, (state) => {
        state.status = 'failed';
      });
  },
});

export default usersSlice.reducer;























//Ticketboard.tsx
import { useEffect, useMemo, useState } from 'react';
import { Plus, Search, Lock, Loader2 } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import {
  fetchTickets,
  createTicket,
  updateTicket,
  moveTicket,
  deleteTicket,
} from '../../store/slices/ticketsSlice';
import { fetchTeamMembers } from '../../store/slices/usersSlice';
import { CURRENT_USER } from '../../mock/mockData';
import type { Ticket, TicketStatus, TicketResolution, TicketPriority } from '../../types/tickets';
import { useToast } from '../common/Toast';
import { COLUMNS, PRIORITY_META, canTransition, OWNER_ONLY_HINT } from './ticketMeta';
import TicketCard from './TicketCard';
import CreateTicketDrawer, { type TicketSubmitPayload } from './CreateTicketDrawer';
import TicketDetailSidebar from './TicketDetailSidebar';
import styles from './TicketBoard.module.scss';

// ─────────────────────────────────────────────────────────────────────────
// Self-contained: no props. Fetches its own tickets and its own assignee
// roster, same as before.
//
// STATIC BUILD (no backend/auth yet): `currentUser` comes straight from the
// mock seed data (`CURRENT_USER`) instead of an auth slice, so the owner-only
// gate below is exercised against a real, stable identity. Once real auth
// exists, replace the `const currentUser = CURRENT_USER;` line with your
// actual selector/hook (e.g. `useAppSelector((s) => s.auth.currentUser)`)
// — nothing else in this component needs to change.
// ─────────────────────────────────────────────────────────────────────────

export default function TicketBoard() {
  const dispatch = useAppDispatch();
  const toast = useToast();
  const { items, status, creating, updatingId, movingIds, deletingId } = useAppSelector(
    (s) => s.tickets
  );
  const currentUser = CURRENT_USER;
  const members = useAppSelector((s) => s.users.items);
  const membersStatus = useAppSelector((s) => s.users.status);

  const [drawerOpen, setDrawerOpen] = useState(false);
  const [editing, setEditing] = useState<Ticket | null>(null);
  const [detail, setDetail] = useState<Ticket | null>(null);

  const [query, setQuery] = useState('');
  const [priorityFilter, setPriorityFilter] = useState<TicketPriority | 'all'>('all');

  const [draggingId, setDraggingId] = useState<string | null>(null);
  const [dragOver, setDragOver] = useState<TicketStatus | null>(null);

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

  const draggingTicket = draggingId ? items.find((t) => t.id === draggingId) ?? null : null;

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

  const handleDrop = (to: TicketStatus) => {
    setDragOver(null);
    const id = draggingId;
    setDraggingId(null);
    if (!id) return;
    const ticket = items.find((t) => t.id === id);
    if (!ticket || ticket.status === to) return;
    // Dropping onto Done defaults to "completed"; use the card menu to discard.
    runMove(id, to, to === 'done' ? 'completed' : null);
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
    <div className={styles['ticket-board']}>
      <header className={styles['ticket-board__header']}>
        <div>
          <h1 className={styles['ticket-board__heading']}>Requirements</h1>
          <p className={styles['ticket-board__sub']}>
            {items.length} ticket{items.length === 1 ? '' : 's'} · drag between columns to
            change status
          </p>
        </div>
        <div className={styles['ticket-board__controls']}>
          <div className={styles['ticket-board__search']}>
            <Search size={15} />
            <input
              value={query}
              onChange={(e) => setQuery(e.target.value)}
              placeholder="Search key, title, label…"
              aria-label="Search tickets"
            />
          </div>
          <select
            className={styles['ticket-board__filter']}
            value={priorityFilter}
            onChange={(e) => setPriorityFilter(e.target.value as TicketPriority | 'all')}
            aria-label="Filter by priority"
          >
            <option value="all">All priorities</option>
            {(Object.keys(PRIORITY_META) as TicketPriority[]).map((p) => (
              <option key={p} value={p}>
                {PRIORITY_META[p].label}
              </option>
            ))}
          </select>
          <button
            className="btn btn-primary"
            onClick={() => {
              setEditing(null);
              setDrawerOpen(true);
            }}
          >
            <Plus size={16} />
            New Requirement
          </button>
        </div>
      </header>

      {status === 'loading' && items.length === 0 ? (
        <div className={styles['ticket-board__loading']}>
          <Loader2 size={20} className={styles['ticket-board__spin']} />
          <span>Loading board…</span>
        </div>
      ) : (
        <div className={styles['ticket-board__columns']}>
          {COLUMNS.map((col) => {
            const list = byStatus[col.status];
            // Would the currently-dragged card be rejected if dropped here?
            const locked =
              !!draggingTicket &&
              dragOver === col.status &&
              !canTransition(draggingTicket, col.status, currentUser.id);
            const active = dragOver === col.status;

            return (
              <section
                key={col.status}
                className={[
                  styles['ticket-board__column'],
                  active ? styles['ticket-board__column--over'] : '',
                  locked ? styles['ticket-board__column--locked'] : '',
                ].join(' ')}
                onDragOver={(e) => {
                  if (!draggingId) return;
                  e.preventDefault();
                  e.dataTransfer.dropEffect = locked ? 'none' : 'move';
                  if (dragOver !== col.status) setDragOver(col.status);
                }}
                onDragLeave={(e) => {
                  // Ignore leaves that bubble from children.
                  if (!e.currentTarget.contains(e.relatedTarget as Node)) {
                    setDragOver((cur) => (cur === col.status ? null : cur));
                  }
                }}
                onDrop={(e) => {
                  e.preventDefault();
                  if (locked) {
                    setDragOver(null);
                    setDraggingId(null);
                    toast.warning(OWNER_ONLY_HINT);
                    return;
                  }
                  handleDrop(col.status);
                }}
              >
                <div className={styles['ticket-board__column-head']}>
                  <span
                    className={styles['ticket-board__column-dot']}
                    style={{ background: col.accent }}
                  />
                  <span className={styles['ticket-board__column-title']}>{col.label}</span>
                  <span className={styles['ticket-board__column-count']}>{list.length}</span>
                  {col.status === 'done' && (
                    <Lock
                      size={12}
                      className={styles['ticket-board__column-lock']}
                      aria-label="Owner-only column"
                    />
                  )}
                </div>

                <div className={styles['ticket-board__column-body']}>
                  {list.map((t) => (
                    <TicketCard
                      key={t.id}
                      ticket={t}
                      currentUser={currentUser}
                      moving={movingIds.includes(t.id)}
                      onOpen={setDetail}
                      onMove={runMove}
                      onDragStart={setDraggingId}
                      onDragEnd={() => {
                        setDraggingId(null);
                        setDragOver(null);
                      }}
                    />
                  ))}

                  {list.length === 0 && (
                    <div className={styles['ticket-board__column-empty']}>
                      {locked ? OWNER_ONLY_HINT : 'Nothing here'}
                    </div>
                  )}
                </div>
              </section>
            );
          })}
        </div>
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
          onMove={runMove}
          onEdit={(t) => {
            setDetail(null);
            setEditing(t);
            setDrawerOpen(true);
          }}
          onDelete={handleDelete}
        />
      )}
    </div>
  );
}
























//Tickets.ts
// ─────────────────────────────────────────────────────────────────────────
// The real, backend-backed implementation. NOT currently wired up — the
// slice imports `mock/ticketsApi.mock.ts` instead until the backend exists.
// Kept here so reconnecting later is a one-line import swap in
// `store/slices/ticketsSlice.ts` (see README "Reconnecting the real backend").
// ─────────────────────────────────────────────────────────────────────────
import api from '../axiosInstance';
import type {
  Ticket,
  CreateTicketRequest,
  UpdateTicketRequest,
  MoveTicketRequest,
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
};
