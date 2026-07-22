---
name: helpdesk-api
description: >-
  Interact with the live helpdesk.artumi.com ticketing system via the
  `helpdesk` CLI — fetch a ticket/issue by number, list issues for a
  client/queue/tag, pick up/drop a ticket's worker, reply to a ticket, and
  change its status/priority/owner/queue. Use
  whenever the user references a ticket code like "ART1234" (unambiguous —
  always this system), or explicitly says "helpdesk"/"helpdesk ticket". A
  bare number or the word "ticket"/"issue" alone is NOT enough to trigger
  this in a project that has its own tracker (GitHub issues, Jira, etc.) —
  see Disambiguation below. Works from any project, not just the helpdesk
  app's own repo. Distinct from working on the helpdesk app's source code —
  this is for using the deployed app as an end user would, through its CLI.
---

# Helpdesk API

Interact with the production helpdesk at `https://helpdesk.artumi.com` through the `helpdesk`
CLI (from the `helpdesk-cli` repo — `ArtumiSystemsLtd/helpdesk-cli`), not raw `curl`. The CLI
wraps the same REST API; full HTTP-level reference (fields, error codes, examples) is in
`references/api.md` alongside this file if you need detail beyond what the CLI surfaces.

## Locating the binary

```bash
HELPDESK=$(command -v helpdesk || echo ~/github.com/Artumi-Systems-Ltd/helpdesk-cli/bin/helpdesk)
```

Use `$HELPDESK` in place of `helpdesk` below in case it isn't on `PATH` yet (the repo's
`install.sh` symlinks it to `~/bin/helpdesk`).

## Authentication

The API key lives at `~/vault/.helpdesk-artumi.com-key`. That directory is an encrypted vault
mount and **may not be mounted** in the current session.

```bash
export HELPDESK_API_KEY=$(cat ~/vault/.helpdesk-artumi.com-key 2>/dev/null | head -1)
```

If the file isn't there or is empty, the vault is locked — tell the user, verbatim:

> UNLOCK TEH VAULT

...then wait for them to mount it before retrying. Don't guess at the key or invent a
workaround.

Read the key fresh each time rather than caching it in the conversation, and never print the
raw key value in output. Set it via the `HELPDESK_API_KEY` env var for each invocation rather
than `helpdesk auth login --key ...` — login persists the key to
`~/.config/helpdesk-cli/config.json` on disk, which outlives this conversation and isn't
necessary here.

You can sanity-check a key without hitting a real ticket:

```bash
HELPDESK_API_KEY=$HELPDESK_API_KEY $HELPDESK auth status
```

## Disambiguation

A code matching a short letter-prefix + number (e.g. `ART1234`, or the older `ISS-4290` style —
usually `ART` in practice) unambiguously means this helpdesk. Treat it as a definite match, no
need to ask.

A bare number or the generic word "ticket"/"issue" is ambiguous in any project that has its own
tracker (GitHub issues, Jira, etc.). In that case, don't assume — ask the user which system they
mean before calling the CLI. Outside such a project (or once the user has said "helpdesk"
explicitly), it's safe to proceed.

## Ticket numbers vs issue ids

Users will refer to tickets by their code (e.g. `ART5123`) or sometimes just the bare number, or
`#4290`. The CLI accepts all three forms directly as the `<id>` argument (it normalizes them
internally) — no need to strip the letter prefix yourself.

## Common operations

Get a ticket by id:

```bash
$HELPDESK issue view ART4290
```

List issues in a queue (needs at least one of `--queue`, `--client`, `--tag`, `--worked-by`; add
`--limit=N` to cap results, default 50 max 200):

```bash
$HELPDESK issue list --queue=44
```

Reply to a ticket (markdown body):

```bash
$HELPDESK issue comment 4290 --body="Reply text here."
```

Reply and close/change status in the same call — `--status` is a word (case-insensitive), not a
number; check the ticket's `status` field via `issue view` first if you're unsure of the exact
wording (e.g. "Fixed", "Cant fix", "Wont fix", "Wishlist", "Duplicate", "Old"):

