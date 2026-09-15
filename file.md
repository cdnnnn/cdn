# Requirements / Tickets — API Specification

Every endpoint the frontend calls, with sample request and response bodies. The
frontend's typed API layer (`api/tickets.ts`, `api/users.ts`, `api/uploads.ts`)
is written against exactly these shapes — if the backend needs to deviate,
update the corresponding `.ts` file to match rather than the other way round.

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

// Returned only by POST /uploads/images (§6) — NOT a field on Ticket. The
// description editor uploads an image the instant it's pasted/dropped/
// inserted and embeds the returned `url` directly as an <img> tag inside
// the ticket's `description` HTML — there is no separate attachments list.
interface TicketAttachment {
  id: string;
  name: string;
  url: string;   // hosted URL, not a data: or blob: URL
  size: number;  // bytes
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
  // lists, and inline <img> tags for any pasted/dropped/inserted images).
  // Sanitize before rendering — see the note under §2/§3.
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
      "description": "Persist the choice per-user and respect prefers-color-scheme on first load.",
      "status": "todo",
      "resolution": null,
      "priority": "medium",
      "owner": { "id": "u1", "name": "Ava Patel" },
      "assignee": { "id": "u2", "name": "Marcus Lee" },
      "labels": ["frontend", "design-system"],
      "attachments": [],
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
  "description": "<p>Persist the choice per-user and respect <strong>prefers-color-scheme</strong> on first load.</p><img src=\"https://cdn.example.com/uploads/att_9f2a.png\" alt=\"mockup.png\">",
  "priority": "medium",
  "labels": ["frontend", "design-system"],
  "assignee_id": "u2"
}
```

All fields except `title` and `priority` are optional. `description` is
**HTML**, not plain text — it comes straight from the frontend's rich-text
editor and can contain `<p>`, `<strong>`, `<em>`, `<ul>`/`<ol>`/`<li>`, and
inline `<img>` tags. Any images the user pasted, dragged in, or inserted via
the editor's toolbar were already uploaded via `POST /uploads/images` (§6)
the moment they were added — their `<img src="...">` already points at a
real hosted URL by the time this request is sent. **Sanitize `description`
before storing/re-serving it** (e.g. strip `<script>`, event handler
attributes, `javascript:` URLs) since it's user-supplied HTML — the frontend
also sanitizes on render (via DOMPurify) as defense in depth, but that isn't
a substitute for server-side sanitization. `assignee_id` may be `null` to
explicitly leave unassigned.

**Response 201** — the full created ticket, with server-assigned `id`,
`key`, `owner`, timestamps, `status: "todo"`, `resolution: null`:

```json
{
  "id": "t1",
  "key": "REQ-1",
  "title": "Add dark-mode toggle to the settings page",
  "description": "<p>Persist the choice per-user and respect <strong>prefers-color-scheme</strong> on first load.</p><img src=\"https://cdn.example.com/uploads/att_9f2a.png\" alt=\"mockup.png\">",
  "status": "todo",
  "resolution": null,
  "priority": "medium",
  "owner": { "id": "u1", "name": "Ava Patel" },
  "assignee": { "id": "u2", "name": "Marcus Lee" },
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

---

## 3. Update ticket metadata

`PATCH /tickets/:id`

Edits title/description/priority/labels/assignee. Does **not** change
`status`/`resolution` — that's §4. Any field omitted from the body is left
unchanged (partial update, not a full replace). As with create (§2),
`description` is sanitized HTML, not plain text — same rules apply.

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

---

## 4. Move ticket status — **the permission-critical endpoint**

`PATCH /tickets/:id/status`

Dedicated endpoint (separate from §3) specifically so the backend can apply
one rule cleanly: **only the ticket's `owner` may set `status: "done"`**.
Anyone may move a ticket among `todo` / `in_progress` / `in_review`. The
frontend's UI already gates this (drag-and-drop, the card menu, and the
detail view's close buttons all disable/reject the action for non-owners) —
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

**Request**: no body.

**Response 200**

```json
{ "status": "ok", "id": "t1" }
```

**Errors**: `403` if the caller isn't allowed to delete (decide your own
policy here — the frontend doesn't currently restrict who sees the delete
button beyond normal access to the detail view); `404` if not found.

