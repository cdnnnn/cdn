//Ticketdetailsidebar.tsx
import { useState, type MouseEvent as ReactMouseEvent } from 'react';
import { X, Pencil, Trash2, Loader2, Lock, Check, Ban, History as HistoryIcon } from 'lucide-react';
import Lightbox from 'yet-another-react-lightbox';
import Zoom from 'yet-another-react-lightbox/plugins/zoom';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import { addTicketComment } from '../../store/slices/ticketsSlice';
import type { Ticket, TicketStatus, TicketResolution, TicketUser } from '../../types/tickets';
import ConfirmDialog from '../common/ConfirmDialog';
import { useToast } from '../common/Toast';
import TicketDescriptionEditor from './TicketDescriptionEditor';
import {
  COLUMNS,
  PRIORITY_META,
  PRIORITY_ICON,
  isOwner,
  isSequentialMove,
  canDropTicket,
  isEmptyHtml,
  sanitizeTicketHtml,
  OWNER_ONLY_HINT,
  OWNER_ONLY_DELETE_HINT,
  SEQUENCE_HINT,
  initials,
  avatarAccent,
} from './ticketMeta';
import styles from './TicketBoard.module.scss';

interface TicketDetailSidebarProps {
  /** Looked up live from the store below — never a cached snapshot, so this
   *  view can't go stale after a comment/move/edit the way passing the
   *  whole `Ticket` object down as a static prop could. */
  ticketId: string;
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

// A history entry's `to_status` alone doesn't distinguish "closed as done"
// from "closed as discarded" — both are `status: 'done'` — so this reads
// `resolution` too when relevant.
const historyStatusLabel = (status: TicketStatus, resolution?: TicketResolution | null) => {
  if (status === 'done') return resolution === 'discarded' ? 'Discarded' : 'Done';
  return COLUMNS.find((c) => c.status === status)?.label ?? status;
};

const historyStatusAccent = (status: TicketStatus, resolution?: TicketResolution | null) => {
  if (status === 'done' && resolution === 'discarded') return '#DC2626';
  return COLUMNS.find((c) => c.status === status)?.accent ?? '#8A909B';
};

const assigneeChangeVerb = (h: { from_assignee: TicketUser | null; to_assignee: TicketUser | null }) => {
  if (!h.from_assignee && h.to_assignee) return `assigned it to ${h.to_assignee.name}`;
  if (h.from_assignee && !h.to_assignee) return `unassigned it (was ${h.from_assignee.name})`;
  if (h.from_assignee && h.to_assignee) return `reassigned it from ${h.from_assignee.name} to ${h.to_assignee.name}`;
  return 'changed the assignee';
};

// Centered modal, same shell as the rest of the app's dialogs (see
// .modal-overlay / .modal in TicketBoard.module.scss). Rendered inline —
// no portal — position:fixed + flex-centering is sufficient here.
export default function TicketDetailSidebar({
  ticketId,
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
  // Selected live from the store on every render — this is the fix for
  // "posting a comment doesn't show up until I refresh": previously this
  // component received a `ticket` object as a prop that was only synced
  // back up from the store via a separate effect in TicketBoard, which is
  // an extra hop that can (and did) go stale. Reading directly from the
  // store here removes that hop entirely.
  const ticket = useAppSelector((s) => s.tickets.items.find((t) => t.id === ticketId));
  const commentingId = useAppSelector((s) => s.tickets.commentingId);
  const [confirmDelete, setConfirmDelete] = useState(false);
  const [commentDraft, setCommentDraft] = useState('');
  const [activeTab, setActiveTab] = useState<'comments' | 'history'>('comments');
  const [zoomSrc, setZoomSrc] = useState<string | null>(null);

  // Description/comments are rendered from a raw HTML string via
  // dangerouslySetInnerHTML, so there's no TipTap NodeView here to wire a
  // click handler to individual images — event delegation on the container
  // does the same job: any click that landed on an <img> opens it in the
  // lightbox.
  const openImageIfClicked = (e: ReactMouseEvent<HTMLDivElement>) => {
    const target = e.target as HTMLElement;
    if (target.tagName === 'IMG') {
      setZoomSrc((target as HTMLImageElement).src);
    }
  };

  if (!ticket) return null; // e.g. deleted from another tab/session

  const owner = isOwner(ticket, currentUser.id);
  const priority = PRIORITY_META[ticket.priority];
  const PriorityIcon = PRIORITY_ICON[ticket.priority];
  const posting = commentingId === ticket.id;
  const comments = ticket.comments ?? [];
  const history = ticket.history ?? [];

  const submitComment = () => {
    if (isEmptyHtml(commentDraft)) return;
    dispatch(addTicketComment({ ticket_id: ticket.id, text: commentDraft }))
      .unwrap()
      .then(() => setCommentDraft(''))
      .catch((e) => toast.error(typeof e === 'string' ? e : 'Could not post comment'));
  };

  return (
    <div className={styles['modal-overlay']} onClick={onClose}>
      <div
        className={styles['modal']}
        role="dialog"
        aria-modal="true"
        aria-label="Ticket detail"
        onClick={(e) => e.stopPropagation()}
      >
        <header className={styles['modal-hdr']}>
          <div>
            <span className={styles['modal-key']}>{ticket.key}</span>
            <span
              className={[
                styles['ticket-card__priority'],
                ticket.priority === 'urgent' ? styles['ticket-card__priority--urgent'] : '',
              ].join(' ')}
              style={{ ['--priority-accent' as string]: priority.accent }}
            >
              <PriorityIcon size={11} strokeWidth={2.75} className={styles['ticket-card__priority-icon']} />
              {priority.label}
            </span>
          </div>
          <div style={{ display: 'flex', alignItems: 'center', gap: '0.4em' }}>
            <button
              type="button"
              className={styles['modal-close']}
              onClick={() => onEdit(ticket)}
              aria-label="Edit ticket"
              title="Edit"
            >
              <Pencil size={14} />
            </button>
            <button
              type="button"
              className={styles['modal-close']}
              onClick={() => setConfirmDelete(true)}
              disabled={deleting || !owner}
              aria-label="Delete ticket"
              title={owner ? 'Delete' : OWNER_ONLY_DELETE_HINT}
            >
              {deleting ? <Loader2 size={14} className={styles['ticket-board__spin']} /> : <Trash2 size={14} />}
            </button>
            <button className={styles['modal-close']} onClick={onClose} aria-label="Close">
              <X size={16} />
            </button>
          </div>
        </header>

        <div className={styles['modal-body']}>
          {/* ---- main: title, description (with inline images), comments ---- */}
          <div className={styles['modal-main']}>
            <h3 className={styles['ticket-detail__title']}>{ticket.title}</h3>

            {!isEmptyHtml(ticket.description) ? (
              <div
                className={styles['ticket-detail__desc']}
                onClick={openImageIfClicked}
                // Description is rich-text HTML from the description editor
                // (images the requester pasted/dropped/inserted are already
                // embedded inline as <img> tags) — sanitize before ever
                // injecting it, since this is otherwise-untrusted content.
                dangerouslySetInnerHTML={{ __html: sanitizeTicketHtml(ticket.description) }}
              />
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

            <div className={styles['ticket-detail__tabs']}>
              <button
                type="button"
                className={[
                  styles['ticket-detail__tab'],
                  activeTab === 'comments' ? styles['ticket-detail__tab--active'] : '',
                ].join(' ')}
                onClick={() => setActiveTab('comments')}
              >
                Comments {comments.length > 0 && `(${comments.length})`}
              </button>
              <button
                type="button"
                className={[
                  styles['ticket-detail__tab'],
                  activeTab === 'history' ? styles['ticket-detail__tab--active'] : '',
                ].join(' ')}
                onClick={() => setActiveTab('history')}
              >
                <HistoryIcon size={12} />
                History {history.length > 0 && `(${history.length})`}
              </button>
            </div>

            {activeTab === 'history' && (
              <div className={styles['ticket-detail__history']}>
                {history.length === 0 && (
                  <p className={styles['ticket-detail__history-empty']}>
                    No changes yet — still in {historyStatusLabel(ticket.status)}.
                  </p>
                )}
                {history.map((h) =>
                  h.type === 'assignee' ? (
                    <div key={h.id} className={styles['ticket-detail__history-item']}>
                      <span
                        className={styles['ticket-detail__history-dot']}
                        style={{ background: h.to_assignee ? avatarAccent(h.to_assignee) : '#8A909B' }}
                      />
                      <span className={styles['ticket-detail__history-text']}>
                        <strong>{h.actor.name}</strong> {assigneeChangeVerb(h)}
                      </span>
                      <span className={styles['ticket-detail__history-time']}>{formatTime(h.created_at)}</span>
                    </div>
                  ) : (
                    <div key={h.id} className={styles['ticket-detail__history-item']}>
                      <span
                        className={styles['ticket-detail__history-dot']}
                        style={{ background: historyStatusAccent(h.to_status, h.resolution) }}
                      />
                      <span className={styles['ticket-detail__history-text']}>
                        <strong>{h.actor.name}</strong> moved{' '}
                        <span className={styles['ticket-detail__history-from']}>
                          {historyStatusLabel(h.from_status)}
                        </span>
                        {' → '}
                        <span
                          className={styles['ticket-detail__history-to']}
                          style={{ color: historyStatusAccent(h.to_status, h.resolution) }}
                        >
                          {historyStatusLabel(h.to_status, h.resolution)}
                        </span>
                      </span>
                      <span className={styles['ticket-detail__history-time']}>{formatTime(h.created_at)}</span>
                    </div>
                  )
                )}
              </div>
            )}

            {activeTab === 'comments' && (
              <div className={styles['ticket-detail__comments']}>
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
                      {/* Comment text is rich-text HTML too (same editor as
                          the description) — sanitize before rendering. */}
                      <div
                        className={styles['ticket-detail__comment-text']}
                        onClick={openImageIfClicked}
                        dangerouslySetInnerHTML={{ __html: sanitizeTicketHtml(c.text) }}
                      />
                    </div>
                  </div>
                ))}

                <div className={styles['ticket-detail__comment-form']}>
                  <TicketDescriptionEditor
                    value={commentDraft}
                    onChange={setCommentDraft}
                    placeholder="Add a comment… paste or drag an image in."
                    disabled={posting}
                    compact
                  />
                  <button
                    type="button"
                    className={styles['ticket-detail__comment-submit']}
                    onClick={submitComment}
                    disabled={posting || isEmptyHtml(commentDraft)}
                  >
                    {posting ? (
                      <Loader2 size={14} className={styles['ticket-board__spin']} />
                    ) : (
                      'Post comment'
                    )}
                  </button>
                </div>
              </div>
            )}
          </div>

          {/* ---- rail: requester/assignee, status, terminal actions ---- */}
          <div className={styles['modal-rail']}>
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
                  <span className={styles['ticket-detail__person-name-text']}>{ticket.owner.name}</span>
                  {owner && <span className={styles['ticket-detail__you']}>you</span>}
                  <span className={`${styles['ticket-role-badge']} ${styles['ticket-role-badge--reporter']}`}>
                    Reporter
                  </span>
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
                    <span className={styles['ticket-detail__person-name-text']}>{ticket.assignee.name}</span>
                    <span className={`${styles['ticket-role-badge']} ${styles['ticket-role-badge--assignee']}`}>
                      Assignee
                    </span>
                  </div>
                ) : (
                  <div className={styles['ticket-detail__person-val']}>
                    <span className={styles['ticket-card__person-avatar--empty']} aria-hidden="true" />
                    <span className={styles['ticket-detail__person-name-text']}>Unassigned</span>
                    <span
                      className={`${styles['ticket-role-badge']} ${styles['ticket-role-badge--unassigned']}`}
                    >
                      Unassigned
                    </span>
                  </div>
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
                  const skipsAhead = !isCurrent && !isSequentialMove(ticket.status, c.status);
                  return (
                    <button
                      key={c.status}
                      type="button"
                      className={[
                        styles['ticket-detail__step'],
                        isCurrent ? styles['ticket-detail__step--current'] : '',
                      ].join(' ')}
                      style={{ ['--step-accent' as string]: c.accent }}
                      disabled={isCurrent || moving || skipsAhead}
                      title={skipsAhead ? SEQUENCE_HINT : undefined}
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
                {(() => {
                  const closeCheck = canDropTicket(ticket, 'done', currentUser.id);
                  const alreadyDone = ticket.status === 'done' && ticket.resolution === 'completed';
                  const alreadyDiscarded = ticket.status === 'done' && ticket.resolution === 'discarded';
                  return (
                    <>
                      <button
                        type="button"
                        className={styles['ticket-detail__done-btn']}
                        disabled={!closeCheck.ok || moving || alreadyDone}
                        title={closeCheck.ok ? undefined : closeCheck.reason}
                        onClick={() => onMove(ticket.id, 'done', 'completed')}
                      >
                        {owner ? <Check size={14} /> : <Lock size={14} />}
                        Mark Done
                      </button>
                      <button
                        type="button"
                        className={styles['ticket-detail__discard-btn']}
                        disabled={!closeCheck.ok || moving || alreadyDiscarded}
                        title={closeCheck.ok ? undefined : closeCheck.reason}
                        onClick={() => onMove(ticket.id, 'done', 'discarded')}
                      >
                        {owner ? <Ban size={14} /> : <Lock size={14} />}
                        Discard
                      </button>
                    </>
                  );
                })()}
              </div>
              {!owner && (
                <p className={styles['ticket-detail__gate-note']} style={{ marginTop: '0.5em' }}>
                  <Lock size={12} /> {OWNER_ONLY_HINT}
                </p>
              )}
              {owner && ticket.status !== 'in_review' && ticket.status !== 'done' && (
                <p className={styles['ticket-detail__gate-note']} style={{ marginTop: '0.5em' }}>
                  {SEQUENCE_HINT}
                </p>
              )}
            </div>
          </div>
        </div>

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
      </div>

      {zoomSrc && (
        <div onClick={(e) => e.stopPropagation()}>
          <Lightbox
            open={!!zoomSrc}
            close={() => setZoomSrc(null)}
            slides={[{ src: zoomSrc }]}
            plugins={[Zoom]}
          />
        </div>
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

  // Fallback bounded height: `flex:1;min-height:0` only produces a real
  // height when an ancestor (the app's .pg-shell) is itself a bounded-
  // height flex container. `height: 100%` is a harmless no-op when that's
  // already true (100% of an already-correct height is the same height),
  // but keeps this component's own columns scrolling internally instead of
  // silently growing with content if it's ever rendered without pg-shell.
  height: 100%;
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
  box-sizing: border-box;
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
// The floating clone rendered inside <DragOverlay> — this is the element
// that actually follows the pointer, giving drag-and-drop its "the card
// itself is moving" feel instead of leaving the source card static.
// `transition: none` is deliberate and important: dnd-kit repositions this
// element every frame via its own transform, and any CSS transition here
// would ease/animate toward each new position instead of snapping to it
// instantly — that's what made the card visibly lag behind the cursor.
// No decorative rotate/scale either, since any extra transform shifts the
// element's visual box relative to its actual (pointer-aligned) position.
.ticket-card--overlay {
  cursor: grabbing;
  box-shadow: $shadow-4;
  opacity: 0.98;
  pointer-events: none;
  transition: none !important;
  transform: none !important;
  &:hover {
    transform: none !important;
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
  display: inline-flex;
  align-items: center;
  gap: 0.3em;
  font-size: 0.78em;
  font-weight: 700;
  letter-spacing: 0.02em;
  padding: 0.22em 0.55em 0.22em 0.45em;
  border-radius: 999px;
  color: var(--priority-accent);
  background: color-mix(in srgb, var(--priority-accent) 12%, transparent);
  border: 1px solid color-mix(in srgb, var(--priority-accent) 28%, transparent);
}
.ticket-card__priority-icon {
  flex: none;
}
.ticket-card__priority--urgent {
  animation: ticket-priority-pulse 1.8s ease-in-out infinite;
}
@keyframes ticket-priority-pulse {
  0%,
  100% {
    box-shadow: 0 0 0 0 color-mix(in srgb, var(--priority-accent) 35%, transparent);
  }
  50% {
    box-shadow: 0 0 0 3px color-mix(in srgb, var(--priority-accent) 0%, transparent);
  }
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
// Still used elsewhere (comment avatars, detail rail) — not card-specific.
.ticket-card__avatar {
  flex-shrink: 0;
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

// ---- card: assignee/requester rows, full name (not just initials) --------
.ticket-card__people {
  display: flex;
  flex-direction: column;
  gap: 0.35em;
  margin-top: 0.15em;
}
.ticket-card__person {
  display: flex;
  align-items: center;
  gap: 0.4em;
  min-width: 0;
}
.ticket-card__person-avatar {
  flex: none;
  width: 1.5em;
  height: 1.5em;
  border-radius: 50%;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  font-size: 0.64em;
  font-weight: 700;
  color: #fff;
}
.ticket-card__person-avatar--owner {
  box-shadow: 0 0 0 1px $line;
}
.ticket-card__person-name {
  flex: 1;
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  font-size: 0.8em;
  color: $ink-2;
  display: flex;
  align-items: center;
  gap: 0.35em;
}
.ticket-card__you {
  flex: none;
  font-size: 0.72em;
  font-weight: 700;
  color: $signal;
  background: $wash;
  padding: 0.05em 0.4em;
  border-radius: 4px;
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


@keyframes ticket-fade-in {
  from { opacity: 0; }
  to { opacity: 1; }
}
@keyframes ticket-modal-in {
  from { opacity: 0; transform: translateY(8px) scale(0.98); }
  to { opacity: 1; transform: none; }
}

// ===========================================================================
// Modal shell — shared by the ticket detail view and the close-confirm
// dialog. Same pattern used elsewhere in the app (see Datasets): a
// full-viewport fixed overlay that centers its content with flexbox, and a
// separate fade-in vs scale-in animation for the scrim and the panel.
// Rendered inline (no portal) — position:fixed + flex centering is enough
// as long as no ancestor sets transform/filter/perspective, which nothing
// in this component tree does.
// ===========================================================================
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
  width: min(1080px, 100%);
  max-height: 88vh;
  display: flex;
  flex-direction: column;
  background: $card;
  border: 1px solid $line;
  border-radius: 18px;
  box-shadow: 0 24px 60px -20px rgba(20, 22, 27, 0.4);
  overflow: hidden;
  animation: ticket-modal-in 0.18s cubic-bezier(0.22, 1, 0.36, 1);
  // Own base size, slightly larger than the page base at very wide
  // viewports — a focused modal reads better a touch bigger than the
  // dense board sitting behind it.
  font-size: 0.8125rem;
  @media (min-width: 1800px) {
    font-size: 1.0625rem;
  }
}
.modal--sm {
  width: min(380px, 100%);
}
.modal-hdr {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.5em;
  padding: 1.1em 1.25em;
  border-bottom: 1px solid $line;
}
.modal-key {
  font-family: $mono;
  font-size: 0.8em;
  font-weight: 700;
  letter-spacing: 0.04em;
  color: $ink-3;
  margin-right: 0.6em;
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

// Two-pane body used by the detail modal: scrollable main content on the
// left, a narrow fixed-width meta rail on the right.
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
  width: 230px;
  border-left: 1px solid $line;
  background: $paper;
  overflow-y: auto;
  padding: 1.1em;
  display: flex;
  flex-direction: column;
  gap: 1.2em;
}

// ===========================================================================
// Detail view content (renders inside .modal-main / .modal-rail above)
// ===========================================================================
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

  p {
    margin: 0 0 0.6em;
    &:last-child {
      margin-bottom: 0;
    }
  }
  ul,
  ol {
    margin: 0 0 0.6em;
    padding-left: 1.4em;
  }
  li {
    margin-bottom: 0.25em;
  }
  strong {
    font-weight: 700;
    color: $ink;
  }
  img {
    display: inline-block;
    width: 84px;
    height: 84px;
    object-fit: cover;
    border-radius: 8px;
    border: 1px solid $line;
    margin: 3px;
    vertical-align: top;
    cursor: zoom-in;
    transition: opacity 0.12s, border-color 0.12s;
    &:hover {
      opacity: 0.85;
      border-color: $ink-3;
    }
  }
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
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 0.3em 0.45em;
  width: 100%;
  max-width: 100%;
  font-size: 0.86em;
  font-weight: 500;
  padding: 0.35em 0.55em;
  border-radius: 14px;
  background: $card;
  border: 1px solid $line;
}
.ticket-detail__person-name-text {
  flex: 1 1 auto;
  min-width: 4em;
  overflow-wrap: anywhere;
  word-break: break-word;
}
.ticket-detail__you {
  flex: none;
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

// ---- main: Comments / History tabs -------------------------------------
.ticket-detail__tabs {
  display: flex;
  align-items: center;
  gap: 0.2em;
  border-bottom: 1px solid $line;
  margin-top: 0.2em;
}
.ticket-detail__tab {
  display: inline-flex;
  align-items: center;
  gap: 0.4em;
  border: 0;
  background: transparent;
  color: $ink-3;
  font-size: 0.84em;
  font-weight: 650;
  padding: 0.6em 0.2em;
  margin-right: 1em;
  cursor: pointer;
  border-bottom: 2px solid transparent;
  transition: color 0.12s, border-color 0.12s;
  &:hover {
    color: $ink;
  }
}
.ticket-detail__tab--active {
  color: $signal;
  border-bottom-color: $signal;
}

// ---- main: history (status-change audit trail) ------------------------------
.ticket-detail__history {
  display: flex;
  flex-direction: column;
  gap: 0.5em;
  margin-top: 0.7em;
}
.ticket-detail__history-item {
  display: flex;
  align-items: baseline;
  gap: 0.55em;
  font-size: 0.86em;
  line-height: 1.5;
}
.ticket-detail__history-dot {
  flex: none;
  align-self: center;
  width: 0.55em;
  height: 0.55em;
  border-radius: 50%;
}
.ticket-detail__history-text {
  flex: 1;
  min-width: 0;
  color: $ink-2;

  strong {
    color: $ink;
    font-weight: 700;
  }
}
.ticket-detail__history-from {
  color: $ink-3;
}
.ticket-detail__history-to {
  font-weight: 650;
}
.ticket-detail__history-time {
  flex: none;
  font-size: 0.82em;
  color: $ink-3;
  white-space: nowrap;
}
.ticket-detail__history-empty {
  font-size: 0.85em;
  color: $ink-3;
  font-style: italic;
  margin: 0;
}

// ---- main: comments ---------------------------------------------------------
.ticket-detail__comments {
  display: flex;
  flex-direction: column;
  gap: 0.9em;
  margin-top: 0.7em;
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
// Comment text is rich-text HTML now (same editor as the description), so
// this needs the same paragraph/list/image handling — not just plain
// pre-wrapped text.
.ticket-detail__comment-text {
  font-size: 0.86em;
  line-height: 1.5;
  color: $ink-2;

  p {
    margin: 0 0 0.5em;
    &:last-child {
      margin-bottom: 0;
    }
  }
  ul,
  ol {
    margin: 0 0 0.5em;
    padding-left: 1.3em;
  }
  li {
    margin-bottom: 0.2em;
  }
  strong {
    font-weight: 700;
    color: $ink;
  }
  img {
    display: inline-block;
    width: 72px;
    height: 72px;
    object-fit: cover;
    border-radius: 7px;
    border: 1px solid $line;
    margin: 2px;
    vertical-align: top;
    cursor: zoom-in;
    transition: opacity 0.12s, border-color 0.12s;
    &:hover {
      opacity: 0.85;
      border-color: $ink-3;
    }
  }
}
.ticket-detail__comment-empty {
  font-size: 0.85em;
  color: $ink-3;
  font-style: italic;
}

// Stacked, not a single row: the composer is a full rich-text editor now
// (paste/drag/insert images, same as the description), so it needs its own
// line — a "Post comment" button sits below it, right-aligned.
.ticket-detail__comment-form {
  display: flex;
  flex-direction: column;
  gap: 0.5em;
}
.ticket-detail__comment-submit {
  align-self: flex-end;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 0.4em;
  padding: 0.5em 0.9em;
  border-radius: 8px;
  border: 1px solid $signal;
  background: $signal;
  color: #fff;
  font-size: 0.82em;
  font-weight: 650;
  cursor: pointer;
  transition: background 0.15s;
  &:hover:not(:disabled) {
    background: $signal-2;
  }
  &:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
}

// ===========================================================================
// Close-ticket confirmation — reuses .modal-overlay / .modal.modal--sm
// above, just with its own inner content.
// ===========================================================================
.close-confirm-body {
  padding: 1.5em 1.5em 1.25em;
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
  gap: 0.3em;
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

@media (max-width: 820px) {
  .ticket-board__header { padding: 20px 18px 16px; flex-direction: column; align-items: flex-start; gap: 10px; }
  .ticket-board__toolbar { padding: 12px 18px; }
  .ticket-board__columns { padding: 0 18px 20px; grid-template-columns: 1fr; }
  .modal-overlay { padding: 12px; }
  .modal { width: 100%; max-height: 94vh; border-radius: 14px; }
  .modal-body { flex-direction: column; overflow-y: auto; }
  .modal-rail { width: auto; border-left: 0; border-top: 1px solid $line; }
}

@media (prefers-reduced-motion: reduce) {
  .modal-overlay,
  .modal,
  .ticket-board__spin,
  .ticket-card__spin {
    animation: none;
  }
}

// ===========================================================================
// Advanced filters — a toggle button in the toolbar that opens a floating
// dropdown panel (assignee / reporter / created date range), all combining
// via AND with each other and with the search box + priority pills. Pure
// client-side filtering over tickets already loaded — no separate request.
// ===========================================================================
.adv-filters {
  position: relative;
}
.adv-filters__toggle {
  display: inline-flex;
  align-items: center;
  gap: 0.4em;
  padding: 0.5em 0.75em;
  border-radius: 999px;
  border: 1px solid $line;
  background: $card;
  color: $ink-2;
  font-size: 0.78em;
  font-weight: 650;
  cursor: pointer;
  transition: border-color 0.15s, color 0.15s, background 0.15s;
  &:hover {
    border-color: $ink-3;
    color: $ink;
  }
}
.adv-filters__toggle--open {
  border-color: $signal;
  color: $signal;
  background: $wash;
}
.adv-filters__badge {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 1.3em;
  height: 1.3em;
  padding: 0 0.35em;
  border-radius: 999px;
  background: $signal;
  color: #fff;
  font-size: 0.72em;
  font-weight: 700;
}
.adv-filters__chevron {
  transition: transform 0.15s;
  .adv-filters__toggle--open & {
    transform: rotate(180deg);
  }
}
.adv-filters__panel {
  position: absolute;
  top: calc(100% + 8px);
  right: 0;
  z-index: 30;
  width: 300px;
  max-width: 90vw;
  background: $card;
  border: 1px solid $line;
  border-radius: 12px;
  box-shadow: 0 14px 32px -12px rgba(20, 22, 27, 0.35);
  padding: 0.9em;
  display: flex;
  flex-direction: column;
  gap: 0.75em;
  animation: ticket-modal-in 0.14s ease both;
}
.adv-filters__field {
  display: flex;
  flex-direction: column;
  gap: 0.35em;
}
.adv-filters__label {
  font-size: 0.72em;
  font-weight: 650;
  text-transform: uppercase;
  letter-spacing: 0.04em;
  color: $ink-3;
}
.adv-filters__date {
  width: 100%;
  border: 1px solid $line;
  border-radius: 8px;
  background: $card;
  color: $ink;
  font-size: 0.85em;
  font-family: inherit;
  padding: 0.55em 0.6em;
  outline: 0;
  transition: border-color 0.15s, box-shadow 0.15s;
  &:focus {
    border-color: $signal;
    box-shadow: 0 0 0 3px $wash;
  }
}
.adv-filters__clear {
  align-self: flex-start;
  display: inline-flex;
  align-items: center;
  gap: 0.35em;
  border: 0;
  background: transparent;
  color: $ink-3;
  font-size: 0.78em;
  font-weight: 600;
  cursor: pointer;
  padding: 0.2em 0;
  margin-top: 0.1em;
  &:hover:not(:disabled) {
    color: $danger;
  }
  &:disabled {
    opacity: 0.4;
    cursor: not-allowed;
  }
}

// ===========================================================================
// Role badges — small pills next to a name marking them as the Reporter or
// Assignee, and an explicit "Unassigned" badge in place of a name when
// there's no assignee, instead of just omitting the row/relying on hover.
// Shared by both the card (.ticket-card__person) and the detail rail
// (.ticket-detail__person-val).
// ===========================================================================
.ticket-role-badge {
  flex: none;
  display: inline-flex;
  align-items: center;
  font-size: 0.62em;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.03em;
  padding: 0.2em 0.5em;
  border-radius: 999px;
  white-space: nowrap;
}
.ticket-role-badge--reporter {
  background: $wash;
  color: $signal;
}
.ticket-role-badge--assignee {
  background: $sky-ink-wash;
  color: $sky-ink;
}
.ticket-role-badge--unassigned {
  background: $ink-wash;
  color: $ink-3;
  border: 1px dashed $line;
}

// Placeholder avatar circle shown instead of initials when there's no
// assignee — a plain dashed ring rather than a solid color, so it reads as
// "empty" at a glance.
.ticket-card__person-avatar--empty {
  flex: none;
  width: 1.5em;
  height: 1.5em;
  border-radius: 50%;
  border: 1.5px dashed $line;
  background: $paper;
}
