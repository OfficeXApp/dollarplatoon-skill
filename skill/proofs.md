# Proofs — submitting, reviewing, and sharing

A proof is the evidence a task was done. It is what triggers payment.

## Contents

- Routes
- Submit a proof
- Drafts — save before sending, or take a proof back
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
| POST | `/gigs/:id/proofs` | Worker | Submit a proof (`draft: true` saves without sending) |
| GET | `/gigs/:id/proofs` | Yes | List proofs, filterable by status |
| GET | `/gigs/:id/proofs/:proof_id` | Owner or submitter | One proof |
| PATCH | `/gigs/:id/proofs/:proof_id/draft` | Submitter | Edit an unsent draft |
| POST | `/gigs/:id/proofs/:proof_id/submit` | Submitter | Send a draft to the client |
| POST | `/gigs/:id/proofs/:proof_id/withdraw` | Submitter | Take a pending proof back to draft |
| DELETE | `/gigs/:id/proofs/:proof_id` | Submitter | Delete a draft |
| PATCH | `/gigs/:id/proofs/:proof_id` | Owner | Approve or reject |
| POST | `/gigs/:id/proofs/:proof_id/report` | Owner | Flag an auto-approved proof |
| PATCH | `/gigs/:id/proofs/:proof_id/tags` | Owner or submitter | Retag |
| PATCH | `/gigs/:id/proofs/:proof_id/alias` | Owner or submitter | Your own private title |
| POST | `/upload/presign` | Yes | Presigned S3 upload URL |

