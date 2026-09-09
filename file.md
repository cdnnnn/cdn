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
import type { Ticket, TicketStatus, TicketResolution, TicketPriority } from '../../types/tickets';
import { useToast } from '../common/Toast';
import { COLUMNS, PRIORITY_META, canTransition, OWNER_ONLY_HINT } from './ticketMeta';
import TicketCard from './TicketCard';
import CreateTicketDrawer, { type TicketSubmitPayload } from './CreateTicketDrawer';
import TicketDetailSidebar from './TicketDetailSidebar';
import styles from './TicketBoard.module.scss';

// ─────────────────────────────────────────────────────────────────────────
// Self-contained: no props. Pulls the signed-in user from the auth slice
// (drives the owner-only "move to Done" gate) and the assignee roster from
// a `users` slice, the same way it already owns its own ticket-fetching.
//
// ASSUMPTION — adjust to match your real store shape if it differs:
//   • state.auth.currentUser : TicketUser   (id, name)
//   • state.users.items      : TicketUser[]
//   • fetchTeamMembers()     : async thunk that populates state.users.items
// If auth instead lives in a context/provider, swap the `useAppSelector`
// below for that context's hook — everything else is unaffected.
// ─────────────────────────────────────────────────────────────────────────

export default function TicketBoard() {
  const dispatch = useAppDispatch();
  const toast = useToast();
  const { items, status, creating, updatingId, movingIds, deletingId } = useAppSelector(
    (s) => s.tickets
  );
  const currentUser = useAppSelector((s) => s.auth.currentUser);
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

  // currentUser drives the owner-only "move to Done" gate everywhere below,
  // so don't render the board until auth has resolved.
  if (!currentUser) {
    return (
      <div className={styles['ticket-board']}>
        <div className={styles['ticket-board__loading']}>
          <Loader2 size={20} className={styles['ticket-board__spin']} />
          <span>Loading…</span>
        </div>
      </div>
    );
  }

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















//Usersslice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import api from '../../api/axiosInstance';
import type { TicketUser } from '../../types/tickets';

// ─────────────────────────────────────────────────────────────────────────
// Minimal roster slice so TicketBoard can be fully self-contained (no props).
// If the app already has a users/team/members slice, delete this file and
// point the two `fetchTeamMembers` / `state.users` references in
// TicketBoard.tsx at the existing one instead — this is a stand-in, not a
// second source of truth.
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

// GET /users — id + name for every assignable teammate
export const fetchTeamMembers = createAsyncThunk('users/fetchAll', () =>
  api.get<{ users: TicketUser[] }>('/users').then((r) => r.data.users ?? [])
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
