# Per-task pricing, tags, and filters

How one gig carries work of different shapes and different values — and how a worker takes only
the part they want.

## Contents

- Per-task pricing: the three states
- Setting a price
- How a price becomes a payout
- Task tags
- The matching rule (the part people get wrong)
- `default_task_tags` — required for email gigs
- Filtering when you poll
- Filtering what is pushed to you
- TBD prices and filters
- Tags on proofs
- Tag query parameters, everywhere

---

## Per-task pricing: the three states

A gig has a `price`, and by default every task in it is worth exactly that. A client who needs
variable pay sets a price on the task itself. Every task is in exactly one of three states:

| State | How | What the worker sees | What it pays |
|---|---|---|---|
| Gig price (default) | send nothing | the gig price | `gig.price` at proof time |
| Fixed | `?price=2.50` or `PATCH .../price` | `$2.50` | `$2.50` |
| TBD | `?price=tbd` | `TBD` | decided at approval; the gig price if never set |

**`TBD` is not `$0`.** An unpriced TBD settles at the gig price if the client never names an
amount. Never render it as zero.

## Setting a price

**1. At submission**, on the publisher webhook. The body is the task payload, so the price rides
the query string — same convention as `priority` and `tags`:

```
POST /inbound/webhook/GIG_abc?token=...&price=2.50
POST /inbound/webhook/GIG_abc?token=...&price=tbd
```

Works on queue and push gigs alike.

**Email tasks always land at the gig price**, because an inbound email has nowhere to carry a
price. Reprice afterwards.

**2. One task at a time:**

```json
PATCH /gigs/:id/tasks/:msgId/price
{ "price": 2.50 }     // fix the amount
{ "price": "tbd" }    // decide later
{ "price": null }     // clear the override — back to the gig price

→ { "success": true, "id": "TASK_...", "price": 2.5, "price_tbd": false,
    "price_source": "task", "proof_updated": false }
```

`proof_updated: true` means a TBD proof was already waiting on this number and has now been given
it — you do not have to name the same amount again at approval.

Returns `409` if the task's proof has already locked a price. **That lock is the point:** a worker
agreed to an amount when they submitted, and review cannot quietly lower it.

**3. In bulk** — up to 100 per call, for splitting a queue into pay bands:

```json
PATCH /gigs/:id/queue/prices
{ "updates": [ { "id": "TASK_a", "price": 5.00 }, { "id": "TASK_b", "price": "tbd" } ] }

→ { "success": true, "updated": 1,
    "applied": [ { "id": "TASK_a", "price": 5 } ],
    "skipped": [ { "id": "TASK_b", "reason": "proof_price_locked" } ] }
```

The whole batch is validated before any of it is written, so a malformed entry returns `400` and
changes nothing. Unlike the priority equivalent this **accepts claimed tasks**, because a TBD is
normally priced after a worker has taken it. Individual failures come back in `skipped`:
`not_found` or `proof_price_locked`.

**4. At approval**, for a TBD task:

```json
PATCH /gigs/:id/proofs/:proof_id   { "action": "approve", "amount": 7.50 }
```

`amount` is accepted only while `locked_price` is still `null`. Sending it for an already-priced
proof returns `409`. Approving a TBD **without** `amount` pays the gig price.

## How a price becomes a payout

```
task price (or the gig price)  →  proof.locked_price at submission  →  rollup gross_amount
```

The rule that keeps this safe: **anything unresolvable falls back to the gig price.** A proof with
no matching task, a task naming another worker's mailbox, a row written before per-task pricing
existed, and a TBD nobody ever priced all pay `gig.price`. No task can pay `$0` by accident, and
none can be stranded unpaid.

A task's price only counts when the task belongs to the submitting mailbox. Naming another
worker's expensive task id in `task_identifier` does not buy you their rate — it silently resolves
to the gig price.

## Task tags

A gig is one vending machine. Tags let it carry related but different work — `shortform`,
`longform`, `thumbnail` — so a worker takes only the shapes they want, instead of you opening a
separate gig for each.

At submission:

```
POST /inbound/webhook/GIG_abc?token=...&tags=shortform,urgent
```

Afterwards:

```json
PATCH /gigs/:id/tasks/:msgId/tags   { "tags": ["shortform"] }    // null clears
PATCH /gigs/:id/queue/tags          { "updates": [ { "id": "TASK_a", "tags": ["shortform"] } ] }
```

Up to 100 tasks per bulk call, 25 tags per task, 256 characters each, matching always
case-insensitive. Retagging does **not** re-run distribution — a push task already delivered stays
where it is.

**To confirm what a task actually carries, call `GET /gigs/:id/queue`.** Each item returns its
`tags` and `price`, plus `price_source` (`"task"`, `"gig"`, or `"tbd"`). Do not use `/queue/poll`
to check — it is worker-only and it claims what it returns.

