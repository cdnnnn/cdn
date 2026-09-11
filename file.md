1. List tickets

GET /tickets

Returns every ticket the current user can see. The frontend currently fetches the full list once and does search/priority filtering client-side — if the dataset grows large, this endpoint can add ?search= / ?priority= / pagination params later without any other contract change.

Request: no body, no query params required.

Response 200

json
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
2. Create ticket

POST /tickets

The caller becomes the ticket's owner server-side — the frontend never sends an owner id, and the backend should ignore one if it's ever present in the body (never trust a client-supplied owner).

Request

json
{
  "title": "Add dark-mode toggle to the settings page",
  "description": "Persist the choice per-user and respect prefers-color-scheme on first load.",
  "priority": "medium",
  "labels": ["frontend", "design-system"],
  "assignee_id": "u2",
  "attachments": [
    { "id": "att_9f2a", "name": "mockup.png", "url": "https://cdn.example.com/uploads/att_9f2a.png", "size": 84213 }
  ]
}

All fields except title and priority are optional. attachments, when present, is the array of already-uploaded attachment objects returned by POST /uploads/images (see §6) — the client uploads images first, then references them here by the metadata that upload returned. assignee_id may be null to explicitly leave unassigned.

Response 201 — the full created ticket, with server-assigned id, key, owner, timestamps, status: "todo", resolution: null:

json
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
  "attachments": [
    { "id": "att_9f2a", "name": "mockup.png", "url": "https://cdn.example.com/uploads/att_9f2a.png", "size": 84213 }
  ],
  "comments": [],
  "created_at": "2026-09-11T10:15:00Z",
  "updated_at": "2026-09-11T10:15:00Z"
}

Errors

Status	When
400	title missing/blank, or priority not one of the four valid values
401	not authenticated
3. Update ticket metadata

PATCH /tickets/:id

Edits title/description/priority/labels/assignee/attachments. Does not change status/resolution — that's §4. Any field omitted from the body is left unchanged (partial update, not a full replace).

Request

json
{
  "title": "Add dark-mode toggle to settings",
  "priority": "high",
  "assignee_id": null
}

Response 200 — the full updated ticket (same shape as §2's response).

Errors

Status	When
400	invalid priority value
403	caller lacks permission to edit this ticket (if your app restricts editing beyond just owner-closes-done)
404	no ticket with that id
4. Move ticket status — the permission-critical endpoint

PATCH /tickets/:id/status

Dedicated endpoint (separate from §3) specifically so the backend can apply one rule cleanly: only the ticket's owner may set status: "done". Anyone may move a ticket among todo / in_progress / in_review. The frontend's UI already gates this (drag-and-drop, the card menu, and the detail view's close buttons all disable/reject the action for non-owners) — that UI gate is a convenience only; this endpoint must re-check ownership itself, since the UI can be bypassed by calling the API directly.

Request — moving to a non-terminal column:

json
{ "status": "in_review" }

Request — closing the ticket (owner only):

json
{ "status": "done", "resolution": "completed" }

resolution is "completed" or "discarded"; required when status is "done", ignored/omitted otherwise. Moving out of done back to an earlier column (if your workflow allows that) should null out resolution.

Response 200 — the full updated ticket.

Errors

Status	When
400	status not a valid value, or status: "done" sent without resolution
403	caller is not the ticket's owner and status is "done" — this is the one that matters most
404	no ticket with that id
json
{ "message": "Only the requester can close this ticket." }
5. Delete ticket

DELETE /tickets/:id

Request: no body.

Response 200

json
{ "status": "ok", "id": "t1" }

Errors: 403 if the caller isn't allowed to delete (decide your own policy here — the frontend doesn't currently restrict who sees the delete button beyond normal access to the detail view); 404 if not found.

6. Upload an image attachment

POST /uploads/images

multipart/form-data, single field named file. Called immediately when a user picks an image in the create/edit form — before the ticket itself is created or saved — so the upload can complete in the background while they keep filling out the rest of the form. The returned object is then included verbatim in the attachments array of §2/§3's request body.

Request (multipart/form-data)

POST /uploads/images
Content-Type: multipart/form-data; boundary=...

------WebKitFormBoundary
Content-Disposition: form-data; name="file"; filename="mockup.png"
Content-Type: image/png

<binary image bytes>
------WebKitFormBoundary--

Response 201

json
{
  "id": "att_9f2a",
  "name": "mockup.png",
  "url": "https://cdn.example.com/uploads/att_9f2a.png",
  "size": 84213
}

Errors

Status	When
400	not an image content-type, or missing file field
413	file too large (frontend already rejects anything over 5MB client-side, but enforce server-side too — never trust the client-side check alone)
7. Add a comment

POST /tickets/:id/comments

The comment's author is the authenticated caller, set server-side — the request body only carries the text.

Request

json
{ "text": "Looks good — can we also cover the Safari edge case from REQ-4?" }

Response 201 — recommended: return the full updated ticket (so the frontend's existing "upsert the whole ticket into state" pattern needs no special-casing for comments):

json
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

If returning the full ticket is expensive (e.g. it has many comments already), returning just the created TicketComment object is also fine — flag it and I'll adjust addTicketComment's reducer in ticketsSlice.ts to append rather than upsert-whole-ticket.

Errors: 400 if text is blank; 404 if the ticket doesn't exist.

8. Team roster (for the assignee picker)

GET /users

Response 200

json
{
  "users": [
    { "id": "u1", "name": "Ava Patel" },
    { "id": "u2", "name": "Marcus Lee" },
    { "id": "u3", "name": "Sofia Nguyen" },
    { "id": "u4", "name": "Jordan Reyes" }
  ]
}

If the app already has a "list teammates" endpoint under a different path (e.g. /team/members, /organization/users), point store/slices/usersSlice.ts at that instead of standing up a duplicate /users route.

Current user — no separate endpoint needed

The frontend does not call a dedicated "current user" endpoint for this feature. The app already authenticates via SSO (see authSlice.ts / ssoLogin) and stores the result at state.auth.user; TicketBoard.tsx reads that directly and adapts it to the { id, name } shape this feature needs (toTicketUser() in ticketMeta.ts). Nothing in this feature needs to log in or fetch identity on its own — it just needs ssoLogin to have already run, which it will have by the time a user can reach this route.

Summary table
#	Method	Path	Purpose
1	GET	/tickets	List all tickets
2	POST	/tickets	Create a ticket
3	PATCH	/tickets/:id	Edit ticket metadata
4	PATCH	/tickets/:id/status	Move status / close (owner-only for done)
5	DELETE	/tickets/:id	Delete a ticket
6	POST	/uploads/images	Upload an image attachment
7	POST	/tickets/:id/comments	Add a comment
8	GET	/users	Team roster for the assignee picker
