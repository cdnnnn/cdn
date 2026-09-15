import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { ticketsApi } from '../../api/endpoints/tickets';
import type {
  Ticket,
  TicketStatus,
  TicketResolution,
  CreateTicketRequest,
  UpdateTicketRequest,
  AddCommentRequest,
} from '../../types/tickets';

type FetchStatus = 'idle' | 'loading' | 'succeeded' | 'failed';

interface MoveArg {
  id: string;
  status: TicketStatus;
  resolution?: TicketResolution | null;
}

interface TicketsState {
  items: Ticket[];
  status: FetchStatus;
  error: string | null;
  creating: boolean;
  updatingId: string | null;
  deletingId: string | null;
  commentingId: string | null;
  // Ids currently mid-move: optimistically applied in `pending`, confirmed in
  // `fulfilled`, rolled back in `rejected`.
  movingIds: string[];
  // Snapshot of {status, resolution} captured at move-start, keyed by id, so a
  // failed transition can be reverted to exactly where the card came from.
  rollback: Record<string, { status: TicketStatus; resolution?: TicketResolution | null }>;
}

const initialState: TicketsState = {
  items: [],
  status: 'idle',
  error: null,
  creating: false,
  updatingId: null,
  deletingId: null,
  commentingId: null,
  movingIds: [],
  rollback: {},
};

export const fetchTickets = createAsyncThunk('tickets/fetchAll', () => ticketsApi.list());

export const createTicket = createAsyncThunk(
  'tickets/create',
  (payload: CreateTicketRequest) => ticketsApi.create(payload)
);

export const updateTicket = createAsyncThunk(
  'tickets/update',
  (payload: UpdateTicketRequest) => ticketsApi.update(payload)
);

// The board moves the card the instant you drop it (see `pending` below) and
// only reconciles with the server response afterward, so drag-and-drop feels
// immediate. A rejection snaps it back.
export const moveTicket = createAsyncThunk(
  'tickets/move',
  (payload: MoveArg) => ticketsApi.move(payload)
);

export const deleteTicket = createAsyncThunk(
  'tickets/delete',
  async (id: string) => {
    const res = await ticketsApi.remove(id);
    return { id: res.id || id };
  }
);

// The comment endpoint's response shape can vary across backends (full
// updated ticket vs. just the created comment vs. some other envelope) —
// rather than trust it and upsert `action.payload` directly (which silently
// does nothing if that assumption is wrong, leaving the new comment
// invisible until something else refetches), refetch the authoritative
// list once the post succeeds. Same "mutate, then refetch" pattern already
// used by createCustomModel elsewhere in this app.
export const addTicketComment = createAsyncThunk(
  'tickets/addComment',
  async (payload: AddCommentRequest, { dispatch }) => {
    await ticketsApi.addComment(payload);
    await dispatch(fetchTickets());
  }
);

const upsert = (list: Ticket[], t: Ticket) => {
  const i = list.findIndex((x) => x.id === t.id);
  if (i === -1) return [t, ...list];
  const next = list.slice();
  next[i] = t;
  return next;
};

const ticketsSlice = createSlice({
  name: 'tickets',
  initialState,
  reducers: {},
  extraReducers: (builder) => {
    builder
      // ---- fetch ----------------------------------------------------------
      .addCase(fetchTickets.pending, (state) => {
        state.status = 'loading';
      })
      .addCase(fetchTickets.fulfilled, (state, action) => {
        state.status = 'succeeded';
        state.items = action.payload ?? [];
      })
      .addCase(fetchTickets.rejected, (state, action) => {
        state.status = 'failed';
        state.error = action.error.message || 'Failed to load tickets';
      })

      // ---- create ---------------------------------------------------------
      .addCase(createTicket.pending, (state) => {
        state.creating = true;
      })
      .addCase(createTicket.fulfilled, (state, action) => {
        state.creating = false;
        state.items = upsert(state.items, action.payload);
      })
      .addCase(createTicket.rejected, (state, action) => {
        state.creating = false;
        state.error = action.error.message || 'Failed to create ticket';
      })

      // ---- update (metadata) ---------------------------------------------
      .addCase(updateTicket.pending, (state, action) => {
        state.updatingId = action.meta.arg.id;
      })
      .addCase(updateTicket.fulfilled, (state, action) => {
        state.updatingId = null;
        state.items = upsert(state.items, action.payload);
      })
      .addCase(updateTicket.rejected, (state, action) => {
        state.updatingId = null;
        state.error = action.error.message || 'Failed to update ticket';
      })

      // ---- move (optimistic) ---------------------------------------------
      .addCase(moveTicket.pending, (state, action) => {
        const { id, status, resolution } = action.meta.arg;
        const t = state.items.find((x) => x.id === id);
        if (!t) return;
        state.rollback[id] = { status: t.status, resolution: t.resolution ?? null };
        t.status = status;
        t.resolution = status === 'done' ? resolution ?? 'completed' : null;
        if (!state.movingIds.includes(id)) state.movingIds.push(id);
      })
      .addCase(moveTicket.fulfilled, (state, action) => {
        const { id } = action.meta.arg;
        state.movingIds = state.movingIds.filter((x) => x !== id);
        delete state.rollback[id];
        state.items = upsert(state.items, action.payload); // trust the server copy
      })
      .addCase(moveTicket.rejected, (state, action) => {
        const { id } = action.meta.arg;
        state.movingIds = state.movingIds.filter((x) => x !== id);
        const snap = state.rollback[id];
        const t = state.items.find((x) => x.id === id);
        if (t && snap) {
          t.status = snap.status;
          t.resolution = snap.resolution ?? null;
        }
        delete state.rollback[id];
        state.error = action.error.message || 'Failed to move ticket';
      })

      // ---- delete ---------------------------------------------------------
      .addCase(deleteTicket.pending, (state, action) => {
        state.deletingId = action.meta.arg;
      })
      .addCase(deleteTicket.fulfilled, (state, action) => {
        state.deletingId = null;
        state.items = state.items.filter((m) => m.id !== action.payload.id);
      })
      .addCase(deleteTicket.rejected, (state, action) => {
        state.deletingId = null;
        state.error = action.error.message || 'Failed to delete ticket';
      })

      // ---- add comment ------------------------------------------------------
      .addCase(addTicketComment.pending, (state, action) => {
        state.commentingId = action.meta.arg.ticket_id;
      })
      .addCase(addTicketComment.fulfilled, (state) => {
        state.commentingId = null;
        // state.items is already up to date — the thunk dispatched
        // fetchTickets() internally, and that action's own .fulfilled
        // reducer (above) already replaced state.items.
      })
      .addCase(addTicketComment.rejected, (state, action) => {
        state.commentingId = null;
        state.error = action.error.message || 'Failed to post comment';
      });
  },
});

export default ticketsSlice.reducer;
