//Ticketdescriptioneditor.tsx
import { useEffect, useRef, type ReactNode } from 'react';
import { useEditor, EditorContent, type Editor } from '@tiptap/react';
import StarterKit from '@tiptap/starter-kit';
import Image from '@tiptap/extension-image';
import Placeholder from '@tiptap/extension-placeholder';
import {
  Bold as BoldIcon,
  Italic as ItalicIcon,
  List as ListIcon,
  ListOrdered,
  ImagePlus,
} from 'lucide-react';
import { useToast } from '../common/Toast';
import styles from './TicketDescriptionEditor.module.scss';

interface TicketDescriptionEditorProps {
  /** HTML content — same shape TipTap emits and consumes. Any images the
   *  user has added are embedded directly as base64 `data:` URIs inside
   *  this string — there's no separate upload step or hosted URL. */
  value: string;
  onChange: (html: string) => void;
  placeholder?: string;
  disabled?: boolean;
  /** Smaller toolbar/padding/max-height — used for the comment composer,
   *  which needs the same paste/drag/insert-image capability as the
   *  description but shouldn't dominate the layout the way a full
   *  description field does. */
  compact?: boolean;
}

// Base64-encoded images run roughly a third larger than the original file,
// so this cap is intentionally stricter than a typical raw-upload limit —
// it exists purely to keep the description field (and the request/response
// bodies carrying it) from ballooning, not to protect an upload endpoint.
const MAX_IMAGE_BYTES = 2 * 1024 * 1024; // 2MB

const readAsDataUrl = (file: File) =>
  new Promise<string>((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = () => resolve(reader.result as string);
    reader.onerror = () => reject(reader.error);
    reader.readAsDataURL(file);
  });

/**
 * A Jira-style rich-text description field: type normally, and images can
 * be pasted straight from the clipboard, dragged in from the desktop, or
 * inserted via the toolbar button — there's no separate "attachments" field
 * and no upload API call. Every image is embedded directly as a base64
 * `data:` URI inside the description's own HTML, so image + text are a
 * single self-contained field with nothing else to fetch or host.
 */
export default function TicketDescriptionEditor({
  value,
  onChange,
  placeholder = 'Context, acceptance criteria, links… paste or drag an image in.',
  disabled = false,
  compact = false,
}: TicketDescriptionEditorProps) {
  const toast = useToast();
  const fileInputRef = useRef<HTMLInputElement | null>(null);

  const editor = useEditor({
    extensions: [
      StarterKit.configure({ heading: false }),
      // allowBase64 is the whole trick here — the stock Image extension
      // otherwise rejects data: URIs and expects a hosted src.
      Image.configure({ inline: false, allowBase64: true }),
      Placeholder.configure({ placeholder }),
    ],
    content: value,
    editable: !disabled,
    onUpdate: ({ editor: e }) => onChange(e.getHTML()),
    editorProps: {
      attributes: {
        class: styles['editor-body'],
      },
      handlePaste: (_view, event) => {
        const files = Array.from(event.clipboardData?.items ?? [])
          .map((item) => item.getAsFile())
          .filter((f): f is File => !!f && f.type.startsWith('image/'));
        if (files.length === 0) return false;
        event.preventDefault();
        files.forEach((f) => insertImage(editor, f, toast));
        return true;
      },
      handleDrop: (_view, event) => {
        const files = Array.from(event.dataTransfer?.files ?? []).filter((f) =>
          f.type.startsWith('image/')
        );
        if (files.length === 0) return false;
        event.preventDefault();
        files.forEach((f) => insertImage(editor, f, toast));
        return true;
      },
    },
  });

  // Keep the editor's content in sync with `value` whenever it changes from
  // outside — e.g. switching "create" → "edit" with a different initial
  // ticket, or the parent clearing the draft after a successful post/submit.
  // The `value !== editor.getHTML()` guard is what keeps this safe to run
  // on every value change instead of just once at mount: on every keystroke
  // the parent's `value` prop becomes exactly what the editor already has,
  // so this is a no-op then — it only actually pushes content in when the
  // change came from outside the editor itself.
  useEffect(() => {
    if (!editor) return;
    if (value !== editor.getHTML()) {
      editor.commands.setContent(value || '', false);
    }
  }, [editor, value]);

  useEffect(() => {
    editor?.setEditable(!disabled);
  }, [editor, disabled]);

  if (!editor) return null;

  const pickImage = () => fileInputRef.current?.click();

  return (
    <div
      className={`${styles['editor']} ${disabled ? styles['editor--disabled'] : ''} ${compact ? styles['editor--compact'] : ''}`}
    >
      <div className={styles['toolbar']}>
        <ToolbarButton
          active={editor.isActive('bold')}
          onClick={() => editor.chain().focus().toggleBold().run()}
          label="Bold"
        >
          <BoldIcon size={14} />
        </ToolbarButton>
        <ToolbarButton
          active={editor.isActive('italic')}
          onClick={() => editor.chain().focus().toggleItalic().run()}
          label="Italic"
        >
          <ItalicIcon size={14} />
        </ToolbarButton>
        <span className={styles['toolbar-sep']} />
        <ToolbarButton
          active={editor.isActive('bulletList')}
          onClick={() => editor.chain().focus().toggleBulletList().run()}
          label="Bullet list"
        >
          <ListIcon size={14} />
        </ToolbarButton>
        <ToolbarButton
          active={editor.isActive('orderedList')}
          onClick={() => editor.chain().focus().toggleOrderedList().run()}
          label="Numbered list"
        >
          <ListOrdered size={14} />
        </ToolbarButton>
        <span className={styles['toolbar-sep']} />
        <ToolbarButton onClick={pickImage} label="Insert image">
          <ImagePlus size={14} />
        </ToolbarButton>
        <input
          ref={fileInputRef}
          type="file"
          accept="image/*"
          hidden
          onChange={(e) => {
            const file = e.target.files?.[0];
            if (file) insertImage(editor, file, toast);
            e.target.value = '';
          }}
        />
      </div>
      <EditorContent editor={editor} className={styles['editor-scroll']} />
    </div>
  );
}

