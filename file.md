//Tiptapresizableimage.tsx
import { useCallback, useRef, type PointerEvent as ReactPointerEvent } from 'react';
import Image from '@tiptap/extension-image';
import { NodeViewWrapper, ReactNodeViewRenderer, type NodeViewProps } from '@tiptap/react';
import styles from './TicketDescriptionEditor.module.scss';

// ─────────────────────────────────────────────────────────────────────────
// An Image node extended with a persisted `width` and a drag-to-resize
// handle in the editor — Confluence-style: drop/paste/insert an image at
// its natural size, then drag the corner handle to resize it. The chosen
// width is written into the saved HTML as an inline `style="width:...px"`
// on the <img> itself (see `renderHTML` below), so it's just a plain,
// portable HTML attribute — the SAME width is what renders back out later
// in the read-only detail view (TicketDetailSidebar.tsx), no special
// handling needed there beyond the ordinary `max-width: 100%` safety net
// that keeps an oversized image from overflowing a narrower container.
//
// No explicit `height` is ever stored — only `width` is user-controlled,
// and `height: auto` (set both here and on the read-only view) lets the
// browser preserve the image's natural aspect ratio automatically, so
// there's no distortion and no separate height math to get wrong.
// ─────────────────────────────────────────────────────────────────────────

const MIN_WIDTH = 80;

function ResizableImageView({ node, updateAttributes, selected }: NodeViewProps) {
  const { src, alt, width } = node.attrs as { src: string; alt?: string; width?: number | null };
  const imgRef = useRef<HTMLImageElement | null>(null);
  const dragRef = useRef<{ startX: number; startWidth: number } | null>(null);

  const onHandlePointerDown = useCallback(
    (e: ReactPointerEvent) => {
      e.preventDefault();
      e.stopPropagation();
      const startWidth = imgRef.current?.getBoundingClientRect().width ?? width ?? 300;
      dragRef.current = { startX: e.clientX, startWidth };

      const onMove = (ev: PointerEvent) => {
        if (!dragRef.current) return;
        const delta = ev.clientX - dragRef.current.startX;
        const next = Math.max(MIN_WIDTH, Math.round(dragRef.current.startWidth + delta));
        updateAttributes({ width: next });
      };
      const onUp = () => {
        dragRef.current = null;
        window.removeEventListener('pointermove', onMove);
        window.removeEventListener('pointerup', onUp);
      };
      window.addEventListener('pointermove', onMove);
      window.addEventListener('pointerup', onUp);
    },
    [updateAttributes, width]
  );

  return (
    <NodeViewWrapper
      as="div"
      className={`${styles['resizable-image']} ${selected ? styles['resizable-image--selected'] : ''}`}
    >
      <img
        ref={imgRef}
        src={src}
        alt={alt || ''}
        style={width ? { width: `${width}px` } : undefined}
        data-resized={width ? 'true' : undefined}
        className={styles['resizable-image__img']}
      />
      {selected && (
        <span
          className={styles['resizable-image__handle']}
          onPointerDown={onHandlePointerDown}
          role="presentation"
          title="Drag to resize"
        />
      )}
    </NodeViewWrapper>
  );
}

export const ResizableImage = Image.extend({
  addAttributes() {
    return {
      ...this.parent?.(),
      width: {
        default: null,
        parseHTML: (el: HTMLElement) => {
          const styleWidth = el.style.width;
          if (styleWidth) {
            const n = parseInt(styleWidth, 10);
            if (!Number.isNaN(n)) return n;
          }
          const attr = el.getAttribute('width');
          return attr ? parseInt(attr, 10) || null : null;
        },
        renderHTML: (attrs: { width?: number | null }) =>
          attrs.width ? { style: `width: ${attrs.width}px` } : {},
      },
    };
  },
  addNodeView() {
    return ReactNodeViewRenderer(ResizableImageView);
  },
});
















//Ticketdescriptioneditor.tsx
import { useEffect, useRef, type ReactNode } from 'react';
import { useEditor, EditorContent, type Editor } from '@tiptap/react';
import StarterKit from '@tiptap/starter-kit';
import { ResizableImage } from './tiptapResizableImage';
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
      // otherwise rejects data: URIs and expects a hosted src. ResizableImage
      // (tiptapResizableImage.tsx) extends it with a drag-to-resize handle
      // and persists the chosen width into the saved HTML.
      ResizableImage.configure({ inline: false, allowBase64: true }),
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



















//Ticketdescription.module.scss
@use '../../styles/_variables' as *;

// Jira-style rich-text description field: a small toolbar + an editable
// area that accepts typed text, pasted/dropped images, and toolbar-
// inserted images. Images are embedded as base64 data: URIs directly in
// the field's own HTML — no upload request, no separate attachments field.

