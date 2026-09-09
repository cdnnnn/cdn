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
import type {
  Ticket,
  TicketStatus,
  TicketResolution,
  TicketPriority,
  TicketUser,
} from '../../types/tickets';
import { useToast } from '../common/Toast';
import { COLUMNS, PRIORITY_META, canTransition, OWNER_ONLY_HINT } from './ticketMeta';
import TicketCard from './TicketCard';
import CreateTicketDrawer, { type TicketSubmitPayload } from './CreateTicketDrawer';
import TicketDetailSidebar from './TicketDetailSidebar';
import styles from './TicketBoard.module.scss';

interface TicketBoardProps {
  /** The signed-in user. Drives the owner-only "move to Done" gate. */
  currentUser: TicketUser;
  /** Optional roster used to populate the assignee picker in the drawer. */
  members?: TicketUser[];
}

export default function TicketBoard({ currentUser, members = [] }: TicketBoardProps) {
  const dispatch = useAppDispatch();
  const toast = useToast();
  const { items, status, creating, updatingId, movingIds, deletingId } = useAppSelector(
    (s) => s.tickets
  );

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


























//Ticketboard.module.scss
@use '../../styles/_variables' as *;

// ===========================================================================
// Ticket board — same ink/paper design system as Providers/Dashboard:
// theme-aware neutrals from _variables, flat accent constants, hover-lift
// cards, mono-ish instrument labels. Font scaling follows the shared
// convention: `.ticket-board` sets one base font-size and everything below
// is expressed in `em`.
// ===========================================================================

$mono: $font-mono;
$radius: 12px;

.ticket-board {
  font-size: 13px;
  display: flex;
  flex-direction: column;
  gap: 1.25em;
  padding: 1.5em 1.75em;
  color: $ink;
  min-height: 0;
  flex: 1;
}

// ---- header ---------------------------------------------------------------
.ticket-board__header {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 1em;
  flex-wrap: wrap;
}
.ticket-board__heading {
  font-size: 1.5em;
  font-weight: 700;
  letter-spacing: -0.01em;
  margin: 0;
}
.ticket-board__sub {
  margin: 0.25em 0 0;
  color: $ink-3;
  font-size: 0.9em;
}
.ticket-board__controls {
  display: flex;
  align-items: center;
  gap: 0.6em;
  flex-wrap: wrap;
}
.ticket-board__search {
  display: flex;
  align-items: center;
  gap: 0.45em;
  padding: 0 0.7em;
  height: 2.6em;
  background: $card;
  border: 1px solid $line;
  border-radius: 8px;
  color: $ink-3;
  transition: border-color 0.15s;
  &:focus-within {
    border-color: $signal;
  }
  input {
    border: 0;
    outline: 0;
    background: transparent;
    color: $ink;
    font-size: 0.9em;
    width: 12em;
    &::placeholder {
      color: $ink-3;
    }
  }
}
.ticket-board__filter {
  height: 2.6em;
  padding: 0 0.7em;
  background: $card;
  border: 1px solid $line;
  border-radius: 8px;
  color: $ink;
  font-size: 0.9em;
  cursor: pointer;
}

// ---- columns --------------------------------------------------------------
.ticket-board__columns {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: 1em;
  align-items: start;
  flex: 1;
  min-height: 0;
}
.ticket-board__column {
  background: $paper;
  border: 1px solid $line-2;
  border-radius: $radius;
  padding: 0.75em;
  display: flex;
  flex-direction: column;
  gap: 0.6em;
  min-height: 8em;
  transition: background 0.15s, border-color 0.15s, box-shadow 0.15s;
}
.ticket-board__column--over {
  border-color: $signal;
  border-style: dashed;
  background: $wash;
  box-shadow: inset 0 0 0 1px $signal;
}
.ticket-board__column--locked {
  border-color: $danger;
  background: $danger-wash;
  box-shadow: inset 0 0 0 1px $danger;
  cursor: not-allowed;
}
.ticket-board__column-head {
  display: flex;
  align-items: center;
  gap: 0.5em;
  padding: 0.1em 0.25em;
}
.ticket-board__column-dot {
  width: 0.6em;
  height: 0.6em;
  border-radius: 50%;
  flex: none;
}
.ticket-board__column-title {
  font-weight: 600;
  font-size: 0.92em;
  letter-spacing: 0.01em;
}
.ticket-board__column-count {
  margin-left: auto;
  min-width: 1.6em;
  height: 1.6em;
  padding: 0 0.4em;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border-radius: 6px;
  background: $ink-wash;
  color: $ink-2;
  font-size: 0.78em;
  font-weight: 600;
  font-family: $mono;
}
.ticket-board__column-lock {
  color: $ink-3;
}
.ticket-board__column-body {
  display: flex;
  flex-direction: column;
  gap: 0.6em;
  min-height: 2em;
}
.ticket-board__column-empty {
  padding: 1.5em 0.5em;
  text-align: center;
  color: $ink-3;
  font-size: 0.82em;
  border: 1px dashed $line;
  border-radius: 8px;
}

// ---- card -----------------------------------------------------------------
.ticket-card {
  --priority-accent: #{$ink-3};
  position: relative;
  background: $card;
  border: 1px solid $line;
  border-left: 3px solid var(--priority-accent);
  border-radius: 10px;
  padding: 0.75em 0.8em;
  display: flex;
  flex-direction: column;
  gap: 0.55em;
  cursor: grab;
  box-shadow: $shadow-2;
  transition: transform 0.12s, box-shadow 0.12s, border-color 0.12s;
  &:hover {
    transform: translateY(-1px);
    box-shadow: $shadow-3;
  }
  &:active {
    cursor: grabbing;
  }
}
.ticket-card--moving {
  opacity: 0.6;
  cursor: default;
}
.ticket-card--discarded {
  opacity: 0.72;
  .ticket-card__title {
    text-decoration: line-through;
    color: $ink-2;
  }
}
.ticket-card__top {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.5em;
}
.ticket-card__key {
  font-family: $mono;
  font-size: 0.72em;
  font-weight: 600;
  letter-spacing: 0.04em;
  color: $ink-3;
}
.ticket-card__top-right {
  display: flex;
  align-items: center;
  gap: 0.4em;
}
.ticket-card__priority {
  --priority-accent: #{$ink-3};
  font-size: 0.68em;
  font-weight: 700;
  letter-spacing: 0.03em;
  text-transform: uppercase;
  padding: 0.25em 0.5em;
  border-radius: 5px;
  color: var(--priority-accent);
  background: color-mix(in srgb, var(--priority-accent) 12%, transparent);
}
.ticket-card__spin {
  animation: spin 1.5s linear infinite;
  color: $signal;
}
.ticket-card__title {
  margin: 0;
  font-size: 0.94em;
  font-weight: 600;
  line-height: 1.35;
  color: $ink;
}
.ticket-card__labels {
  display: flex;
  flex-wrap: wrap;
  gap: 0.35em;
}
.ticket-card__label {
  font-size: 0.7em;
  padding: 0.2em 0.5em;
  border-radius: 5px;
  background: $ink-wash;
  color: $ink-2;
}
.ticket-card__foot {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.5em;
  margin-top: 0.1em;
}
.ticket-card__resolution {
  font-size: 0.68em;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.03em;
  padding: 0.2em 0.5em;
  border-radius: 5px;
}
.ticket-card__resolution--done {
  color: $ok;
  background: $ok-wash;
}
.ticket-card__resolution--discarded {
  color: $rose-ink;
  background: $rose-ink-wash;
}
.ticket-card__avatars {
  display: flex;
  align-items: center;
  margin-left: auto;
}
.ticket-card__avatar {
  width: 1.7em;
  height: 1.7em;
  border-radius: 50%;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  font-size: 0.62em;
  font-weight: 700;
  color: #fff;
  border: 2px solid $card;
  & + & {
    margin-left: -0.5em;
  }
}
.ticket-card__avatar--owner {
  box-shadow: 0 0 0 1px $line;
}

// ---- card menu ------------------------------------------------------------
.ticket-card__menu-wrap {
  position: relative;
}
.ticket-card__menu-btn {
  border: 0;
  background: transparent;
  color: $ink-3;
  padding: 0.15em;
  border-radius: 5px;
  cursor: pointer;
  display: inline-flex;
  &:hover {
    background: $ink-wash;
    color: $ink;
  }
}
.ticket-card__menu {
  position: absolute;
  right: 0;
  top: 1.7em;
  z-index: 20;
  min-width: 12em;
  background: $card;
  border: 1px solid $line;
  border-radius: 9px;
  box-shadow: $shadow-3;
  padding: 0.35em;
  display: flex;
  flex-direction: column;
  gap: 0.1em;
  animation: drawerIn 0.12s ease both;
}
.ticket-card__menu-label {
  font-size: 0.66em;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  color: $ink-3;
  padding: 0.35em 0.55em 0.15em;
}
.ticket-card__menu-item {
  display: flex;
  align-items: center;
  gap: 0.5em;
  width: 100%;
  border: 0;
  background: transparent;
  color: $ink;
  font-size: 0.82em;
  text-align: left;
  padding: 0.5em 0.55em;
  border-radius: 6px;
  cursor: pointer;
  &:hover:not(:disabled) {
    background: $wash;
  }
  &:disabled {
    color: $ink-3;
    cursor: not-allowed;
  }
}
.ticket-card__menu-item--danger:not(:disabled) {
  color: $danger;
  &:hover {
    background: $danger-wash;
  }
}
.ticket-card__menu-dot {
  width: 0.55em;
  height: 0.55em;
  border-radius: 50%;
  flex: none;
}
.ticket-card__menu-lead {
  flex: none;
}
.ticket-card__menu-check {
  margin-left: auto;
  color: $signal;
}
.ticket-card__menu-sep {
  height: 1px;
  background: $line-2;
  margin: 0.2em 0.3em;
}

// ---- loading --------------------------------------------------------------
.ticket-board__loading {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.6em;
  padding: 4em;
  color: $ink-3;
}
.ticket-board__spin {
  animation: spin 1.5s linear infinite;
  color: $signal;
}

// ===========================================================================
// Detail sidebar (shares this module, like ProviderModelsSidebar shares
// Providers.module.scss)
// ===========================================================================
.ticket-detail__overlay {
  position: fixed;
  inset: 0;
  bottom: $footer-height;
  background: rgba(17, 24, 39, 0.4);
  z-index: 100;
}
.ticket-detail {
  position: fixed;
  top: 0;
  right: 0;
  bottom: $footer-height;
  width: 400px;
  max-width: 100%;
  background: $surface;
  box-shadow: $shadow-4;
  z-index: 101;
  display: flex;
  flex-direction: column;
  font-size: 13px;
  animation: drawerIn 0.25s ease both;
}
.ticket-detail__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.5em;
  padding: 1em 1.1em;
  border-bottom: 1px solid $line;
}
.ticket-detail__key {
  font-family: $mono;
  font-size: 0.8em;
  font-weight: 700;
  letter-spacing: 0.04em;
  color: $ink-3;
  margin-right: 0.6em;
}
.ticket-detail__header-actions {
  display: flex;
  align-items: center;
  gap: 0.2em;
}
.ticket-detail__body {
  padding: 1.1em;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 1em;
}
.ticket-detail__title {
  margin: 0;
  font-size: 1.15em;
  font-weight: 700;
  line-height: 1.35;
}
.ticket-detail__desc {
  margin: 0;
  color: $ink-2;
  font-size: 0.9em;
  line-height: 1.55;
  white-space: pre-wrap;
}
.ticket-detail__desc--empty {
  margin: 0;
  color: $ink-3;
  font-size: 0.88em;
  font-style: italic;
}
.ticket-detail__people {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 0.75em;
  padding: 0.85em;
  background: $paper;
  border-radius: 10px;
}
.ticket-detail__person {
  display: flex;
  flex-direction: column;
  gap: 0.35em;
}
.ticket-detail__person-label {
  font-size: 0.68em;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: $ink-3;
}
.ticket-detail__person-val {
  display: flex;
  align-items: center;
  gap: 0.45em;
  font-size: 0.88em;
  font-weight: 500;
}
.ticket-detail__you {
  font-size: 0.72em;
  font-weight: 700;
  color: $signal;
  background: $wash;
  padding: 0.1em 0.4em;
  border-radius: 4px;
}
.ticket-detail__muted {
  color: $ink-3;
}
.ticket-detail__section-label {
  display: flex;
  align-items: center;
  gap: 0.4em;
  font-size: 0.72em;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: $ink-3;
  margin-top: 0.25em;
}
.ticket-detail__stepper {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 0.4em;
}
.ticket-detail__step {
  --step-accent: #{$signal};
  border: 1px solid $line;
  background: $card;
  color: $ink-2;
  font-size: 0.8em;
  font-weight: 600;
  padding: 0.6em 0.4em;
  border-radius: 8px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.3em;
  transition: border-color 0.12s, background 0.12s, color 0.12s;
  &:hover:not(:disabled) {
    border-color: var(--step-accent);
    color: $ink;
  }
  &:disabled {
    cursor: default;
  }
}
.ticket-detail__step--current {
  border-color: var(--step-accent);
  background: color-mix(in srgb, var(--step-accent) 12%, transparent);
  color: var(--step-accent);
}
.ticket-detail__terminal {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 0.5em;
}
.ticket-detail__done-btn,
.ticket-detail__discard-btn {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.4em;
  padding: 0.65em;
  border-radius: 8px;
  font-size: 0.85em;
  font-weight: 600;
  cursor: pointer;
  border: 1px solid transparent;
  transition: filter 0.12s, opacity 0.12s;
  &:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
  &:not(:disabled):hover {
    filter: brightness(0.96);
  }
}
.ticket-detail__done-btn {
  background: $ok;
  color: #fff;
}
.ticket-detail__discard-btn {
  background: $card;
  border-color: $danger;
  color: $danger;
}
.ticket-detail__gate-note {
  display: flex;
  align-items: center;
  gap: 0.4em;
  margin: 0;
  font-size: 0.78em;
  color: $ink-3;
}





