function ToolbarButton({
  children,
  onClick,
  active,
  label,
}: {
  children: ReactNode;
  onClick: () => void;
  active?: boolean;
  label: string;
}) {
  return (
    <button
      type="button"
      className={`${styles['toolbar-btn']} ${active ? styles['toolbar-btn--active'] : ''}`}
      onMouseDown={(e) => e.preventDefault()} // keep editor selection/focus intact
      onClick={onClick}
      aria-label={label}
      title={label}
    >
      {children}
    </button>
  );
}

// Reads the file straight to a base64 data: URI (no network round-trip) and
// inserts it as an <img> at the cursor. Purely local — nothing to upload,
// nothing that can fail on the network, just FileReader.
function insertImage(editor: Editor | null, file: File, toast: ReturnType<typeof useToast>) {
  if (!editor) return;
  if (file.size > MAX_IMAGE_BYTES) {
    toast.error(`"${file.name}" is over 2MB — pick a smaller image.`);
    return;
  }
  readAsDataUrl(file)
    .then((dataUrl) => {
      editor.chain().focus().setImage({ src: dataUrl, alt: file.name }).run();
    })
    .catch(() => {
      toast.error(`Couldn't read "${file.name}".`);
    });
}















//Ticketdetailsidebar.tsx
import { useState } from 'react';
import DOMPurify from 'dompurify';
import { X, Pencil, Trash2, Loader2, Lock, Check, Ban } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import { addTicketComment } from '../../store/slices/ticketsSlice';
import type { Ticket, TicketStatus, TicketResolution, TicketUser } from '../../types/tickets';
import ConfirmDialog from '../common/ConfirmDialog';
import { useToast } from '../common/Toast';
import TicketDescriptionEditor from './TicketDescriptionEditor';
import {
  COLUMNS,
  PRIORITY_META,
  isOwner,
  isSequentialMove,
  canDropTicket,
  isEmptyHtml,
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

  if (!ticket) return null; // e.g. deleted from another tab/session

  const owner = isOwner(ticket, currentUser.id);
  const priority = PRIORITY_META[ticket.priority];
  const posting = commentingId === ticket.id;
  const comments = ticket.comments ?? [];

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
              className={styles['ticket-card__priority']}
              style={{ ['--priority-accent' as string]: priority.accent }}
            >
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
                dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(ticket.description ?? '') }}
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
                      {/* Comment text is rich-text HTML too (same editor as
                          the description) — sanitize before rendering. */}
                      <div
                        className={styles['ticket-detail__comment-text']}
                        dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(c.text) }}
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
            </div>
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











//Ticketmeta.ts
import type { TicketStatus, TicketPriority, Ticket, TicketUser } from '../../types/tickets';

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














//Ticketcard.ts
import { useDraggable } from '@dnd-kit/core';
import { Loader2 } from 'lucide-react';
import type { Ticket, TicketUser } from '../../types/tickets';
import { PRIORITY_META, isOwner, initials, avatarAccent } from './ticketMeta';
import styles from './TicketBoard.module.scss';

interface TicketCardProps {
  ticket: Ticket;
  currentUser: TicketUser;
  moving?: boolean;
  onOpen: (ticket: Ticket) => void;
  /** True only for the clone rendered inside <DragOverlay> — static, no
   *  drag hook of its own. */
  overlay?: boolean;
  /** Overlay mode only: the exact width (px) of the card that's actually
   *  being dragged, captured at drag-start. Without this the clone sizes
   *  itself independently and can end up narrower/wider than the real
   *  card, which is what makes a dragged card look offset from the
   *  cursor instead of feeling like the card itself is being carried. */
  overlayWidth?: number;
}