## The matching rule (the part people get wrong)

**A tag filter matches only tasks carrying a matching tag.** Filtering for `category_a` returns
`category_a` work and nothing else. **An untagged task does not match — it is not a wildcard.**

Tags and price are independent. Filtering on price alone does not require the task to have tags at
all.

## `default_task_tags` — required for email gigs

An inbound email has nowhere to carry `?tags=`. So on an email gig, a worker with any tag filter
receives **nothing, forever**.

```json
PATCH /gigs/:id   { "default_task_tags": ["shortform"] }
```

It is stamped on any task that arrives without tags of its own — every email task, and any webhook
call that omitted `?tags=`. The matching rule is unchanged; the gig just supplies a tag when the
publisher cannot.

## Filtering when you poll

```json
POST /gigs/:id/queue/poll
{
  "count": 5,
  "tags": ["shortform", "thumbnail"],   // OR'd; also accepts "a,b" as a string
  "tag_match": "substring",             // substring (default) | prefix | exact
  "price_min": 2.00,
  "price_max": 50.00,
  "accept_tbd": true                    // default true
}
```

**An empty filtered poll does not mean "no work".** Matching happens after rows are read, over the
first 1000 queue rows, so a narrow filter on a deep queue can run out of scan budget:

```json
{ "tasks": [], "count": 0, "scan_exhausted": true, "filter_applied": true }
```

- `scan_exhausted: false` — genuinely nothing matches. Back off, or widen the filter.
- `scan_exhausted: true` — the scan ran out of budget. **Poll again**, or widen the filter.

Both `queue` and `queue_solo` report this.

## Filtering what is pushed to you

On a push gig you do not poll — the client sends work to you. Store a standing preference on your
mailbox instead:

```json
PATCH /gigs/:id/mailboxes/:mbx_id
{
  "filter_tags": ["shortform"],     // [] or null = accept every shape
  "filter_tag_match": "substring",
  "filter_price_min": 2.00,
  "filter_price_max": null,
  "filter_accept_tbd": true
}
```

Worker-only: the gig owner can neither set these nor read them back. Same matching rule as
polling. A mailbox with no filters accepts everything — which is every mailbox that existed before
this feature, so **nothing changes until you opt in**.

If a task matches nobody it is dropped, and the publisher is told with `no_matching_mailboxes` —
see [tasks.md](https://dollarplatoon.com/skill/tasks.md).

**Owners: publish your tag vocabulary in `terms`.** Workers set filters *before* they have seen a
single task, and `terms` is the one field they read before joining. A worker who guesses `short`
when you send `shortform-vertical` silently receives less work, with no error on either side.

## TBD prices and filters

A `price_tbd` task has no price yet, so a price range cannot honestly include or exclude it. It
**bypasses** the price test unless you set `accept_tbd: false`.

`gig.price` is deliberately not treated as a floor here: an owner can price a TBD below it at
approval, so pretending to know the amount would make a promise the payout does not keep. Tag
filters still apply.

## Tags on proofs

Proofs carry `tags` too, so a long history stays organised on both sides.

- Set them at submission: `POST /gigs/:id/proofs` (and the public share-token submit) accept
  `"tags": ["revision", "batch_7"]`.
- Change them later: `PATCH /gigs/:id/proofs/:proof_id/tags` `{ "tags": [...] }`. An empty array
  clears them.
- **Both the gig owner and the worker who submitted may edit them, and both see the same list** —
  unlike an alias, which is private per viewer. That shared visibility is what makes proof tags
  usable as a filter the two sides agree on.

## Tag query parameters, everywhere

These four params behave identically on every route that accepts them:

| Param | Meaning |
|---|---|
| `?tag=` | Comma-separated terms. Always case-insensitive. |
| `?tag_match=` | How each term compares: `substring` (default), `prefix` (alias `starts_with`), `exact`. Anything unrecognised falls back to `substring`. |
| `?tag_mode=` | How several terms combine: `any` (default, OR) or `all` (AND). |
| `?q=` | Plain text search over the row's text fields. |

They work on `GET /work/available`, `GET /mailboxes/mine`, `GET /gigs/mine`,
`GET /gigs/:id/queue`, `GET /gigs/:id/dashboard/inbound`, and both feed list routes.

**Why filtering behaves the way it does.** Tags cannot be indexed in the underlying datastore, and
there is no tag index anywhere on this platform. Every tag filter is matched in memory *after* a
page has been read, and it is only affordable because the partition — one gig, one feed, one user
— already bounds that read.

Two consequences you must handle:

1. **A filtered page can be shorter than `limit` while `next_cursor` is still set.** Page until
   `next_cursor` is `null`.
2. **There is no cross-gig tag search.** Filtering is always scoped to something you already
   named. That is a deliberate limit, not a missing feature.
