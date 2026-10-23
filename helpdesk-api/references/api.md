# Issues API

HTTP API for issues (tickets), queues, clients and wiki pages. Supports
fetching issues (`GET /api/issues`), creating a new issue (`POST
/api/issues`), updating an issue's status, priority, owner, hold date or
critical date (`PATCH /api/issues`), picking up or dropping an issue's worker (`POST
/api/issues/{id}/pickup`, `POST /api/issues/{id}/drop`), adding a reply to an
existing issue (`POST /api/issues/{id}/messages`), listing the queues visible
to the API key's user (`GET /api/queues`), listing active clients (`GET
/api/clients`), downloading a message's attachments and embedded images
(`GET /api/attachments/{id}`, `GET /api/uploads/attachments/{path}`,
`GET /api/messages/{id}/images/{n}`), and
fetching a wiki page's raw markdown source (`GET
/api/wiki/{wikiSlug}/{pageSlug}`).

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

**Client-portal users can't use the API at all.** A key belonging to a user
with a `clientid` set (i.e. a client-portal account, not a staff account) is
rejected outright, on every endpoint, before any endpoint-specific check
runs — the API is a staff/automation surface only.

Every endpoint below can also fail authentication itself, before any of its
own checks run:

| Status | Code             | When                                                |
|--------|------------------|------------------------------------------------------|
| 401    | `unauthorized`   | Missing/invalid key, revoked key, or inactive user  |
| 401    | `token_expired`  | Key exists and is otherwise valid, but has passed its expiry date/time |
| 403    | `forbidden`      | The key's user has a `clientid` set (client-portal user) |

These aren't repeated in each endpoint's own error table below.

### Minting a key

Admins can generate and manage keys from the **API Keys** page in the admin
panel (`/api-keys`) — for any user, including themselves. It shows every
key's status (Active/Expired/Revoked), usage count, and last-used time, and
lets an admin revoke (deactivate) or delete a key.

Keys can also still be created via a CLI script:

```
php arcode/modules/HelpDesk/scripts/CreateApiKey.php <username> [label]
```

Either way, the raw key is shown/printed once. It's stored only as a hash,
so if you lose it you must mint a new one.

A key can optionally be given an expiry date/time when it's generated. An
expired key is rejected the same as an invalid one, but with a distinct
error code — see **Errors** below.

## `GET /api/issues`

Exactly one of the following query parameters is required:

| Param        | Type    | Description                                                  |
|--------------|---------|----------------------------------------------------------------|
| `id`         | integer | Fetch a single issue by its issue id                          |
| `client`     | integer | List issues belonging to a client id                          |
| `queue`      | integer | List issues in a queue id                                      |
| `tag`        | string  | List issues with a given tag, or comma-separated tags (issues must have *all* of them) |
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
      "critical_at": null,
      "merged_into": null,
      "created_at": "2010-11-25T14:10:00+00:00",
      "updated_at": "2010-11-26T09:02:15+00:00"
    }
  ]
}
```