// Move actions live only in the detail view's status stepper and Done/Discard
// buttons (plus drag-and-drop) — there's no per-card "⋯" quick-move menu.
export default function TicketCard({
  ticket,
  currentUser,
  moving = false,
  onOpen,
  overlay = false,
  overlayWidth,
}: TicketCardProps) {
  const owner = isOwner(ticket, currentUser.id);
  const priority = PRIORITY_META[ticket.priority];

  // The real drag-and-drop wiring. Listeners go on the card's root element;
  // dnd-kit's activation distance means an ordinary click (open the card)
  // still fires normally — a drag only "activates" once the pointer has
  // moved a few pixels. No live `transform` is applied here — the source
  // card stays put (just dimmed via isDragging); the <DragOverlay> clone in
  // TicketBoard is what actually follows the cursor.
  const { attributes, listeners, setNodeRef, isDragging } = useDraggable({
    id: ticket.id,
    disabled: overlay || moving,
  });

  return (
    <article
      ref={overlay ? undefined : setNodeRef}
      {...(overlay ? {} : attributes)}
      {...(overlay ? {} : listeners)}
      className={[
        styles['ticket-card'],
        moving ? styles['ticket-card--moving'] : '',
        isDragging ? styles['ticket-card--dragging'] : '',
        overlay ? styles['ticket-card--overlay'] : '',
        ticket.resolution === 'discarded' ? styles['ticket-card--discarded'] : '',
      ].join(' ')}
      style={{
        ['--priority-accent' as string]: priority.accent,
        ...(overlay
          ? { width: overlayWidth, flexShrink: 0 }
          : { touchAction: 'none' }),
      }}
      onClick={() => !overlay && onOpen(ticket)}
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
          {moving && <Loader2 size={14} className={styles['ticket-card__spin']} />}
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

      {ticket.resolution && (
        <footer className={styles['ticket-card__foot']}>
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
        </footer>
      )}

      <div className={styles['ticket-card__people']}>
        {ticket.assignee && (
          <div className={styles['ticket-card__person']} title={`Assignee: ${ticket.assignee.name}`}>
            <span
              className={styles['ticket-card__person-avatar']}
              style={{ background: avatarAccent(ticket.assignee) }}
            >
              {initials(ticket.assignee)}
            </span>
            <span className={styles['ticket-card__person-name']}>{ticket.assignee.name}</span>
          </div>
        )}
        <div
          className={styles['ticket-card__person']}
          title={`Requester: ${ticket.owner.name}${owner ? ' (you)' : ''}`}
        >
          <span
            className={`${styles['ticket-card__person-avatar']} ${styles['ticket-card__person-avatar--owner']}`}
            style={{ background: avatarAccent(ticket.owner) }}
          >
            {initials(ticket.owner)}
          </span>
          <span className={styles['ticket-card__person-name']}>
            {ticket.owner.name}
            {owner && <span className={styles['ticket-card__you']}>you</span>}
          </span>
        </div>
      </div>
    </article>
  );
}

















//Ticketadvancedfilters.tsx
import { useEffect, useRef } from 'react';
import { X, Filter as FilterIcon, ChevronDown } from 'lucide-react';
import TicketSelect, { type TicketSelectOption } from './TicketSelect';
import styles from './TicketBoard.module.scss';

export interface AdvancedFilterState {
  assigneeId: string; // '' = all, 'unassigned' = no assignee, else a user id
  reporterId: string; // '' = all, else a user id
  dateFrom: string; // '' or 'YYYY-MM-DD'
  dateTo: string; // '' or 'YYYY-MM-DD'
}

export const EMPTY_ADVANCED_FILTERS: AdvancedFilterState = {
  assigneeId: '',
  reporterId: '',
  dateFrom: '',
  dateTo: '',
};

export const countActiveFilters = (f: AdvancedFilterState): number =>
  [f.assigneeId, f.reporterId, f.dateFrom, f.dateTo].filter(Boolean).length;

interface TicketAdvancedFiltersProps {
  open: boolean;
  onToggleOpen: () => void;
  value: AdvancedFilterState;
  onChange: (next: AdvancedFilterState) => void;
  assigneeOptions: TicketSelectOption[];
  reporterOptions: TicketSelectOption[];
}

/**
 * A collapsible advanced-filter panel — created date range, assignee, and
 * reporter — that all combine together (AND) with each other and with the
 * search box / priority pills already in the toolbar. Everything here
 * filters the ticket list already loaded in memory; no separate API call,
 * since the board already has every ticket's full data client-side.
 */
