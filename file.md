import { useCallback, useRef, type PointerEvent as ReactPointerEvent } from 'react';
import Image from '@tiptap/extension-image';
import { mergeAttributes } from '@tiptap/core';
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
        // Deliberately NOT relying on TipTap's per-attribute renderHTML
        // merge here (returning {} — i.e. "this attribute contributes
        // nothing on its own"). The width is instead written explicitly in
        // the node-level renderHTML() override below, which reads the raw
        // `node.attrs.width` directly — a more direct, unambiguous path
        // than depending on the attribute-merge internals to carry it
        // through correctly.
        renderHTML: () => ({}),
      },
    };
  },
  // Node-level override: builds the actual <img src="..." style="width:
  // ...px"> tag that `editor.getHTML()` serializes — this is exactly what
  // ends up saved in the ticket's `description` field, and exactly what
  // the read-only detail view renders back out later.
  renderHTML({ node, HTMLAttributes }) {
    const width = node.attrs.width as number | null;
    const attrs = mergeAttributes(this.options.HTMLAttributes, HTMLAttributes);
    if (width) {
      const existingStyle = typeof attrs.style === 'string' ? attrs.style : '';
      attrs.style = `width: ${width}px;${existingStyle ? ` ${existingStyle}` : ''}`;
    }
    return ['img', attrs];
  },
  addNodeView() {
    return ReactNodeViewRenderer(ResizableImageView);
  },
});