$mono: $font-mono;
$sans: $font-body;

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
// Smaller footprint for the comment composer — same capability (toolbar,
// paste/drag/insert images), just less visually dominant than the full
// description field.
.editor--compact {
  .toolbar {
    padding: 0.25em 0.35em;
  }
  .toolbar-btn {
    width: 1.6em;
    height: 1.6em;
  }
  .editor-body {
    padding: 0.5em 0.6em;
    font-size: 0.86em;
    min-height: 3.8em;
  }
  .editor-scroll {
    max-height: 180px;
  }
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
}

// ---- resizable images ---------------------------------------------------
// Confluence-style: an image drops in at its natural size (no artificial
// cap beyond the editor's own width), and a drag handle appears in the
// bottom-right corner whenever the image is selected — dragging it sets an
// explicit width (only width; height follows automatically via `height:
// auto` below, so the aspect ratio can never get distorted). That width is
// what gets saved into the ticket's description HTML, so the image renders
// at the exact same size later in the read-only detail view.
.resizable-image {
  position: relative;
  display: inline-block;
  max-width: 100%;
  line-height: 0; // avoids a stray gap under the image from inline baseline alignment
  margin: 0.4em 0;
}
.resizable-image--selected {
  outline: 2px solid $signal;
  outline-offset: 2px;
  border-radius: 8px;
}
.resizable-image__img {
  display: block;
  max-width: 100%;
  height: auto;
  border-radius: 8px;
  border: 1px solid $line;
}
.resizable-image__handle {
  position: absolute;
  right: -5px;
  bottom: -5px;
  width: 13px;
  height: 13px;
  background: $signal;
  border: 2px solid $card;
  border-radius: 3px;
  cursor: nwse-resize;
  box-shadow: 0 1px 3px rgba(20, 22, 27, 0.35);
  touch-action: none;
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

  // No separate "Title is required" message — the submit button is simply
  // disabled while the title is empty, which already communicates that.
  const valid = title.trim().length > 0;

  const addLabel = () => {
    const v = labelDraft.trim();
    if (!v) return;
    if (!labels.includes(v)) setLabels((prev) => [...prev, v]);
    setLabelDraft('');
  };

  const submit = () => {
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
                className={`${styles['field-input']} ${styles['field-input--lg']}`}
                value={title}
                onChange={(e) => setTitle(e.target.value)}
                placeholder="Short summary of the requirement"
              />
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



















//Createticketdrawer.module.scss
@use '../../styles/_variables' as *;

// Centered modal — same shell convention used across the app (see
// Datasets.module.scss's __modal-overlay/__modal, and TicketBoard's
// .modal-overlay/.modal). Rendered inline, no portal: a full-viewport
// fixed overlay that flex-centers its panel.

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;

@keyframes ticket-fade-in {
  from { opacity: 0; }
  to { opacity: 1; }
}
@keyframes ticket-modal-in {
  from { opacity: 0; transform: translateY(8px) scale(0.98); }
  to { opacity: 1; transform: none; }
}
@keyframes ticket-spin {
  to { transform: rotate(360deg); }
}

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
  width: min(880px, 100%);
  max-height: 88vh;
  display: flex;
  flex-direction: column;
  background: $card;
  border: 1px solid $line;
  border-radius: 18px;
  box-shadow: 0 24px 60px -20px rgba(20, 22, 27, 0.4);
  overflow: hidden;
  animation: ticket-modal-in 0.18s cubic-bezier(0.22, 1, 0.36, 1);
  // Own base size, a touch larger than the page base at very wide
  // viewports — a focused modal reads better slightly bigger than the
  // dense board sitting behind it.
  font-size: 0.8125rem;
  @media (min-width: 1800px) {
    font-size: 1.0625rem;
  }
}

// ---- header -----------------------------------------------------------
.modal-hdr {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1.1em 1.25em;
  border-bottom: 1px solid $line;
}
.modal-header-text {
  display: flex;
  flex-direction: column;
  gap: 0.15em;
}
.modal-eyebrow {
  font-family: $mono;
  font-size: 0.68em;
  font-weight: 700;
  letter-spacing: 0.1em;
  text-transform: uppercase;
  color: $signal;
}
.modal-title {
  font-family: $display;
  font-size: 1.2em;
  font-weight: 700;
  color: $ink;
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

// ---- two-column body ----------------------------------------------------
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
  width: 260px;
  border-left: 1px solid $line;
  background: $paper;
  overflow-y: auto;
  padding: 1.25em;
  display: flex;
  flex-direction: column;
  gap: 1.1em;
}

.field {
  display: flex;
  flex-direction: column;
  gap: 0.4em;
}
.field-label {
  font-size: 0.78em;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.04em;
  color: $ink-2;
}
.field-req {
  color: $danger;
}
.field-input {
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
.field-input--lg {
  font-size: 1.15em;
  font-weight: 600;
  padding: 0.6em 0.7em;
}
.field-input--error {
  border-color: $danger;
  &:focus {
    box-shadow: 0 0 0 3px $danger-wash;
  }
}
.field-error {
  font-size: 0.78em;
  color: $danger;
}
.field-note {
  margin-top: auto;
  display: flex;
  align-items: center;
  gap: 0.4em;
  font-size: 0.76em;
  color: $ink-3;
  padding-top: 0.8em;
  border-top: 1px dashed $line;
}

// ---- chip input -----------------------------------------------------------
.chip-input {
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
.chip {
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
.chip-field {
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
.modal-foot {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 0.6em;
  padding: 1em 1.25em;
  border-top: 1px solid $line;
}
.modal-cancel {
  padding: 9px 16px;
  border: 1px solid $line;
  border-radius: 10px;
  background: $card;
  color: $ink-2;
  font-family: $sans;
  font-size: 0.9em;
  font-weight: 650;
  cursor: pointer;
  transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;
  &:hover:not(:disabled) { border-color: $ink-3; color: $ink; background: $paper; }
  &:disabled { opacity: 0.5; cursor: not-allowed; }
}
.modal-submit {
  display: inline-flex;
  align-items: center;
  gap: 7px;
  padding: 9px 18px;
  border: 1px solid transparent;
  border-radius: 10px;
  background: $signal;
  color: #fff;
  font-family: $sans;
  font-size: 0.9em;
  font-weight: 650;
  cursor: pointer;
  transition: background 0.15s ease;
  &:hover:not(:disabled) { background: $signal-2; }
  &:disabled { opacity: 0.6; cursor: not-allowed; }
}
.spin {
  animation: ticket-spin 0.8s linear infinite;
}

@media (max-width: 768px) {
  .modal-overlay { padding: 12px; }
  .modal { width: 100%; max-height: 94vh; border-radius: 14px; }
  .modal-body { flex-direction: column; overflow-y: auto; }
  .modal-rail { width: auto; border-left: 0; border-top: 1px solid $line; }
}

@media (prefers-reduced-motion: reduce) {
  .modal-overlay,
  .modal,
  .spin {
    animation: none;
  }
}

// ===========================================================================
// Custom select — replaces the native <select> with themed dropdown chrome
// (used for Priority / Assignee in the rail).
// ===========================================================================
.select {
  position: relative;
}
.select-trigger {
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.5em;
  border: 1px solid $line;
  border-radius: 8px;
  background: $card;
  color: $ink;
  font-size: 0.92em;
  font-family: inherit;
  padding: 0.62em 0.7em;
  cursor: pointer;
  transition: border-color 0.15s, box-shadow 0.15s, background 0.15s;

  &:hover:not(:disabled) {
    border-color: $ink-3;
  }
  &:disabled {
    opacity: 0.55;
    cursor: not-allowed;
  }
}
.select-trigger--open {
  border-color: $signal;
  box-shadow: 0 0 0 3px $wash;
}
.select-value {
  display: flex;
  align-items: center;
  gap: 0.5em;
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  font-weight: 500;
}
.select-placeholder {
  color: $ink-3;
  font-weight: 400;
}
.select-caret {
  flex-shrink: 0;
  color: $ink-3;
  transition: transform 0.15s;
  .select-trigger--open & {
    transform: rotate(180deg);
  }
}

.select-menu {
  position: absolute;
  top: calc(100% + 6px);
  left: 0;
  right: 0;
  z-index: 20;
  max-height: 15em;
  overflow-y: auto;
  background: $card;
  border: 1px solid $line;
  border-radius: 10px;
  box-shadow: 0 14px 32px -12px rgba(20, 22, 27, 0.35);
  padding: 0.35em;
  display: flex;
  flex-direction: column;
  gap: 0.1em;
  animation: ticket-modal-in 0.12s ease both;
}
.select-empty {
  padding: 0.6em 0.7em;
  font-size: 0.86em;
  color: $ink-3;
  text-align: center;
}
.select-option {
  display: flex;
  align-items: center;
  gap: 0.55em;
  width: 100%;
  border: 0;
  background: transparent;
  color: $ink;
  font-size: 0.9em;
  font-family: inherit;
  text-align: left;
  padding: 0.55em 0.6em;
  border-radius: 7px;
  cursor: pointer;
  transition: background 0.12s;

  &:hover {
    background: $paper;
  }
}
.select-option--active {
  background: $wash;
  font-weight: 600;
  color: $signal;
}
.select-option--highlight {
  background: $paper;
}
.select-option--active.select-option--highlight {
  background: $wash;
  box-shadow: inset 0 0 0 1px rgba(43, 43, 245, 0.25);
}
.select-option-label {
  flex: 1;
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.select-check {
  flex-shrink: 0;
  color: $signal;
}
.select-dot {
  flex-shrink: 0;
  width: 1.5em;
  height: 1.5em;
  border-radius: 50%;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  font-size: 0.62em;
  font-weight: 700;
  color: #fff;
  // used both as a plain priority-color dot (no glyph) and as a tiny
  // avatar-style badge (with initials as glyph) for the assignee list
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
    display: block;
    max-width: 100%;
    height: auto;
    border-radius: 8px;
    border: 1px solid $line;
    margin: 0.5em 0;
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
    height: auto;
    border-radius: 8px;
    border: 1px solid $line;
    margin: 0.4em 0;
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
