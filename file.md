import { useEffect, useRef, useState } from 'react';
import Image from '@tiptap/extension-image';
import { NodeViewWrapper, ReactNodeViewRenderer, type NodeViewProps } from '@tiptap/react';
import styles from './TicketDescriptionEditor.module.scss';

// ─────────────────────────────────────────────────────────────────────────
// An Image node extended with persisted `width`/`height`, set via explicit
// number inputs rather than a drag handle. Click an image to select it and
// a small "W × H" control bar appears; type a value and press Apply (or
// just tab out / press Enter) to resize it, or leave both blank and it
// stays at its natural size. This replaces an earlier drag-to-resize
// implementation that turned out to be unreliable — a single deliberate
// "commit" click firing exactly one attribute update is far easier to
// reason about (and debug) than dozens of rapid-fire updates during a
// pointer drag, which could race against the editor's own content-sync
// logic in TicketDescriptionEditor.tsx.
//
// Both dimensions are stored as plain numeric HTML attributes on the <img>
// itself — `width="450" height="300"`, not a CSS style — combined with
// `max-width: 100%; height: auto;` in this file's CSS (and the identical
// rule in TicketBoard.module.scss for the read-only view), so the browser
// renders it at exactly that saved width × height wherever there's room,
// and only scales both dimensions down together — preserving the same
// aspect ratio — if the container is narrower than that.
//
// `width`/`height` are contributed via each attribute's own `renderHTML`
// (the standard, documented way to add an attribute to an existing TipTap
// node), NOT by overriding the node's own `renderHTML` — that was tried
// once and broke image rendering entirely, since it silently discarded the
// base Image extension's own src/alt handling.
// ─────────────────────────────────────────────────────────────────────────

const MIN_DIMENSION = 20;

function ResizableImageView({ node, updateAttributes, selected }: NodeViewProps) {
  const { src, alt, width, height } = node.attrs as {
    src: string;
    alt?: string;
    width?: number | null;
    height?: number | null;
  };
  const imgRef = useRef<HTMLImageElement | null>(null);
  const [wInput, setWInput] = useState(width != null ? String(width) : '');
  const [hInput, setHInput] = useState(height != null ? String(height) : '');

  // Keep the inputs in sync if the underlying attrs ever change from
  // outside this control (e.g. undo/redo, or loading a different ticket).
  useEffect(() => {
    setWInput(width != null ? String(width) : '');
    setHInput(height != null ? String(height) : '');
  }, [width, height]);

  const naturalRatio = () => {
    const img = imgRef.current;
    return img && img.naturalWidth && img.naturalHeight ? img.naturalHeight / img.naturalWidth : null;
  };

  // One deliberate commit: parses whatever's currently in the two inputs
  // and writes both attributes in a single updateAttributes call. Blank
  // input on either side means "use natural size" for that dimension — if
  // only one of the two is filled in, the other is derived from the
  // image's real aspect ratio so it can't come out stretched.
  const applySize = () => {
    const wRaw = wInput.trim();
    const hRaw = hInput.trim();
    let w = wRaw ? Math.max(MIN_DIMENSION, parseInt(wRaw, 10) || 0) : null;
    let h = hRaw ? Math.max(MIN_DIMENSION, parseInt(hRaw, 10) || 0) : null;
    const ratio = naturalRatio();
    if (w && !h && ratio) h = Math.round(w * ratio);
    if (h && !w && ratio) w = Math.round(h / ratio);
    updateAttributes({ width: w, height: h });
    setWInput(w != null ? String(w) : '');
    setHInput(h != null ? String(h) : '');
  };

  const resetSize = () => {
    setWInput('');
    setHInput('');
    updateAttributes({ width: null, height: null });
  };

  return (
    <NodeViewWrapper
      as="div"
      className={`${styles['resizable-image']} ${selected ? styles['resizable-image--selected'] : ''}`}
    >
      <img
        ref={imgRef}
        src={src}
        alt={alt || ''}
        width={width ?? undefined}
        height={height ?? undefined}
        data-resized={width ? 'true' : undefined}
        className={styles['resizable-image__img']}
      />
      {selected && (
        // contentEditable={false} + stopping mousedown keeps ProseMirror
        // from treating clicks/typing in these inputs as document edits or
        // losing the image's selection while you're using them.
        <div
          className={styles['resizable-image__controls']}
          contentEditable={false}
          onMouseDown={(e) => e.stopPropagation()}
        >
          <label className={styles['resizable-image__field']}>
            <span>W</span>
            <input
              type="number"
              min={MIN_DIMENSION}
              value={wInput}
              placeholder="auto"
              onChange={(e) => setWInput(e.target.value)}
              onKeyDown={(e) => {
                if (e.key === 'Enter') applySize();
              }}
            />
          </label>
          <span className={styles['resizable-image__times']}>×</span>
          <label className={styles['resizable-image__field']}>
            <span>H</span>
            <input
              type="number"
              min={MIN_DIMENSION}
              value={hInput}
              placeholder="auto"
              onChange={(e) => setHInput(e.target.value)}
              onKeyDown={(e) => {
                if (e.key === 'Enter') applySize();
              }}
            />
          </label>
          <button type="button" className={styles['resizable-image__apply']} onClick={applySize}>
            Apply
          </button>
          <button type="button" className={styles['resizable-image__reset']} onClick={resetSize}>
            Reset
          </button>
        </div>
      )}
    </NodeViewWrapper>
  );
}

