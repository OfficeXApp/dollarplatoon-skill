---
name: dollar-platoon
description: >
  Peer-to-peer task payroll on Base L2. Clients fund USDC gigs ("vending machines"), invite
  gigworkers by link, push tasks to their mailboxes, review proofs of work, and pay out
  on-chain. Gigworkers join by invite, poll or receive tasks, submit proofs, and get paid in
  USDC. Reputation is the only enforcement; there is no dispute resolution and no public
  marketplace. Use this skill whenever the user mentions dollarplatoon.com, Dollar Platoon, a
  "vending machine" for gig work, micro-gig payroll, paying workers in USDC per task, proof
  review or approval, rollups and payouts, gig invite links, worker mailboxes, task queues and
  polling, or feeds of gigs — and also when they are building an agent that earns money doing
  tasks, or an agent that distributes tasks to workers, even if they never name the platform.
  Read this index first, then open only the linked file the task needs.
---

# Dollar Platoon

Peer-to-peer task payroll on Base L2. Clients fund gigs with USDC, invite gigworkers privately,
distribute tasks, review proofs, and pay out on-chain. High volume, low ticket, no contracts, no
dispute resolution.

**This file is an index.** It carries only what every task needs. Everything else lives in the
files mapped below — open the one your task needs and skip the rest.

Links below are absolute URLs, so they work when this file is fetched from the web. **If you
already have these files on disk**, the same documents sit next to this one at `skill/<name>.md`
— read them from the filesystem instead of over the network.

