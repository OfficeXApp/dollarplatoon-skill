---
name: dollarplatoon-skill
description: >
  Peer-to-peer task payroll on Base L2. Clients fund USDC gigs ("vending machines"), invite
  gigworkers by link, push tasks to their mailboxes, review proofs of work, and pay out
  on-chain. Gigworkers join by invite, poll or receive tasks, submit proofs, and get paid in
  USDC. One mode inverts this: an "order machine" is a shop whose owner does the work, funded
  per order by the buyer who places it. Settlement is recorded in an event ledger and nothing is
  scored or rated; there is no dispute resolution and no public marketplace. Use this skill whenever the user mentions
  dollarplatoon.com, Dollar Platoon, a "vending machine" for gig work, micro-gig payroll, paying
  workers in USDC per task, proof review or approval, rollups and payouts, gig invite links,
  worker mailboxes, task queues and polling, feeds of gigs, escrowing or pre-funding a task on
  chain before it is worked, or placing and funding an order against a shop — and also when they
  are building an agent that earns money doing tasks, or an agent that distributes tasks to
  workers, even if they never name the platform.
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
— read them from the filesystem instead of over the network. **If you fetched this file from
staging**, the same documents sit at `skill/<name>.md` on that host too; the links below point at
production's identical copy, so read siblings from the host you started on if that matters to you.

