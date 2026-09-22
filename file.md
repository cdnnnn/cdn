npm install @dnd-kit/core @tiptap/react @tiptap/pm @tiptap/starter-kit @tiptap/extension-image @tiptap/extension-placeholder dompurify








import { useCallback, useRef, type PointerEvent as ReactPointerEvent } from 'react';
import Image from '@tiptap/extension-image';
import { NodeViewWrapper, ReactNodeViewRenderer, type NodeViewProps } from '@tiptap/react';
import styles from './TicketDescriptionEditor.module.scss';

// ─────────────────────────────────────────────────────────────────────────
// An Image node extended with persisted `width`/`height` and a drag-to-
// resize handle in the editor — Confluence-style: drop/paste/insert an
// image at its natural size, then drag the corner handle to resize it.
//
// Both dimensions are stored as plain numeric HTML attributes on the <img>
// itself — `width="450" height="300"`, not a CSS style — the standard
// approach for a responsive image that still reserves its exact box.
// Combined with `max-width: 100%; height: auto;` in this file's CSS (and
// the identical rule in TicketBoard.module.scss for the read-only view),
// the browser renders it at exactly that saved width × height wherever
// there's room, and only scales both dimensions down together — preserving
// the same aspect ratio — if the container is narrower than that.
//
// IMPORTANT: `width`/`height` are contributed via each attribute's own
// `renderHTML`, NOT by overriding the node's own `renderHTML` — the node-
// level renderHTML is left completely alone (inherited from the base Image
// extension), so `src`/`alt` rendering is untouched. An earlier version of
// this file replaced the node-level renderHTML entirely, which broke image
// rendering outright — this per-attribute approach is the standard,
// documented way to add an attribute to an existing TipTap node without
// risking exactly that regression.
//
// Height is never resized independently — it's always derived from the
// image's own natural aspect ratio (naturalWidth/naturalHeight) at the
// moment you drag, so a resize can never stretch or squash the image.
// ─────────────────────────────────────────────────────────────────────────

const MIN_WIDTH = 80;

function ResizableImageView({ node, updateAttributes, selected }: NodeViewProps) {
  const { src, alt, width, height } = node.attrs as {
    src: string;
    alt?: string;
    width?: number | null;
    height?: number | null;
  };
  const imgRef = useRef<HTMLImageElement | null>(null);
  const dragRef = useRef<{ startX: number; startWidth: number; ratio: number | null } | null>(null);

  const onHandlePointerDown = useCallback(
    (e: ReactPointerEvent) => {
      e.preventDefault();
      e.stopPropagation();
      const img = imgRef.current;
      const startWidth = img?.getBoundingClientRect().width ?? width ?? 300;
      // Locked in once at drag-start from the image's real decoded pixel
      // dimensions — every width during this drag derives its paired
      // height from this same ratio, so the image is never stretched.
      const ratio = img && img.naturalWidth ? img.naturalHeight / img.naturalWidth : null;
      dragRef.current = { startX: e.clientX, startWidth, ratio };

      const onMove = (ev: PointerEvent) => {
        if (!dragRef.current) return;
        const delta = ev.clientX - dragRef.current.startX;
        const nextWidth = Math.max(MIN_WIDTH, Math.round(dragRef.current.startWidth + delta));
        const nextHeight = dragRef.current.ratio ? Math.round(nextWidth * dragRef.current.ratio) : null;
        updateAttributes({ width: nextWidth, height: nextHeight });
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
        width={width ?? undefined}
        height={height ?? undefined}
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
