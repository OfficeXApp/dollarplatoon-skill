# For clients — getting work done and paying for it

You have work. This is how you turn it into a funded gig, get people into it, and pay them.

## Contents

- The whole flow in one script
- Step 1 — create the gig
- Step 2 — fund it (and why 110%)
- Step 3 — get workers in
- Step 4 — send tasks
- Step 5 — review proofs
- Step 6 — pay out
- Choosing a distribution mode
- Common client mistakes
- Where to go next

---

## The whole flow in one script

Runnable end to end. Each step links to the reference file that explains it properly.

```bash
KEY=$DOLLAR_PLATOON_API_KEY
API=https://dollarplatoon.com/api

# 1. Create a gig. It comes back with an invite link and an inbound webhook already wired.
GIG=$(curl -s -X POST $API/gigs -H "x-api-key: $KEY" -H "Content-Type: application/json" -d '{
  "title": "Reddit comments for launch",
  "price": 0.50,
  "terms": "Comment genuinely on the linked thread. Tags used here: reddit, urgent.",
  "distribution": "queue",
  "queue_order": "fifo",
  "review_timeout": 172800
}')
GIG_ID=$(echo "$GIG" | jq -r .gig.id)
TOKEN=$(echo "$GIG" | jq -r .gig.webhook | sed 's/.*token=//')
echo "$GIG" | jq -r .gig.invite_url          # send this to workers

# 2. Fund it. Budget 110% of what you expect to pay out.
curl -s -X POST $API/gigs/$GIG_ID/deposit -H "x-api-key: $KEY" \
  -H "Content-Type: application/json" -d '{"wallet_alias_id":"'$ALIAS'","amount":110}'

# 3. Push a task into the queue.
curl -s -X POST "$API/inbound/webhook/$GIG_ID?token=$TOKEN&subject=Comment+task&tags=reddit" \
  -H "Content-Type: application/json" \
  -d '{"thread_url":"https://reddit.com/r/example/comments/abc","comment":"..."}'

# 4. Read what came back in. Proofs arrive here.
curl -s "$API/gigs/$GIG_ID/dashboard" -H "x-api-key: $KEY" | jq '.proofs[] | {id, status, task_identifier}'

# 5. Approve one, with a note the worker will read.
curl -s -X PATCH $API/gigs/$GIG_ID/proofs/$PROOF_ID -H "x-api-key: $KEY" \
  -H "Content-Type: application/json" -d '{"action":"approve","feedback":"Nice work."}'

# 6. Pay. The daily cron also does this on its own.
curl -s -X POST $API/gigs/$GIG_ID/rollups -H "x-api-key: $KEY"
```

---

## Step 1 — create the gig