export default function TicketAdvancedFilters({
  open,
  onToggleOpen,
  value,
  onChange,
  assigneeOptions,
  reporterOptions,
}: TicketAdvancedFiltersProps) {
  const rootRef = useRef<HTMLDivElement | null>(null);
  const activeCount = countActiveFilters(value);

  const set = (patch: Partial<AdvancedFilterState>) => onChange({ ...value, ...patch });

  useEffect(() => {
    if (!open) return;
    const onDoc = (e: MouseEvent) => {
      if (rootRef.current && !rootRef.current.contains(e.target as Node)) onToggleOpen();
    };
    document.addEventListener('mousedown', onDoc);
    return () => document.removeEventListener('mousedown', onDoc);
  }, [open, onToggleOpen]);

  return (
    <div className={styles['adv-filters']} ref={rootRef}>
      <button
        type="button"
        className={`${styles['adv-filters__toggle']} ${open ? styles['adv-filters__toggle--open'] : ''}`}
        onClick={onToggleOpen}
      >
        <FilterIcon size={13} />
        Advanced filters
        {activeCount > 0 && <span className={styles['adv-filters__badge']}>{activeCount}</span>}
        <ChevronDown size={13} className={styles['adv-filters__chevron']} />
      </button>

      {open && (
        <div className={styles['adv-filters__panel']}>
          <div className={styles['adv-filters__field']}>
            <span className={styles['adv-filters__label']}>Assignee</span>
            <TicketSelect
              value={value.assigneeId}
              options={assigneeOptions}
              placeholder="All assignees"
              onChange={(v) => set({ assigneeId: v })}
              aria-label="Filter by assignee"
            />
          </div>

          <div className={styles['adv-filters__field']}>
            <span className={styles['adv-filters__label']}>Reporter</span>
            <TicketSelect
              value={value.reporterId}
              options={reporterOptions}
              placeholder="All reporters"
              onChange={(v) => set({ reporterId: v })}
              aria-label="Filter by reporter"
            />
          </div>

          <div className={styles['adv-filters__field']}>
            <span className={styles['adv-filters__label']}>Created from</span>
            <input
              type="date"
              className={styles['adv-filters__date']}
              value={value.dateFrom}
              max={value.dateTo || undefined}
              onChange={(e) => set({ dateFrom: e.target.value })}
              aria-label="Created on or after"
            />
          </div>

          <div className={styles['adv-filters__field']}>
            <span className={styles['adv-filters__label']}>Created to</span>
            <input
              type="date"
              className={styles['adv-filters__date']}
              value={value.dateTo}
              min={value.dateFrom || undefined}
              onChange={(e) => set({ dateTo: e.target.value })}
              aria-label="Created on or before"
            />
          </div>

          <button
            type="button"
            className={styles['adv-filters__clear']}
            onClick={() => onChange(EMPTY_ADVANCED_FILTERS)}
            disabled={activeCount === 0}
          >
            <X size={12} />
            Clear filters
          </button>
        </div>
      )}
    </div>
  );
}














