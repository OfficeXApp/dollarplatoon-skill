# Queues — polling, ordering, and assigning work

Two queue distributions share every endpoint here. They differ in one thing: whether workers
compete for the same task.

## Contents

- `queue` versus `queue_solo`
- Queue order
- Routes
- Polling for tasks
- Claiming one named task (share link)
- Declining a task
- Setting priority
- Direct assignment to one worker
- Private task details
- Returning an assigned task
- Hiring for a high-value task

---

## `queue` versus `queue_solo`

| | `queue` (shared) | `queue_solo` (single player) |
|---|---|---|
| Who can claim a task | The first worker to poll it | Every worker, independently |
| Effect on other workers | Claiming removes it from their queue | None — nothing you do is visible to them |
| What you receive | The task itself | Your own private copy |
| Proofs per task | One, gig-wide | One per worker |
| Cost per task | price × 1 | price × number of workers who take it |
| Task leaves the queue when | Someone claims it | It hits `max_claims_per_task` (never, if unlimited) |

**The `queue_solo` cost warning.** Ten queued tasks in a solo gig with five workers is fifty
payouts, not ten. `max_claims_per_task` bounds it — and it is also the *only* thing that ever
drains a solo queue, because claiming does not consume the task.

## Queue order

Set `queue_order` on the gig.

- **`fifo`** (default) — oldest first.
- **`lifo`** — newest first.
- **`priority`** — you set the order. Each task carries a `priority` number; lower goes first, and
  tasks sharing a number are served oldest-first. So 0, 1, 2, 5, 7, 23, 4689, 99999 are polled in
  exactly that order.
- **`random`** — each poll draws at random from the tasks still queued. There is no position, so
  `priority` numbers are ignored while this mode is on. A poll samples up to the first 1000 queued
  tasks.

New tasks default to priority **1000**, leaving room to insert above and below without renumbering
anything. Priorities are non-negative whole numbers up to 9999999999.

Think of priority as a **virtual arrival time**: a task at 0 behaves as though it arrived before
everything else. That is why `fifo` honours priorities (oldest virtual arrival first) and `lifo`
reverses them. Setting `queue_order: "priority"` declares the queue is meant to be hand-ordered —
it turns on the priority column in the dashboard — but reordering works in any mode.

