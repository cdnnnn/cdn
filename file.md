// VirtualList.tsx
//
// Minimal fixed-row-height windowing, tuned for very large lists (10k+ rows).
//
//  - Only rows in/near the viewport are mounted.
//  - Scroll position lives in a ref; React re-renders only when the scroll
//    crosses a "block" boundary (BLOCK rows), not on every scroll frame. The
//    overscan is larger than a block, so the viewport is always covered
//    between re-renders — scrolling itself is handled by the browser.
//  - Rows are positioned with transform (compositor-friendly) and isolated
//    with CSS containment so one row's layout can't invalidate the others.
//  - overscroll-behavior: contain stops the scroll from chaining into the
//    outer page column when the list hits its top/bottom.
import { useCallback, useEffect, useReducer, useRef } from 'react';
import type { ReactNode } from 'react';

const BLOCK = 5; // rows per re-render step

interface VirtualListProps<T> {
  items: T[];
  // Height of one row in px, INCLUDING the gap below it.
  rowHeight: number;
  // Gap between rows in px (subtracted from the rendered row height).
  gap?: number;
  // The list grows with its content up to this height, then scrolls.
  maxHeight: number;
  // Extra rows mounted above/below the viewport. Keep >= BLOCK.
  overscan?: number;
  getKey: (item: T, index: number) => string | number;
  renderRow: (item: T, index: number) => ReactNode;
  className?: string;
}

export default function VirtualList<T>({
  items, rowHeight, gap = 0, maxHeight, overscan = 10, getKey, renderRow, className,
}: VirtualListProps<T>) {
  const ref = useRef<HTMLDivElement>(null);
  const rafRef = useRef<number | null>(null);
  const scrollTopRef = useRef(0);
  const blockRef = useRef(0);
  const [, rerender] = useReducer((n: number) => n + 1, 0);

  const totalHeight = Math.max(0, items.length * rowHeight - gap);
  const viewportHeight = Math.min(totalHeight, maxHeight);
  const visibleRows = Math.ceil(viewportHeight / rowHeight) + 1;

  const onScroll = useCallback(() => {
    if (rafRef.current !== null) return;
    rafRef.current = requestAnimationFrame(() => {
      rafRef.current = null;
      const el = ref.current;
      if (!el) return;
      scrollTopRef.current = el.scrollTop;
      const block = Math.floor(el.scrollTop / (rowHeight * BLOCK));
      if (block !== blockRef.current) {
        blockRef.current = block;
        rerender();
      }
    });
  }, [rowHeight]);

  useEffect(() => () => {
    if (rafRef.current !== null) cancelAnimationFrame(rafRef.current);
  }, []);

  // If the list shrinks, keep the scroll position valid.
  useEffect(() => {
    const el = ref.current;
    const max = Math.max(0, totalHeight - viewportHeight);
    if (el && el.scrollTop > max) {
      el.scrollTop = max;
      scrollTopRef.current = max;
      blockRef.current = Math.floor(max / (rowHeight * BLOCK));
      rerender();
    }
  }, [totalHeight, viewportHeight, rowHeight]);

  const startBlockRow = Math.floor(scrollTopRef.current / (rowHeight * BLOCK)) * BLOCK;
  const start = Math.max(0, startBlockRow - overscan);
  const end = Math.min(items.length, startBlockRow + BLOCK + visibleRows + overscan);

  const rows: ReactNode[] = [];
  for (let i = start; i < end; i++) {
    rows.push(
      <div
        key={getKey(items[i], i)}
        style={{
          position: 'absolute',
          top: 0,
          left: 0,
          right: 0,
          height: rowHeight - gap,
          transform: `translateY(${i * rowHeight}px)`,
          contain: 'layout paint style',
        }}
      >
        {renderRow(items[i], i)}
      </div>
    );
  }

  return (
    <div
      ref={ref}
      className={className}
      onScroll={onScroll}
      style={{
        height: viewportHeight,
        overflowY: 'auto',
        position: 'relative',
        overscrollBehavior: 'contain',
        contain: 'layout paint',
      }}
    >
      <div style={{ height: totalHeight, position: 'relative' }}>{rows}</div>
    </div>
  );
}