> **The "Owner" column reads differently on an order machine.** On an `inbound_order` gig the gig
> owner is the *vendor being judged*, so the verdict, the report and the proof reads belong to the
> **participant who paid**. The withdraw route is refused outright. Each difference is noted
> inline below and set out in full in
> [orders.md](https://dollarplatoon.com/skill/orders.md).

## Submit a proof

```json
POST /gigs/:id/proofs
{
  "mailbox_id": "MBX_01HX...",
  "task_identifier": "TASK_01HX...",
  "proofs": ["https://reddit.com/r/...", "https://s3.../screenshot.png"],
  "tags": ["batch_7"],
  "private_note": "Licence key: ABC-123-XYZ",
  "asking_price": 50
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

## Name your own price

A gig with `allow_price_offers: true` lets the worker quote their own number:

```json
POST /gigs/:id/proofs   { ..., "asking_price": 50 }

→ { "proof": { "id": "PROOF_01HX...", "locked_price": 10, "asking_price": 50 } }
```

**An ask is a quote, never a payout.** `locked_price` still holds the task or gig price. The
client accepts by approving with `amount` equal to `asking_price`; the proof then records
`price_source: "worker"`. The client can also approve at the gig price, approve at a third
number, or reject the proof.

**Silence pays the gig price.** A review that times out settles at `locked_price`, so an ask can
never be won by a client who stops answering. Bulk approval in the dashboard does the same.

On a gig without the flag, `asking_price` returns `400`. Set it with
`PATCH /gigs/:id  { "allow_price_offers": true }`.

The `warning` field appears when the gig's `available_funds` is below the task price. The proof is
still accepted and can still be approved, but it cannot be paid until the client deposits more.

## Drafts — save before sending, or take a proof back

`draft` is a proof the client has not been shown. Two things put a proof there, and they are the
same state afterwards:

- **Save before sending** — `POST /gigs/:id/proofs` with `"draft": true`
- **Withdraw** — `POST /gigs/:id/proofs/:proof_id/withdraw` on a proof still `pending`

```json
POST /gigs/:id/proofs
{ "mailbox_id": "MBX_01HX...", "task_identifier": "TASK_01HX...",
  "proofs": ["work in progress"], "draft": true }

→ { "proof": { "id": "PROOF_01HX...", "status": "draft", "timeout_at": null },
    "warning": "This is a draft. The client cannot see it until you POST .../submit." }
```

**A draft claims its task, exactly like a submission.** That is the point — two workers cannot
both draft the same task and then both try to send it. It also means the task counts against your
`max_open_tasks` cap and your rate-limit window while the draft sits there.

**Nothing happens to a draft on its own.** No review clock, no auto-approval, no `proof_webhook`,
no payout, and it is not counted in `proofs_submitted`. The client cannot see it at all: it is
filtered out of their dashboard, their proof list, and `GET /gigs/:id/proofs/:proof_id` returns
`404` to them.

### Editing and sending

```json
PATCH /gigs/:id/proofs/:proof_id/draft
{ "proofs": ["https://..."], "private_note": "Licence key: ABC-123", "asking_price": 50 }

→ { "success": true, "status": "draft", "updated": ["proofs", "private_note", "asking_price"] }
```

Send only the fields you are changing. `private_note` and `asking_price` accept `null` to clear.
`task_identifier` cannot be changed — delete the draft and start again against the other task.

```json
POST /gigs/:id/proofs/:proof_id/submit

→ { "success": true, "status": "pending", "timeout_at": "2026-08-22T...",
    "locked_price": 2.5, "resubmitted": false }
```

**The price is snapshotted at submit, not when the draft was saved.** A draft can sit for days
while the client reprices the task, so the number you agree to is the one showing when you send.
The client may also reprice a task freely while it carries only a draft.

### Withdrawing

```json
POST /gigs/:id/proofs/:proof_id/withdraw

→ { "success": true, "status": "draft", "withdrawn_from": "pending" }
```

**The web app calls this button "Undo".** It is the same endpoint — the label avoids reading as a
withdrawal of money.

**Only a `pending` proof can be withdrawn.** Once the client has approved or rejected it, the
verdict is theirs — withdrawing a rejection would erase it from your record. Anything already
reviewed returns `409`.

**Resending restarts the review clock from zero.** The client gets a full `review_timeout` window
on work they are seeing for the first time. The webhook fires again with `"resubmitted": true`,
but `proofs_submitted` is credited only once, on the first send, and only the first send writes
a `proof_submitted` event.

### Deleting

```json
DELETE /gigs/:id/proofs/:proof_id      // drafts only; a submitted proof is a permanent record

→ { "success": true, "id": "PROOF_01HX...", "deleted": true }
```

Deleting frees the `task_identifier` so a fresh proof can be started against it. Your claim on
the task is left alone — abandon it and `task_timeout` recycles it as normal.

**A draft is discarded for you when the task leaves you.** Declining it, reporting it, an owner
recycling or reassigning it, and the expiry sweep all delete your draft, because otherwise it
would block the next worker's proof against that task.

## The proof lifecycle

```
draft  (no clock, invisible to the client — POST with draft:true, or withdraw a pending proof)
  ├─ deleted           (worker action, or the task leaves them)
  └─ submit ─┐
             ↓
submitted  (locked_price snapshot, timeout_at set)
  ├─ draft             (worker withdraws; pending only, and only before a payout)
  ├─ approved          (client action)            → rolled up → paid on-chain → paid_out_at set
  ├─ timeout_approved  (cron, after review_timeout) → same rollup path
  ├─ rejected          (needs a rejection_tag; returns the task to be done again by default)
  │    approved ⇄ rejected: the client may change the verdict until a rollup carries the proof,
  │                         and never once the rejection returned the task
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
{ "action": "approve", "amount": 50 }                                    // accept a worker's ask
{ "action": "reject", "rejection_tag": "incomplete", "feedback": "Screenshot doesn't match" }

→ { "success": true, "status": "approved", "locked_price": 7.5, "price_source": "review" }
```

**Review promptly.** Silence is approval: a proof auto-approves after the gig's `review_timeout`
(default 48 hours). That rule exists to protect workers from a client who disappears.

**On an order machine the reviewer is the participant who paid, not the gig owner.** The owner is
the vendor who did the work, and this route refuses them — the authority is resolved from the
task's buyer. `review_timeout` is also the vendor's only structural protection there, which is why
that mode forbids `-1` (manual review, no auto-approval) and anything under an hour.

**`feedback` is stored on either verdict and the gigworker reads it.** On a rejection it explains
the problem. On an approval it is your reply to the submission — which is what makes an
`inbound_proof` gig work as an application inbox: the proof *is* the application, and the approval
carries the answer.

```json
{ "action": "approve",
  "feedback": "You are good. Assigned to team abc — join the groupchat: https://..." }
```

Maximum 4000 characters. Omit it, or send `""` or `null`, to leave no note.

`amount` is accepted on a proof whose `locked_price` is still `null`, and on a proof that carries
an `asking_price`. Sending it for any other already-priced proof returns `409` — the price a
worker agreed to at submission cannot be lowered at review. A worker's own ask reopens the
number because they, not the client, named it.

**Automate it** by setting `proof_webhook_url` on the gig: every submission is POSTed there, so
your own validator or AI agent can check the work before you look at it.

## A rejection returns the task by default

Rejecting a proof sends its task back out to be done again, unless you say otherwise. The work
you asked for is still work you want; before this, a rejected task was closed for good and
nobody — not another worker, not you — could ever pick it up.

```json
PATCH /gigs/:id/proofs/:proof_id

{ "action": "reject", "rejection_tag": "incomplete" }                   // task goes back out
{ "action": "reject", "rejection_tag": "not_selected", "requeue": false }  // task closes too

→ { "success": true, "status": "rejected", "requeued": true, "task_identifier": "TASK_..." }
```

**`requeue` defaults to `true`.** Existing automation that rejects proofs now returns tasks
without asking. Send `"requeue": false` wherever a rejection is meant to end the task — hiring
gigs that reject every applicant but one are the usual case.

Where the task actually lands depends on the gig's distribution:

| Distribution | Where the task goes |
|---|---|
| `queue` | back into the queue at its own position, this worker excluded |
| `queue_solo` | this worker's copy is dropped, the claim slot returns to the template |
| assigned task | back to you as `UNASSIGNED`, brief intact — never into a shared queue |
| `push` | offered to another mailbox that accepts it |

**A returned rejection is final.** Another worker may already hold that task, so approving the
old proof afterwards would pay twice for one task. A second `PATCH` on it returns `409`:

```json
{ "error": "This rejection returned the task to be done again, so its verdict can no longer change",
  "status": "rejected", "task_released_at": "2026-08-21T10:00:00.000Z" }
```

Read the proof to check before you ask — the owner's copy carries `task_released_at`, and
`can_revise` is `false` with `revision_blocked_by_requeue: true`.

**When there is nothing to return**, the rejection still stands and `requeued` comes back
`false` with a reason. That happens on `push` and `inbound_proof` gigs where the
`task_identifier` is free-form — a URL, a ticket number — and names no task row:

```json
→ { "success": true, "status": "rejected", "requeued": false,
    "requeue_error": "This gig's task identifier does not name a task that can be returned" }
```

Nothing is lost in that case: the proof keeps its undo, exactly as a `requeue: false` rejection
does.

**One live proof per task.** A task may collect several rejected attempts over its life, but
only one proof owns it at a time. Every other rule follows from that — a released attempt no
longer blocks a new submission, no longer stops you recycling or repricing or editing the task,
and no longer counts against the worker's open-task cap.

## Change a verdict (undo an accidental approve or reject)

Send the same `PATCH` again with the other verdict. There is no separate undo endpoint.

```json
PATCH /gigs/:id/proofs/:proof_id

{ "action": "approve", "feedback": "Rejected this by mistake — approved." }   // was rejected
{ "action": "reject", "rejection_tag": "incomplete" }                          // was approved

→ { "success": true, "status": "approved", "revised": true, "previous_status": "rejected" }
```

**The window closes when a payout picks the proof up.** The daily cron rolls approved proofs into
a payout, and sweeps rejected ones into a $0 rollup so they stop being reconsidered. Once either
happens the verdict is final and a second `PATCH` returns `409`:

```json
{ "error": "This proof was already settled in a payout and its verdict can no longer change",
  "status": "approved", "rollup_id": "ROLLUP_...", "rollup_status": "paid" }
```

So an undo is same-day work. To check before you ask, read the single proof — the owner's copy
carries `can_revise` on any `approved` or `rejected` proof:

```json
GET /gigs/:id/proofs/:proof_id
→ { "status": "rejected", "can_revise": true, "revision_blocked_by_rollup_id": null }
```

A rollup is not the only thing that closes the window. A rejection that returned its task is
final from the moment it was made — see "A rejection returns the task by default" above.

What a revision changes:

- `feedback` is replaced by what you send now, and **cleared if you omit it**. The old note
  belonged to the verdict you are replacing.
- `rejection_tag` is cleared when you change to `approve`, and required when you change to
  `reject`.
- Every verdict writes its own event and none are overwritten, so a revision is visible as a
  revision. A reader wanting one verdict per proof takes the latest by timestamp.
- `locked_price` follows the normal rule: `amount` is accepted only on a proof still priced
  `null` or carrying an `asking_price`.

`timeout_approved` cannot be revised. The cron approved it, not you — dispute it with
`POST .../report` instead.

## Rejection tags and what they cost

A `rejection_tag` is required when rejecting. It is recorded on the rejection's event and is
the reason anyone reading the ledger sees. It carries no score — there is no score — so the
column below is about what each tag MEANS, not what it costs:

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

Do not reach for `not_selected` to soften a genuine quality problem. The ledger is the only
signal this platform publishes, and mislabelling bad work removes the one thing everyone else
depends on.

## Report an auto-approved proof

```json
POST /gigs/:id/proofs/:proof_id/report   → { "success": true, "status": "reported" }
```

Works only on `timeout_approved` proofs — the ones that were approved because you missed the
window. Reported proofs are excluded from rollups and will not be paid.

**On an order machine `report` works in both directions**, and the row records which
(`reported_by: "client" | "gigworker"`). It has to: the clock approving a delivery nobody reviewed
is precisely the buyer's problem, and reporting is their only remedy when the timeout ruled
against them. An owner-only report there would point the remedy at the wrong party.

## Private delivery (`private_note`)

`proofs` is visible to the client the moment you submit. `private_note` is not. It is the
escrowed half of the delivery — the licence key, the password, the download link — and the
client cannot read it until they have actually paid for that proof.

```json
POST /gigs/:id/proofs   { ..., "private_note": "Licence key: ABC-123-XYZ" }
POST /public/submit-proof { ..., "private_note": "Licence key: ABC-123-XYZ" }
```

Optional, at most 8000 characters, on both submit routes. A blank string means "none".

Every proof response carries two fields for it, and the three states are readable from
`private_note` **alone** — you never have to cross-check the boolean to know where you stand:

| `private_note` | `private_note_locked` | Meaning |
|---|---|---|
| `""` | `false` | The worker attached no note. Nothing is coming, now or ever. |
| `null` | `true` | A note exists and is being withheld. Pay for the proof and read it again. |
| `"..."` | `false` | Released. |

The empty string is the point: `""` means *nothing was attached* and `null` means *you have not
paid yet*. Reporting `null` for both would leave a client who has already paid unable to tell a
worker who sent nothing from a note they still have to unlock. Both values are falsy, so
`if (!private_note)` still means "there is nothing to read right now".

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

**On an order machine this field is not optional practice — it is the vendor's only protection.**
The buyer reads `proofs[]` at review time and may undo the whole order, deposit and all, right up
until they approve. So put the deliverable in `private_note` and let `proofs[]` carry evidence
only: a watermarked preview, a hash, a word count, a description. Enough to rule on, not enough to
use. `POST .../proofs/:proof_id/withdraw` is refused in that mode for the same reason — the buyer
has already paid and may already have read it, so un-delivering would leave them holding a funded
order with nothing against it. See [orders.md](https://dollarplatoon.com/skill/orders.md).

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
| POST | `/public/upload-presign` | No | Presigned upload URL (any type, 100MB max) |

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
{ "filename": "delivery.zip", "content_type": "application/zip", "prefix": "proofs", "content_length": 5242880 }
→ { "presigned_url": "https://s3...", "url": "https://s3...", "key": "proofs/...", "bucket": "...", "max_bytes": 104857600 }
```

PUT the file to `presigned_url`, then put the returned `url` in your proof's `proofs` array.
Prefixes: `"proofs"` (default), `"avatars"`, `"gig-icons"`. The presigned URL expires in one hour.

Any file type is accepted — image, document, video, archive, design file. The limit is **100MB
per file**. Use `application/octet-stream` when you do not know the content type.

Send `content_length` with the exact byte count of the file. The API refuses a larger value with
`413`, and it signs the count into the URL, so the upload fails if the byte count does not match.
`content_length` is optional for older clients, but without it the size is not enforced.

The key holds a random UUID, so nobody can guess it. The bucket blocks public access; a proof URL
is signed for one hour when the API returns it. Files expire after 365 days.
