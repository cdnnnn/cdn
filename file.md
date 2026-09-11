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
import type { Ticket, TicketStatus, TicketResolution, TicketPriority } from '../../types/tickets';
import { useToast } from '../common/Toast';
import { COLUMNS, PRIORITY_META, canTransition, OWNER_ONLY_HINT, toTicketUser } from './ticketMeta';
import TicketCard from './TicketCard';
import TicketColumn from './TicketColumn';
import CreateTicketDrawer, { type TicketSubmitPayload } from './CreateTicketDrawer';
import TicketDetailSidebar from './TicketDetailSidebar';
import TicketCloseConfirm from './TicketCloseConfirm';
import styles from './TicketBoard.module.scss';

// ─────────────────────────────────────────────────────────────────────────
// Self-contained: no props. Fetches its own tickets and its own assignee
// roster. Current-user identity comes from the app's existing authSlice
// (state.auth.user, an SsoLoginResult) — this board never dispatches a
// login itself, it just reads whatever the app's SSO flow already put
// there and adapts it to the minimal { id, name } shape this feature needs
// via `toTicketUser` (see ticketMeta.ts — adjust its field-name guesses to
// match your real SsoLoginResult).
// ─────────────────────────────────────────────────────────────────────────

export default function TicketBoard() {
  const dispatch = useAppDispatch();
  const toast = useToast();
  const { items, status, creating, updatingId, movingIds, deletingId } = useAppSelector(
    (s) => s.tickets
  );
  const currentUser = useAppSelector((s) => toTicketUser(s.auth.user));
  const authSliceStatus = useAppSelector((s) => s.auth.status);
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
  // once this guard has passed. In practice this should resolve near-
  // instantly since the app's SSO flow almost certainly finishes before a
  // user can reach this route at all — this guard is just a safety net for
  // the brief window (or a genuinely logged-out edge case) where it hasn't.
  if (!currentUser) {
    return (
      <div className={`page-enter pg-shell ${styles['ticket-board']}`}>
        <div className={styles['ticket-board__loading']}>
          <Loader2 size={20} className={styles['ticket-board__spin']} />
          <span>
            {authSliceStatus === 'failed'
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












//Ticketmeta.ts
import type { TicketStatus, TicketPriority, Ticket, TicketUser } from '../../types/tickets';

// ─────────────────────────────────────────────────────────────────────────
// Adapter from the app's existing SsoLoginResult (state.auth.user, from
// authSlice.ts) to this feature's minimal TicketUser shape ({ id, name }).
//
// ⚠️ ADJUST THIS — I don't have SsoLoginResult's actual field names (it's
// defined in your own `../../types`), so this tries the common variants an
// SSO payload tends to use. Replace the fallback chains below with the real
// field names once you confirm them; everything downstream (the owner
// check, avatar initials, comment authorship) only ever touches `.id` and
// `.name`, so this one function is the only place that needs to change.
// ─────────────────────────────────────────────────────────────────────────
export function toTicketUser(sso: Record<string, unknown> | null | undefined): TicketUser | null {
  if (!sso) return null;
  const id = (sso.id ?? sso.userId ?? sso.user_id ?? sso.sub) as string | undefined;
  const name = (sso.name ?? sso.fullName ?? sso.full_name ?? sso.displayName ?? sso.email) as
    | string
    | undefined;
  if (!id || !name) return null;
  return { id, name };
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

/** Can `currentUserId` move `ticket` into `target`? */
export const canTransition = (
  ticket: Ticket,
  target: TicketStatus,
  currentUserId: string
): boolean => {
  if (target === 'done') return isOwner(ticket, currentUserId);
  return true;
};

export const OWNER_ONLY_HINT = 'Only the requester can close this ticket.';

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
