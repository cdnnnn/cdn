//Ticketdescriptioneditior.tsx
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






















//Ticketdescriptioneditor.module.scss
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
  /** Rich-text HTML from the description editor. Any images the user
   *  pasted/dropped/inserted are embedded inline as base64
   *  `<img src="data:image/...">` tags — there's no separate upload step,
   *  hosted URL, or attachments array; image + text are one field.
   *  Sanitize before rendering (see TicketDetailSidebar.tsx). */
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