## Routes

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/gigs/:id/queue/poll` | Worker | Claim tasks |
| POST | `/gigs/:id/queue/:msgId/claim` | Worker | Claim ONE named task |
| POST | `/gigs/:id/queue/:msgId/decline` | Worker | Skip a task, for you only |
| GET | `/gigs/:id/queue` | Member | List queued tasks with `tags`, `price`, `priority`, `declined_count` |
| GET | `/gigs/:id/reserved` | Owner or member | Tasks held back from the queue. Owner sees all; a worker sees their own |
| GET | `/gigs/:id/tasks/:msgId` | Owner, holder, or member | One task with its full body |
| PATCH | `/gigs/:id/tasks/:msgId/availability` | Owner | `open`, `reserved` (with `reserved_for`), or `view_only` |
| PATCH | `/gigs/:id/tasks/:msgId/priority` | Owner | Move one queued task |
| PATCH | `/gigs/:id/queue/priorities` | Owner | Reorder up to 100 tasks in one call |
| PATCH | `/gigs/:id/tasks/:msgId/price` | Owner | Set what one task pays |
| PATCH | `/gigs/:id/queue/prices` | Owner | Price up to 100 tasks in one call |
| PATCH | `/gigs/:id/tasks/:msgId/tags` | Owner | Retag one task |
| PATCH | `/gigs/:id/queue/tags` | Owner | Retag up to 100 tasks in one call |
| POST | `/gigs/:id/tasks/:msgId/assign` | Owner | Give a task to one named worker |
| PATCH | `/gigs/:id/tasks/:msgId/private-details` | Owner | The brief only the holder sees |
| PATCH | `/gigs/:id/tasks/:msgId/alias` | Owner, holder, member | Your own private title for a task |
| DELETE | `/gigs/:id/tasks/:taskId` | Owner | Delete a task |

`GET /gigs/:id/queue` accepts `?tag=`, `?tag_match=`, and `?tag_mode=`. It is the owner's way to
confirm that `?tags=` and `?price=` landed — **do not use `/queue/poll` to check**, because it is
worker-only and it claims what it returns.

## Polling for tasks

```json
POST /gigs/:id/queue/poll   { "count": 2 }   // default 2, max 20
```

```json
{
  "tasks": [
    { "id": "TASK_01HX...", "type": "webhook", "subject": "...", "payload": "...",
      "forwarded_at": "...", "claimed_at": "...",
      "price": 2.50, "price_tbd": false,
      "tags": ["shortform"],
      "reserved_for_me": false,
      "source_task_id": null }
  ],
  "count": 1,
  "scan_exhausted": false,
  "filter_applied": false,
  "rate_limit": { "count": 5, "minutes": 60, "source": "gig", "used": 3, "remaining": 2, "retry_at": null },
  "open_tasks": { "max_open_tasks": 3, "source": "gig", "open": 1, "remaining": 2 }
}
```

Returns tasks in the configured order, skipping anything you have already taken, proven, or
declined. Tasks are never pushed to a mailbox in a queue gig — you must poll.

**Read `price` and `tags` per task, not per gig.** The gig price is only a default.

**Your own reservations come first.** A task the client reserved for you is served ahead of the
shared queue, and comes back with `reserved_for_me: true`. It ignores the queue order and your
own tag filter — the client named you for that task, so a standing preference does not drop it.
It does not dodge your limits: it costs a slot like any other claim. A gig can hold tasks
reserved for you indefinitely; nobody else is ever offered them. See
[tasks.md](https://dollarplatoon.com/skill/tasks.md), and `GET /gigs/:id/reserved` to look at
what is waiting for you without claiming it.

**Poll after every proof.** Submitting does not fetch more work. A claim and its proof together
cost one slot against your rate limit, not two.

**Two ceilings, not one.** `rate_limit` is per window and refills with time. `open_tasks` counts
what you hold unproven right now and never refills on its own — submit or skip to free a slot.
Both trim `count` down rather than failing the poll; both return `429` only at zero.

**In `queue_solo`**, each returned task is a private copy with its own `id`. Use that `id` as your
`task_identifier` — never `source_task_id`, which is shared and will be rejected. Nobody competes
with you, so a poll can never fail with "already claimed"; it returns nothing only when you have
taken everything.

**`scan_exhausted` tells you which kind of empty you got.** Matching happens after rows are read,
over a bounded scan:

- `false` with no tasks — genuinely nothing matches. Back off or widen the filter.
- `true` — the scan ran out of budget. **Poll again**; it will make further progress.

Both queue types report it. A solo claim leaves the task in place, so the poll has to scan past
everything you already hold, which is what makes the budget reachable on large solo queues.

Past your rate limit, polling returns `429` with `retry_at`. Claims are capped to your remaining
allowance — asking for 10 with 2 left returns 2.

Filtering a poll by tags and price: see
[pricing-and-tags.md](https://dollarplatoon.com/skill/pricing-and-tags.md).

## Claiming one named task (share link)

A poll asks for whatever the queue order offers next. This asks for **one exact task**:

```json
POST /gigs/:id/queue/:msgId/claim   { "mailbox_id": "MBX_..." }
→ { "success": true, "mailbox_id": "MBX_...", "task": { ... }, "rate_limit": {...}, "open_tasks": {...} }
```

`mailbox_id` is optional. The response `task` is shaped exactly like one entry of a poll,
`private_details` included — the task is yours now.

**The link.** Every task has a page at `https://dollarplatoon.com/claim/:gigId/:taskId`. The owner
copies it from the task's detail pane in the dashboard and sends it to a worker. Opening it and
pressing **Accept this task** calls the endpoint above. This is the pull half of `?assign_to=`:
`assign_to` pushes the task at a named worker; the link lets a worker you chose take it themselves.

**Membership is the credential — but the link can carry the invite.** There is no per-task token.
A visitor who has not joined the gig sees a join link instead of the task, so the URL is safe to
paste into a chat of workers who all belong to the gig. The first to open it wins.

Add `?invite=<token>` — a gig invite token from `POST /gigs/:id/invites` — and the link works for
somebody who has **not** joined yet: they are offered the gig, and joining sends them straight back
to this task instead of to a mailbox list.

```
https://dollarplatoon.com/claim/GIG_01HX.../TASK_01KV...?invite=abc123def456
```

Use an **unlimited** invite (`max_uses: null`) for a link that goes to more than one person; a
3-use token is dead on the fourth reader. Never use an email-bound invite: it names one person, so
everybody else who follows the link is refused. The dashboard's **Share task…** dialog picks the
right one for you and turns the invite on by default.

A **reserved** task is the common case for a link like this: no poll offers it at all, so the
link is the only way in and it can sit in somebody's inbox for a week without another worker
sweeping it up. A reservation can also name one worker, and then only that worker's poll and only
their claim will take it.

A **view-only** task cannot be claimed at all. Its link is `/task/:gig_id/:task_id` (the `/claim/`
form still works and shows the same page), and it exists so one link can go to many people without
the first of them taking the work.

