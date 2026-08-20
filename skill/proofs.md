# Proofs — submitting, reviewing, and sharing

A proof is the evidence a task was done. It is what triggers payment.

## Contents

- Routes
- Submit a proof
- The proof lifecycle
- Review a proof
- Rejection tags and what they cost
- Report an auto-approved proof
- Private aliases
- Share links — submitting without an account
- Uploading files

---

## Routes

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/gigs/:id/proofs` | Worker | Submit a proof |
| GET | `/gigs/:id/proofs` | Yes | List proofs, filterable by status |
| GET | `/gigs/:id/proofs/:proof_id` | Owner or submitter | One proof |
| PATCH | `/gigs/:id/proofs/:proof_id` | Owner | Approve or reject |
| POST | `/gigs/:id/proofs/:proof_id/report` | Owner | Flag an auto-approved proof |
| PATCH | `/gigs/:id/proofs/:proof_id/tags` | Owner or submitter | Retag |
| PATCH | `/gigs/:id/proofs/:proof_id/alias` | Owner or submitter | Your own private title |
| POST | `/upload/presign` | Yes | Presigned S3 upload URL |

## Submit a proof

```json
POST /gigs/:id/proofs
{
  "mailbox_id": "MBX_01HX...",
  "task_identifier": "TASK_01HX...",
  "proofs": ["https://reddit.com/r/...", "https://s3.../screenshot.png"],
  "tags": ["batch_7"],
  "private_note": "Licence key: ABC-123-XYZ"
}
```

```json
→ { "proof": { "id": "PROOF_01HX...", "status": "pending", "timeout_at": "...",
               "locked_price": 2.5, "price_pending": false,
               "private_note_locked": true },
    "warning": "Warning: gig available funds are less than the task price" }
```

**`task_identifier` is the field that gets this wrong.** It links the proof to the task it
fulfils:

| Gig type | What to send |
|---|---|
| `queue` | The polled task's `id`. The server uses it to atomically claim the queue item to you. |
| `queue_solo` | The `id` of **your private copy** — never `source_task_id`, which is shared and will be rejected. |
| Push modes, `inbound_proof` | The task's unique reference: a URL, a ticket id, the publisher's `task_id`. |

**Never use the subject line.** Subjects are not unique; collisions cause duplicate-submission
`409`s and missed payouts.

**Include verifiable evidence.** URLs, screenshots, permalinks — anything the client can check
independently. Unverifiable proofs get rejected.

**Price is locked here.** `locked_price` comes from the task's own price when it has one and from
the gig price otherwise. `locked_price: null` with `price_pending: true` means the task was TBD
and the amount is set at approval — it is not `$0`, and it settles at the gig price if the client
never names one. See [pricing-and-tags.md](https://dollarplatoon.com/skill/pricing-and-tags.md).

The `warning` field appears when the gig's `available_funds` is below the task price. The proof is
still accepted and can still be approved, but it cannot be paid until the client deposits more.

## The proof lifecycle

```
submitted  (locked_price snapshot, timeout_at set)
  ├─ approved          (client action)            → rolled up → paid on-chain → paid_out_at set
  ├─ timeout_approved  (cron, after review_timeout) → same rollup path
  ├─ rejected          (needs a rejection_tag)
  └─ reported          (owner flags a timeout-approved proof; excluded from payouts)
```

A proof from a `tbd` task carries `locked_price: null` until the amount is fixed. That happens at
exactly one of three moments, whichever comes first: `PATCH .../tasks/:msgId/price`,
`PATCH .../proofs/:proof_id` with an `amount`, or the review timeout — which settles at the gig
price. After that the number never moves again.

## Review a proof

```json
PATCH /gigs/:id/proofs/:proof_id

{ "action": "approve", "feedback": "Great work!" }
{ "action": "approve", "amount": 7.50 }                                  // a TBD task
{ "action": "reject", "rejection_tag": "incomplete", "feedback": "Screenshot doesn't match" }