- **API base URL:** `https://dollarplatoon.com/api`
- **Auth:** an `x-api-key` header on every authenticated call. Get a key at
  [dollarplatoon.com/settings](https://dollarplatoon.com/settings).
- **Staging:** `https://staging.dollarplatoon.com/api` — same code, Base Sepolia, its own
  accounts, and test money you cannot lose. **Build here first:**
  [skill/staging.md](https://dollarplatoon.com/skill/staging.md).

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
| Order machine, shop | A gig whose `distribution` is `inbound_order` — see below |

**One machine runs backwards, and it is called an order machine.** On a gig whose `distribution`
is `inbound_order`, the owner is the **vendor who does the work**, and an outside **participant**
sends the task, funds it with their own per-order USDC deposit, and is the only party who may
approve it. Every "the client sends work and pays for it" sentence in this skill is false there.
It is documented on its own page — [skill/orders.md](https://dollarplatoon.com/skill/orders.md) —
and the pages it contradicts say so where it matters.

**An order machine can also be free.** `list_price: 0` opens a shop that takes orders and touches
no contract at all: no deposit, no Treasury, no gas, and the buyer needs no wallet. It is a
different machine rather than a cheaper one — approval, not a payout, is what releases the
vendor's withheld deliverable, and approval is final. Paid shops still have a $0.02 floor, and
anything between the two is refused. See
[skill/orders.md](https://dollarplatoon.com/skill/orders.md).

**Your role is a property of the MACHINE, not of your session.** There is no client/worker toggle
and no persona in any URL: for each machine you are its Owner or a Participant in it. One account,
one set of pages. See [skill/web-pages.md](https://dollarplatoon.com/skill/web-pages.md).

**On screen it says Vending Machine; in the URL it says `gigs`.** The app's pages live at `/gigs`,
`/gigs/:id` and so on — the path follows the API and the entity, and the label follows the
product. Older `/machine*`, `/client/*` and `/gigworker/*` links all still redirect, carrying
their query string. Note that the SINGULAR `/gig/:id` is a different page: the public gig view,
with `/gig/:id/join?invite=` as its invite landing.

---

## Start here

**Building for a client** — someone who has work and wants it done:
→ [skill/clients.md](https://dollarplatoon.com/skill/clients.md)

**Building for a gigworker** — someone who does work and wants to be paid, human or AI agent:
→ [skill/gigworkers.md](https://dollarplatoon.com/skill/gigworkers.md)

**Building against an order machine** — a shop you buy from, or a shop you run:
→ [skill/orders.md](https://dollarplatoon.com/skill/orders.md)

**Just need the mechanics** — auth, id format, paging, errors:
→ [skill/quickstart.md](https://dollarplatoon.com/skill/quickstart.md)

**Standing up an integration** — base URLs, test money, what is not production-ready:
→ [skill/staging.md](https://dollarplatoon.com/skill/staging.md)

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
[skill/payouts.md](https://dollarplatoon.com/skill/payouts.md). This is also the field that
releases a proof's `private_note`: a gigworker can withhold the licence key, password or
download link until the money lands, and the client sees only `private_note_locked: true` until
then. Approving a proof does not open it.

**5. Budget 110% of payouts.** The worker receives the full amount and the platform fee is
charged **on top** from the gig balance. A $100 payout costs the gig $110. Funds are locked once
deposited — there is no withdrawal.

> **A gig can commit that 110% per task, before anyone works it.** With `task_escrow` on, each
> task's USDC is deposited against that task alone as the task is created, and no other task's
> payout can reach it. A worker reads `escrow_funded` and `deposit_id` on the task and can verify
> the money on chain before starting. Off by default and off on every gig that has ever existed;
> a client turning it on is opting into creating a task SPENDING money. See
> [skill/tasks.md](https://dollarplatoon.com/skill/tasks.md).

> **On an order machine every one of these five changes.** The task price comes from the buyer's
> deposit rather than the gig; the fee comes **out of** that deposit rather than on top, so a
> $0.50 order pays the vendor $0.45; and the deposit can be undone by either party right up until
> approval. On a **free** order machine (`list_price: 0`) there is no deposit, no fee and no
> payout at all — the buyer's approval settles the order and releases the deliverable. Check
> `gig.distribution` and `gig.list_price` before applying rules 3 and 5, and read
> [skill/orders.md](https://dollarplatoon.com/skill/orders.md).

---

## Where everything is

Each file below is self-contained and linked directly from here. Open what you need.

### Doing the work

| File | What is in it |
|---|---|
| [skill/quickstart.md](https://dollarplatoon.com/skill/quickstart.md) | API key, base URL, the `PREFIX_ULID` id format, pagination and error conventions, rate limits. Read once. |
| [skill/clients.md](https://dollarplatoon.com/skill/clients.md) | The client playbook end to end: create a gig, fund it, invite workers, send tasks, review proofs, pay out. Includes a runnable walkthrough. |
| [skill/gigworkers.md](https://dollarplatoon.com/skill/gigworkers.md) | The gigworker playbook end to end, including the agent loop for working many machines at once without wasting polls. |
| [skill/orders.md](https://dollarplatoon.com/skill/orders.md) | **Order machines (`inbound_order`)** — the inverted mode, both sides of it: the shopfront price, the fee-inclusive deposit, publish-with-deposit, **free shops (`list_price: 0`, no chain at all)**, the undo either party may press until approval, why the deliverable belongs in `private_note`, and the whole list of what the mode refuses. |
| [skill/staging.md](https://dollarplatoon.com/skill/staging.md) | Building against staging: base URLs, Base Sepolia and MockUSDC, getting an account and test money, and an honest list of what is not production-ready. |

### API reference, by domain

| File | What is in it |
|---|---|
| [skill/gigs.md](https://dollarplatoon.com/skill/gigs.md) | Gigs, invite links, mailboxes (joining and leaving), worker rate limits, task expiry, funding, the dashboard. |
| [skill/tasks.md](https://dollarplatoon.com/skill/tasks.md) | Getting tasks INTO a gig: the publisher webhook, drafts a client can save before publishing, reserved tasks that no poll offers (optionally held for one named worker), view-only tasks nobody can claim, the comment thread on a task and who can read it, running a bidding round and giving the task to the winner privately, inbound email, **task escrow — funding one task on chain before anyone works it**, distribution modes, and the payload formats that serve humans and agents at once. |
| [skill/queue.md](https://dollarplatoon.com/skill/queue.md) | Queue and single-player queue: polling, claiming, declining, hand-ordering, assigning a task to one named worker, task links that carry a gig invite, private briefs, hiring for a high-value job. |
| [skill/pricing-and-tags.md](https://dollarplatoon.com/skill/pricing-and-tags.md) | Per-task pricing including TBD, task tags, and every filter — how one gig carries several shapes of work. |
| [skill/proofs.md](https://dollarplatoon.com/skill/proofs.md) | Submitting proofs, drafts a worker can save or withdraw a submission back into, reviewing them, changing a verdict before the payout, rejection tags and what they cost, private aliases, the `private_note` that is released only after payout, and share links that let someone submit without an account. |
| [skill/payouts.md](https://dollarplatoon.com/skill/payouts.md) | Rollups, the fee, how to tell whether a proof was really paid, wallets, and the event ledger. |
| [skill/feeds.md](https://dollarplatoon.com/skill/feeds.md) | Feeds: invite-only networks holding a registry of vending machines and a notification stream. |
| [skill/web-pages.md](https://dollarplatoon.com/skill/web-pages.md) | Every shareable and embeddable page — the machine pages, the order pages, the two links every task has (a read-only one and a claim one) and the invite they can carry, the notifications bell, autologin deep links, whitelabel params, iframe rules, and the map from the old `/client/*` and `/gigworker/*` URLs to the ones you should build. |

### Context

| File | What is in it |
|---|---|
| [skill/platform.md](https://dollarplatoon.com/skill/platform.md) | How the system works underneath: wallets and gas, the treasury contract, the event ledger, security tokens, and the risk and liability terms. |
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

Every settled action writes an immutable event against a wallet, readable at
`GET /reputation/:wallet/events`. That ledger is the only signal this platform publishes — it does
not score or rate anyone. There is no dispute resolution, and no funds can be reversed.

An **order machine** runs the same picture backwards:

```
Participant (buys)                            Vendor (owns the machine, does the work)
  │                                               │
  ├─ joins by invite, gets a mailbox              │
  ├─ saves a draft order through the webhook      │
  ├─ publishes it WITH a USDC deposit ──────────▶ │ receives it in their own mailbox
  │        (the deposit and the send are one call)├─ does the work
  │◀──────────────────────────────────────────────┤ delivers a proof, deliverable withheld
  ├─ reads the evidence                           │
  ├─ may UNDO and take the deposit back ──────────┤ …and so may the vendor, until here
  ├─ approves (or the review timeout does)  ◀── the point of no return
  ├─ either party triggers the rollup             │
  └─ USDC pays out on Base ─────────────────────▶ │ paid_out_at is stamped
                                                  └─ private_note is released to the buyer
```

On a **free** machine (`list_price: 0`) the same picture stops three lines early. There is no
deposit to send and none to undo — cancelling simply closes the order — and no rollup and no
payout. The buyer's approval is the last step, and it is what releases `private_note`. Because
nothing follows it to close the verdict, that approval is **final**:

```
  ├─ publishes it with NO deposit ──────────────▶ │ receives it, does the work
  │◀──────────────────────────────────────────────┤ delivers a proof, deliverable withheld
  ├─ may CANCEL, and so may the vendor, until here
  └─ approves (or the review timeout does)  ◀── the point of no return
                                                  └─ private_note is released to the buyer
```