//Ticketcard.tsx
import { useEffect, useRef, useState } from 'react';
import { MoreHorizontal, Loader2, Lock, Check, Ban } from 'lucide-react';
import type { Ticket, TicketStatus, TicketResolution, TicketUser } from '../../types/tickets';
import {
  COLUMNS,
  PRIORITY_META,
  isOwner,
  OWNER_ONLY_HINT,
  initials,
  avatarAccent,
} from './ticketMeta';
import styles from './TicketBoard.module.scss';

interface TicketCardProps {
  ticket: Ticket;
  currentUser: TicketUser;
  moving?: boolean;
  onOpen: (ticket: Ticket) => void;
  onMove: (id: string, status: TicketStatus, resolution?: TicketResolution | null) => void;
  onDragStart: (id: string) => void;
  onDragEnd: () => void;
}

export default function TicketCard({
  ticket,
  currentUser,
  moving = false,
  onOpen,
  onMove,
  onDragStart,
  onDragEnd,
}: TicketCardProps) {
  const [menuOpen, setMenuOpen] = useState(false);
  const menuRef = useRef<HTMLDivElement | null>(null);
  const owner = isOwner(ticket, currentUser.id);
  const priority = PRIORITY_META[ticket.priority];

  useEffect(() => {
    if (!menuOpen) return;
    const onDoc = (e: MouseEvent) => {
      if (menuRef.current && !menuRef.current.contains(e.target as Node)) setMenuOpen(false);
    };
    document.addEventListener('mousedown', onDoc);
    return () => document.removeEventListener('mousedown', onDoc);
  }, [menuOpen]);

  const move = (status: TicketStatus, resolution?: TicketResolution | null) => {
    setMenuOpen(false);
    if (status === ticket.status && !resolution) return;
    onMove(ticket.id, status, resolution);
  };

  return (
    <article
      className={[
        styles['ticket-card'],
        moving ? styles['ticket-card--moving'] : '',
        ticket.resolution === 'discarded' ? styles['ticket-card--discarded'] : '',
      ].join(' ')}
      style={{ ['--priority-accent' as string]: priority.accent }}
      draggable={!moving}
      onDragStart={(e) => {
        e.dataTransfer.setData('text/plain', ticket.id);
        e.dataTransfer.effectAllowed = 'move';
        onDragStart(ticket.id);
      }}
      onDragEnd={onDragEnd}
      onClick={() => onOpen(ticket)}
    >
      <header className={styles['ticket-card__top']}>
        <span className={styles['ticket-card__key']}>{ticket.key}</span>
        <div className={styles['ticket-card__top-right']}>
          <span
            className={styles['ticket-card__priority']}
            style={{ ['--priority-accent' as string]: priority.accent }}
          >
            {priority.label}
          </span>
          {moving ? (
            <Loader2 size={14} className={styles['ticket-card__spin']} />
          ) : (
            <div className={styles['ticket-card__menu-wrap']} ref={menuRef}>
              <button
                type="button"
                className={styles['ticket-card__menu-btn']}
                aria-label="Move ticket"
                onClick={(e) => {
                  e.stopPropagation();
                  setMenuOpen((v) => !v);
                }}
              >
                <MoreHorizontal size={15} />
              </button>
              {menuOpen && (
                <div
                  className={styles['ticket-card__menu']}
                  role="menu"
                  onClick={(e) => e.stopPropagation()}
                >
                  <div className={styles['ticket-card__menu-label']}>Move to</div>
                  {COLUMNS.filter((c) => c.status !== 'done').map((c) => (
                    <button
                      key={c.status}
                      type="button"
                      role="menuitem"
                      className={styles['ticket-card__menu-item']}
                      disabled={c.status === ticket.status}
                      onClick={() => move(c.status)}
                    >
                      <span
                        className={styles['ticket-card__menu-dot']}
                        style={{ background: c.accent }}
                      />
                      {c.label}
                      {c.status === ticket.status && (
                        <Check size={13} className={styles['ticket-card__menu-check']} />
                      )}
                    </button>
                  ))}
                  <div className={styles['ticket-card__menu-sep']} />
                  <button
                    type="button"
                    role="menuitem"
                    className={styles['ticket-card__menu-item']}
                    disabled={!owner}
                    title={owner ? undefined : OWNER_ONLY_HINT}
                    onClick={() => owner && move('done', 'completed')}
                  >
                    {owner ? (
                      <Check size={13} className={styles['ticket-card__menu-lead']} />
                    ) : (
                      <Lock size={13} className={styles['ticket-card__menu-lead']} />
                    )}
                    Mark Done
                  </button>
                  <button
                    type="button"
                    role="menuitem"
                    className={`${styles['ticket-card__menu-item']} ${styles['ticket-card__menu-item--danger']}`}
                    disabled={!owner}
                    title={owner ? undefined : OWNER_ONLY_HINT}
                    onClick={() => owner && move('done', 'discarded')}
                  >
                    {owner ? (
                      <Ban size={13} className={styles['ticket-card__menu-lead']} />
                    ) : (
                      <Lock size={13} className={styles['ticket-card__menu-lead']} />
                    )}
                    Discard
                  </button>
                </div>
              )}
            </div>
          )}
        </div>
      </header>

      <h4 className={styles['ticket-card__title']}>{ticket.title}</h4>

      {(ticket.labels ?? []).length > 0 && (
        <div className={styles['ticket-card__labels']}>
          {(ticket.labels ?? []).slice(0, 4).map((l) => (
            <span key={l} className={styles['ticket-card__label']}>
              {l}
            </span>
          ))}
        </div>
      )}

      <footer className={styles['ticket-card__foot']}>
        {ticket.resolution && (
          <span
            className={[
              styles['ticket-card__resolution'],
              ticket.resolution === 'discarded'
                ? styles['ticket-card__resolution--discarded']
                : styles['ticket-card__resolution--done'],
            ].join(' ')}
          >
            {ticket.resolution === 'discarded' ? 'Discarded' : 'Completed'}
          </span>
        )}
        <div className={styles['ticket-card__avatars']}>
          {ticket.assignee && (
            <span
              className={styles['ticket-card__avatar']}
              style={{ background: avatarAccent(ticket.assignee) }}
              title={`Assignee: ${ticket.assignee.name}`}
            >
              {initials(ticket.assignee)}
            </span>
          )}
          <span
            className={`${styles['ticket-card__avatar']} ${styles['ticket-card__avatar--owner']}`}
            style={{ background: avatarAccent(ticket.owner) }}
            title={`Requester: ${ticket.owner.name}${owner ? ' (you)' : ''}`}
          >
            {initials(ticket.owner)}
          </span>
        </div>
      </footer>
    </article>
  );
}

























