//Ticketmeta.ts
import DOMPurify from 'dompurify';
import { Flame, ArrowUp, Minus, ArrowDown } from 'lucide-react';
import type { TicketStatus, TicketPriority, Ticket, TicketUser } from '../../types/tickets';

// ─────────────────────────────────────────────────────────────────────────
// Sanitizes rich-text HTML (description / comment text) before it's ever
// injected via dangerouslySetInnerHTML. `ADD_ATTR: ['style']` is load-
// bearing, not decorative: a resized image's width lives entirely in
// `style="width: …px"` on the <img> tag (see tiptapResizableImage.tsx) — the
// live editor never goes through this function at all (its resize preview
// is a plain React `style` prop, unaffected by DOMPurify), so a resize that
// looks correct while creating/editing but reverts to full size once you
// view the saved ticket means the `style` attribute got stripped right
// here. Forcing it into the allow-list explicitly, rather than trusting
// DOMPurify's default config, is what actually fixes that — don't remove
// this option even if it looks redundant against whatever DOMPurify's
// current default happens to allow.
export const sanitizeTicketHtml = (html: string | null | undefined): string =>
  DOMPurify.sanitize(html ?? '', { ADD_ATTR: ['style'] });

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
 *  plain falsy/blank check isn't enough — strip tags and check what's left. */
export const isEmptyHtml = (html?: string | null): boolean => {
  if (!html) return true;
  return html.replace(/<[^>]*>/g, '').trim().length === 0;

};
















//Ticketdetialsidebar.tsx
import { useState } from 'react';
import { X, Pencil, Trash2, Loader2, Lock, Check, Ban, History as HistoryIcon } from 'lucide-react';
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
    </div>
  );
}
