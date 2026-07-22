---
name: helpdesk-api
description: >-
  Interact with the live helpdesk.artumi.com ticketing system via its HTTP
  API — fetch a ticket/issue by number, list issues for a client/queue/tag,
  pick up/drop a ticket's worker, reply to a ticket, and change its
  status/priority/owner/queue. Use
  whenever the user references a ticket code like "ART1234" (unambiguous —
  always this system), or explicitly says "helpdesk"/"helpdesk ticket". A
  bare number or the word "ticket"/"issue" alone is NOT enough to trigger
  this in a project that has its own tracker (GitHub issues, Jira, etc.) —
  see Disambiguation below. Works from any project, not just the helpdesk
  app's own repo. Distinct from working on the helpdesk app's source code —
  this is for using the deployed app as an end user would, through its API.
---

# Helpdesk API

Interact with the production helpdesk at `https://helpdesk.artumi.com` through its REST API.
Full endpoint reference (all fields, error codes, examples) is in
`references/api.md` alongside this file — read it if you need detail beyond the summary below.

## Authentication

The API key lives at `~/vault/.helpdesk-artumi.com-key`. That directory is an encrypted vault
mount and **may not be mounted** in the current session.

```bash
API_KEY=$(cat ~/vault/.helpdesk-artumi.com-key 2>/dev/null | head -1)
```

If the file isn't there or is empty, the vault is locked — tell the user, verbatim:

> UNLOCK TEH VAULT

...then wait for them to mount it before retrying. Don't guess at the key or invent a
workaround.

Read the key fresh each time rather than caching it in the conversation, and never print the
raw key value in output.

Send it as `Authorization: Bearer $API_KEY`.

## Disambiguation

A code matching a short letter-prefix + number (e.g. `ART1234`, or the older `ISS-4290` style —
usually `ART` in practice) unambiguously means this helpdesk. Treat it as a definite match, no
need to ask.

A bare number or the generic word "ticket"/"issue" is ambiguous in any project that has its own
tracker (GitHub issues, Jira, etc.). In that case, don't assume — ask the user which system they
mean before calling the API. Outside such a project (or once the user has said "helpdesk"
explicitly), it's safe to proceed.

## Ticket numbers vs issue ids

Users will refer to tickets by their code (e.g. `ART5123`) or sometimes just the bare number.
The API's `GET /api/issues` only filters by numeric `id`, not by code — strip the letter prefix
and use the trailing number as `id` unless that fails, in which case ask the user to confirm the
id.

## Common operations

Get a ticket by id:

```bash
curl -s -H "Authorization: Bearer $API_KEY" \
  "https://helpdesk.artumi.com/api/issues?id=4290"
```

List issues in a queue (add `&limit=N` to cap results, default 50 max 200):

```bash
curl -s -H "Authorization: Bearer $API_KEY" \
  "https://helpdesk.artumi.com/api/issues?queue=44"
```

Reply to a ticket (markdown body):

```bash
curl -s -X POST \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"body":"Reply text here."}' \
  "https://helpdesk.artumi.com/api/issues/4290/messages"
```

Reply and close/change status in the same call — `status` is a word (case-insensitive), not a
number; check the ticket's `status` field via a `GET` first if you're unsure of the exact wording
(e.g. "Fixed", "Cant fix", "Wont fix", "Wishlist", "Duplicate", "Old"):

```bash
curl -s -X POST \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"body":"Fixed and confirmed working.","status":"Fixed"}' \
  "https://helpdesk.artumi.com/api/issues/4290/messages"
```

Change status/priority/owner without adding a reply — `priority` is also a word
(Low/Medium/High/Critical), and `owner` takes a username, not a numeric id:

```bash
curl -s -X PATCH \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"status":"Fixed","priority":"Critical","owner":"jdoe"}' \
  "https://helpdesk.artumi.com/api/issues?id=4290"
```

Create a new ticket — `queueid` is the queue's name (case-insensitive), not its numeric id:

```bash
curl -s -X POST \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"issuename":"Title","body":"Opening message","queueid":"Support"}' \
  "https://helpdesk.artumi.com/api/issues"
```

`worker_id` (who is actively working the ticket right now, distinct from `owner_id` who's
case-managing it) is not settable via `PATCH` — it has its own pick-up/drop endpoints:

Pick up a ticket before working on it — claims `worker_id` for the API key's user:

```bash
curl -s -X POST -H "Authorization: Bearer $API_KEY" \
  "https://helpdesk.artumi.com/api/issues/4290/pickup"
```

If someone else already has it, this returns `409 conflict` rather than clobbering their claim.
Don't silently retry with `steal` — tell the user someone else is already working the ticket and
let them decide. Only steal if they explicitly say to take it over (e.g. that person is out/gone):

```bash
curl -s -X POST -H "Authorization: Bearer $API_KEY" -H "Content-Type: application/json" \
  -d '{"steal":true}' "https://helpdesk.artumi.com/api/issues/4290/pickup"
```

Drop it when you're done (also happens automatically if you close the ticket — any status change
away from `"Open"` clears `worker_id` for you):

```bash
curl -s -X POST -H "Authorization: Bearer $API_KEY" \
  "https://helpdesk.artumi.com/api/issues/4290/drop"
```

A `GET`/`POST`/`PATCH` response's `worked` (bool) and `worked_by_you` (bool) fields tell you
whether a ticket is claimed and by whom, without having to inspect the raw `worker_id`.

## Workflow for "get ticket N and work on it"

1. `GET /api/issues?id=N` to fetch full details (status, priority, owner, worked/worker, timestamps).
2. Summarize the ticket for the user in plain language before taking any action — don't reply
   or change status unprompted.
3. Once the user decides to actually act on the ticket (not just look at it), `POST
   /api/issues/{id}/pickup` first, before replying/closing/reassigning — this is what "working on
   a ticket" means to the API and prevents someone else picking it up underneath you. Skip this
   for read-only asks ("what's the status of ART4290?").
   - If pickup 409s, tell the user who's already on it (from the `GET`) rather than stealing
     automatically.
4. When the user decides what to do (reply, reassign owner, close, reprioritize), make the
   corresponding call and show the resulting `issue` object back so they can confirm it worked.
5. Replying and changing status/queue can be done in one `POST /messages` call — prefer that
   over two separate requests when the user wants both. Closing the ticket this way also drops
   `worker_id` automatically, so there's no need for a separate `drop` call in that case.

## Errors

`{"error": {"code": "...", "message": "..."}}` with a matching HTTP status — `401` means the key
is missing/invalid, `403` a permissions issue on the queue, `404` the id doesn't exist or isn't
visible to this key's user, `400` a malformed request. Surface the `message` to the user rather
than just the status code.
