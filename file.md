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
  align-items: start;
  flex: 1;
  min-height: 0;
  overflow-y: auto;
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

@media (max-width: 768px) {
  .ticket-board__header { padding: 20px 18px 16px; flex-direction: column; align-items: flex-start; gap: 10px; }
  .ticket-board__toolbar { padding: 12px 18px; }
  .ticket-board__columns { padding: 0 18px 20px; grid-template-columns: 1fr; }
}
