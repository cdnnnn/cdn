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
  // Set right before every onChange this component fires itself (typing,
  // toolbar actions, applying an image resize, etc). The sync effect below
  // checks this to tell "the editor just told us about its own change" apart
  // from "the parent handed us a genuinely new value from outside" (e.g.
  // clearing the comment box after posting, or switching to a different
  // ticket's description). Without this distinction, the effect compares
  // `value` against `editor.getHTML()` on every single change this
  // component itself causes — any timing or serialization quirk in that
  // echo loop can end up calling `setContent()` right after an edit and
  // silently reverting it, which is exactly the kind of bug a resized
  // image not sticking looks like.
  const isInternalChange = useRef(false);

  const editor = useEditor({
    extensions: [
      StarterKit.configure({ heading: false }),
      // allowBase64 is the whole trick here — the stock Image extension
      // otherwise rejects data: URIs and expects a hosted src. ResizableImage
      // (tiptapResizableImage.tsx) extends it with explicit width/height
      // resize controls and persists the chosen size into the saved HTML.
      ResizableImage.configure({ inline: false, allowBase64: true }),
      Placeholder.configure({ placeholder }),
    ],
    content: value,
    editable: !disabled,
    onUpdate: ({ editor: e }) => {
      isInternalChange.current = true;
      onChange(e.getHTML());
    },
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

  // Only pushes `value` into the editor when the change genuinely came from
  // outside this component (see isInternalChange above) — e.g. the parent
  // clearing the draft after a successful post/submit, or switching to a
  // different ticket's description/comment.
  useEffect(() => {
    if (!editor) return;
    if (isInternalChange.current) {
      isInternalChange.current = false;
      return;
    }
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
