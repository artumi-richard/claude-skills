# Issues API

HTTP API for issues (tickets). Supports fetching issues (`GET /api/issues`),
creating a new issue (`POST /api/issues`), updating an issue's status,
priority or owner (`PATCH /api/issues`), picking up or dropping an issue's
worker (`POST /api/issues/{id}/pickup`, `POST /api/issues/{id}/drop`), and
adding a reply to an existing issue (`POST /api/issues/{id}/messages`).

## Authentication

Every request must include an API key, either as a bearer token or a custom
header:

```
Authorization: Bearer <key>
```

or

```
X-API-Key: <key>
```

A key authenticates as a specific user, and results are scoped exactly as
they would be for that user in the web UI (queue visibility, admin vs.
non-admin, etc.) — the API does not expose anything the underlying user
account couldn't already see.

### Minting a key

Keys are created via a CLI script (no admin UI yet):

```
php arcode/modules/HelpDesk/scripts/CreateApiKey.php <username> [label]
```

This prints the raw key once. It's stored only as a hash, so if you lose it
you must mint a new one.

## `GET /api/issues`

Exactly one of the following query parameters is required:

| Param        | Type    | Description                                                  |
|--------------|---------|----------------------------------------------------------------|
| `id`         | integer | Fetch a single issue by its issue id                          |
| `client`     | integer | List issues belonging to a client id                          |
| `queue`      | integer | List issues in a queue id                                      |
| `tag`        | string  | List issues with a given tag                                   |
| `worked_by`  | integer | List issues a user made a tracked change to (status, priority, worker, queue, hold date, critical-at, or a comment) on a given day |

`client`, `tag` and `worked_by` may be combined with `queue` in the same
request (they're ANDed together); `id` is exclusive — if present, everything
else is ignored.

An optional `date` parameter (`YYYY-MM-DD`) scopes `worked_by` to a specific
day — it's only valid alongside `worked_by` and defaults to today (server
local time) if omitted.

An optional `limit` parameter caps the number of results for list-style
requests (`client`/`queue`/`tag`/`worked_by`) — default 50, max 200. Results
are ordered newest first. `id` requests always return a single issue.

### Response shape

Success responses are always `{"issues": [...]}` — a single-element array for
`id` lookups, a list for filtered lookups.

```json
{
  "issues": [
    {
      "id": 4290,
      "name": "Printer not working",
      "code": "ISS-4290",
      "client_id": 2,
      "queue_id": 44,
      "worker_id": 9,
      "worked": true,
      "worked_by_you": false,
      "owner_id": 9,
      "priority": "Normal",
      "status": "Open",
      "type": 1,
      "hold_until": null,
      "merged_into": null,
      "created_at": "2010-11-25T14:10:00+00:00",
      "updated_at": "2010-11-26T09:02:15+00:00"
    }
  ]
}
```

Fields referencing other records (`client_id`, `queue_id`, `worker_id`,
`owner_id`) are raw ids, not resolved names. Timestamps are ISO 8601. Any of
the id fields, `hold_until`, and `merged_into` may be `null`. `status` and
`priority` are words (e.g. `"Open"`, `"Fixed"`, `"Critical"`) — there are no
numeric status/priority ids in the API.

`worked` is an explicit boolean mirroring `worker_id !== null` — someone is
actively working the issue right now. `worked_by_you` is `true` only when
that someone is the API key's own user. `worker_id` is not directly settable
via `PATCH` — see **Picking up and dropping an issue** below, which is the
only way to change it. Whenever an issue's `status` is changed away from
`"Open"` (via `PATCH` or `POST .../messages`), `worker_id` is automatically
cleared, the same as closing a ticket in the web UI.

### Errors

Errors are returned as `{"error": {"code": "...", "message": "..."}}` with a
matching HTTP status:

| Status | Code           | When                                                          |
|--------|----------------|----------------------------------------------------------------|
| 400    | `bad_request`  | None of `id`/`client`/`queue`/`tag`/`worked_by` supplied; `date` supplied without `worked_by`; invalid `worked_by` or `date` |
| 401    | `unauthorized` | Missing, invalid, or inactive API key                          |
| 403    | `forbidden`    | `queue` filter refers to a queue you don't have access to      |
| 404    | `not_found`    | `id` doesn't exist, or refers to an issue you can't see        |