Both are in [tasks.md](https://dollarplatoon.com/skill/tasks.md).

**Same limits as a poll.** The proof rate limit and the open-task cap both apply and both return
`429`. A link only chooses *which* task a worker spends a slot on; it never buys them an extra one.

**Answers.** Every failure carries a machine-readable `reason`:

| Code | `reason` | Meaning |
|------|----------|---------|
| 403 | `not_joined` | The caller has no mailbox in this gig |
| 403 | `mailbox_inactive` | Joined, but their mailbox is not active |
| 409 | `taken` | Another worker claimed it first, or the last solo slot went |
| 409 | `assigned_elsewhere` | The owner assigned it to a named worker |
| 409 | `withdrawn` | The owner took it back (`UNASSIGNED`) |
| 409 | `exhausted` | `queue_solo`: `max_claims_per_task` is used up |
| 409 | `other_worker` | `queue_solo`: this id is another worker's copy — link the template |
| 409 | `already_proven` | A proof was already submitted against it |
| 409 | `view_only` | The client published it for reading only — nobody can claim it |
| 409 | `reserved_for_other` | Reserved for a different worker. Nobody took it; it was never on offer to you |
| 404 | `not_found` | Deleted |

Re-opening the link after a successful claim returns `200` with `already_held: true`. It claims
nothing a second time and spends no rate-limit slot, so the link is safe to bookmark.

## Declining a task

```json
POST /gigs/:id/queue/:msgId/decline   → { "success": true }
```

Marks the task skipped **for you only**. Idempotent, free, and invisible to other workers — the
task keeps the position the owner gave it rather than being pushed to the back. Use it whenever a
polled task is not suitable so future polls return fresh items instead of the same head of queue.

In `queue_solo`, pass the `id` of your own copy: it is discarded, its claim slot returns to the
shared task so other workers are unaffected, and the task is retired for you permanently.

The owner sees a `declined_count` per task and can prune genuinely unworkable items.

## Setting priority

Three equivalent ways — they write the same number.

**1. At submission**, on the publisher webhook (the body is the payload, so this rides the query
string):

```
POST /inbound/webhook/GIG_abc?token=...&priority=0
→ { "status": "queued", "message_id": "TASK_...", "priority": 0 }
```

**2. One task:**

```json
PATCH /gigs/:id/tasks/:msgId/priority   { "priority": 0 }
→ { "success": true, "id": "TASK_...", "priority": 0 }
```

**3. In bulk** — up to 100 per call, for rearranging a queue programmatically:

```json
PATCH /gigs/:id/queue/priorities
{ "updates": [ { "id": "TASK_a", "priority": 0 }, { "id": "TASK_b", "priority": 10 } ] }

→ { "success": true, "updated": 1,
    "applied": [ { "id": "TASK_a", "priority": 0 } ],
    "skipped": [ { "id": "TASK_b", "reason": "already_claimed" } ] }
```

The whole batch is validated before anything is written, so a malformed entry returns `400` and
changes nothing. Individual tasks that cannot move are reported in `skipped` rather than failing
the batch: `not_found`, `already_claimed`, or `not_in_queue` (a solo task that hit
`max_claims_per_task`).

Reordering is cheap — position lives in the task's index key, so each move is a single write that
never touches the tasks you left alone. Rewriting a 100-task order every tick is reasonable.

**Only queued tasks move.** Once claimed, a task has left the queue and a single-task PATCH
returns `409`. Reordering never disturbs work in progress.

## Direct assignment to one worker

A queue hands work to whoever polls first. That is right for a $2 task and wrong for one expensive
job, because "polled first" says nothing about competence.

At creation:

```
POST /inbound/webhook/GIG_abc?token=...&assign_to=MBX_01KR5BENF2CK39Y82B2MGQ1ND6
POST /inbound/webhook/GIG_abc?token=...&assign_to=worker@example.com
→ { "status": "assigned", "message_id": "TASK_01KV...", "mailbox_id": "MBX_01KR5..." }
```

Or on a task that already exists:

```json
POST /gigs/:id/tasks/:msgId/assign   { "assign_to": "worker@example.com" }
```

`assign_to` takes a **mailbox id** or the worker's **account email** (their login address, not the
contact email on the mailbox). Failures return `400` with a `reason`: `assignee_not_found`,
`assignee_not_active`, `assignee_ambiguous`.

Works on queue and push gigs alike. On a queue gig the task never enters the queue, so no other
worker ever sees it.

**What assignment overrides.** An assigned task ignores that worker's own `filter_tags` and price
filters and skips `round_robin` rotation — you named this person, so their standing preferences do
not apply. It also clears `declined_by`, so someone who previously skipped the task can receive it.