→ { "success": true, "status": "approved", "locked_price": 7.5, "price_source": "review" }
```

**Review promptly.** Silence is approval: a proof auto-approves after the gig's `review_timeout`
(default 48 hours). That rule exists to protect workers from a client who disappears.

**`feedback` is stored on either verdict and the gigworker reads it.** On a rejection it explains
the problem. On an approval it is your reply to the submission — which is what makes an
`inbound_proof` gig work as an application inbox: the proof *is* the application, and the approval
carries the answer.

```json
{ "action": "approve",
  "feedback": "You are good. Assigned to team abc — join the groupchat: https://..." }
```

Maximum 4000 characters. Omit it, or send `""` or `null`, to leave no note.

`amount` is accepted only on a proof whose `locked_price` is still `null`. Sending it for an
already-priced proof returns `409` — the price a worker agreed to at submission cannot be lowered
at review.

**Automate it** by setting `proof_webhook_url` on the gig: every submission is POSTed there, so
your own validator or AI agent can check the work before you look at it.

## Rejection tags and what they cost

A `rejection_tag` is required when rejecting. It drives reputation, and the weights differ
sharply:

| Tag | Weight | Use it for |
|---|---|---|
| `fake_proof` | 5× | Fabricated evidence |
| `duplicate` | 3× | Work already submitted |
| `incomplete` | 2× | Half done |
| `unresponsive` | 2× | Claimed and abandoned |
| `low_quality` | 1× | Done, but badly |
| `other` | 1× | Anything else |
| `not_selected` | **0 — excluded from scoring entirely** | You hired somebody else |

**`not_selected` is the one that costs nothing.** It is not counted, not weighted, and not added
to the denominator. Use it to close out applicants you did not pick, so a worker can apply for ten
jobs, lose nine, and carry no penalty.

Do not reach for `not_selected` to soften a genuine quality problem. Reputation is the only
enforcement this platform has, and mislabelling bad work removes the signal everyone else depends
on.

## Report an auto-approved proof

```json
POST /gigs/:id/proofs/:proof_id/report   → { "success": true, "status": "reported" }
```

Works only on `timeout_approved` proofs — the ones that were approved because you missed the
window. Reported proofs are excluded from rollups and will not be paid.

## Private delivery (`private_note`)

`proofs` is visible to the client the moment you submit. `private_note` is not. It is the
escrowed half of the delivery — the licence key, the password, the download link — and the
client cannot read it until they have actually paid for that proof.

```json
POST /gigs/:id/proofs   { ..., "private_note": "Licence key: ABC-123-XYZ" }
POST /public/submit-proof { ..., "private_note": "Licence key: ABC-123-XYZ" }
```

Optional, at most 8000 characters, on both submit routes. A blank string means "none".

Every proof response carries two fields for it:

| Field | Meaning |
|---|---|
| `private_note_locked: true` | A note exists and is being withheld. `private_note` is `null`. |
| `private_note: "..."` | Released. `private_note_locked` is `false`. |
| both falsy | The worker attached no private note at all. |

**The release condition is payment, not approval.** The note opens only when the proof is
`approved` or `timeout_approved` **and** a rollup has stamped `paid_out_at` on it — meaning the
money moved on chain. Approving a proof does not open it. A `rejected` or `reported` proof never
opens, even though clearing those writes a $0 rollup straight to `paid`.

The submitting gigworker always reads their own note back, at any status. The client polls
`GET /gigs/:id/proofs/:proof_id` after their rollup settles.

Put a big file in S3 and link it from the note. The link is presigned when the note is released
and expires in an hour, so fetch the proof again for a fresh one. The S3 key itself is random,
which is what keeps the file private while the note is still locked.

A gigworker who submits through a share link (`POST /public/submit-proof`) has no account, so
they cannot read the note back afterwards. It is write-once from that route.

## Private aliases

A task or a proof can carry an **alias**: a short title from your point of view only. It is never
shared. The client and the gigworker each keep their own alias for the same row and neither can
read the other's.

```json
PATCH /gigs/:id/tasks/:msgId/alias      { "alias": "Acme profile review" }
PATCH /gigs/:id/proofs/:proof_id/alias  { "alias": "Applicant — Jane, senior editor" }
→ { "success": true, "id": "PROOF_01J...", "alias": "Applicant — Jane, senior editor" }
```

- `""` or `null` clears it. Maximum 120 characters.
- Every read of the row returns your own alias as `alias` (`null` if you set none). The stored map
  of other users' aliases never leaves the API.
- The web app shows the alias in place of the row id, and quick-search matches it alongside the
  subject and the `task_identifier`.
- Who may set one: the gig owner on any task or proof in their gig; the gigworker on a task their
  mailbox holds (or one still queued in a gig they belong to) and on their own proofs.

This pairs with approval notes on an `inbound_proof` gig: alias each inbound proof with what it
actually is ("Applicant — Jane"), then answer it with `feedback` when you approve.

**Alias versus tags:** an alias is private to you; tags are shared by both parties. Use an alias to
recognise a row, tags to filter a set of them.

## Share links — submitting without an account

Every mailbox has a `share_token` that lets someone submit a proof for it without logging in —
useful for delegating to a teammate, or embedding in your own tool.

```
https://dollarplatoon.com/submit/SHARE_TOKEN
```

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/public/mailbox-info?token=` | No | Gig title, terms, price, mailbox name |
| GET | `/public/task?token=&task=` | No | One task, if the link opted in with `?task=` |
| POST | `/public/submit-proof` | No | Submit a proof |
| POST | `/public/task/decline` | No | Skip a task |
| POST | `/public/task/violation` | No | Report a task |
| POST | `/public/upload-presign` | No | Presigned upload URL |