`client`/`tag` filters that simply match nothing return `{"issues": []}`
rather than an error.

## `POST /api/issues`

Creates a new issue (ticket) plus its opening message. The issue is created as
the user the API key belongs to, in exactly the same way as the web "New
Ticket" form — so the key's user must have add permission on the target queue.

Send a JSON body with `Content-Type: application/json`. (Form-encoded bodies
with the same field names also work.)

| Field       | Type    | Required | Description                                             |
|-------------|---------|----------|---------------------------------------------------------|
| `issuename` | string  | yes      | Ticket title                                            |
| `body`      | string  | yes      | Opening message body (HTML allowed)                     |
| `queueid`   | string  | yes      | Name (case-insensitive) of the queue to create the ticket in — must be one you can add to |
| `clientid`  | integer | no       | Associate the ticket with a client                      |
| `priority`  | string  | no       | Priority word (case-insensitive) — Low/Medium/High/Critical; defaults to Medium |
| `owner`     | string  | no       | Username (case-insensitive) of the case-managing owner  |
| `tags`      | string  | no       | Comma-separated tags; lowercased and de-duplicated      |

Other fields (`worker`, `issuestatus`, `issuetype`, `issuecode`) are not
settable at creation time — new issues are always created Open, as a normal
ticket, with a server-generated code. `issuestatus` (along with `priority`
and `owner`) can be changed afterwards via `PATCH /api/issues` — see below.
`worker` is set via `POST /api/issues/{id}/pickup`, not at creation time —
see **Picking up and dropping an issue** below.

### Response

On success the endpoint returns `201 Created` with the newly created issue
under a singular `issue` key, in the same shape as a `GET` result element:

```json
{
  "issue": {
    "id": 5123,
    "name": "Printer not working",
    "code": "ART5123",
    "client_id": null,
    "queue_id": 44,
    "worker_id": null,
    "owner_id": null,
    "priority": "Medium",
    "status": "Open",
    "type": 1,
    "hold_until": "2026-07-10T14:10:00+00:00",
    "merged_into": null,
    "created_at": "2026-07-10T14:10:00+00:00",
    "updated_at": "2026-07-10T14:10:00+00:00"
  }
}
```

### Errors

Uses the same `{"error": {"code": "...", "message": "..."}}` shape as the GET
endpoint:

| Status | Code           | When                                                            |
|--------|----------------|----------------------------------------------------------------|
| 400    | `bad_request`  | Missing/blank `issuename`, `body` or `queueid`; invalid JSON body or priority |
| 401    | `unauthorized` | Missing, invalid, or inactive API key                          |
| 403    | `forbidden`    | You don't have add permission on the requested `queueid`       |

## `PATCH /api/issues`

Updates the owner, status and/or priority of an existing issue. (`PUT` is
also accepted as an alias for `PATCH`.) The key's user must have **manage**
permission on the issue's queue — the same permission that gates the
status/priority/owner controls in the web UI.

The issue id is passed as the `id` query parameter, e.g.
`PATCH /api/issues?id=4290`. Fields to change are sent in the request body —
send a JSON body with `Content-Type: application/json` (form-encoded bodies
with the same field names also work, except unassigning an owner, which
requires JSON `null` — see below).

| Field      | Type         | Description                                                        |
|------------|--------------|------------------------------------------------------------------------|
| `status`   | string       | New status word (case-insensitive) — one of the values from the `status` field on a `GET` response, e.g. `"Open"`, `"Fixed"` |
| `priority` | string       | New priority word (case-insensitive) — Low/Medium/High/Critical     |
| `owner`    | string\|null | Username (case-insensitive) of the case-managing owner. `null` (JSON only) unassigns the owner |

At least one of the three fields must be present. Only the fields supplied
are changed; omitted fields are left as-is. Every change is recorded in the
issue's history, same as an edit made through the web UI. There is no way to
(re)assign `worker` via `PATCH` — that's a separate pick-up/drop workflow,
not a plain reassignment field, described next. Setting `status` to anything
other than `"Open"` automatically clears `worker_id` if it was set.

### Response

On success returns `200 OK` with the updated issue under the `issue` key, in
the same shape as a `GET`/`POST` result.

### Errors