---

## 6. Upload an image

`POST /uploads/images`

`multipart/form-data`, single field named `file`. There is **no separate
"attachments" UI** — this is called directly by the description editor the
instant a user **pastes** an image from the clipboard, **drags** one in from
their desktop, or picks one via the editor's toolbar image button. It
uploads in the background while a local preview shows immediately in the
editor; once this endpoint responds, the returned `url` replaces the local
preview and becomes the `src` of an `<img>` tag embedded directly in the
ticket's `description` HTML (see §2/§3) — the response is never stored or
referenced separately from that HTML.

**Request** (`multipart/form-data`)

```
POST /uploads/images
Content-Type: multipart/form-data; boundary=...

------WebKitFormBoundary
Content-Disposition: form-data; name="file"; filename="mockup.png"
Content-Type: image/png

<binary image bytes>
------WebKitFormBoundary--
```

**Response 201**

```json
{
  "id": "att_9f2a",
  "name": "mockup.png",
  "url": "https://cdn.example.com/uploads/att_9f2a.png",
  "size": 84213
}
```

`id` and `size` aren't used by the frontend beyond this response (there's no
attachments list to key them into) — `url` is the only field that actually
ends up persisted, embedded in the description HTML. Still worth returning
all three, in case that changes later.

**Errors**

| Status | When |
| --- | --- |
| 400 | not an image content-type, or missing `file` field |
| 413 | file too large (frontend already rejects anything over 5MB client-side, but enforce server-side too — never trust the client-side check alone) |

---

## 7. Add a comment

`POST /tickets/:id/comments`

The comment's `author` is the authenticated caller, set server-side — the
request body only carries the text.

**Request**

```json
{ "text": "Looks good — can we also cover the Safari edge case from REQ-4?" }
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
  "owner": { "id": "u2", "name": "Marcus Lee" },
  "assignee": { "id": "u1", "name": "Ava Patel" },
  "labels": ["bug", "models"],
  "attachments": [],
  "comments": [
    {
      "id": "c1",
      "author": { "id": "u1", "name": "Ava Patel" },
      "text": "Looks good — can we also cover the Safari edge case from REQ-4?",
      "created_at": "2026-09-11T10:20:00Z"
    }
  ],
  "created_at": "2026-08-18T14:20:00Z",
  "updated_at": "2026-09-11T10:20:00Z"
}
```

> If returning the full ticket is expensive (e.g. it has many comments
> already), returning just the created `TicketComment` object is also fine
> — flag it and I'll adjust `addTicketComment`'s reducer in
> `ticketsSlice.ts` to append rather than upsert-whole-ticket.

**Errors**: `400` if `text` is blank; `404` if the ticket doesn't exist.

---

## 8. Team roster (for the assignee picker)

`GET /users`

**Response 200**

```json
{
  "users": [
    { "id": "u1", "name": "Ava Patel" },
    { "id": "u2", "name": "Marcus Lee" },
    { "id": "u3", "name": "Sofia Nguyen" },
    { "id": "u4", "name": "Jordan Reyes" }
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
its `id` field, not a separate internal numeric/UUID user id. If the backend
identifies users differently internally, populate `TicketUser.id` with the
user's username specifically when serializing these fields, or the owner
check will never match for anyone.

---

## Summary table

| # | Method | Path | Purpose |
| --- | --- | --- | --- |
| 1 | GET | `/tickets` | List all tickets |
| 2 | POST | `/tickets` | Create a ticket |
| 3 | PATCH | `/tickets/:id` | Edit ticket metadata |
| 4 | PATCH | `/tickets/:id/status` | Move status / close (owner-only for `done`) |
| 5 | DELETE | `/tickets/:id` | Delete a ticket |
| 6 | POST | `/uploads/images` | Upload an image attachment |
| 7 | POST | `/tickets/:id/comments` | Add a comment |
| 8 | GET | `/users` | Team roster for the assignee picker |