**Restrictions.**

- `?priority=` is rejected alongside `?assign_to=` — an assigned task has no queue position.
- `409` if the task already has a proof.
- `400` on a `queue_solo` template or a worker's copy of one. Those belong to the solo claim-slot
  machinery; post new work with `?assign_to=` instead.
- Assigning a task the worker already holds is supported, and is how you **confirm** a
  self-claimed task as assigned. It returns `unchanged: true` and preserves their claim.

## Private task details

`private_details` is the part of the brief only the holder sees — the real spec, asset links,
credentials. Advertise the job in the payload; keep the substance here.

```json
PATCH /gigs/:id/tasks/:msgId/private-details   { "private_details": "Full brief..." }
PATCH /gigs/:id/tasks/:msgId/private-details   { "private_details": null }   // clears
```

Maximum **4000 characters**. The limit is deliberate: the value is stored inline and never
offloaded to object storage, which is what keeps it out of presigned-URL reach.

| Caller | Sees it |
|---|---|
| Gig owner | Yes, everywhere |
| The mailbox currently holding the task | Yes — in the poll response, their inbound list, and `GET /gigs/:id/tasks/:msgId` |
| Any other gig member browsing `GET /gigs/:id/queue` | No |
| A share link (`/submit/:token`, `GET /public/task`) | No |
| The task-delivery webhook | No — the brief does not exist yet when the task is created |

**This stops browsing, not harvesting.** A worker can claim a task, read the brief, and decline;
they keep what they read. `max_claims_per_task`, proof rate limits, and `declined_by` bound how
often anyone can do that — but a reassignment clears those markers. Do not treat `private_details`
as confidentiality.

## Returning an assigned task

A worker who cannot do an assigned task declines it as usual:

```json
POST /gigs/:id/queue/:msgId/decline
→ { "success": true, "skipped": true, "returned_to_owner": true }
```

The task goes back to the **owner**, not into the shared queue — `mailbox_id` becomes
`"UNASSIGNED"`. It keeps its `private_details`, disappears from every queue and inbox, and only
the owner can see it. Hand it to somebody else with `POST /gigs/:id/tasks/:msgId/assign`.

`extend` and `recycle` both reject an `UNASSIGNED` task — there is no holder and no clock.

**Push-gig limitation:** decline works on queue gigs only. An assignee on a push gig cannot hand a
task back; only the owner's `recycle` can move it.

## Hiring for a high-value task

There is no special "job posting" feature. Compose the pieces that exist.

**1. Post free application tasks.**

```
POST /inbound/webhook/GIG_abc?token=...&price=0&tags=type:interested
```

Describe the job generically and ask for whatever you will judge on — a portfolio link, a short
plan, a sample. On a `queue_solo` gig set `max_claims_per_task: N` and post **one** task: it hands
a private copy to each of N applicants. On a shared `queue` gig post N separate tasks.

**2. Applicants apply by submitting a `$0` proof.** It costs them nothing and pays nothing.

**3. Review, and REJECT them all with `not_selected` — including the winner's.**

```json
PATCH /gigs/:id/proofs/:proof_id
{ "action": "reject", "rejection_tag": "not_selected", "requeue": false }
```

**`"requeue": false` is not optional here.** A rejection returns its task to be done again by
default, which is right for real work and wrong for an application: leaving it out re-posts the
application task and invites the next applicant to apply for a job you have already filled.

This is the counter-intuitive step, and it matters twice over:

- `not_selected` is excluded from reputation scoring entirely, so nobody is penalised for applying
  and losing. Any other tag **would** cost them.
- **Rejected `$0` proofs get cleared by a rollup. Approved `$0` proofs never do.** A payout run
  skips any mailbox whose approved total is `$0`, so those rows are rescanned forever and grow
  without bound. Approving applications is the trap; rejecting them is the clean path.

**4. Post the real job, assigned to the person you chose.**

```
POST /inbound/webhook/GIG_abc?token=...&price=500&assign_to=winner@example.com
→ { "status": "assigned", "message_id": "TASK_01KV..." }

PATCH /gigs/GIG_abc/tasks/TASK_01KV.../private-details   { "private_details": "The real brief..." }
```

Only they can see the brief, and nobody else can take the job.

**Optional: gate the gig as well as the task.** `join_policy: "invite"` plus
`requires_approval: true` holds new members at `pending_approval` until you approve them, and
applicants can put a portfolio link in `notes` when they join. This raises the floor for the whole
gig — but it does not solve allocation on its own, since an approved-but-mediocre worker still
polls as fast as anyone else.