| Status | Code           | When                                                                 |
|--------|----------------|------------------------------------------------------------------------|
| 400    | `bad_request`  | Missing `id`; none of `status`/`priority`/`owner` supplied; invalid `status`, `priority` or `owner`; invalid JSON body |
| 401    | `unauthorized` | Missing, invalid, or inactive API key                                |
| 403    | `forbidden`    | You don't have manage permission on the issue's queue                |
| 404    | `not_found`    | `id` doesn't exist, or refers to an issue you can't see               |

## Picking up and dropping an issue

`worker_id` tracks who is *actively working* an issue right now, separately
from `owner_id` (who's case-managing it). Only one user can work an issue at
a time. If you're building something that acts on a ticket (replying,
investigating, fixing), pick it up first — this stops someone else picking
up the same ticket without realizing it's already being handled, and other
tools/users can see `worked`/`worked_by_you` on a `GET` to know it's spoken
for.

### `POST /api/issues/{id}/pickup`

Sets `worker_id` to the API key's own user. The key's user only needs
**view** access to the issue's queue (the same visibility check as `GET`).

If the issue is unworked, or already worked by you, this succeeds
immediately (repeat calls are safe — picking up your own ticket again is a
no-op). If it's currently worked by someone else, this fails with `409
conflict` unless you pass `"steal": true` in the JSON body, in which case
the key's user additionally needs **steal** permission on the issue's
queue — the same permission that gates the "Steal" confirmation in the web
UI. Stealing is meant for recovering a ticket someone picked up and never
dropped (e.g. they went home), not routine reassignment.

```json
{"steal": true}
```

`steal` is optional and defaults to `false`. Omit the body (or send `{}`)
for a normal pick-up.

#### Response

On success returns `200 OK` with the updated issue under the `issue` key.

#### Errors

| Status | Code           | When                                                                 |
|--------|----------------|------------------------------------------------------------------------|
| 400    | `bad_request`  | Missing/invalid `id`; invalid JSON body                              |
| 401    | `unauthorized` | Missing, invalid, or inactive API key                                |
| 403    | `forbidden`    | Issue is worked by someone else, `steal` was `true`, but you lack steal permission on the queue |
| 404    | `not_found`    | `id` doesn't exist, or refers to an issue you can't see               |
| 409    | `conflict`     | Issue is worked by someone else and `steal` wasn't `true`             |

### `POST /api/issues/{id}/drop`

Clears `worker_id`, releasing the issue for someone else to pick up. No
request body is needed. If nobody is currently working the issue, this is a
no-op (`200 OK`, not an error). If someone *else* is working it, this fails
with `409 conflict` — you can only drop your own pick-up (use `pickup` with
`steal` if you need to take over from them, then `drop`).

#### Response

On success returns `200 OK` with the updated issue under the `issue` key.

#### Errors

| Status | Code           | When                                                                 |
|--------|----------------|------------------------------------------------------------------------|
| 400    | `bad_request`  | Missing/invalid `id`                                                  |
| 401    | `unauthorized` | Missing, invalid, or inactive API key                                |
| 404    | `not_found`    | `id` doesn't exist, or refers to an issue you can't see               |
| 409    | `conflict`     | Issue is currently worked by someone else                             |

## `POST /api/issues/{id}/messages`

Adds a reply/comment to an existing issue — the API equivalent of the
"Comment" box on a ticket's page. The key's user must have **comment**
permission on the issue's queue. The new message is recorded as markdown,
attributed to the key's user, and (like a normal reply) bumps the issue's
`updated_at` and adds an entry to its history.

Optionally, in the same request, you can also change the issue's `status`
and/or move it to a different `queue_id` — this requires **manage**
permission on the issue's current queue (and, if moving queues, on the
target queue too). This is useful for e.g. replying and closing a ticket in
one call.

Send a JSON body with `Content-Type: application/json`. (Form-encoded bodies
with the same field names also work.)

| Field      | Type    | Required | Description                                                      |
|------------|---------|----------|--------------------------------------------------------------------|
| `body`     | string  | yes      | Reply body (markdown)                                              |
| `status`   | string  | no       | New status word (case-insensitive) — requires manage permission on the issue's queue |
| `queue_id` | string  | no       | Name (case-insensitive) of the queue to move the issue to — requires manage permission on both the current and target queue |

There is no internal/private note flag — every message added this way is
visible to anyone who can view the issue, including a client-portal user if
the issue has a client, same as a reply made through the web UI. There's
also no attachment support via the API yet.

### Response

On success returns `201 Created` with the updated issue under the `issue`
key, in the same shape as a `GET`/`POST`/`PATCH` result — including the new
message appended to `messages`.

### Errors

| Status | Code           | When                                                                 |
|--------|----------------|------------------------------------------------------------------------|
| 400    | `bad_request`  | Missing/blank `body`; invalid `status` or `queue_id`; invalid JSON body |
| 401    | `unauthorized` | Missing, invalid, or inactive API key                                |
| 403    | `forbidden`    | You don't have comment permission on the issue's queue, or (when `status`/`queue_id` are supplied) you don't have manage permission on the relevant queue(s) |
| 404    | `not_found`    | `id` doesn't exist, or refers to an issue you can't see               |

## Examples

Fetch a single issue by id:

```bash
curl -H "Authorization: Bearer $API_KEY" \
  'https://helpdesk.artumi.com/api/issues?id=4290'
