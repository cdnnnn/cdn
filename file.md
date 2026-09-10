//Ticketdetailsidebar.tsx
import { useState } from 'react';
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
    </>
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

@keyframes sheetUpIn {
  from { transform: translateY(24px); opacity: 0; }
  to { transform: translateY(0); opacity: 1; }
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
  // Slightly below page body size — dense enough for a kanban card without
  // reading oversized next to the column chrome around it.
  font-size: 0.92em;
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
// Detail sheet — fixed-height panel pinned to the bottom, inset from the
// app's left nav. The area above it (down to where the sheet starts) is the
// dimmed backdrop; the sheet itself has square corners, not rounded ones.
// ===========================================================================
$sheet-height: 580px;
$sheet-left: 267px;

.ticket-detail__overlay {
  position: fixed;
  top: 0;
  left: $sheet-left;
  right: 0;
  bottom: calc(#{$footer-height} + #{$sheet-height});
  background: rgba(17, 24, 39, 0.4);
  z-index: 100;
}
.ticket-detail {
  position: fixed;
  left: $sheet-left;
  right: 0;
  bottom: $footer-height;
  height: $sheet-height;
  background: $surface;
  border-top: 1px solid $line;
  box-shadow: $shadow-4;
  z-index: 101;
  display: flex;
  flex-direction: column;
  animation: sheetUpIn 0.22s cubic-bezier(0.22, 0.72, 0.16, 1) both;
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
}




















//Createticketdrawer.tsx
import { useEffect, useMemo, useRef, useState } from 'react';
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

  return (
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
    </>
  );
}




















//Createticketdrawer.module.scss
@use '../../styles/_variables' as *;

// Bottom sheet — same fixed geometry as the ticket-detail sheet in
// TicketBoard.module.scss (267px inset from the left nav, 580px tall,
// square corners), so create/edit/detail all feel like the same surface
// sliding up from the same place.

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;

$sheet-height: 580px;
$sheet-left: 267px;

@keyframes sheetUpIn {
  from { transform: translateY(24px); opacity: 0; }
  to { transform: translateY(0); opacity: 1; }
}

.sheet__overlay {
  position: fixed;
  top: 0;
  left: $sheet-left;
  right: 0;
  bottom: calc(#{$footer-height} + #{$sheet-height});
  background: rgba(17, 24, 39, 0.4);
  z-index: 100;
}
.sheet {
  position: fixed;
  left: $sheet-left;
  right: 0;
  bottom: $footer-height;
  height: $sheet-height;
  background: $surface;
  border-top: 1px solid $line;
  box-shadow: $shadow-4;
  z-index: 101;
  display: flex;
  flex-direction: column;
  font-size: 13px;
  animation: sheetUpIn 0.22s cubic-bezier(0.22, 0.72, 0.16, 1) both;
}

// ---- header -----------------------------------------------------------
.sheet__header {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1.1em 1.25em;
  border-bottom: 1px solid $line;
}
.sheet__header-text {
  display: flex;
  flex-direction: column;
  gap: 0.15em;
}
.sheet__eyebrow {
  font-family: $mono;
  font-size: 0.68em;
  font-weight: 700;
  letter-spacing: 0.1em;
  text-transform: uppercase;
  color: $signal;
}
.sheet__title {
  font-size: 1.2em;
  font-weight: 700;
  color: $ink;
}

// ---- two-column body ----------------------------------------------------
.sheet__body {
  flex: 1;
  min-height: 0;
  display: flex;
}
.sheet__main {
  flex: 1;
  min-width: 0;
  overflow-y: auto;
  padding: 1.25em;
  display: flex;
  flex-direction: column;
  gap: 1.1em;
}
.sheet__rail {
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

.sheet__field {
  display: flex;
  flex-direction: column;
  gap: 0.4em;
}
.sheet__label {
  font-size: 0.78em;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.04em;
  color: $ink-2;
}
.sheet__req {
  color: $danger;
}
.sheet__input,
.sheet__textarea {
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
.sheet__input--lg {
  font-size: 1.15em;
  font-weight: 600;
  padding: 0.6em 0.7em;
}
.sheet__textarea {
  resize: vertical;
  line-height: 1.55;
  flex: 1;
}
.sheet__input--error {
  border-color: $danger;
  &:focus {
    box-shadow: 0 0 0 3px $danger-wash;
  }
}
.sheet__error {
  font-size: 0.78em;
  color: $danger;
}
.sheet__meta-note {
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
.sheet__attach-zone {
  display: flex;
  flex-wrap: wrap;
  gap: 0.6em;
}
.sheet__attach-thumb {
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
.sheet__attach-remove {
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
.sheet__attach-meta {
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
.sheet__attach-add {
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
.sheet__chip-input {
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
.sheet__chip {
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
.sheet__chip-field {
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
.sheet__footer {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 0.6em;
  padding: 1em 1.25em;
  border-top: 1px solid $line;
  background: $surface;
}
.sheet__spin {
  animation: spin 1.5s linear infinite;
}



















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















//Ticketsapi.mock.ts
import type {
  Ticket,
  TicketComment,
  CreateTicketRequest,
  UpdateTicketRequest,
  MoveTicketRequest,
  AddCommentRequest,
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
      attachments: payload.attachments ?? [],
      comments: [],
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
    if (rest.attachments !== undefined) t.attachments = rest.attachments;
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

  // Real backend: POST /tickets/:id/comments. Appends a comment authored by
  // the current user and returns the updated ticket.
  addComment: ({ ticket_id, text }: AddCommentRequest) => {
    const t = findOrThrow(ticket_id);
    const comment: TicketComment = {
      id: `c${Date.now()}`,
      author: CURRENT_USER,
      text,
      created_at: new Date().toISOString(),
    };
    t.comments = [...(t.comments ?? []), comment];
    t.updated_at = new Date().toISOString();
    return wait({ ...t });
  },
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

export interface TicketAttachment {
  id: string;
  name: string;
  /** Data URL in this static/mock build; a real backend would return a hosted URL. */
  url: string;
  size: number;
}

export interface TicketComment {
  id: string;
  author: TicketUser;
  text: string;
  created_at: string;
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
  attachments?: TicketAttachment[];
  comments?: TicketComment[];
  created_at: string;
  updated_at: string;
}

export interface CreateTicketRequest {
  title: string;
  description?: string;
  priority: TicketPriority;
  labels?: string[];
  assignee_id?: string | null;
  attachments?: TicketAttachment[];
}

/** Partial edit of an existing ticket (title/description/priority/labels/assignee/attachments). */
export interface UpdateTicketRequest extends Partial<CreateTicketRequest> {
  id: string;
}

export interface AddCommentRequest {
  ticket_id: string;
  text: string;
}

/** Status transitions go through their own endpoint so the backend can apply
 *  the owner-only rule for entering `done`. */
export interface MoveTicketRequest {
  id: string;
  status: TicketStatus;
  resolution?: TicketResolution | null;
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