//ragslice.ts
import { createSlice, createAsyncThunk, original } from '@reduxjs/toolkit';
import { ragApi } from '../../api/endpoints/rag';
import type { CreateDatasetParams, RagFile, RagStatusResponse } from '../../api/endpoints/rag';

type AsyncStatus = 'idle' | 'loading' | 'succeeded' | 'failed';

// With thousands of files, every poll/refetch would otherwise replace every
// file object (and re-freeze/re-render all of them) even when nothing
// changed. Reuse the previous object for any file whose visible fields are
// unchanged, and reuse the previous ARRAY when nothing changed at all, so
// downstream memoization (React.memo rows, useMemo) can skip the work.
function reuseFiles(prev: RagFile[] | undefined, next: RagFile[]): RagFile[] {
  if (!prev || prev.length === 0) return next;
  const byId = new Map<number, RagFile>();
  for (const f of prev) byId.set(f.id, f);

  let identical = prev.length === next.length;
  const merged = next.map((f, i) => {
    const p = byId.get(f.id);
    const unchanged =
      !!p &&
      p.status === f.status &&
      p.error_message === f.error_message &&
      p.updated_at === f.updated_at &&
      p.filename === f.filename &&
      p.size === f.size;
    if (!unchanged || prev[i] !== p) identical = false;
    return unchanged ? p : f;
  });
  return identical ? prev : merged;
}

interface RagState {
  createStatus: AsyncStatus;
  createError: string | null;

  uploadedFiles: RagFile[];
  filesStatus: AsyncStatus;
  filesError: string | null;

  uploadStatus: AsyncStatus;
  uploadError: string | null;

  processingStartStatus: AsyncStatus;
  processingStartError: string | null;

  status: RagStatusResponse | null;
  statusFetchStatus: AsyncStatus;
  statusFetchError: string | null;
}

const initialState: RagState = {
  createStatus: 'idle',
  createError: null,

  uploadedFiles: [],
  filesStatus: 'idle',
  filesError: null,

  uploadStatus: 'idle',
  uploadError: null,

  processingStartStatus: 'idle',
  processingStartError: null,

  status: null,
  statusFetchStatus: 'idle',
  statusFetchError: null,
};

// POST /datasets/create (eval_type: rag) — returns the new dataset id.
export const createRagDataset = createAsyncThunk(
  'rag/create',
  async (params: CreateDatasetParams, { rejectWithValue }) => {
    try {
      return await ragApi.create(params);
    } catch (err) {
      const message = err instanceof Error ? err.message : 'Failed to create dataset';
      return rejectWithValue(message);
    }
  }
);

// GET /datasets/uploaded_files, filtered to this dataset.
export const fetchRagUploadedFiles = createAsyncThunk(
  'rag/fetchUploadedFiles',
  async (datasetId: string, { rejectWithValue }) => {
    try {
      return await ragApi.listUploadedFiles(datasetId);
    } catch (err) {
      const message = err instanceof Error ? err.message : 'Failed to load uploaded files';
      return rejectWithValue(message);
    }
  }
);

// POST /datasets/upload-rag?dataset_id= — one batch of files.
export const uploadRagFiles = createAsyncThunk(
  'rag/uploadFiles',
  async ({ datasetId, files }: { datasetId: string; files: File[] }, { rejectWithValue }) => {
    try {
      await ragApi.uploadFiles(datasetId, files);
    } catch (err) {
      const message = err instanceof Error ? err.message : 'Failed to upload files';
      return rejectWithValue(message);
    }
  }
);

// GET /datasets/rag_processing?dataset_id=
export const startRagProcessing = createAsyncThunk(
  'rag/startProcessing',
  async (datasetId: string, { rejectWithValue }) => {
    try {
      await ragApi.startProcessing(datasetId);
    } catch (err) {
      const message = err instanceof Error ? err.message : 'Failed to start processing';
      return rejectWithValue(message);
    }
  }
);