Full field reference: [gigs.md](https://dollarplatoon.com/skill/gigs.md).

The four decisions that matter, because changing them later is disruptive:

**`distribution`** — how tasks reach workers. See the table further down. If you are unsure,
`queue` with `fifo` is the safe default: workers pull work when they are ready, and nothing is
pushed to someone who is asleep.

**`price`** — what one task pays by default. Individual tasks can override it, and a task can be
priced `tbd` and settled at approval. See
[pricing-and-tags.md](https://dollarplatoon.com/skill/pricing-and-tags.md). For market rates by
task type, see [prices.md](https://dollarplatoon.com/skill/prices.md).

**`review_timeout`** — seconds before an unreviewed proof **auto-approves**. Default 48 hours.
This protects workers from a client who disappears. If you cannot commit to reviewing, either
raise it or accept that unreviewed work gets paid.

**`terms`** — the one field a worker reads before joining. Put your tag vocabulary here. A worker
sets their filters before they have seen a single task, so if you send `shortform-vertical` and
they guess `short`, they silently receive nothing and neither of you gets an error.

Every new gig is created with `join_policy: "invite"` and a **default unlimited invite link**,
returned as `invite_url`. There is no public marketplace — `GET /gigs` returns `410`.

## Step 2 — fund it (and why 110%)

```json
POST /gigs/:id/deposit   { "wallet_alias_id": "...", "amount": 100 }
→ { "tx_hash": "0x...", "available_funds": 100 }
```

- The worker receives the **full** amount they earned. The platform fee is charged **on top**
  from the gig balance. A $100 payout costs the gig $110.
- **Funds are locked.** There is no withdrawal function. USDC leaves a gig only as worker
  payouts. Deposit conservatively and top up.
- **A gig cannot go into debt.** A rollup pre-checks `available_funds >= gross + fee` and fails
  entirely if the gig cannot cover it.
- Underfunding does not block submissions. Workers can still submit; the proof is approved and
  simply cannot be paid until you deposit. That is a bad look — fund first.

## Step 3 — get workers in

Gigs are private networks. People join through an invite link.

```json
POST /gigs/:id/invites   { "max_uses": 1, "email": "worker@example.com", "label": "for Alice" }
→ { "invite": { "token": "a1b2c3d4e5f6", "invite_url": "https://dollarplatoon.com/gig/GIG_.../join?invite=..." } }
```

Two fields give you every mode you need:

- `max_uses`: `1` for one person, `N` for a cohort, `null` for an open link you can post.
- `email`: bind the link to one address, or `null` for anyone holding it. An email-bound invite
  is also a **pre-approval** — that worker skips `pending_approval` even on a gig that requires it.

Revoke with `DELETE /gigs/:id/invites/:token`. Use consumption is atomic, so concurrent joins
cannot race past `max_uses`.

To reach many workers at once through one relationship, publish the gig to a **feed** —
[feeds.md](https://dollarplatoon.com/skill/feeds.md).

## Step 4 — send tasks

Full reference: [tasks.md](https://dollarplatoon.com/skill/tasks.md).

Prefer the webhook over email. It is instant, it takes structured data, and it accepts the query
params that carry a task's price, tags, priority, and assignee:

```bash
POST /inbound/webhook/:gig_id?token=...&price=2.50&tags=shortform&priority=0
```

For a human without an account — a teammate, a partner — send them the **Insert Task page**
instead: `https://dollarplatoon.com/insert/{GIG_ID}?token={SECURITY_TOKEN}`. Same destination, no
login. See [web-pages.md](https://dollarplatoon.com/skill/web-pages.md).

## Step 5 — review proofs

Full reference: [proofs.md](https://dollarplatoon.com/skill/proofs.md).

```json
PATCH /gigs/:id/proofs/:proof_id
{ "action": "approve", "feedback": "Great work!" }
{ "action": "reject", "rejection_tag": "incomplete", "feedback": "Screenshot doesn't match" }
```

- **Review promptly.** Silence approves after `review_timeout`.
- **Always send a `rejection_tag`.** It is written to the rejection's event and is the reason
  anyone reading the worker's ledger will see. It costs them no score — there is no score — but
  it is the only signal other clients get, so label honestly.
- **Use `not_selected` when you simply hired someone else.** It says "did not get the job", not
  "did bad work", which is what makes free application tasks safe to run.
- **A rejection returns the task by default.** The work goes back out so somebody else can do
  it. Send `"requeue": false` to close the task with the proof — which is what you want on a
  hiring gig, where rejecting the other applicants must not re-post the job. A returned
  rejection is final: another worker may hold that task now, so its verdict cannot change.
- **`feedback` is read by the worker on approvals too.** On an `inbound_proof` gig the proof *is*
  the application, so the approval is where you answer: "you're in, join the groupchat: <link>".
- **Changed your mind? Send the same `PATCH` with the other verdict.** An accidental reject
  becomes an approval, and the reverse — until a payout picks the proof up, which the daily cron
  does. After that the call returns `409`.
- **A pending proof can disappear.** A worker may withdraw their own submission back to a private
  draft for as long as you have not reviewed it, and it then leaves your dashboard entirely. You
  never see a draft. If they send it again the webhook fires a second time carrying
  `"resubmitted": true`, and your review window restarts from that moment. Reviewing a proof ends
  this — once you have approved or rejected, the verdict is yours alone.
- Automate it by setting `proof_webhook_url` on the gig and routing submissions to your own
  validator or agent.

## Step 6 — pay out

Full reference: [payouts.md](https://dollarplatoon.com/skill/payouts.md).

```json
POST /gigs/:id/rollups
→ { "rollups": [...], "available_funds": 44.50, "retried_stuck": 1, "skipped_below_minimum": [...] }
```

A daily cron does this on its own; the manual call is for paying immediately.

**Never re-create a failed rollup by hand.** A payout that fails — or one that merely took more
than 20 seconds to confirm — is retried automatically, reusing the *same* rollup, and the chain
is checked first so nobody is paid twice. Treat a single `failed` as "not settled yet", never as
"lost". Triggering a second payout for the same proofs is how you pay twice.

---

## Choosing a distribution mode

| Mode | Behaviour | Reach for it when |
|---|---|---|
| `queue` | Shared queue. The first worker to poll a task gets it. | The default. High volume, interchangeable work. |
| `queue_solo` | Every worker gets their own private copy of each task. | You want the same task done by N people (surveys, ratings, redundancy). **Cost is price × workers.** |
| `round_robin` | Pushed, rotating fairly through active mailboxes. | Even workload across a known roster. |
| `random` | Pushed to one mailbox at random. | Simple spread, no fairness guarantee needed. |
| `priority_weighted` | Pushed, weighted by each mailbox's `priority` (1–10). | You want your best workers to get more. |
| `free_for_all` | Pushed to every active mailbox. | Announcements, or races where you want the first result. |
| `inbound_proof` | No tasks at all; workers submit proofs directly. | Applications, bounties, anything where the submission *is* the work. |
| `inbound_order` | **Inverted.** You do the work; outsiders send and fund each order. | You are selling something, not buying it. See [orders.md](https://dollarplatoon.com/skill/orders.md). |

`queue_solo` is the one to think twice about: ten tasks and five workers is fifty payouts, not
ten. Bound it with `max_claims_per_task`.

`inbound_order` is not a variant of this page at all — it is the other side of the counter. If
you pick it, nothing on this page applies: you do not fund the gig, you do not send tasks, and you
do not approve anything. Read [orders.md](https://dollarplatoon.com/skill/orders.md) first and
decide deliberately.

## Common client mistakes

- **Approving `$0` application proofs.** A rollup skips any mailbox whose approved total is `$0`,
  so those rows are rescanned forever and grow without bound. **Reject** applications with
  `not_selected` instead — same outcome for the applicant, clean ledger for you.
- **Letting a queue hand a $500 job to whoever polls first.** Use assignment. See the hiring
  walkthrough in [queue.md](https://dollarplatoon.com/skill/queue.md).
- **Putting the real brief in the public payload.** Advertise in the payload, keep the substance
  in `private_details`, which only the holder sees.
- **Rotating the security token and forgetting the integrations.** Rotation invalidates the old
  email address, webhook URL, and every Insert Task link at once.
- **Assuming a worker sees your tags.** If your gig takes tasks by email, set
  `default_task_tags` — an email has nowhere to carry `?tags=`, so filtered workers get nothing.

## Where to go next

- Gig fields, invites, mailboxes, funding → [gigs.md](https://dollarplatoon.com/skill/gigs.md)
- Sending tasks and payload formats → [tasks.md](https://dollarplatoon.com/skill/tasks.md)
- Ordering, assigning, and pricing work → [queue.md](https://dollarplatoon.com/skill/queue.md),
  [pricing-and-tags.md](https://dollarplatoon.com/skill/pricing-and-tags.md)
- Dashboards and embeds you can hand to a partner →
  [web-pages.md](https://dollarplatoon.com/skill/web-pages.md)