Fields referencing other records (`client_id`, `queue_id`, `worker_id`,
`owner_id`) are raw ids, not resolved names. Timestamps are ISO 8601. Any of
the id fields, `hold_until`, `critical_at` and `merged_into` may be `null`.
`hold_until` is when an on-hold issue comes back into the queue (a past or
`null` value means it isn't on hold); `critical_at` is when an issue will be
automatically escalated to Critical priority. `status` and
`priority` are words (e.g. `"Open"`, `"Completed"`, `"Critical"`) — there are no
numeric status/priority ids in the API.

Each issue also carries `relations` and `messages`. A message looks like:

```json
{
  "id": 81234,
  "subject": null,
  "body": "Here's the error:\n\n![shot](/uploads/attachments/pasted/AbCdEf0123456789ghij.png)",
  "type": 4,
  "type_label": "Markdown",
  "from_user_id": 9,
  "from_email": null,
  "created_at": "2026-10-22T11:04:03+01:00",
  "attachments": [
    {
      "id": 5501,
      "name": "Screen Shot.png",
      "is_image": true,
      "size": 48211,
      "url": "/api/attachments/5501",
      "thumb_url": "/api/uploads/attachments/imagethumb_5501_Xy12Ab34Cd.jpg"
    }
  ],
  "embedded_images": [
    "/api/uploads/attachments/pasted/AbCdEf0123456789ghij.png"
  ]
}
```

`attachments` are the files uploaded with the message. `name` is the
uploader's own filename; `size` is in bytes (`null` if the file has gone
missing from disk, in which case `url` 404s); `is_image` says whether it's a
displayable image (png/jpg/gif/webp/bmp/avif); `thumb_url` is a smaller
rendition for images, or `null`. `embedded_images` are the images in the
`body` itself, which have no attachment row: pasted images and ones embedded
via a data URI (see **Embedding images in a message**), listed as
`/api/uploads/attachments/...`, followed by any images still stored inline as
base64 data URIs (as older HTML messages' pasted screenshots are), listed as
`/api/messages/{id}/images/{n}`.
All of these URLs are relative to the site root and need the API key; see
**Downloading attachments and images** below.

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
| `body`      | string  | yes      | Opening message body (markdown) — can embed images as base64 data URIs, see **Embedding images in a message** below |
| `queueid`   | string  | yes      | Name (case-insensitive) of the queue to create the ticket in — must be one you can add to |
| `clientid`  | integer | yes      | Client id to associate the ticket with (every ticket has a client) |
| `priority`  | string  | no       | Priority word (case-insensitive) — Low/Medium/High/Critical; defaults to Medium |
| `owner`     | string  | no       | Username (case-insensitive) of the case-managing owner  |
| `tags`      | string  | no       | Comma-separated tags; lowercased and de-duplicated      |
| `hold_until` | string | no       | Put the ticket on hold until this date/time — see **Dates** below; defaults to now (not on hold) |
| `critical_at` | string | no      | Escalate the ticket to Critical priority at this date/time — see **Dates** below |

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
    "critical_at": null,
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
| 400    | `bad_request`  | Missing/blank `issuename`, `body`, `queueid` or `clientid`; invalid `clientid`; invalid JSON body or priority; invalid or too-distant `hold_until`/`critical_at`; an embedded image in `body` (see **Embedding images in a message**) has an unsupported type, invalid base64, or is over 25MB |
| 401    | `unauthorized` | Missing, invalid, or inactive API key                          |
| 403    | `forbidden`    | You don't have add permission on the requested `queueid`       |

## `PATCH /api/issues`

Updates the owner, status, priority, hold date and/or critical date of an
existing issue. (`PUT` is
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
| `status`   | string       | New status word (case-insensitive) — one of the values from the `status` field on a `GET` response, e.g. `"Open"`, `"Completed"`. The old name `"Fixed"` is still accepted for the Completed status |
| `priority` | string       | New priority word (case-insensitive) — Low/Medium/High/Critical     |
| `owner`    | string\|null | Username (case-insensitive) of the case-managing owner. `null` (JSON only) unassigns the owner |
| `hold_until` | string\|null | Put the issue on hold until this date/time — see **Dates** below. `null` or `""` takes it off hold. Only allowed if the issue is (or this request makes it) Open |
| `critical_at` | string\|null | Escalate the issue to Critical priority at this date/time — see **Dates** below. `null` or `""` removes the escalation |

At least one of these fields must be present. Only the fields supplied
are changed; omitted fields are left as-is. Every change is recorded in the
issue's history, same as an edit made through the web UI. There is no way to
(re)assign `worker` via `PATCH` — that's a separate pick-up/drop workflow,
not a plain reassignment field, described next. Setting `status` to anything
other than `"Open"` automatically clears `worker_id` if it was set.

### Dates

`hold_until` and `critical_at` (on both `POST` and `PATCH`) take an ISO 8601
date/time, e.g. `"2026-11-03T09:00:00+00:00"`. A value without an offset
(`"2026-11-03 09:00"`, or a bare date, which means midnight) is read as UK
time (Europe/London). Relative expressions understood by PHP's `strtotime()`
— `"+2 weeks"`, `"tomorrow 09:00"`, `"next monday"` — are also accepted.

A date can be **no more than 1 year and 1 week from now**; anything later is
rejected with `400 bad_request`. Past dates are allowed: a past `hold_until`
just means "not on hold", and a past `critical_at` escalates the issue to
Critical on the next hourly escalation run.

### Response

On success returns `200 OK` with the updated issue under the `issue` key, in
the same shape as a `GET`/`POST` result.

### Errors

| Status | Code           | When                                                                 |
|--------|----------------|------------------------------------------------------------------------|
| 400    | `bad_request`  | Missing `id`; none of `status`/`priority`/`owner`/`hold_until`/`critical_at` supplied; invalid `status`, `priority` or `owner`; invalid or too-distant `hold_until`/`critical_at` (see **Dates**); `hold_until` on an issue that isn't Open; invalid JSON body |
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
the issue has a client, same as a reply made through the web UI. There's no
multipart file-upload attachment support via the API, but see **Embedding
images in a message** below for inline images.

### Response

On success returns `201 Created` with the updated issue under the `issue`
key, in the same shape as a `GET`/`POST`/`PATCH` result — including the new
message appended to `messages`.

### Errors

| Status | Code           | When                                                                 |
|--------|----------------|------------------------------------------------------------------------|
| 400    | `bad_request`  | Missing/blank `body`; invalid `status` or `queue_id`; invalid JSON body; an embedded image (see below) has an unsupported type, invalid base64, or is over 25MB |
| 401    | `unauthorized` | Missing, invalid, or inactive API key                                |
| 403    | `forbidden`    | You don't have comment permission on the issue's queue, or (when `status`/`queue_id` are supplied) you don't have manage permission on the relevant queue(s) |
| 404    | `not_found`    | `id` doesn't exist, or refers to an issue you can't see               |

## Embedding images in a message

Since there's no multipart form here, an image is embedded directly in the
markdown `body` as a base64 [data
URI](https://developer.mozilla.org/en-US/docs/Web/URI/Reference/Schemes/data),
using ordinary markdown image syntax:

```
![screenshot](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAAB...)
```

This works in both `POST /api/issues`' `body` (the opening message) and
`POST /api/issues/{id}/messages`' `body` (a reply). Every matching image is
decoded and written to the same shared uploads directory the web UI's own
pasted-image uploads use (the `attachments` disk's `pasted/` folder — see
`App\Filament\Forms\Components\MessageBodyEditor`), and the markdown is
rewritten in place to reference the stored file's URL instead of the data
URI, e.g.:

```
![screenshot](/uploads/attachments/pasted/AbCdEf0123456789ghij.png)
```

No separate `Attachment` row is created for an embedded image — it's treated
the same as a pasted image in the rich editor (referenced only from the
message body), not a discrete ticket attachment.

Supported types: `image/png`, `image/jpeg`, `image/gif`, `image/webp`,
`image/bmp`, `image/avif` (deliberately no `image/svg+xml` — SVG can carry
script and is excluded from inline rendering everywhere else in the app for
the same reason). Each embedded image is capped at 25MB, matching the web
UI's own attachment size limit. An unsupported type, invalid base64, or an
oversized image fails the whole request with `400 bad_request` — nothing is
saved, including the rest of the message — rather than silently dropping
just the image. A markdown image that's already a plain URL (not a data URI)
is left untouched.

## Downloading attachments and images

The `/uploads/attachments/...` and `/attachment/{id}` URLs the web UI uses
need a logged-in browser session, so they don't work with an API key. The
API has its own equivalents, with the same permission check — the key's user
must be able to view the issue the file belongs to:

- `GET /api/attachments/{id}` — an attachment by id, as listed in a
  message's `attachments[].url`. Served under the uploader's own filename.
- `GET /api/uploads/attachments/{path}` — a file by its path on disk, as
  listed in `embedded_images` and `attachments[].thumb_url`. Any
  `/uploads/attachments/...` URL found in a message `body` works here by
  prefixing it with `/api`.
- `GET /api/messages/{id}/images/{n}` — the `n`th (from 0) base64 data-URI
  image in message `id`'s body, decoded, as listed in `embedded_images`.
  Only png/jpeg/gif/webp/bmp/avif images count towards `n`; an SVG data URI
  is never listed or served.

The response body is the raw file. Images, PDFs, plain text and browser-
playable video are sent with their real `Content-Type` and an `inline`
`Content-Disposition`; anything else (including SVG and HTML) is sent as
`application/octet-stream` with `attachment`. `Range` requests are honoured.

Pasted images (under `pasted/`) aren't tied to any one issue, so any staff
API key can fetch them by path.

### Errors

| Status | Code           | When                                                          |
|--------|----------------|------------------------------------------------------------------|
| 401    | `unauthorized` | Missing, invalid, or inactive API key                          |
| 404    | `not_found`    | No such attachment/file, the file is missing from disk, or it belongs to an issue you can't see |

## `GET /api/queues`

Lists the queues visible to the API key's user — the same set they'd see in
the web UI's queue picker (all active queues for an Admin, or the active
queues they have viewer access to otherwise). Takes no parameters.

### Response shape

```json
{
  "queues": [
    {
      "id": 44,
      "name": "Bugs"
    },
    {
      "id": 41,
      "name": "Support"
    }
  ]
}
```

Queues are returned sorted by `name` (case-insensitive). There's no way to
fetch a single queue by id, or to include inactive queues.

### Errors

| Status | Code           | When                                  |
|--------|----------------|------------------------------------------|
| 401    | `unauthorized` | Missing, invalid, or inactive API key |

## `GET /api/clients`

Lists active clients — id and name. Takes no parameters. Scope matches the
client picker on the New Ticket / Advanced Reply forms
(`App\Filament\Pages\NewIssue`): every active client, not restricted to
Admins the way the admin-only Clients resource (`/clients`) is — any staff
API user (see **Authentication** above for who counts as "staff") gets the
full list.

### Response shape

```json
{
  "clients": [
    {
      "id": 2,
      "name": "Example Client Ltd"
    },
    {
      "id": 6,
      "name": "Another Client Inc"
    }
  ]
}
```

Clients are returned sorted by `name`. Inactive clients are excluded, and
there's no way to fetch a single client by id or to include inactive ones.

### Errors

| Status | Code           | When                                  |
|--------|----------------|------------------------------------------|
| 401    | `unauthorized` | Missing, invalid, or inactive API key |

## `GET /api/wiki/{wikiSlug}/{pageSlug}`

Fetches a single wiki page's raw markdown **source** — not the rendered
HTML the web UI shows. Pages are identified the same way the web UI
addresses them (`/wiki/{wikiSlug}/{pageSlug}`): a wiki's `slug`, then a
page's `slug` within that wiki, both matched case-insensitively (a
mixed-case `pageSlug` still resolves — the response's own `slug` field is
always the canonical lower-cased one).

Visibility matches the web UI exactly: a wiki not visible to the key's user
(client/public/company/personal rules — see `App\Models\Wiki::visibleTo()`)
404s the same way a hidden issue does, rather than revealing whether it
exists.

### Response shape

```json
{
  "page": {
    "wiki": "engineering",
    "slug": "deploy-process",
    "title": "Deploy Process",
    "body": "# Deploy Process\n\nRun `bin/deploy.sh` from `main`...",
    "is_index": false,
    "created_at": "2026-01-10T09:00:00+00:00",
    "updated_at": "2026-03-02T14:22:00+00:00"
  }
}
```

`body` is exactly what's stored — the same markdown the Edit Wiki Page form
would show, not the HTML `ViewWikiPage` renders it into.

### Errors

Uses the same `{"error": {"code": "...", "message": "..."}}` shape as the
issues endpoints:

| Status | Code           | When                                                          |
|--------|----------------|------------------------------------------------------------------|
| 401    | `unauthorized` | Missing, invalid, or inactive API key                          |
| 404    | `not_found`    | No wiki with that slug, the wiki isn't visible to you, or no page with that slug exists in it |

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

List issues carrying *both* tags (tags are ANDed, and are matched
lowercased/trimmed):

```bash
curl -H "Authorization: Bearer $API_KEY" \
  'https://helpdesk.artumi.com/api/issues?tag=urgent,hardware'
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
  -d '{"owner":"jdoe","status":"Completed","priority":"Critical"}' \
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
  -d '{"body":"Fixed and confirmed working.","status":"Completed"}' \
  'https://helpdesk.artumi.com/api/issues/4290/messages'
```

List the queues you can see (e.g. to look up a `queueid` name for `POST
/api/issues`):

```bash
curl -H "Authorization: Bearer $API_KEY" \
  'https://helpdesk.artumi.com/api/queues'
```

Add a reply with an embedded image (a base64 PNG data URI — see **Embedding
images in a message**):

```bash
curl -X POST \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"body":"Here'"'"'s a screenshot:\n\n![screenshot](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNk+A8AAQUBAScY42YAAAAASUVORK5CYII=)"}' \
  'https://helpdesk.artumi.com/api/issues/4290/messages'
```

List the clients you can see:

```bash
curl -H "Authorization: Bearer $API_KEY" \
  'https://helpdesk.artumi.com/api/clients'
```

Download an attachment (the id comes from a message's `attachments`):

```bash
curl -H "Authorization: Bearer $API_KEY" -OJ \
  'https://helpdesk.artumi.com/api/attachments/5501'
```

Download an image embedded in a message body:

```bash
curl -H "Authorization: Bearer $API_KEY" -o shot.png \
  'https://helpdesk.artumi.com/api/uploads/attachments/pasted/AbCdEf0123456789ghij.png'
```

Fetch a wiki page's markdown source:

```bash
curl -H "Authorization: Bearer $API_KEY" \
  'https://helpdesk.artumi.com/api/wiki/engineering/deploy-process'
```

Missing/bad key:

```bash
curl -H "Authorization: Bearer wrongkey" \
  'https://helpdesk.artumi.com/api/issues?id=4290'
# 401 {"error":{"code":"unauthorized","message":"Missing or invalid API key"}}
```

A key belonging to a client-portal user (has a `clientid`):

```bash
curl -H "Authorization: Bearer $CLIENT_USER_API_KEY" \
  'https://helpdesk.artumi.com/api/queues'
# 403 {"error":{"code":"forbidden","message":"Client-portal users cannot use the API"}}
```
