//Ticketdescriptioneditor.tsx
import { useEffect, useRef, type ReactNode } from 'react';
import { useEditor, EditorContent, type Editor } from '@tiptap/react';
import StarterKit from '@tiptap/starter-kit';
import Placeholder from '@tiptap/extension-placeholder';
import {
  Bold as BoldIcon,
  Italic as ItalicIcon,
  List as ListIcon,
  ListOrdered,
  ImagePlus,
} from 'lucide-react';
import { UploadImage } from './tiptapUploadImage';
import { uploadsApi } from '../../api/endpoints/uploads';
import { useToast } from '../common/Toast';
import styles from './TicketDescriptionEditor.module.scss';

interface TicketDescriptionEditorProps {
  /** HTML content — same shape TipTap emits and consumes. */
  value: string;
  onChange: (html: string) => void;
  placeholder?: string;
  disabled?: boolean;
}

const MAX_IMAGE_BYTES = 5 * 1024 * 1024; // 5MB

/**
 * A Jira-style rich-text description field: type normally, and images can
 * be pasted straight from the clipboard, dragged in from the desktop, or
 * inserted via the toolbar button — there's no separate "attachments"
 * upload field anywhere else in the form. Each image uploads in the
 * background (via POST /uploads/images) the moment it's added; a local
 * preview shows immediately and swaps to the real hosted URL once the
 * upload finishes, so the rest of the form never has to wait on it.
 */