// GET /datasets/status?dataset_id= — polled while the studio is open.
export const fetchRagStatus = createAsyncThunk(
  'rag/fetchStatus',
  async (datasetId: string, { rejectWithValue }) => {
    try {
      return await ragApi.fetchStatus(datasetId);
    } catch (err) {
      const message = err instanceof Error ? err.message : 'Failed to load processing status';
      return rejectWithValue(message);
    }
  }
);

const ragSlice = createSlice({
  name: 'rag',
  initialState,
  reducers: {
    resetRagCreate: (state) => {
      state.createStatus = 'idle';
      state.createError = null;
    },
    // Called when the studio closes — wipes everything scoped to the
    // dataset that was open so a later session starts clean.
    resetRagStudio: (state) => {
      state.uploadedFiles = [];
      state.filesStatus = 'idle';
      state.filesError = null;
      state.uploadStatus = 'idle';
      state.uploadError = null;
      state.processingStartStatus = 'idle';
      state.processingStartError = null;
      state.status = null;
      state.statusFetchStatus = 'idle';
      state.statusFetchError = null;
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(createRagDataset.pending, (state) => {
        state.createStatus = 'loading';
        state.createError = null;
      })
      .addCase(createRagDataset.fulfilled, (state) => {
        state.createStatus = 'succeeded';
      })
      .addCase(createRagDataset.rejected, (state, action) => {
        state.createStatus = 'failed';
        state.createError = (action.payload as string) || action.error.message || 'Failed to create dataset';
      })

      .addCase(fetchRagUploadedFiles.pending, (state) => {
        state.filesStatus = 'loading';
        state.filesError = null;
      })
      .addCase(fetchRagUploadedFiles.fulfilled, (state, action) => {
        state.filesStatus = 'succeeded';
        const prevFiles = original(state)?.uploadedFiles;
        state.uploadedFiles = reuseFiles(prevFiles, Array.isArray(action.payload) ? action.payload : []);
      })
      .addCase(fetchRagUploadedFiles.rejected, (state, action) => {
        state.filesStatus = 'failed';
        state.filesError = (action.payload as string) || action.error.message || 'Failed to load uploaded files';
      })

      .addCase(uploadRagFiles.pending, (state) => {
        state.uploadStatus = 'loading';
        state.uploadError = null;
      })
      .addCase(uploadRagFiles.fulfilled, (state) => {
        state.uploadStatus = 'succeeded';
      })
      .addCase(uploadRagFiles.rejected, (state, action) => {
        state.uploadStatus = 'failed';
        state.uploadError = (action.payload as string) || action.error.message || 'Failed to upload files';
      })

      .addCase(startRagProcessing.pending, (state) => {
        state.processingStartStatus = 'loading';
        state.processingStartError = null;
      })
      .addCase(startRagProcessing.fulfilled, (state) => {
        state.processingStartStatus = 'succeeded';
      })
      .addCase(startRagProcessing.rejected, (state, action) => {
        state.processingStartStatus = 'failed';
        state.processingStartError = (action.payload as string) || action.error.message || 'Failed to start processing';
      })

      .addCase(fetchRagStatus.pending, (state) => {
        // Deliberately doesn't flip statusFetchStatus to 'loading' on every
        // poll tick — only the very first fetch should show a loading
        // state; background polls shouldn't disrupt what's on screen.
        if (state.statusFetchStatus === 'idle') state.statusFetchStatus = 'loading';
      })
      .addCase(fetchRagStatus.fulfilled, (state, action) => {
        state.statusFetchStatus = 'succeeded';
        state.statusFetchError = null;
        if (!action.payload) {
          state.status = null;
        } else {
          const prevFiles = original(state)?.status?.files;
          state.status = {
            ...action.payload,
            files: reuseFiles(prevFiles, Array.isArray(action.payload.files) ? action.payload.files : []),
          };
        }
      })
      .addCase(fetchRagStatus.rejected, (state, action) => {
        // Same reasoning — don't blow away a working status view just
        // because one background poll failed transiently.
        if (!state.status) {
          state.statusFetchStatus = 'failed';
          state.statusFetchError = (action.payload as string) || action.error.message || 'Failed to load status';
        }
      });
  },
});

export const { resetRagCreate, resetRagStudio } = ragSlice.actions;
export default ragSlice.reducer;