```

List issues for a client:

```bash
curl -H "Authorization: Bearer $API_KEY" \
  'https://helpdesk.artumi.com/api/issues?client=2'
```

List issues in a queue, capped at 10 results:

```bash
curl -H "Authorization: Bearer $API_KEY" \
  'https://helpdesk.artumi.com/api/issues?queue=44&limit=10'
```

List issues with a given tag:

```bash
curl -H "Authorization: Bearer $API_KEY" \
  'https://helpdesk.artumi.com/api/issues?tag=urgent'
```

List issues user 9 worked on today:

```bash
curl -H "Authorization: Bearer $API_KEY" \
  'https://helpdesk.artumi.com/api/issues?worked_by=9'
```

List issues user 9 worked on a specific day:

```bash
curl -H "Authorization: Bearer $API_KEY" \
  'https://helpdesk.artumi.com/api/issues?worked_by=9&date=2026-07-15'
```

Create a new issue:

```bash
curl -X POST \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"issuename":"Printer not working","body":"The printer on floor 2 is jammed.","queueid":"Support","priority":"High","tags":"hardware, urgent"}' \
  'https://helpdesk.artumi.com/api/issues'
```

Assign an owner, and change status and priority:

```bash
curl -X PATCH \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"owner":"jdoe","status":"Fixed","priority":"Critical"}' \
  'https://helpdesk.artumi.com/api/issues?id=4290'
```

Unassign an owner (requires JSON body — `owner` set to `null`):

```bash
curl -X PATCH \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"owner":null}' \
  'https://helpdesk.artumi.com/api/issues?id=4290'
```

Pick up an issue before working on it:

```bash
curl -X POST \
  -H "Authorization: Bearer $API_KEY" \
  'https://helpdesk.artumi.com/api/issues/4290/pickup'
# 409 conflict if someone else already has it:
# {"error":{"code":"conflict","message":"Issue is already being worked by another user; retry with steal=true to take it over"}}
```

Steal an issue someone else picked up and abandoned (requires steal
permission on the queue):

```bash
curl -X POST \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"steal":true}' \
  'https://helpdesk.artumi.com/api/issues/4290/pickup'
```

Drop an issue once you're done (or handing it off):

```bash
curl -X POST \
  -H "Authorization: Bearer $API_KEY" \
  'https://helpdesk.artumi.com/api/issues/4290/drop'
```

Add a reply to an issue:

```bash
curl -X POST \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"body":"Fixed — the jam was cleared and the printer reprinted the queued jobs."}' \
  'https://helpdesk.artumi.com/api/issues/4290/messages'
```

Reply and mark the ticket fixed in one call (see the `status` field on a
`GET` response for the full list of valid words):

```bash
curl -X POST \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"body":"Fixed and confirmed working.","status":"Fixed"}' \
  'https://helpdesk.artumi.com/api/issues/4290/messages'
```

Missing/bad key:

```bash
curl -H "Authorization: Bearer wrongkey" \
  'https://helpdesk.artumi.com/api/issues?id=4290'
# 401 {"error":{"code":"unauthorized","message":"Missing or invalid API key"}}
```
