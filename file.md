//Ticketboard.tsx
import { useEffect, useMemo, useState } from 'react';
import { Plus, Search, Lock, Loader2, Layers, ListFilter } from 'lucide-react';
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
import TicketCloseConfirm from './TicketCloseConfirm';
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

  // Set whenever a move would land a ticket in the terminal column; renders
  // TicketCloseConfirm instead of moving immediately. Cleared on choose/cancel.
  const [pendingClose, setPendingClose] = useState<{ id: string; key: string; title: string } | null>(
    null
  );

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

  const handleDrop = (to: TicketStatus) => {
    setDragOver(null);
    const id = draggingId;
    setDraggingId(null);
    if (!id) return;
    const ticket = items.find((t) => t.id === id);
    if (!ticket || ticket.status === to) return;
    requestMove(id, to);
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
                      dragging={draggingId === t.id}
                      onOpen={setDetail}
                      onMove={requestMove}
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
















//Ticketboard.module.scss
@use '../../styles/_variables' as *;

// ===========================================================================
// Ticket board — same ink/paper design system as Providers/Dashboard:
// theme-aware neutrals from _variables, flat accent constants, hover-lift
// cards, mono-ish instrument labels.
//
// Header/toolbar structure and font-scaling convention are copied 1:1 from
// Providers.module.scss: `.ticket-board` sets one base font-size that every
// descendant `em` value is relative to, bumped to 1rem at wide (>1800px)
// viewports so the whole page reads larger on big monitors without any
// individual rule changing.
// ===========================================================================

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;
$radius:  12px;

@keyframes modalIn {
  from { transform: translateY(8px) scale(0.98); opacity: 0; }
  to { transform: translateY(0) scale(1); opacity: 1; }
}

$soft: 0 1px 2px rgba(20, 22, 27, 0.05);
$lift: 0 14px 30px -14px rgba(20, 22, 27, 0.22);

// base font-size the board's internal `em` scale is built on — same value
// Providers uses, so the two pages feel identical in density.
$board-base-font: 0.8125rem;

%micro {
  font-family: $mono;
  font-size: 0.8462em; // 0.6875rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}

.ticket-board {
  // master scale control — every em-based font-size below responds to this
  font-size: $board-base-font;

  @media (min-width: 1800px) {
    font-size: 1rem;
  }

  display: flex;
  flex-direction: column;
  min-height: 0;
  flex: 1;
  color: $ink;
}

// ---- header -----------------------------------------------------------
.ticket-board__header {
  flex-shrink: 0;
  display: flex;
  align-items: flex-end;
  justify-content: space-between;
  gap: 1rem;
  padding: 24px 32px 20px;
  margin-bottom: 20px;
  border-bottom: 1px solid $line;
  background: $card;

  h1 {
    font-family: $display;
    font-size: 1.8462em; // 1.5rem / 0.8125rem
    font-weight: 800;
    letter-spacing: -0.02em;
    color: $ink;
    line-height: 1.2;
  }
}

.ticket-board__header-eyebrow {
  @extend %micro;
  display: flex;
  align-items: center;
  gap: 8px;
  color: $signal;
  margin-bottom: 6px;

  &::before {
    content: '';
    width: 16px;
    height: 2px;
    border-radius: 2px;
    background: $signal;
  }
}

.ticket-board__header-sub {
  margin-top: 4px;
  font-size: 1.0385em; // 0.84375rem / 0.8125rem
  color: $ink-2;
}

.ticket-board__header-meta {
  flex-shrink: 0;
  display: inline-flex;
  align-items: center;
  gap: 7px;
  padding: 7px 13px;
  border-radius: 999px;
  border: 1px solid $line;
  background: $paper;
  font-family: $mono;
  font-size: 0.8846em; // 0.71875rem / 0.8125rem
  font-weight: 700;
  letter-spacing: 0.06em;
  text-transform: uppercase;
  color: $ink-2;
  white-space: nowrap;
  margin-bottom: 3px;
}

// ---- toolbar ------------------------------------------------------------
.ticket-board__toolbar {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 14px;
  padding: 14px 32px;
  background: $card;
  border-bottom: 1px solid $line;
  margin-bottom: 20px;
  flex-wrap: wrap;
}

.ticket-board__search {
  position: relative;
  flex: 1;
  max-width: 340px;
  min-width: 200px;

  svg {
    position: absolute;
    top: 50%;
    left: 13px;
    transform: translateY(-50%);
    color: $ink-3;
    pointer-events: none;
  }

  input {
    width: 100%;
    border: 1.5px solid $line;
    border-radius: 10px;
    padding: 9px 12px 9px 38px;
    font-size: 1.0385em; // 0.84375rem / 0.8125rem
    font-family: $sans;
    color: $ink;
    background: $paper;
    transition: border-color 0.15s ease, background 0.15s ease;

    &::placeholder { color: $ink-3; }
    &:focus {
      outline: none;
      border-color: $signal;
      background: $card;
    }
  }
}

.ticket-board__toolbar-right {
  display: flex;
  align-items: center;
  gap: 14px;
  flex-wrap: wrap;
}

.ticket-board__filter-group {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 4px;
  background: $paper;
  border: 1px solid $line;
  border-radius: 999px;
}

.ticket-board__toolbar-label {
  flex-shrink: 0;
  display: inline-flex;
  align-items: center;
  gap: 5px;
  padding: 5px 10px 5px 11px;
  @extend %micro;
  font-size: 0.7692em; // 0.625rem / 0.8125rem
  color: $ink-3;
  white-space: nowrap;
}

.ticket-board__filter-pill {
  padding: 6px 13px;
  border: 0;
  border-radius: 999px;
  background: transparent;
  color: $ink-2;
  font-size: 0.9615em; // 0.78125rem / 0.8125rem
  font-weight: 650;
  cursor: pointer;
  transition: all 0.15s ease;

  &:hover { color: $ink; }

  &--on {
    background: $card;
    color: $signal;
    box-shadow: $soft;
  }
}

.ticket-board__toolbar-divider {
  flex-shrink: 0;
  width: 1px;
  height: 26px;
  background: $line;
}

.ticket-board__add-btn {
  display: inline-flex;
  align-items: center;
  gap: 7px;
  padding: 9px 15px;
  border: 1px solid $signal;
  border-radius: 10px;
  background: $signal;
  color: #fff;
  font-family: $sans;
  font-size: 1em; // 0.8125rem / 0.8125rem (base)
  font-weight: 650;
  cursor: pointer;
  box-shadow: $soft;
  transition: background 0.16s ease, border-color 0.16s ease, transform 0.16s ease, box-shadow 0.16s ease;

  &:hover { background: $signal-2; border-color: $signal-2; transform: translateY(-1px); box-shadow: $lift; }
}

// ---- columns --------------------------------------------------------------
.ticket-board__columns {
  padding: 0 32px 28px;
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: 1em;
  align-items: stretch;
  flex: 1;
  min-height: 0;
}
.ticket-board__column {
  height: 100%;
  min-height: 0;
  background: $paper;
  border: 1px solid $line-2;
  border-radius: $radius;
  padding: 0.75em;
  display: flex;
  flex-direction: column;
  gap: 0.6em;
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
  flex-shrink: 0;
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
  flex: 1;
  min-height: 0;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 0.6em;
  // small inset so the scrollbar doesn't sit flush against the cards
  padding-right: 0.15em;
  margin-right: -0.15em;
}
.ticket-board__column-empty {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  text-align: center;
  color: $ink-3;
  font-size: 0.82em;
  border: 1px dashed $line;
  border-radius: 8px;
  padding: 1em;
}

// ---- card -----------------------------------------------------------------
.ticket-card {
  --priority-accent: #{$ink-3};
  // Slightly below page body size — dense enough for a kanban card without
  // reading oversized next to the column chrome around it.
  font-size: 0.92em;
  position: relative;
  background: $card;
  border: 1px solid $line;
  border-radius: 10px;
  padding: 0.75em 0.8em;
  display: flex;
  flex-direction: column;
  gap: 0.55em;
  cursor: grab;
  box-shadow: $shadow-2;
  transition: transform 0.12s, box-shadow 0.12s, border-color 0.12s, opacity 0.12s;
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
.ticket-card--dragging {
  opacity: 0.35;
  border-style: dashed;
  border-color: $signal;
  box-shadow: none;
  transform: scale(0.98);
  cursor: grabbing;
  &:hover {
    transform: scale(0.98);
  }
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
  font-size: 0.85em;
  font-weight: 700;
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
  font-size: 0.8em;
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
  font-size: 1.03em;
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
  font-size: 0.82em;
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
  font-size: 0.78em;
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
  font-size: 0.72em;
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
// Detail modal — centered dialog (not a bottom sheet), sized to fit the
// two-pane main/rail layout below.
// ===========================================================================
$detail-width: 980px;
$detail-max-height: 700px;

.ticket-detail__overlay {
  position: fixed;
  inset: 0;
  bottom: $footer-height;
  background: rgba(17, 24, 39, 0.5);
  z-index: 100;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 28px;
}
.ticket-detail {
  position: relative;
  width: min(#{$detail-width}, 100%);
  max-height: min(#{$detail-max-height}, 100%);
  background: $surface;
  border: 1px solid $line;
  border-radius: 14px;
  box-shadow: $shadow-4;
  z-index: 101;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  // Own base size, slightly larger than the page base at very wide
  // viewports — a focused modal reads better a touch bigger than the
  // dense board sitting behind it.
  font-size: 0.8125rem;
  @media (min-width: 1800px) {
    font-size: 1.0625rem;
  }
  animation: modalIn 0.16s ease both;
}
.ticket-detail__header {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.5em;
  padding: 1em 1.25em;
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

// Two-pane body: scrollable content on the left, a narrow fixed-width meta
// rail on the right holding requester/assignee/status/actions — so those
// no longer stretch to the panel's full width.
.ticket-detail__body {
  flex: 1;
  min-height: 0;
  display: flex;
}
.ticket-detail__main {
  flex: 1;
  min-width: 0;
  overflow-y: auto;
  padding: 1.1em 1.25em;
  display: flex;
  flex-direction: column;
  gap: 1.1em;
}
.ticket-detail__rail {
  flex: none;
  width: 230px;
  border-left: 1px solid $line;
  background: $paper;
  overflow-y: auto;
  padding: 1.1em;
  display: flex;
  flex-direction: column;
  gap: 1.2em;
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

// ---- rail: compact people chips (auto-width, not stretched) --------------
.ticket-detail__people {
  display: flex;
  flex-direction: column;
  gap: 0.8em;
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
  display: inline-flex;
  align-items: center;
  gap: 0.45em;
  width: fit-content;
  max-width: 100%;
  font-size: 0.86em;
  font-weight: 500;
  padding: 0.3em 0.55em 0.3em 0.3em;
  border-radius: 999px;
  background: $card;
  border: 1px solid $line;
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
  font-size: 0.68em;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: $ink-3;
}

// ---- rail: status — compact auto-width pills, not a full-width grid ------
.ticket-detail__stepper {
  display: flex;
  flex-wrap: wrap;
  gap: 0.4em;
}
.ticket-detail__step {
  --step-accent: #{$signal};
  flex: none;
  border: 1px solid $line;
  background: $card;
  color: $ink-2;
  font-size: 0.78em;
  font-weight: 600;
  padding: 0.45em 0.65em;
  border-radius: 999px;
  cursor: pointer;
  display: inline-flex;
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

// ---- rail: terminal actions — compact auto-width buttons, side by side ---
.ticket-detail__terminal {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5em;
}
.ticket-detail__done-btn,
.ticket-detail__discard-btn {
  flex: none;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 0.4em;
  padding: 0.5em 0.8em;
  border-radius: 999px;
  font-size: 0.8em;
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
  align-items: flex-start;
  gap: 0.4em;
  margin: 0;
  font-size: 0.75em;
  line-height: 1.4;
  color: $ink-3;
}

// ---- main: attachments -----------------------------------------------------
.ticket-detail__attachments {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(72px, 1fr));
  gap: 0.5em;
}
.ticket-detail__attachment {
  position: relative;
  aspect-ratio: 1;
  border-radius: 8px;
  overflow: hidden;
  border: 1px solid $line;
  background: $paper;
  display: block;

  img {
    width: 100%;
    height: 100%;
    object-fit: cover;
  }
}

// ---- main: comments ---------------------------------------------------------
.ticket-detail__comments {
  display: flex;
  flex-direction: column;
  gap: 0.9em;
}
.ticket-detail__comment {
  display: flex;
  gap: 0.6em;
}
.ticket-detail__comment-body {
  flex: 1;
  min-width: 0;
  background: $paper;
  border: 1px solid $line-2;
  border-radius: 10px;
  padding: 0.6em 0.75em;
}
.ticket-detail__comment-head {
  display: flex;
  align-items: baseline;
  gap: 0.5em;
  margin-bottom: 0.2em;
}
.ticket-detail__comment-author {
  font-size: 0.85em;
  font-weight: 700;
  color: $ink;
}
.ticket-detail__comment-time {
  font-size: 0.72em;
  color: $ink-3;
}
.ticket-detail__comment-text {
  margin: 0;
  font-size: 0.86em;
  line-height: 1.5;
  color: $ink-2;
  white-space: pre-wrap;
}
.ticket-detail__comment-empty {
  font-size: 0.85em;
  color: $ink-3;
  font-style: italic;
}
.ticket-detail__comment-form {
  display: flex;
  gap: 0.6em;
  align-items: flex-start;
}
.ticket-detail__comment-input {
  flex: 1;
  min-height: 2.6em;
  max-height: 8em;
  resize: vertical;
  border: 1px solid $line;
  border-radius: 10px;
  padding: 0.55em 0.7em;
  font-family: $sans;
  font-size: 0.86em;
  color: $ink;
  background: $card;
  transition: border-color 0.15s, box-shadow 0.15s;
  &::placeholder { color: $ink-3; }
  &:focus {
    outline: none;
    border-color: $signal;
    box-shadow: 0 0 0 3px $wash;
  }
}
.ticket-detail__comment-send {
  flex: none;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 2.6em;
  height: 2.6em;
  border-radius: 10px;
  border: 1px solid $signal;
  background: $signal;
  color: #fff;
  cursor: pointer;
  transition: background 0.15s;
  &:hover:not(:disabled) { background: $signal-2; }
  &:disabled { opacity: 0.5; cursor: not-allowed; }
}

@media (max-width: 768px) {
  .ticket-board__header { padding: 20px 18px 16px; flex-direction: column; align-items: flex-start; gap: 10px; }
  .ticket-board__toolbar { padding: 12px 18px; }
  .ticket-board__columns { padding: 0 18px 20px; grid-template-columns: 1fr; }
  .ticket-detail__overlay { padding: 0; }
  .ticket-detail { width: 100%; max-height: 100%; border-radius: 0; border: 0; }
  .ticket-detail__body { flex-direction: column; overflow-y: auto; }
  .ticket-detail__rail { width: auto; border-left: 0; border-top: 1px solid $line; }
}


// ===========================================================================
// Close-ticket confirmation — small centered modal asking the user to pick
// Done or Discard before a ticket enters the terminal column.
// ===========================================================================
.close-confirm__overlay {
  position: fixed;
  inset: 0;
  bottom: $footer-height;
  background: rgba(17, 24, 39, 0.5);
  z-index: 110;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 24px;
}
.close-confirm {
  position: relative;
  width: min(360px, 100%);
  background: $surface;
  border: 1px solid $line;
  border-radius: 14px;
  box-shadow: $shadow-4;
  z-index: 111;
  padding: 1.5em 1.5em 1.25em;
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
  gap: 0.3em;
  font-size: 13px;
  animation: modalIn 0.16s ease both;
}
.close-confirm__x {
  position: absolute;
  top: 0.8em;
  right: 0.8em;
  border: 0;
  background: transparent;
  color: $ink-3;
  padding: 0.3em;
  border-radius: 6px;
  cursor: pointer;
  &:hover {
    background: $ink-wash;
    color: $ink;
  }
}
.close-confirm__key {
  font-family: $mono;
  font-size: 0.75em;
  font-weight: 700;
  letter-spacing: 0.05em;
  color: $ink-3;
}
.close-confirm__title {
  margin: 0.3em 0 0;
  font-size: 1.15em;
  font-weight: 700;
  color: $ink;
}
.close-confirm__msg {
  margin: 0;
  font-size: 0.9em;
  color: $ink-2;
  max-width: 26em;
}
.close-confirm__actions {
  display: flex;
  gap: 0.6em;
  width: 100%;
  margin-top: 1em;
}
.close-confirm__done,
.close-confirm__discard {
  flex: 1;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 0.4em;
  padding: 0.7em 0.8em;
  border-radius: 10px;
  font-size: 0.9em;
  font-weight: 650;
  cursor: pointer;
  border: 1px solid transparent;
  transition: filter 0.12s, transform 0.12s;
  &:hover {
    filter: brightness(0.96);
    transform: translateY(-1px);
  }
}
.close-confirm__done {
  background: $ok;
  color: #fff;
}
.close-confirm__discard {
  background: $card;
  border-color: $danger;
  color: $danger;
}
.close-confirm__cancel {
  margin-top: 0.6em;
  border: 0;
  background: transparent;
  color: $ink-3;
  font-size: 0.84em;
  cursor: pointer;
  padding: 0.3em 0.6em;
  &:hover {
    color: $ink;
  }
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
  dragging?: boolean;
  onOpen: (ticket: Ticket) => void;
  onMove: (id: string, status: TicketStatus, resolution?: TicketResolution | null) => void;
  onDragStart: (id: string) => void;
  onDragEnd: () => void;
}

export default function TicketCard({
  ticket,
  currentUser,
  moving = false,
  dragging = false,
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
        dragging ? styles['ticket-card--dragging'] : '',
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












//Ticketdetailssidebar.tsx
import { useState } from 'react';
import { createPortal } from 'react-dom';
import { X, Pencil, Trash2, Loader2, Lock, Check, Ban, Send, Paperclip } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import { addTicketComment } from '../../store/slices/ticketsSlice';
import type { Ticket, TicketStatus, TicketResolution, TicketUser } from '../../types/tickets';
import ConfirmDialog from '../common/ConfirmDialog';
import { useToast } from '../common/Toast';
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

const formatTime = (iso: string) => {
  try {
    return new Date(iso).toLocaleString(undefined, {
      month: 'short',
      day: 'numeric',
      hour: 'numeric',
      minute: '2-digit',
    });
  } catch {
    return iso;
  }
};

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
  const dispatch = useAppDispatch();
  const toast = useToast();
  const commentingId = useAppSelector((s) => s.tickets.commentingId);
  const [confirmDelete, setConfirmDelete] = useState(false);
  const [commentDraft, setCommentDraft] = useState('');

  const owner = isOwner(ticket, currentUser.id);
  const priority = PRIORITY_META[ticket.priority];
  const posting = commentingId === ticket.id;
  const comments = ticket.comments ?? [];

  const submitComment = () => {
    const text = commentDraft.trim();
    if (!text) return;
    dispatch(addTicketComment({ ticket_id: ticket.id, text }))
      .unwrap()
      .then(() => setCommentDraft(''))
      .catch((e) => toast.error(typeof e === 'string' ? e : 'Could not post comment'));
  };

  return createPortal(
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
          {/* ---- main: title, description, attachments, comments ---- */}
          <div className={styles['ticket-detail__main']}>
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

            {(ticket.attachments ?? []).length > 0 && (
              <div>
                <div className={styles['ticket-detail__section-label']}>
                  <Paperclip size={11} />
                  Attachments ({ticket.attachments!.length})
                </div>
                <div className={styles['ticket-detail__attachments']} style={{ marginTop: '0.6em' }}>
                  {ticket.attachments!.map((a) => (
                    <a
                      key={a.id}
                      className={styles['ticket-detail__attachment']}
                      href={a.url}
                      target="_blank"
                      rel="noreferrer"
                      title={a.name}
                    >
                      <img src={a.url} alt={a.name} />
                    </a>
                  ))}
                </div>
              </div>
            )}

            <div>
              <div className={styles['ticket-detail__section-label']}>
                Comments {comments.length > 0 && `(${comments.length})`}
              </div>
              <div className={styles['ticket-detail__comments']} style={{ marginTop: '0.7em' }}>
                {comments.length === 0 && (
                  <p className={styles['ticket-detail__comment-empty']}>
                    No comments yet — start the discussion.
                  </p>
                )}
                {comments.map((c) => (
                  <div key={c.id} className={styles['ticket-detail__comment']}>
                    <span
                      className={styles['ticket-card__avatar']}
                      style={{ background: avatarAccent(c.author) }}
                    >
                      {initials(c.author)}
                    </span>
                    <div className={styles['ticket-detail__comment-body']}>
                      <div className={styles['ticket-detail__comment-head']}>
                        <span className={styles['ticket-detail__comment-author']}>
                          {c.author.name}
                        </span>
                        <span className={styles['ticket-detail__comment-time']}>
                          {formatTime(c.created_at)}
                        </span>
                      </div>
                      <p className={styles['ticket-detail__comment-text']}>{c.text}</p>
                    </div>
                  </div>
                ))}

                <div className={styles['ticket-detail__comment-form']}>
                  <textarea
                    className={styles['ticket-detail__comment-input']}
                    value={commentDraft}
                    onChange={(e) => setCommentDraft(e.target.value)}
                    onKeyDown={(e) => {
                      if (e.key === 'Enter' && (e.metaKey || e.ctrlKey)) {
                        e.preventDefault();
                        submitComment();
                      }
                    }}
                    placeholder="Add a comment… (⌘/Ctrl + Enter to send)"
                    disabled={posting}
                  />
                  <button
                    type="button"
                    className={styles['ticket-detail__comment-send']}
                    onClick={submitComment}
                    disabled={posting || !commentDraft.trim()}
                    aria-label="Post comment"
                  >
                    {posting ? (
                      <Loader2 size={15} className={styles['ticket-board__spin']} />
                    ) : (
                      <Send size={15} />
                    )}
                  </button>
                </div>
              </div>
            </div>
          </div>

          {/* ---- rail: requester/assignee, status, terminal actions ---- */}
          <div className={styles['ticket-detail__rail']}>
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
                {ticket.assignee ? (
                  <div className={styles['ticket-detail__person-val']}>
                    <span
                      className={styles['ticket-card__avatar']}
                      style={{ background: avatarAccent(ticket.assignee) }}
                    >
                      {initials(ticket.assignee)}
                    </span>
                    {ticket.assignee.name}
                  </div>
                ) : (
                  <span className={styles['ticket-detail__muted']}>Unassigned</span>
                )}
              </div>
            </div>

            <div>
              <div className={styles['ticket-detail__section-label']}>
                Status
                {moving && <Loader2 size={13} className={styles['ticket-board__spin']} />}
              </div>
              <div className={styles['ticket-detail__stepper']} style={{ marginTop: '0.5em' }}>
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
            </div>

            <div>
              <div className={styles['ticket-detail__section-label']}>Close ticket</div>
              <div className={styles['ticket-detail__terminal']} style={{ marginTop: '0.5em' }}>
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
                <p className={styles['ticket-detail__gate-note']} style={{ marginTop: '0.5em' }}>
                  <Lock size={12} /> {OWNER_ONLY_HINT}
                </p>
              )}
            </div>
          </div>
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
    </>,
    document.body
  );
}


















//Createticketdrawer.tsx
import { useEffect, useMemo, useRef, useState } from 'react';
import { createPortal } from 'react-dom';
import { X, Loader2, ImagePlus, FileText } from 'lucide-react';
import type { Ticket, TicketAttachment, TicketPriority, TicketUser } from '../../types/tickets';
import { PRIORITY_META } from './ticketMeta';
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
  /** Return a Promise to keep the sheet open on failure and close it on success. */
  onSubmit: (payload: TicketSubmitPayload) => Promise<unknown> | void;
}

const MAX_ATTACHMENT_BYTES = 5 * 1024 * 1024; // 5MB per image, static/mock build

const readAsDataUrl = (file: File) =>
  new Promise<string>((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = () => resolve(reader.result as string);
    reader.onerror = () => reject(reader.error);
    reader.readAsDataURL(file);
  });

const formatSize = (bytes: number) =>
  bytes < 1024 * 1024 ? `${Math.round(bytes / 1024)} KB` : `${(bytes / (1024 * 1024)).toFixed(1)} MB`;

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
  const [attachments, setAttachments] = useState<TicketAttachment[]>(initialTicket?.attachments ?? []);
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

  const titleError = touched && !title.trim() ? 'Title is required' : '';
  const valid = title.trim().length > 0;

  const addLabel = () => {
    const v = labelDraft.trim();
    if (!v) return;
    if (!labels.includes(v)) setLabels((prev) => [...prev, v]);
    setLabelDraft('');
  };

  const handleFiles = async (fileList: FileList | null) => {
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
    const next: TicketAttachment[] = await Promise.all(
      files.map(async (f) => ({
        id: `a${Date.now()}${Math.random().toString(36).slice(2, 7)}`,
        name: f.name,
        url: await readAsDataUrl(f),
        size: f.size,
      }))
    );
    setAttachments((prev) => [...prev, ...next]);
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
      attachments: attachments.length ? attachments : undefined,
    });
  };

  const heading = mode === 'edit' ? 'Edit requirement' : 'New requirement';
  const cta = mode === 'edit' ? 'Save changes' : 'Create requirement';

  const priorities = useMemo(() => Object.keys(PRIORITY_META) as TicketPriority[], []);

  return createPortal(
    <>
      <div className={styles['sheet__overlay']} onClick={() => !submitting && onClose()} />
      <aside className={styles['sheet']} role="dialog" aria-modal="true" aria-label={heading}>
        <header className={styles['sheet__header']}>
          <div className={styles['sheet__header-text']}>
            <span className={styles['sheet__eyebrow']}>{mode === 'edit' ? 'Editing' : 'New'}</span>
            <span className={styles['sheet__title']}>{heading}</span>
          </div>
          <button
            className="btn btn-sm btn-ghost"
            onClick={onClose}
            disabled={submitting}
            aria-label="Close"
          >
            <X size={16} />
          </button>
        </header>

        <div className={styles['sheet__body']}>
          {/* ---- main column: title, description, attachments ---- */}
          <div className={styles['sheet__main']}>
            <label className={styles['sheet__field']}>
              <span className={styles['sheet__label']}>
                Title <span className={styles['sheet__req']}>*</span>
              </span>
              <input
                ref={firstFieldRef}
                className={`${styles['sheet__input']} ${styles['sheet__input--lg']} ${titleError ? styles['sheet__input--error'] : ''}`}
                value={title}
                onChange={(e) => setTitle(e.target.value)}
                onBlur={() => setTouched(true)}
                placeholder="Short summary of the requirement"
              />
              {titleError && <span className={styles['sheet__error']}>{titleError}</span>}
            </label>

            <label className={styles['sheet__field']}>
              <span className={styles['sheet__label']}>Description</span>
              <textarea
                className={styles['sheet__textarea']}
                value={description}
                onChange={(e) => setDescription(e.target.value)}
                rows={7}
                placeholder="Context, acceptance criteria, links…"
              />
            </label>

            <div className={styles['sheet__field']}>
              <span className={styles['sheet__label']}>Attachments</span>
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
              <div className={styles['sheet__attach-zone']}>
                {attachments.map((a) => (
                  <div key={a.id} className={styles['sheet__attach-thumb']}>
                    <img src={a.url} alt={a.name} />
                    <button
                      type="button"
                      className={styles['sheet__attach-remove']}
                      onClick={() => setAttachments((prev) => prev.filter((x) => x.id !== a.id))}
                      aria-label={`Remove ${a.name}`}
                    >
                      <X size={11} />
                    </button>
                    <span className={styles['sheet__attach-meta']}>{formatSize(a.size)}</span>
                  </div>
                ))}
                <button
                  type="button"
                  className={styles['sheet__attach-add']}
                  onClick={() => fileInputRef.current?.click()}
                >
                  <ImagePlus size={18} />
                  Add image
                </button>
              </div>
              {attachError && <span className={styles['sheet__error']}>{attachError}</span>}
            </div>
          </div>

          {/* ---- meta column: priority, assignee, labels ---- */}
          <div className={styles['sheet__rail']}>
            <label className={styles['sheet__field']}>
              <span className={styles['sheet__label']}>Priority</span>
              <select
                className={styles['sheet__input']}
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

            <label className={styles['sheet__field']}>
              <span className={styles['sheet__label']}>Assignee</span>
              <select
                className={styles['sheet__input']}
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

            <div className={styles['sheet__field']}>
              <span className={styles['sheet__label']}>Labels</span>
              <div className={styles['sheet__chip-input']}>
                {labels.map((l) => (
                  <span key={l} className={styles['sheet__chip']}>
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
                  className={styles['sheet__chip-field']}
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
              <div className={styles['sheet__meta-note']}>
                <FileText size={12} />
                {initialTicket.key} · created{' '}
                {new Date(initialTicket.created_at).toLocaleDateString()}
              </div>
            )}
          </div>
        </div>

        <footer className={styles['sheet__footer']}>
          <button className="btn btn-ghost" onClick={onClose} disabled={submitting}>
            Cancel
          </button>
          <button className="btn btn-primary" onClick={submit} disabled={submitting || !valid}>
            {submitting ? (
              <>
                <Loader2 size={15} className={styles['sheet__spin']} />
                Saving…
              </>
            ) : (
              cta
            )}
          </button>
        </footer>
      </aside>
    </>,
    document.body
  );
}













//Ticketcloseconfirm.tsx
import { createPortal } from 'react-dom';
import { Check, Ban, X } from 'lucide-react';
import type { TicketResolution } from '../../types/tickets';
import styles from './TicketBoard.module.scss';

interface TicketCloseConfirmProps {
  ticketKey: string;
  title: string;
  onCancel: () => void;
  onConfirm: (resolution: TicketResolution) => void;
}

/**
 * Asks the user to explicitly choose Done or Discard before a ticket enters
 * the terminal column — fired from drag-and-drop onto Done, the card menu,
 * and the detail view's close actions alike, so there's one confirmation
 * moment no matter how the move was triggered.
 */
export default function TicketCloseConfirm({
  ticketKey,
  title,
  onCancel,
  onConfirm,
}: TicketCloseConfirmProps) {
  return createPortal(
    <>
      <div className={styles['close-confirm__overlay']} onClick={onCancel} />
      <div className={styles['close-confirm']} role="alertdialog" aria-modal="true" aria-label="Close ticket">
        <button className={styles['close-confirm__x']} onClick={onCancel} aria-label="Cancel">
          <X size={14} />
        </button>
        <span className={styles['close-confirm__key']}>{ticketKey}</span>
        <h4 className={styles['close-confirm__title']}>How should this ticket close?</h4>
        <p className={styles['close-confirm__msg']}>&ldquo;{title}&rdquo;</p>

        <div className={styles['close-confirm__actions']}>
          <button
            type="button"
            className={styles['close-confirm__done']}
            onClick={() => onConfirm('completed')}
            autoFocus
          >
            <Check size={16} />
            Mark Done
          </button>
          <button
            type="button"
            className={styles['close-confirm__discard']}
            onClick={() => onConfirm('discarded')}
          >
            <Ban size={16} />
            Discard
          </button>
        </div>

        <button type="button" className={styles['close-confirm__cancel']} onClick={onCancel}>
          Cancel
        </button>
      </div>
    </>,
    document.body
  );
}