export default function TicketDescriptionEditor({
  value,
  onChange,
  placeholder = 'Context, acceptance criteria, links… paste or drag an image in.',
  disabled = false,
}: TicketDescriptionEditorProps) {
  const toast = useToast();
  const fileInputRef = useRef<HTMLInputElement | null>(null);

  const editor = useEditor({
    extensions: [
      StarterKit.configure({ heading: false }),
      UploadImage.configure({ inline: false, allowBase64: false }),
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
        files.forEach((f) => insertAndUpload(editor, f, toast));
        return true;
      },
      handleDrop: (_view, event) => {
        const files = Array.from(event.dataTransfer?.files ?? []).filter((f) =>
          f.type.startsWith('image/')
        );
        if (files.length === 0) return false;
        event.preventDefault();
        files.forEach((f) => insertAndUpload(editor, f, toast));
        return true;
      },
    },
  });

  // Keep the editor's content in sync if `value` changes from outside (e.g.
  // switching from "create" to "edit" mode with a different initial ticket)
  // without fighting the user's own typing on every render.
  useEffect(() => {
    if (!editor) return;
    if (value !== editor.getHTML()) {
      editor.commands.setContent(value || '', false);
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [editor]);

  useEffect(() => {
    editor?.setEditable(!disabled);
  }, [editor, disabled]);

  if (!editor) return null;

  const pickImage = () => fileInputRef.current?.click();

  return (
    <div className={`${styles['editor']} ${disabled ? styles['editor--disabled'] : ''}`}>
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
            if (file) insertAndUpload(editor, file, toast);
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

// Inserts a local preview immediately (so the image shows up the instant
// it's pasted/dropped/picked), then uploads in the background and swaps
// the preview for the real hosted URL once it's done — or removes the node
// and toasts an error if the upload fails.
function insertAndUpload(editor: Editor | null, file: File, toast: ReturnType<typeof useToast>) {
  if (!editor) return;
  if (file.size > MAX_IMAGE_BYTES) {
    toast.error(`"${file.name}" is over 5MB — pick a smaller image.`);
    return;
  }

  const tempId = `up_${Date.now()}_${Math.random().toString(36).slice(2, 7)}`;
  const localUrl = URL.createObjectURL(file);

  // TipTap's built-in `setImage` command type only knows about
  // src/alt/title — cast is needed to also pass our extension's custom
  // data-uploading/data-temp-id attributes through.
  editor
    .chain()
    .focus()
    .setImage({
      src: localUrl,
      alt: file.name,
      'data-uploading': 'true',
      'data-temp-id': tempId,
      // eslint-disable-next-line @typescript-eslint/no-explicit-any
    } as any)
    .run();

  uploadsApi
    .uploadImage(file)
    .then((uploaded) => {
      replaceUploadingImage(editor, tempId, { src: uploaded.url, 'data-uploading': null });
      URL.revokeObjectURL(localUrl);
    })
    .catch(() => {
      removeUploadingImage(editor, tempId);
      URL.revokeObjectURL(localUrl);
      toast.error(`Couldn't upload "${file.name}".`);
    });
}

function findUploadingNode(editor: Editor, tempId: string) {
  let found: { pos: number; node: ReturnType<Editor['state']['doc']['nodeAt']> } | null = null;
  editor.state.doc.descendants((node, pos) => {
    if (found) return false;
    if (node.attrs?.['data-temp-id'] === tempId) {
      found = { pos, node };
      return false;
    }
    return true;
  });
  return found;
}

function replaceUploadingImage(editor: Editor, tempId: string, patch: Record<string, unknown>) {
  const found = findUploadingNode(editor, tempId);
  if (!found || !found.node) return;
  editor
    .chain()
    .command(({ tr }) => {
      tr.setNodeMarkup(found.pos, undefined, { ...found.node!.attrs, ...patch });
      return true;
    })
    .run();
}

function removeUploadingImage(editor: Editor, tempId: string) {
  const found = findUploadingNode(editor, tempId);
  if (!found || !found.node) return;
  editor
    .chain()
    .command(({ tr }) => {
      tr.delete(found.pos, found.pos + found.node!.nodeSize);
      return true;
    })
    .run();
}















//Ticketdescriptioneditor.module.scss
@use '../../styles/_variables' as *;

// Jira-style rich-text description field: a small toolbar + an editable
// area that accepts typed text, pasted/dropped images, and toolbar-
// inserted images — no separate attachments field anywhere else.

$mono: $font-mono;
$sans: $font-body;

@keyframes editor-spin {
  to { transform: rotate(360deg); }
}

.editor {
  border: 1px solid $line;
  border-radius: 8px;
  background: $card;
  overflow: hidden;
  transition: border-color 0.15s, box-shadow 0.15s;

  &:focus-within {
    border-color: $signal;
    box-shadow: 0 0 0 3px $wash;
  }
}
.editor--disabled {
  opacity: 0.6;
  pointer-events: none;
}

// ---- toolbar ----------------------------------------------------------
.toolbar {
  display: flex;
  align-items: center;
  gap: 0.2em;
  padding: 0.4em 0.5em;
  border-bottom: 1px solid $line;
  background: $paper;
}
.toolbar-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 1.9em;
  height: 1.9em;
  border: 0;
  border-radius: 6px;
  background: transparent;
  color: $ink-2;
  cursor: pointer;
  transition: background 0.12s, color 0.12s;
  &:hover {
    background: $ink-wash;
    color: $ink;
  }
}
.toolbar-btn--active {
  background: $wash;
  color: $signal;
}
.toolbar-sep {
  width: 1px;
  height: 1.2em;
  background: $line;
  margin: 0 0.3em;
}

// ---- editable body ------------------------------------------------------
.editor-scroll {
  max-height: 260px;
  overflow-y: auto;
}
.editor-body {
  padding: 0.7em 0.8em;
  font-family: $sans;
  font-size: 0.92em;
  line-height: 1.6;
  color: $ink;
  outline: none;
  min-height: 5.5em;

  p {
    margin: 0 0 0.5em;
    &:last-child {
      margin-bottom: 0;
    }
  }
  ul,
  ol {
    margin: 0 0 0.5em;
    padding-left: 1.4em;
  }
  li {
    margin-bottom: 0.2em;
  }
  strong {
    font-weight: 700;
  }

  // TipTap's placeholder-free empty-state: shown via a pseudo-element on
  // the first empty paragraph, driven by ProseMirror's own `is-empty`
  // class plus the `data-placeholder` we set in editorProps.attributes.
  p.is-editor-empty:first-child::before {
    content: attr(data-placeholder);
    float: left;
    height: 0;
    color: $ink-3;
    pointer-events: none;
  }

  img {
    display: block;
    max-width: 100%;
    max-height: 320px;
    height: auto;
    border-radius: 8px;
    border: 1px solid $line;
    margin: 0.4em 0;
    object-fit: contain;
  }

  // Local-preview placeholder while an image is still uploading — dimmed
  // and desaturated is the upload-in-progress cue. (A proper spinner/badge
  // overlay would need a custom TipTap NodeView around the image node;
  // this simpler CSS-only cue covers the same "something's happening"
  // signal without that extra machinery.)
  img[data-uploading='true'] {
    position: relative;
    opacity: 0.55;
    filter: grayscale(30%);
    cursor: wait;
  }
}





















//Tiptapuploadimage.ts
import Image from '@tiptap/extension-image';

// Extends the stock Image extension with two extra attributes so a pasted/
// dropped/inserted image can render immediately (from a local blob: URL)
// while it uploads in the background, then get swapped to the real hosted
// URL once the upload finishes — see TicketDescriptionEditor.tsx for where
// these attributes actually get written and read.
export const UploadImage = Image.extend({
  addAttributes() {
    return {
      ...this.parent?.(),
      'data-uploading': {
        default: null,
        parseHTML: (el) => el.getAttribute('data-uploading'),
        renderHTML: (attrs) =>
          attrs['data-uploading'] ? { 'data-uploading': attrs['data-uploading'] } : {},
      },
      'data-temp-id': {
        default: null,
        parseHTML: (el) => el.getAttribute('data-temp-id'),
        renderHTML: (attrs) => (attrs['data-temp-id'] ? { 'data-temp-id': attrs['data-temp-id'] } : {}),
      },
    };
  },
});
























//Ticketdetailsidebar.tsx
import { useState } from 'react';
import DOMPurify from 'dompurify';
import { X, Pencil, Trash2, Loader2, Lock, Check, Ban, Send } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import { addTicketComment } from '../../store/slices/ticketsSlice';
import type { Ticket, TicketStatus, TicketResolution, TicketUser } from '../../types/tickets';
import ConfirmDialog from '../common/ConfirmDialog';
import { useToast } from '../common/Toast';
import {
  COLUMNS,
  PRIORITY_META,
  isOwner,
  isSequentialMove,
  canDropTicket,
  isEmptyHtml,
  OWNER_ONLY_HINT,
  SEQUENCE_HINT,
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

// Centered modal, same shell as the rest of the app's dialogs (see
// .modal-overlay / .modal in TicketBoard.module.scss). Rendered inline —
// no portal — position:fixed + flex-centering is sufficient here.
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
              disabled={deleting}
              aria-label="Delete ticket"
              title="Delete"
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

















//Createticketdrawer.tsx
import { useEffect, useMemo, useRef, useState } from 'react';
import { X, Loader2, FileText } from 'lucide-react';
import type { Ticket, TicketPriority, TicketUser } from '../../types/tickets';
import { PRIORITY_META, initials, avatarAccent } from './ticketMeta';
import TicketSelect from './TicketSelect';
import TicketDescriptionEditor from './TicketDescriptionEditor';
import styles from './CreateTicketDrawer.module.scss';

/** What the drawer hands back on submit — no id/status, the board adds those.
 *  `description` is HTML from the rich-text editor (images the user pasted/
 *  dropped/inserted are already embedded as <img src="..."> pointing at
 *  their real uploaded URLs — there's no separate attachments array). */
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
  /** Return a Promise to keep the modal open on failure and close it on success. */
  onSubmit: (payload: TicketSubmitPayload) => Promise<unknown> | void;
}

// Centered modal, same shell as the rest of the app's dialogs (see
// .modal-overlay / .modal in this file's SCSS module). Rendered inline —
// no portal.
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
      description: description || undefined,
      priority,
      labels: labels.length ? labels : undefined,
      assignee_id: assigneeId || null,
    });
  };

  const heading = mode === 'edit' ? 'Edit requirement' : 'New requirement';
  const cta = mode === 'edit' ? 'Save changes' : 'Create requirement';

  const priorityOptions = useMemo(
    () =>
      (Object.keys(PRIORITY_META) as TicketPriority[]).map((p) => ({
        value: p,
        label: PRIORITY_META[p].label,
        accent: PRIORITY_META[p].accent,
      })),
    []
  );

  const assigneeOptions = useMemo(
    () => [
      { value: '', label: 'Unassigned' },
      ...members.map((m) => ({
        value: m.id,
        label: m.name,
        accent: avatarAccent(m),
        glyph: initials(m),
      })),
    ],
    [members]
  );

  return (
    <div className={styles['modal-overlay']} onClick={() => !submitting && onClose()}>
      <div
        className={styles['modal']}
        role="dialog"
        aria-modal="true"
        aria-label={heading}
        onClick={(e) => e.stopPropagation()}
      >
        <header className={styles['modal-hdr']}>
          <div className={styles['modal-header-text']}>
            <span className={styles['modal-eyebrow']}>{mode === 'edit' ? 'Editing' : 'New'}</span>
            <span className={styles['modal-title']}>{heading}</span>
          </div>
          <button
            className={styles['modal-close']}
            onClick={onClose}
            disabled={submitting}
            aria-label="Close"
          >
            <X size={16} />
          </button>
        </header>

        <div className={styles['modal-body']}>
          {/* ---- main column: title, description ---- */}
          <div className={styles['modal-main']}>
            <label className={styles['field']}>
              <span className={styles['field-label']}>
                Title <span className={styles['field-req']}>*</span>
              </span>
              <input
                ref={firstFieldRef}
                className={`${styles['field-input']} ${styles['field-input--lg']} ${titleError ? styles['field-input--error'] : ''}`}
                value={title}
                onChange={(e) => setTitle(e.target.value)}
                onBlur={() => setTouched(true)}
                placeholder="Short summary of the requirement"
              />
              {titleError && <span className={styles['field-error']}>{titleError}</span>}
            </label>

            <div className={styles['field']}>
              <span className={styles['field-label']}>Description</span>
              <TicketDescriptionEditor
                value={description}
                onChange={setDescription}
                disabled={submitting}
              />
            </div>
          </div>

          {/* ---- meta column: priority, assignee, labels ---- */}
          <div className={styles['modal-rail']}>
            <div className={styles['field']}>
              <span className={styles['field-label']}>Priority</span>
              <TicketSelect
                value={priority}
                options={priorityOptions}
                onChange={(v) => setPriority(v as TicketPriority)}
                aria-label="Priority"
              />
            </div>

            <div className={styles['field']}>
              <span className={styles['field-label']}>Assignee</span>
              <TicketSelect
                value={assigneeId}
                options={assigneeOptions}
                placeholder="Unassigned"
                onChange={setAssigneeId}
                disabled={members.length === 0}
                aria-label="Assignee"
              />
            </div>

            <div className={styles['field']}>
              <span className={styles['field-label']}>Labels</span>
              <div className={styles['chip-input']}>
                {labels.map((l) => (
                  <span key={l} className={styles['chip']}>
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
                  className={styles['chip-field']}
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
              <div className={styles['field-note']}>
                <FileText size={12} />
                {initialTicket.key} · created{' '}
                {new Date(initialTicket.created_at).toLocaleDateString()}
              </div>
            )}
          </div>
        </div>

        <footer className={styles['modal-foot']}>
          <button className={styles['modal-cancel']} onClick={onClose} disabled={submitting}>
            Cancel
          </button>
          <button className={styles['modal-submit']} onClick={submit} disabled={submitting || !valid}>
            {submitting ? (
              <>
                <Loader2 size={15} className={styles['spin']} />
                Saving…
              </>
            ) : (
              cta
            )}
          </button>
        </footer>
      </div>
    </div>
  );
}
















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

/** The shape returned by POST /uploads/images — used only by the
 *  description editor (TicketDescriptionEditor.tsx) when a user pastes,
 *  drags, or inserts an image. There is no ticket-level attachments array;
 *  uploaded images are embedded directly as <img> tags inside the ticket's
 *  `description` HTML. */
export interface TicketAttachment {
  id: string;
  name: string;
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
  /** Rich-text HTML from the description editor — pasted/dropped/inserted
   *  images are embedded inline as <img src="..."> pointing at their real
   *  uploaded URLs. Sanitize before rendering (see TicketDetailSidebar.tsx). */
  description?: string;
  status: TicketStatus;
  resolution?: TicketResolution | null;
  priority: TicketPriority;
  /** The requester. Only this user may move the ticket into `done`. */
  owner: TicketUser;
  assignee?: TicketUser | null;
  labels?: string[];
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
}

/** Partial edit of an existing ticket (title/description/priority/labels/assignee). */
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