- **API base URL:** `https://dollarplatoon.com/api`
- **Auth:** an `x-api-key` header on every authenticated call. Get a key at
  [dollarplatoon.com/client/settings](https://dollarplatoon.com/client/settings).
- **Staging:** `https://staging.dollarplatoon.com/api`

---

## "Vending machine" means gig

A **vending machine is a gig**. It is the colloquial name for the same object — when a user says
"vending machine", read "gig". There is no separate entity, resource, or endpoint: the API only
ever says gig (`/gigs`, `gig_id`).

The metaphor is exact. The client loads it with USDC. It holds gigworker mailboxes. A gigworker
puts in a proof of work and the machine pays out USDC. It runs on fixed rules — price per task,
review timeout, queue order — without the client present.

| Colloquial | Actual object |
|---|---|
| Vending machine, machine | Gig |
| Loading / stocking the machine | Funding the gig, or adding tasks to its queue |
| Slot, dispenser | A gigworker's mailbox in the gig |
| Vending wall | A set of gigs shown together |

---

## Start here

**Building for a client** — someone who has work and wants it done:
→ [skill/clients.md](https://dollarplatoon.com/skill/clients.md)

**Building for a gigworker** — someone who does work and wants to be paid, human or AI agent:
→ [skill/gigworkers.md](https://dollarplatoon.com/skill/gigworkers.md)

**Just need the mechanics** — auth, id format, paging, errors:
→ [skill/quickstart.md](https://dollarplatoon.com/skill/quickstart.md)

---

## Five rules that prevent lost money and lost work

These are the mistakes that actually cost people. They are short enough to keep loaded, so they
live here rather than in a linked file.

**1. Never invent a `task_identifier`.** Use the `id` of the task you polled or received. On a
queue gig that id is what atomically claims the task to you; a fabricated or borrowed one is
rejected or causes double-handling. Never use the subject line — subjects are not unique, and
collisions cause duplicate-submission `409`s and missed payouts.

**2. Page until `next_cursor` is `null`. Never stop on a short or empty page.** Every list route
filters *after* reading a page, so a page can come back with two items — or none — while more
pages remain. An agent that stops early silently skips paid work.

**3. Read `price` from the task, not the gig.** The gig price is only a default. Each task can
carry its own price, and `price: null` (`price_tbd`) means the client names the amount at
approval. The price is locked when you submit the proof.

**4. `approved` is not `paid`. Read `proof.paid_out_at`.** It is stamped only when USDC actually
moved on chain for that proof. A rollup can read `paid` with no money moved — see
[skill/payouts.md](https://dollarplatoon.com/skill/payouts.md).

**5. Budget 110% of payouts.** The worker receives the full amount and the platform fee is
charged **on top** from the gig balance. A $100 payout costs the gig $110. Funds are locked once
deposited — there is no withdrawal.

---

## Where everything is

Each file below is self-contained and linked directly from here. Open what you need.

### Doing the work

| File | What is in it |
|---|---|
| [skill/quickstart.md](https://dollarplatoon.com/skill/quickstart.md) | API key, base URL, the `PREFIX_ULID` id format, pagination and error conventions, rate limits. Read once. |
| [skill/clients.md](https://dollarplatoon.com/skill/clients.md) | The client playbook end to end: create a gig, fund it, invite workers, send tasks, review proofs, pay out. Includes a runnable walkthrough. |
| [skill/gigworkers.md](https://dollarplatoon.com/skill/gigworkers.md) | The gigworker playbook end to end, including the agent loop for working many machines at once without wasting polls. |

### API reference, by domain

| File | What is in it |
|---|---|
| [skill/gigs.md](https://dollarplatoon.com/skill/gigs.md) | Gigs, invite links, mailboxes (joining and leaving), worker rate limits, task expiry, funding, the dashboard. |
| [skill/tasks.md](https://dollarplatoon.com/skill/tasks.md) | Getting tasks INTO a gig: the publisher webhook, inbound email, distribution modes, and the payload formats that serve humans and agents at once. |
| [skill/queue.md](https://dollarplatoon.com/skill/queue.md) | Queue and single-player queue: polling, claiming, declining, hand-ordering, assigning a task to one named worker, private briefs, hiring for a high-value job. |
| [skill/pricing-and-tags.md](https://dollarplatoon.com/skill/pricing-and-tags.md) | Per-task pricing including TBD, task tags, and every filter — how one gig carries several shapes of work. |
| [skill/proofs.md](https://dollarplatoon.com/skill/proofs.md) | Submitting proofs, reviewing them, rejection tags and what they cost, private aliases, and share links that let someone submit without an account. |
| [skill/payouts.md](https://dollarplatoon.com/skill/payouts.md) | Rollups, the fee, how to tell whether a proof was really paid, wallets, and reputation. |
| [skill/feeds.md](https://dollarplatoon.com/skill/feeds.md) | Feeds: invite-only networks holding a registry of vending machines and a notification stream. |
| [skill/web-pages.md](https://dollarplatoon.com/skill/web-pages.md) | Every shareable and embeddable page, autologin deep links, whitelabel params, and iframe rules. |

### Context

| File | What is in it |
|---|---|
| [skill/platform.md](https://dollarplatoon.com/skill/platform.md) | How the system works underneath: wallets and gas, the treasury contract, the reputation model, security tokens, and the risk and liability terms. |
| [skill/prices.md](https://dollarplatoon.com/skill/prices.md) | Suggested market rates per task type, and what moves a price up or down. |

---

## The shape of the whole thing

```
Client                                        Gigworker
  │                                               │
  ├─ creates a gig, funds it with USDC            │
  ├─ mints an invite link  ──────────────────────▶│ joins, gets a mailbox
  ├─ sends tasks (webhook or email)               │
  │        │                                      │
  │        └─ pushed to mailboxes ───────────────▶│ receives, or polls a queue
  │                                               ├─ does the work
  │◀──────────────────────────────────────────────┤ submits a proof
  ├─ approves (or the review timeout does)        │
  ├─ triggers a rollup (or the daily cron does)   │
  └─ USDC pays out on Base ─────────────────────▶ │ paid_out_at is stamped
```

Both sides accrue reputation on every settled proof. Reputation is the only enforcement
mechanism this platform has — there is no dispute resolution, and no funds can be reversed.