```bash
$HELPDESK issue comment 4290 --body="Fixed and confirmed working." --status="Fixed"
```

`issue comment` also takes `--queue=NAME` to move the ticket to another queue while replying.

Change status/priority/owner without adding a reply — `--priority` is also a word
(Low/Medium/High/Critical), and `--owner` takes a username, not a numeric id. Use
`--remove-owner` to clear the owner:

```bash
$HELPDESK issue edit 4290 --status="Fixed" --priority="Critical" --owner="jdoe"
```

Create a new ticket — `--queue` is the queue's name (case-insensitive), not its numeric id:

```bash
$HELPDESK issue create --title="Title" --body="Opening message" --queue="Support"
```

Optional flags on `issue create`: `--client=ID`, `--priority=P`, `--owner=USER`,
`--tags=a,b`.

`worker_id` (who is actively working the ticket right now, distinct from `owner_id` who's
case-managing it) isn't settable via `issue edit` — it has its own pick-up/drop subcommands:

Pick up a ticket before working on it — claims `worker_id` for the API key's user:

```bash
$HELPDESK issue pickup 4290
```

If someone else already has it, the CLI exits with status 1 and prints a message to stderr
rather than clobbering their claim. Don't silently retry with `--steal` — tell the user someone
else is already working the ticket and let them decide. Only steal if they explicitly say to
take it over (e.g. that person is out/gone):

```bash
$HELPDESK issue pickup 4290 --steal
```

Drop it when you're done (also happens automatically if you close the ticket — any status change
away from `"Open"` clears `worker_id` for you):

```bash
$HELPDESK issue drop 4290
```

The "Worked:" line in `issue view`/`issue edit`/etc. output tells you whether a ticket is
claimed and by whom (`yes (you)` vs `yes`), without having to inspect the raw `worker_id`.

## Workflow for "get ticket N and work on it"

1. If the user is asking you to work the ticket (as opposed to a pure read-only question like
   "what's the status of ART4290?"), run `issue pickup N` **first, immediately** — before
   fetching details, reading, thinking about, or deciding anything about the ticket. "Work the
   ticket"/"pick up ART4290"/"look into ART4290 and fix it" all count; a bare "what does
   ART4290 say?" does not. Picking up early is what claims the ticket for you and prevents
   someone else from picking it up underneath you while you're still forming an understanding of
   the issue — don't wait until you're ready to comment or change status to do this.
   - If pickup fails (exit 1, "already being worked by another user"), tell the user who's
     already on it (from the `issue view` output, fetched next) rather than stealing
     automatically.
2. `issue view N` to fetch full details (status, priority, owner, worked, timestamps, messages).
3. Summarize the ticket for the user in plain language before taking any other action — don't
   reply or change status unprompted.
4. If, after understanding the ticket, it turns out this isn't something you can actually do
   (out of scope, needs info/access you don't have, unclear what's being asked, etc.), run
   `issue drop N` and tell the user why, rather than leaving it claimed while idle.
5. When the user decides what to do (reply, reassign owner, close, reprioritize), run the
   corresponding command and show the resulting issue output back so they can confirm it worked.
6. Replying and changing status/queue can be done in one `issue comment` call — prefer that
   over two separate commands when the user wants both. Closing the ticket this way also drops
   `worker_id` automatically, so there's no need for a separate `issue drop` call in that case.

## Errors

The CLI prints `Error (CODE): message` to stderr and exits 1 on API errors. `401`-class errors
mean the key is missing/invalid, `403` a permissions issue on the queue, `404` the id doesn't
exist or isn't visible to this key's user, `400` a malformed request. Surface the message to the
user rather than just the exit code. `AbstractApiCommand` prints a clear "No API key configured"
message (exit 1) if `HELPDESK_API_KEY` wasn't set — treat that the same as a locked vault.