// Reads a saved size back from either the plain `width`/`height` HTML
// attribute this extension writes, or (for backward compatibility with
// anything saved by an earlier version of this extension) the older
// `style="width: …px"` form.
const parseSizeAttr = (el: HTMLElement, attr: 'width' | 'height') => {
  const direct = el.getAttribute(attr);
  if (direct) {
    const n = parseInt(direct, 10);
    if (!Number.isNaN(n)) return n;
  }
  const styleValue = attr === 'width' ? el.style.width : el.style.height;
  if (styleValue) {
    const n = parseInt(styleValue, 10);
    if (!Number.isNaN(n)) return n;
  }
  return null;
};

export const ResizableImage = Image.extend({
  addAttributes() {
    return {
      ...this.parent?.(),
      width: {
        default: null,
        parseHTML: (el: HTMLElement) => parseSizeAttr(el, 'width'),
        renderHTML: (attrs: { width?: number | null }) =>
          attrs.width ? { width: attrs.width } : {},
      },
      height: {
        default: null,
        parseHTML: (el: HTMLElement) => parseSizeAttr(el, 'height'),
        renderHTML: (attrs: { height?: number | null }) =>
          attrs.height ? { height: attrs.height } : {},
      },
    };
  },
  addNodeView() {
    return ReactNodeViewRenderer(ResizableImageView);
  },
});





















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
// cap beyond the editor's own width). Click it to select it and a small
// "W × H" control bar appears below it — type a size and press Apply, or
// leave both blank for natural size. That width/height is what gets saved
// into the ticket's description HTML, so the image renders at the exact
// same size later in the read-only detail view.
.resizable-image {
  position: relative;
  display: inline-block;
  max-width: 100%;
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
.resizable-image__controls {
  display: flex;
  align-items: center;
  gap: 0.4em;
  margin-top: 0.4em;
  padding: 0.35em 0.45em;
  background: $paper;
  border: 1px solid $line;
  border-radius: 8px;
  width: fit-content;
}
.resizable-image__field {
  display: flex;
  align-items: center;
  gap: 0.3em;
  font-size: 0.72em;
  font-weight: 700;
  color: $ink-3;

  input {
    width: 4.2em;
    border: 1px solid $line;
    border-radius: 5px;
    background: $card;
    color: $ink;
    font-size: 1em;
    font-family: inherit;
    padding: 0.3em 0.4em;
    outline: 0;
    -moz-appearance: textfield;

    &::-webkit-outer-spin-button,
    &::-webkit-inner-spin-button {
      -webkit-appearance: none;
      margin: 0;
    }
    &:focus {
      border-color: $signal;
      box-shadow: 0 0 0 2px $wash;
    }
  }
}
.resizable-image__times {
  color: $ink-3;
  font-size: 0.75em;
}
.resizable-image__apply,
.resizable-image__reset {
  border: 0;
  border-radius: 5px;
  font-size: 0.72em;
  font-weight: 700;
  padding: 0.35em 0.6em;
  cursor: pointer;
}
.resizable-image__apply {
  background: $signal;
  color: #fff;
  &:hover {
    background: $signal-2;
  }
}
.resizable-image__reset {
  background: transparent;
  color: $ink-3;
  &:hover {
    color: $ink;
    background: $ink-wash;
  }
}