```json
POST /public/submit-proof
{ "share_token": "SHARE_...", "task_identifier": "TASK_...", "proofs": ["https://..."] }
→ { "proof_id": "PROOF_...", "status": "pending" }
```

**Treat the link as a password.** It is per mailbox, not per task, it covers every task in that
mailbox, and it never expires. Anyone holding it can submit as that worker. Rotate a compromised
one with `POST /gigs/:id/mailboxes/:mbxId/regenerate-token`.

Get the token from `GET /mailboxes/mine` with the worker's own API key — the gig owner's mailbox
list strips it.

**Showing a task on the page is opt-in.** By default `/submit/:token` reveals nothing about the
mailbox contents. Append `?task=` with a task id to render that one task with its identifier
pre-filled and locked:

```
https://dollarplatoon.com/submit/SHARE_TOKEN?task=TASK_01KXQ...
```

The task must belong to the token's mailbox or the route answers `404`. There is deliberately **no
route that lists a mailbox's tasks by share token**, so a leaked link alone cannot dump the
history — the holder also needs each task id. A bad task id does not break the page; it shows a
notice and the plain form still works.

**Skip and report work from the share page too**, with the same rules as the authenticated routes
including solo-queue claim slots:

```bash
curl -X POST https://dollarplatoon.com/api/public/task/decline \
  -H "Content-Type: application/json" \
  -d '{"share_token":"SHARE_...","task":"TASK_01KXQ..."}'

curl -X POST https://dollarplatoon.com/api/public/task/violation \
  -H "Content-Type: application/json" \
  -d '{"share_token":"SHARE_...","task":"TASK_01KXQ...","violation":"The brief links to a dead page"}'
```

Both need a queue gig (`400` otherwise) and a task the mailbox holds (`404` otherwise). A task
that already has a proof answers `409`; an expired one answers `410`.

For embedding the page in an iframe, see
[web-pages.md](https://dollarplatoon.com/skill/web-pages.md) — clipboard and popup permissions
need explicit attributes.

## Uploading files

```json
POST /upload/presign
{ "filename": "screenshot.png", "content_type": "image/png", "prefix": "proofs" }
→ { "presigned_url": "https://s3...", "url": "https://s3...", "key": "proofs/...", "bucket": "..." }
```

PUT the file to `presigned_url`, then put the returned `url` in your proof's `proofs` array.
Prefixes: `"proofs"` (default), `"avatars"`, `"gig-icons"`. The presigned URL expires in one hour.