//Createticketdrawer.tsx
import { useEffect, useMemo, useRef, useState } from 'react';
import { X, Loader2, Plus } from 'lucide-react';
import type { Ticket, TicketPriority, TicketUser } from '../../types/tickets';
import { PRIORITY_META } from './ticketMeta';
import styles from './CreateTicketDrawer.module.scss';

/** What the drawer hands back on submit — no id/status, the board adds those. */
export interface TicketSubmitPayload {
  title: string;
  description?: string;
  priority: TicketPriority;
  labels?: string[];
  assignee_id?: string | null;
}

interface CreateTicketDrawerProps {
  mode: 'create' | 'edit';
  initialTicket?: Ticket;
  members?: TicketUser[];
  submitting?: boolean;
  onClose: () => void;
  /** Return a Promise to keep the drawer open on failure and close it on success. */
  onSubmit: (payload: TicketSubmitPayload) => Promise<unknown> | void;
}

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
  const [touched, setTouched] = useState(false);

  const firstFieldRef = useRef<HTMLInputElement | null>(null);
  useEffect(() => {
    firstFieldRef.current?.focus();
  }, []);

  // Close on Escape, matching the other overlays.
  useEffect(() => {
    const onKey = (e: KeyboardEvent) => {
      if (e.key === 'Escape' && !submitting) onClose();
    };
    document.addEventListener('keydown', onKey);
    return () => document.removeEventListener('keydown', onKey);
  }, [onClose, submitting]);

  const titleError = touched && !title.trim() ? 'Title is required' : '';
  const valid = title.trim().length > 0;

  const addLabel = () => {
    const v = labelDraft.trim();
    if (!v) return;
    if (!labels.includes(v)) setLabels((prev) => [...prev, v]);
    setLabelDraft('');
  };

  const submit = () => {
    setTouched(true);
    if (!valid) return;
    onSubmit({
      title: title.trim(),
      description: description.trim() || undefined,
      priority,
      labels: labels.length ? labels : undefined,
      assignee_id: assigneeId || null,
    });
  };

  const heading = mode === 'edit' ? 'Edit requirement' : 'New requirement';
  const cta = mode === 'edit' ? 'Save changes' : 'Create requirement';

  const priorities = useMemo(() => Object.keys(PRIORITY_META) as TicketPriority[], []);

  return (
    <>
      <div className={styles['drawer__overlay']} onClick={() => !submitting && onClose()} />
      <aside className={styles['drawer']} role="dialog" aria-modal="true" aria-label={heading}>
        <header className={styles['drawer__header']}>
          <div className={styles['drawer__title']}>{heading}</div>
          <button
            className="btn btn-sm btn-ghost"
            onClick={onClose}
            disabled={submitting}
            aria-label="Close"
          >
            <X size={16} />
          </button>
        </header>

        <div className={styles['drawer__body']}>
          <label className={styles['drawer__field']}>
            <span className={styles['drawer__label']}>
              Title <span className={styles['drawer__req']}>*</span>
            </span>
            <input
              ref={firstFieldRef}
              className={`${styles['drawer__input']} ${titleError ? styles['drawer__input--error'] : ''}`}
              value={title}
              onChange={(e) => setTitle(e.target.value)}
              onBlur={() => setTouched(true)}
              placeholder="Short summary of the requirement"
            />
            {titleError && <span className={styles['drawer__error']}>{titleError}</span>}
          </label>

          <label className={styles['drawer__field']}>
            <span className={styles['drawer__label']}>Description</span>
            <textarea
              className={styles['drawer__textarea']}
              value={description}
              onChange={(e) => setDescription(e.target.value)}
              rows={4}
              placeholder="Context, acceptance criteria, links…"
            />
          </label>

          <div className={styles['drawer__row']}>
            <label className={styles['drawer__field']}>
              <span className={styles['drawer__label']}>Priority</span>
              <select
                className={styles['drawer__input']}
                value={priority}
                onChange={(e) => setPriority(e.target.value as TicketPriority)}
              >
                {priorities.map((p) => (
                  <option key={p} value={p}>
                    {PRIORITY_META[p].label}
                  </option>
                ))}
              </select>
            </label>

            <label className={styles['drawer__field']}>
              <span className={styles['drawer__label']}>Assignee</span>
              <select
                className={styles['drawer__input']}
                value={assigneeId}
                onChange={(e) => setAssigneeId(e.target.value)}
                disabled={members.length === 0}
              >
                <option value="">Unassigned</option>
                {members.map((m) => (
                  <option key={m.id} value={m.id}>
                    {m.name}
                  </option>
                ))}
              </select>
            </label>
          </div>

          <div className={styles['drawer__field']}>
            <span className={styles['drawer__label']}>Labels</span>
            <div className={styles['drawer__chip-input']}>
              {labels.map((l) => (
                <span key={l} className={styles['drawer__chip']}>
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
                className={styles['drawer__chip-field']}
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
                placeholder={labels.length ? 'Add another…' : 'Type a label, press Enter'}
              />
              {labelDraft.trim() && (
                <button
                  type="button"
                  className={styles['drawer__chip-add']}
                  onClick={addLabel}
                  aria-label="Add label"
                >
                  <Plus size={13} />
                </button>
              )}
            </div>
          </div>
        </div>

        <footer className={styles['drawer__footer']}>
          <button className="btn btn-ghost" onClick={onClose} disabled={submitting}>
            Cancel
          </button>
          <button className="btn btn-primary" onClick={submit} disabled={submitting || !valid}>
            {submitting ? (
              <>
                <Loader2 size={15} className={styles['drawer__spin']} />
                Saving…
              </>
            ) : (
              cta
            )}
          </button>
        </footer>
      </aside>
    </>
  );
}




























//Createticketdrawer.module.scss
@use '../../styles/_variables' as *;

// Right-anchored drawer, same shell + animation as AddCustomModelDrawer.
// `.drawer` sets the base font-size; descendants scale in `em`.

$mono: $font-mono;

.drawer__overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: $footer-height;
  background: rgba(17, 24, 39, 0.4);
  z-index: 100;
  display: flex;
  justify-content: flex-end;
}
.drawer {
  position: fixed;
  top: 0;
  right: 0;
  bottom: $footer-height;
  width: 440px;
  max-width: 100%;
  background: $surface;
  box-shadow: $shadow-4;
  z-index: 101;
  display: flex;
  flex-direction: column;
  font-size: 13px;
  animation: drawerIn 0.25s ease both;
}

.drawer__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1.1em 1.25em;
  border-bottom: 1px solid $line;
}
.drawer__title {
  font-size: 1.15em;
  font-weight: 700;
  color: $ink;
}