//Ticketboard.tsx
import { useEffect, useMemo, useState } from 'react';
import {
  DndContext,
  DragOverlay,
  PointerSensor,
  useSensor,
  useSensors,
  closestCenter,
  MeasuringStrategy,
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
import { COLUMNS, PRIORITY_META, canDropTicket, toTicketUser, initials, avatarAccent } from './ticketMeta';
import TicketCard from './TicketCard';
import TicketColumn from './TicketColumn';
import TicketAdvancedFilters, {
  EMPTY_ADVANCED_FILTERS,
  type AdvancedFilterState,
} from './TicketAdvancedFilters';
import CreateTicketDrawer, { type TicketSubmitPayload } from './CreateTicketDrawer';
import TicketDetailSidebar from './TicketDetailSidebar';
import TicketCloseConfirm from './TicketCloseConfirm';
import styles from './TicketBoard.module.scss';

// Columns are static (no layout shift) during a drag, so measuring droppable
// rects once at drag-start — instead of dnd-kit's default of continuously
// re-measuring every frame while dragging — cuts out unnecessary work on
// each pointer move and keeps the drag feeling smooth rather than janky.
const MEASURING = { droppable: { strategy: MeasuringStrategy.BeforeDragging } };

// ─────────────────────────────────────────────────────────────────────────
// Self-contained: no props. Fetches its own tickets and its own assignee
// roster. Current-user identity comes from the app's existing authSlice
// (state.auth.user, an SsoLoginResult) — this board never dispatches a
// login itself, it just reads whatever the app's SSO flow already put
// there and adapts it to the minimal { id, name } shape this feature needs
// via `toTicketUser` (see ticketMeta.ts).
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
  // Only the id is kept here — TicketDetailSidebar selects the live ticket
  // straight out of the store by this id, so it can never go stale (e.g.
  // after posting a comment) the way a locally-cached ticket snapshot could.
  const [detailId, setDetailId] = useState<string | null>(null);

  const [query, setQuery] = useState('');
  const [priorityFilter, setPriorityFilter] = useState<TicketPriority | 'all'>('all');
  const [advFiltersOpen, setAdvFiltersOpen] = useState(false);
  const [advFilters, setAdvFilters] = useState<AdvancedFilterState>(EMPTY_ADVANCED_FILTERS);

  // The ticket id currently being dragged, driving both the floating
  // DragOverlay clone and each column's locked/over highlighting.
  const [activeId, setActiveId] = useState<string | null>(null);

  // The dragged card's actual on-screen width, captured the instant the
  // drag starts. Without this, the floating overlay clone (see DragOverlay
  // below) sizes itself independently and can end up a different width than
  // the real card — which is exactly what made the drag "ghost" look offset
  // from the cursor instead of feeling like the card itself. Height isn't
  // captured separately: same content + same styles at the same width
  // naturally produces the same height.
  const [activeWidth, setActiveWidth] = useState<number | undefined>(undefined);

  // Set whenever a move would land a ticket in the terminal column; renders
  // TicketCloseConfirm instead of moving immediately. Cleared on choose/cancel.
  const [pendingClose, setPendingClose] = useState<{ id: string; key: string; title: string } | null>(
    null
  );

  // A short activation distance keeps ordinary clicks (opening a card)
  // working normally — a drag only "activates" once the pointer has moved
  // a few pixels past its starting point.
  const sensors = useSensors(useSensor(PointerSensor, { activationConstraint: { distance: 6 } }));

  useEffect(() => {
    if (status === 'idle') dispatch(fetchTickets());
  }, [status, dispatch]);

  useEffect(() => {
    if (membersStatus === 'idle') dispatch(fetchTeamMembers());
  }, [membersStatus, dispatch]);

  const activeTicket = activeId ? items.find((t) => t.id === activeId) ?? null : null;

  const filtered = useMemo(() => {
    const q = query.trim().toLowerCase();
    // Inclusive day-range: "from" starts at local midnight, "to" runs
    // through the end of that day, so picking the same date for both
    // includes everything created on that day.
    const from = advFilters.dateFrom ? new Date(`${advFilters.dateFrom}T00:00:00`) : null;
    const to = advFilters.dateTo ? new Date(`${advFilters.dateTo}T23:59:59.999`) : null;

    return items.filter((t) => {
      if (priorityFilter !== 'all' && t.priority !== priorityFilter) return false;

      if (advFilters.assigneeId === 'unassigned' && t.assignee) return false;
      if (
        advFilters.assigneeId &&
        advFilters.assigneeId !== 'unassigned' &&
        t.assignee?.id !== advFilters.assigneeId
      )
        return false;

      if (advFilters.reporterId && t.owner.id !== advFilters.reporterId) return false;

      if (from || to) {
        const created = new Date(t.created_at);
        if (from && created < from) return false;
        if (to && created > to) return false;
      }

      if (!q) return true;
      return (
        t.title.toLowerCase().includes(q) ||
        t.key.toLowerCase().includes(q) ||
        (t.labels ?? []).some((l) => l.toLowerCase().includes(q))
      );
    });
  }, [items, query, priorityFilter, advFilters]);

  // Assignee options: the known team roster plus an explicit "Unassigned"
  // choice. Reporter options: every distinct requester actually present on
  // a ticket right now — derived from `items`, not a separate request,
  // since the board already has every ticket's full data loaded.
  const assigneeFilterOptions = useMemo(
    () => [
      { value: 'unassigned', label: 'Unassigned' },
      ...members.map((m) => ({
        value: m.id,
        label: m.name,
        accent: avatarAccent(m),
        glyph: initials(m),
      })),
    ],
    [members]
  );

  const reporterFilterOptions = useMemo(() => {
    const seen = new Map<string, (typeof items)[number]['owner']>();
    for (const t of items) {
      if (!seen.has(t.owner.id)) seen.set(t.owner.id, t.owner);
    }
    return Array.from(seen.values())
      .sort((a, b) => a.name.localeCompare(b.name))
      .map((u) => ({ value: u.id, label: u.name, accent: avatarAccent(u), glyph: initials(u) }));
  }, [items]);

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
    if (!ticket || (ticket.status === to && !resolution)) return;
    const check = canDropTicket(ticket, to, currentUser.id);
    if (!check.ok) {
      toast.warning(check.reason ?? "That move isn't allowed.");
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

  // Entry point for every move request (drag, detail-view stepper/close
  // buttons). Moves into the terminal column always pause for an explicit
  // Done/Discard choice instead of executing immediately. Every other move
  // is checked against the sequence + owner rules before it's attempted.
  const requestMove = (id: string, to: TicketStatus) => {
    const ticket = items.find((t) => t.id === id);
    if (!ticket) return;
    const check = canDropTicket(ticket, to, currentUser.id);
    if (!check.ok) {
      toast.warning(check.reason ?? "That move isn't allowed.");
      return;
    }
    if (to === 'done') {
      setPendingClose({ id, key: ticket.key, title: ticket.title });
      return;
    }
    runMove(id, to);
  };

  const handleDragStart = (event: DragStartEvent) => {
    setActiveId(String(event.active.id));
    setActiveWidth(event.active.rect.current.initial?.width);
  };

  const handleDragEnd = (event: DragEndEvent) => {
    const { active, over } = event;
    setActiveId(null);
    setActiveWidth(undefined);
    if (!over) return;
    const to = over.id as TicketStatus;
    const ticket = items.find((t) => t.id === active.id);
    if (!ticket || ticket.status === to) return;
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
        setDetailId(null);
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

          <TicketAdvancedFilters
            open={advFiltersOpen}
            onToggleOpen={() => setAdvFiltersOpen((v) => !v)}
            value={advFilters}
            onChange={setAdvFilters}
            assigneeOptions={assigneeFilterOptions}
            reporterOptions={reporterFilterOptions}
          />

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
          measuring={MEASURING}
          onDragStart={handleDragStart}
          onDragEnd={handleDragEnd}
          onDragCancel={() => {
            setActiveId(null);
            setActiveWidth(undefined);
          }}
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
                onOpen={(t) => setDetailId(t.id)}
              />
            ))}
          </div>

          {/* The floating clone that actually follows the cursor — sized to
              exactly match the source card's width (see handleDragStart) so
              it stays pixel-aligned under the pointer instead of looking
              offset. Rendered directly as DragOverlay's child (no extra
              wrapper div) so there's no nested box-sizing ambiguity between
              what dnd-kit measured and what's actually drawn. */}
          <DragOverlay dropAnimation={{ duration: 180, easing: 'cubic-bezier(0.18, 0.67, 0.6, 1.22)' }}>
            {activeTicket ? (
              <TicketCard
                ticket={activeTicket}
                currentUser={currentUser}
                onOpen={() => {}}
                overlay
                overlayWidth={activeWidth}
              />
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

      {detailId && (
        <TicketDetailSidebar
          ticketId={detailId}
          currentUser={currentUser}
          moving={movingIds.includes(detailId)}
          deleting={deletingId === detailId}
          onClose={() => setDetailId(null)}
          onMove={requestMove}
          onEdit={(t) => {
            setDetailId(null);
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
// Still used elsewhere (comment avatars, detail rail) — not card-specific.
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
  width: min(980px, 100%);
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
    display: block;
    max-width: 100%;
    max-height: 420px;
    height: auto;
    border-radius: 8px;
    border: 1px solid $line;
    margin: 0.5em 0;
    object-fit: contain;
    cursor: zoom-in;
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
    display: block;
    max-width: 100%;
    max-height: 280px;
    height: auto;
    border-radius: 8px;
    border: 1px solid $line;
    margin: 0.4em 0;
    object-fit: contain;
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

















# Requirements / Tickets — API Specification

Every endpoint the frontend calls, with sample request and response bodies. The
frontend's typed API layer (`api/tickets.ts`, `api/users.ts`) is written against
exactly these shapes — if the backend needs to deviate, update the corresponding
`.ts` file to match rather than the other way round.

**Base URL**: all paths below are relative to whatever base URL the shared
`axiosInstance` is already configured with elsewhere in the app (e.g.
`/api`). Auth (session cookie or `Authorization` header) is assumed to be
attached by that same shared instance — none of these endpoints do their own
auth handshake.

**Common types** referenced throughout:

```ts
type TicketStatus = 'todo' | 'in_progress' | 'in_review' | 'done';
type TicketResolution = 'completed' | 'discarded'; // only set when status = 'done'
type TicketPriority = 'low' | 'medium' | 'high' | 'urgent';

interface TicketUser {
  id: string;
  name: string;
}

interface TicketComment {
  id: string;
  author: TicketUser;
  text: string;
  created_at: string; // ISO 8601
}

interface Ticket {
  id: string;
  key: string;                 // e.g. "REQ-42", server-assigned, unique, sequential
  title: string;
  // Rich-text HTML from the description editor (paragraphs, bold/italic,
  // lists) with any pasted/dropped/inserted images embedded inline as
  // base64 <img src="data:image/..."> tags. There is no attachments field
  // and no image-hosting endpoint — image + text are one self-contained
  // field. Sanitize before rendering — see the note under §2/§3.
  description?: string;
  status: TicketStatus;
  resolution?: TicketResolution | null;
  priority: TicketPriority;
  owner: TicketUser;           // the requester — only they may close the ticket
  assignee?: TicketUser | null;
  labels?: string[];
  comments?: TicketComment[];
  created_at: string;          // ISO 8601
  updated_at: string;          // ISO 8601
}
```

**Error format** (assumed): every non-2xx response returns a JSON body with a
human-readable `message`, which the frontend surfaces directly in toasts:

```json
{ "message": "You do not have permission to close this ticket." }
```

If your API's actual error envelope differs (e.g. `{"error": {...}}` or a
`detail` field), tell me and I'll adjust the frontend's error-unwrapping
instead of asking you to change the backend to match this doc.

---

## ⚠️ Important: `description` and comment `text` can be large

Because images are embedded as base64 rather than uploaded separately and
referenced by URL, a description or comment with a few screenshots in it can
easily run into the hundreds of KB to low MB — base64 encoding alone adds
~33% over the original file size on top of whatever images the user pastes
in. This applies equally to `Ticket.description` (§2/§3) and
`TicketComment.text` (§6), since both go through the same rich-text editor
with the same paste/drag/insert-image capability. Practical implications for
the backend:

- **Request body size limits** — make sure whatever sits in front of these
  endpoints (reverse proxy, framework body-parser, API gateway) allows a
  request body of at least a few MB, not just a typical small-JSON default.
- **Database column types** — both `description` and each comment's `text`
  need a `TEXT`/`LONGTEXT` (or equivalent) column, not a short `VARCHAR`.
- **List endpoint (§1) response size** — if descriptions and comments
  routinely carry images, `GET /tickets` returning every ticket's *full*
  description and *every* comment (images and all) could get expensive as
  the ticket count grows. If that becomes a real problem, the cleanest fix
  is having §1 return a truncated/stripped description and omit comments
  entirely, with the frontend fetching the full ticket (comments included)
  lazily when its detail view actually opens — that would need a
  `GET /tickets/:id` endpoint added and a small frontend change to call it
  on demand. Flag it if you want that; not needed at current expected scale.

The frontend caps individual images at 2MB client-side (stricter than a
typical raw-upload limit specifically because of the base64 size penalty),
but that's a soft UX guard, not a substitute for the request-size headroom
above.

---

## 1. List tickets

`GET /tickets`

Returns every ticket the current user can see. The frontend currently
fetches the full list once and does search/priority filtering client-side —
if the dataset grows large, this endpoint can add `?search=` / `?priority=`
/ pagination params later without any other contract change.

**Request**: no body, no query params required.

**Response 200**

```json
{
  "tickets": [
    {
      "id": "t1",
      "key": "REQ-1",
      "title": "Add dark-mode toggle to the settings page",
      "description": "<p>Persist the choice per-user and respect <strong>prefers-color-scheme</strong> on first load.</p>",
      "status": "todo",
      "resolution": null,
      "priority": "medium",
      "owner": { "id": "ava.patel", "name": "Ava Patel" },
      "assignee": { "id": "marcus.lee", "name": "Marcus Lee" },
      "labels": ["frontend", "design-system"],
      "comments": [],
      "created_at": "2026-08-20T09:00:00Z",
      "updated_at": "2026-08-20T09:00:00Z"
    }
  ]
}
```

---

## 2. Create ticket

`POST /tickets`

The caller becomes the ticket's `owner` server-side — the frontend never
sends an owner id, and the backend should ignore one if it's ever present in
the body (never trust a client-supplied owner).

**Request**

```json
{
  "title": "Add dark-mode toggle to the settings page",
  "description": "<p>Persist the choice per-user and respect <strong>prefers-color-scheme</strong> on first load.</p><img src=\"data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAA...\" alt=\"mockup.png\">",
  "priority": "medium",
  "labels": ["frontend", "design-system"],
  "assignee_id": "marcus.lee"
}
```

All fields except `title` and `priority` are optional. `description` is
**HTML**, not plain text — it comes straight from the frontend's rich-text
editor and can contain `<p>`, `<strong>`, `<em>`, `<ul>`/`<ol>`/`<li>`, and
inline `<img>` tags whose `src` is a base64 `data:` URI (see the size note
above). **Sanitize `description` before storing/re-serving it** (e.g. strip
`<script>`, event handler attributes, `javascript:` URLs) since it's
user-supplied HTML — the frontend also sanitizes on render (via DOMPurify)
as defense in depth, but that isn't a substitute for server-side
sanitization. `assignee_id` may be `null` to explicitly leave unassigned.

**Response 201** — the full created ticket, with server-assigned `id`,
`key`, `owner`, timestamps, `status: "todo"`, `resolution: null`:

```json
{
  "id": "t1",
  "key": "REQ-1",
  "title": "Add dark-mode toggle to the settings page",
  "description": "<p>Persist the choice per-user and respect <strong>prefers-color-scheme</strong> on first load.</p><img src=\"data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAA...\" alt=\"mockup.png\">",
  "status": "todo",
  "resolution": null,
  "priority": "medium",
  "owner": { "id": "ava.patel", "name": "Ava Patel" },
  "assignee": { "id": "marcus.lee", "name": "Marcus Lee" },
  "labels": ["frontend", "design-system"],
  "comments": [],
  "created_at": "2026-09-11T10:15:00Z",
  "updated_at": "2026-09-11T10:15:00Z"
}
```

**Errors**

| Status | When |
| --- | --- |
| 400 | `title` missing/blank, or `priority` not one of the four valid values |
| 401 | not authenticated |
| 413 | request body too large (see the size note above) |

---

## 3. Update ticket metadata

`PATCH /tickets/:id`

Edits title/description/priority/labels/assignee. Does **not** change
`status`/`resolution` — that's §4. Any field omitted from the body is left
unchanged (partial update, not a full replace). As with create (§2),
`description` is sanitized HTML with possible embedded base64 images — same
rules and size considerations apply.

**Request**

```json
{
  "title": "Add dark-mode toggle to settings",
  "priority": "high",
  "assignee_id": null
}
```

**Response 200** — the full updated ticket (same shape as §2's response).

**Errors**

| Status | When |
| --- | --- |
| 400 | invalid `priority` value |
| 403 | caller lacks permission to edit this ticket (if your app restricts editing beyond just owner-closes-done) |
| 404 | no ticket with that id |
| 413 | request body too large |

---

## 4. Move ticket status — **the permission-critical endpoint**

`PATCH /tickets/:id/status`

Dedicated endpoint (separate from §3) specifically so the backend can apply
one rule cleanly: **only the ticket's `owner` may set `status: "done"`**.
Anyone may move a ticket among `todo` / `in_progress` / `in_review`, one
column at a time (no skipping ahead — the frontend enforces this client-side
too, but it's a UX nicety, not something this endpoint needs to validate).
The frontend's UI already gates the owner rule (drag-and-drop, and the
detail view's close buttons disable/reject the action for non-owners) —
**that UI gate is a convenience only; this endpoint must re-check ownership
itself**, since the UI can be bypassed by calling the API directly.

**Request** — moving to a non-terminal column:

```json
{ "status": "in_review" }
```

**Request** — closing the ticket (owner only):

```json
{ "status": "done", "resolution": "completed" }
```

`resolution` is `"completed"` or `"discarded"`; required when `status` is
`"done"`, ignored/omitted otherwise. Moving *out* of `done` back to an
earlier column (if your workflow allows that) should null out `resolution`.

**Response 200** — the full updated ticket.

**Errors**

| Status | When |
| --- | --- |
| 400 | `status` not a valid value, or `status: "done"` sent without `resolution` |
| 403 | **caller is not the ticket's owner and `status` is `"done"`** — this is the one that matters most |
| 404 | no ticket with that id |

```json
{ "message": "Only the requester can close this ticket." }
```

---

## 5. Delete ticket

`DELETE /tickets/:id`

**Only the ticket's `owner` (requester) may delete it** — same rule as closing a ticket
(§4). The frontend disables the delete button for anyone else, but as always with the
UI gates in this feature, **that's a convenience only; this endpoint must re-check
ownership server-side**, since the button being disabled doesn't stop a direct API call.

**Request**: no body.

**Response 200**

```json
{ "status": "ok", "id": "t1" }
```

**Errors**: `403` if the caller isn't the ticket's owner; `404` if not found.

---

## 6. Add a comment

`POST /tickets/:id/comments`

The comment's `author` is the authenticated caller, set server-side — the
request body only carries the text. Like `description` (§2/§3), `text` is
**rich-text HTML from the same comment editor**, not plain text — it can
contain `<p>`, `<strong>`, `<em>`, lists, and inline `<img src="data:image/...">`
tags for any images the commenter pasted, dragged in, or inserted. The same
sanitize-server-side requirement and size implications from the callout at
the top of this doc apply here too.

**Request**

```json
{ "text": "<p>Looks good — can we also cover the Safari edge case from <strong>REQ-4</strong>?</p>" }
```

**Response 201** — recommended: return the **full updated ticket** (so the
frontend's existing "upsert the whole ticket into state" pattern needs no
special-casing for comments):

```json
{
  "id": "t2",
  "key": "REQ-2",
  "title": "Custom model discovery times out on slow endpoints",
  "status": "in_progress",
  "resolution": null,
  "priority": "high",
  "owner": { "id": "marcus.lee", "name": "Marcus Lee" },
  "assignee": { "id": "ava.patel", "name": "Ava Patel" },
  "labels": ["bug", "models"],
  "comments": [
    {
      "id": "c1",
      "author": { "id": "ava.patel", "name": "Ava Patel" },
      "text": "<p>Looks good — can we also cover the Safari edge case from <strong>REQ-4</strong>?</p>",
      "created_at": "2026-09-11T10:20:00Z"
    }
  ],
  "created_at": "2026-08-18T14:20:00Z",
  "updated_at": "2026-09-11T10:20:00Z"
}
```

> If returning the full ticket is expensive (e.g. it has many comments
> already, or a large embedded-image description), returning just the
> created `TicketComment` object is also fine — flag it and I'll adjust
> `addTicketComment`'s reducer in `ticketsSlice.ts` to append rather than
> upsert-whole-ticket.

**Errors**: `400` if `text` is blank; `404` if the ticket doesn't exist.

---

## 7. Team roster (for the assignee picker)

`GET /users`

**Response 200**

```json
{
  "users": [
    { "id": "ava.patel", "name": "Ava Patel" },
    { "id": "marcus.lee", "name": "Marcus Lee" },
    { "id": "sofia.nguyen", "name": "Sofia Nguyen" },
    { "id": "jordan.reyes", "name": "Jordan Reyes" }
  ]
}
```

If the app already has a "list teammates" endpoint under a different path
(e.g. `/team/members`, `/organization/users`), point `store/slices/usersSlice.ts`
at that instead of standing up a duplicate `/users` route.

---

## Current user — no separate endpoint needed

The frontend does **not** call a dedicated "current user" endpoint for this
feature. The app already authenticates via SSO (see `authSlice.ts` /
`ssoLogin`) and stores the result at `state.auth.user` — an `SsoLoginResult`
with `username` and `profileName` fields. `TicketBoard.tsx` reads that
directly and adapts it to the `{ id, name }` shape this feature needs via
`toTicketUser()` in `ticketMeta.ts`, mapping `username → id` and
`profileName → name`. Nothing in this feature needs to log in or fetch
identity on its own — it just needs `ssoLogin` to have already run, which it
will have by the time a user can reach this route.

**⚠️ Backend requirement this implies**: since the owner-only close rule is
checked client-side as `ticket.owner.id === currentUser.id`, and
`currentUser.id` is now the SSO **`username`**, every `TicketUser` object the
API returns (`owner`, `assignee`, comment `author` — see the "Common types"
section at the top of this doc) must use that same **`username`** value as
its `id` field, not a separate internal numeric/UUID user id (this is why
the sample ids throughout this doc look like `"ava.patel"` rather than
`"u1"`). If the backend identifies users differently internally, populate
`TicketUser.id` with the user's username specifically when serializing
these fields, or the owner check will never match for anyone.

---

## Summary table

| # | Method | Path | Purpose |
| --- | --- | --- | --- |
| 1 | GET | `/tickets` | List all tickets |
| 2 | POST | `/tickets` | Create a ticket |
| 3 | PATCH | `/tickets/:id` | Edit ticket metadata |
| 4 | PATCH | `/tickets/:id/status` | Move status / close (owner-only for `done`) |
| 5 | DELETE | `/tickets/:id` | Delete a ticket |
| 6 | POST | `/tickets/:id/comments` | Add a comment |
| 7 | GET | `/users` | Team roster for the assignee picker |

No image-upload endpoint — images are embedded as base64 directly in
`description`, so there's nothing else for the backend to host or serve.