.drawer__body {
  flex: 1;
  overflow-y: auto;
  padding: 1.25em;
  display: flex;
  flex-direction: column;
  gap: 1.1em;
}
.drawer__row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 0.9em;
}
.drawer__field {
  display: flex;
  flex-direction: column;
  gap: 0.4em;
}
.drawer__label {
  font-size: 0.78em;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.04em;
  color: $ink-2;
}
.drawer__req {
  color: $danger;
}
.drawer__input,
.drawer__textarea {
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
.drawer__textarea {
  resize: vertical;
  line-height: 1.5;
}
.drawer__input--error {
  border-color: $danger;
  &:focus {
    box-shadow: 0 0 0 3px $danger-wash;
  }
}
.drawer__error {
  font-size: 0.78em;
  color: $danger;
}

// ---- chip input -----------------------------------------------------------
.drawer__chip-input {
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
.drawer__chip {
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
.drawer__chip-field {
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
.drawer__chip-add {
  border: 0;
  background: $wash;
  color: $signal;
  display: inline-flex;
  padding: 0.25em;
  border-radius: 6px;
  cursor: pointer;
}

.drawer__footer {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 0.6em;
  padding: 1em 1.25em;
  border-top: 1px solid $line;
}
.drawer__spin {
  animation: spin 1.5s linear infinite;
}



























//Ticketdetailsidebar.tsx
import { useState } from 'react';
import { X, Pencil, Trash2, Loader2, Lock, Check, Ban } from 'lucide-react';
import type { Ticket, TicketStatus, TicketResolution, TicketUser } from '../../types/tickets';
import ConfirmDialog from '../common/ConfirmDialog';
import {
  COLUMNS,
  PRIORITY_META,
  isOwner,
  OWNER_ONLY_HINT,
  initials,
  avatarAccent,
} from './ticketMeta';
import styles from './TicketBoard.module.scss';

interface TicketDetailSidebarProps {
  ticket: Ticket;
  currentUser: TicketUser;
  moving?: boolean;
  deleting?: boolean;
  onClose: () => void;
  onMove: (id: string, status: TicketStatus, resolution?: TicketResolution | null) => void;
  onEdit: (ticket: Ticket) => void;
  onDelete: (id: string) => void;
}

export default function TicketDetailSidebar({
  ticket,
  currentUser,
  moving = false,
  deleting = false,
  onClose,
  onMove,
  onEdit,
  onDelete,
}: TicketDetailSidebarProps) {
  const [confirmDelete, setConfirmDelete] = useState(false);
  const owner = isOwner(ticket, currentUser.id);
  const priority = PRIORITY_META[ticket.priority];

  return (
    <>
      <div className={styles['ticket-detail__overlay']} onClick={onClose} />
      <aside className={styles['ticket-detail']} role="dialog" aria-label="Ticket detail">
        <header className={styles['ticket-detail__header']}>
          <div>
            <span className={styles['ticket-detail__key']}>{ticket.key}</span>
            <span
              className={styles['ticket-card__priority']}
              style={{ ['--priority-accent' as string]: priority.accent }}
            >
              {priority.label}
            </span>
          </div>
          <div className={styles['ticket-detail__header-actions']}>
            <button
              type="button"
              className="btn btn-sm btn-ghost"
              onClick={() => onEdit(ticket)}
              aria-label="Edit ticket"
              title="Edit"
            >
              <Pencil size={15} />
            </button>
            <button
              type="button"
              className="btn btn-sm btn-ghost"
              onClick={() => setConfirmDelete(true)}
              disabled={deleting}
              aria-label="Delete ticket"
              title="Delete"
            >
              {deleting ? <Loader2 size={15} className={styles['ticket-board__spin']} /> : <Trash2 size={15} />}
            </button>
            <button className="btn btn-sm btn-ghost" onClick={onClose} aria-label="Close">
              <X size={16} />
            </button>
          </div>
        </header>

        <div className={styles['ticket-detail__body']}>
          <h3 className={styles['ticket-detail__title']}>{ticket.title}</h3>

          {ticket.description ? (
            <p className={styles['ticket-detail__desc']}>{ticket.description}</p>
          ) : (
            <p className={styles['ticket-detail__desc--empty']}>No description.</p>
          )}

          {(ticket.labels ?? []).length > 0 && (
            <div className={styles['ticket-card__labels']}>
              {(ticket.labels ?? []).map((l) => (
                <span key={l} className={styles['ticket-card__label']}>
                  {l}
                </span>
              ))}
            </div>
          )}

          <div className={styles['ticket-detail__people']}>
            <div className={styles['ticket-detail__person']}>
              <span className={styles['ticket-detail__person-label']}>Requester</span>
              <div className={styles['ticket-detail__person-val']}>
                <span
                  className={styles['ticket-card__avatar']}
                  style={{ background: avatarAccent(ticket.owner) }}
                >
                  {initials(ticket.owner)}
                </span>
                {ticket.owner.name}
                {owner && <span className={styles['ticket-detail__you']}>you</span>}
              </div>
            </div>
            <div className={styles['ticket-detail__person']}>
              <span className={styles['ticket-detail__person-label']}>Assignee</span>
              <div className={styles['ticket-detail__person-val']}>
                {ticket.assignee ? (
                  <>
                    <span
                      className={styles['ticket-card__avatar']}
                      style={{ background: avatarAccent(ticket.assignee) }}
                    >
                      {initials(ticket.assignee)}
                    </span>
                    {ticket.assignee.name}
                  </>
                ) : (
                  <span className={styles['ticket-detail__muted']}>Unassigned</span>
                )}
              </div>
            </div>
          </div>

          <div className={styles['ticket-detail__section-label']}>
            Status
            {moving && <Loader2 size={13} className={styles['ticket-board__spin']} />}
          </div>

          {/* Status stepper — the terminal actions are gated to the owner. */}
          <div className={styles['ticket-detail__stepper']}>
            {COLUMNS.filter((c) => c.status !== 'done').map((c) => {
              const isCurrent = ticket.status === c.status;
              return (
                <button
                  key={c.status}
                  type="button"
                  className={[
                    styles['ticket-detail__step'],
                    isCurrent ? styles['ticket-detail__step--current'] : '',
                  ].join(' ')}
                  style={{ ['--step-accent' as string]: c.accent }}
                  disabled={isCurrent || moving}
                  onClick={() => onMove(ticket.id, c.status)}
                >
                  {isCurrent && <Check size={13} />}
                  {c.label}
                </button>
              );
            })}
          </div>

          <div className={styles['ticket-detail__terminal']}>
            <button
              type="button"
              className={styles['ticket-detail__done-btn']}
              disabled={!owner || moving || (ticket.status === 'done' && ticket.resolution === 'completed')}
              title={owner ? undefined : OWNER_ONLY_HINT}
              onClick={() => onMove(ticket.id, 'done', 'completed')}
            >
              {owner ? <Check size={14} /> : <Lock size={14} />}
              Mark Done
            </button>
            <button
              type="button"
              className={styles['ticket-detail__discard-btn']}
              disabled={!owner || moving || (ticket.status === 'done' && ticket.resolution === 'discarded')}
              title={owner ? undefined : OWNER_ONLY_HINT}
              onClick={() => onMove(ticket.id, 'done', 'discarded')}
            >
              {owner ? <Ban size={14} /> : <Lock size={14} />}
              Discard
            </button>
          </div>
          {!owner && (
            <p className={styles['ticket-detail__gate-note']}>
              <Lock size={12} /> {OWNER_ONLY_HINT}
            </p>
          )}
        </div>
      </aside>

      {confirmDelete && (
        <ConfirmDialog
          title="Delete this ticket?"
          message={`"${ticket.title}" will be permanently removed. This can't be undone.`}
          confirmLabel="Delete"
          tone="danger"
          loading={deleting}
          onCancel={() => setConfirmDelete(false)}
          onConfirm={() => onDelete(ticket.id)}
        />
      )}
    </>
  );
}

























//Ticketmeta.ts
import type { TicketStatus, TicketPriority, Ticket, TicketUser } from '../../types/tickets';

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























//Ticketsslice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { ticketsApi } from '../../api/endpoints/tickets';
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





























//Tickets.ts
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


















//Tickets.ts
// ─────────────────────────────────────────────────────────────────────────
// Ticket domain types.
//
// Kept in their own file rather than widening the shared ../../types barrel,
// same approach used for CustomModelRequestWithParams in api/endpoints/models.
// Re-export these from your central `types` index if you'd rather import them
// alongside Model/Provider.
// ─────────────────────────────────────────────────────────────────────────

/** The four board columns. `done` is the single terminal column; whether the
 *  work was completed or dropped is captured by `resolution`. */
export type TicketStatus = 'todo' | 'in_progress' | 'in_review' | 'done';

/** Only meaningful when status === 'done'. */
export type TicketResolution = 'completed' | 'discarded';

export type TicketPriority = 'low' | 'medium' | 'high' | 'urgent';

export interface TicketUser {
  id: string;
  name: string;
}

export interface Ticket {
  id: string;
  /** Human-friendly key shown on the card, e.g. "REQ-42". Server-assigned. */
  key: string;
  title: string;
  description?: string;
  status: TicketStatus;
  resolution?: TicketResolution | null;
  priority: TicketPriority;
  /** The requester. Only this user may move the ticket into `done`. */
  owner: TicketUser;
  assignee?: TicketUser | null;
  labels?: string[];
  created_at: string;
  updated_at: string;
}

export interface CreateTicketRequest {
  title: string;
  description?: string;
  priority: TicketPriority;
  labels?: string[];
  assignee_id?: string | null;
}

/** Partial edit of an existing ticket (title/description/priority/labels/assignee). */
export interface UpdateTicketRequest extends Partial<CreateTicketRequest> {
  id: string;
}

/** Status transitions go through their own endpoint so the backend can apply
 *  the owner-only rule for entering `done`. */
export interface MoveTicketRequest {
  id: string;
  status: TicketStatus;
  resolution?: TicketResolution | null;
}
