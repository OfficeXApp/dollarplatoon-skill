# Order machines (`inbound_order`) — the shop that does the work

One distribution mode inverts every assumption the rest of this skill makes. Read this page
before you build anything against a gig whose `distribution` is `inbound_order`, because the
sentence "the gig owner sends the work and pays for it" — true on every other page — is false
here in both halves.

## Contents

- The inversion, in one table
- Routes
- Open a shop — the vendor's side
- `list_price`, and why `gig.price` is pinned to 0
- Free shops (`list_price: 0`)
- Fee-INCLUSIVE pricing: what the deposit actually buys
- The order lifecycle
- Place an order — the participant's side, end to end
- Hand the buyer a payment link — two calls, one URL
- Undo: either party, until approval
- `private_note` is the vendor's only protection
- Approve, then settle, then reveal
- Report and comments both work in both directions
- What this mode refuses, and why a `409` is an answer
- Fields you will see on an order
- What is still missing

---

## The inversion, in one table

| | Every other mode | `inbound_order` |
|---|---|---|
| Sends the task | the gig owner | an outside **participant** |
| Does the work | a mailbox holder | the **vendor** — who *is* the gig owner |
| Approves | the gig owner | **the participant, and nobody else** |
| Funds it | the gig owner, once, into a shared pot | the participant, **per task**, at publish |
| Fee | 10% charged **on top** of the payout | 10% taken **out of** the deposit |
| Gets paid | the mailbox wallet | the **vendor's** mailbox wallet |
| Can undo the money | nobody — deposits are one-way | **either party**, until approval |
| Triggers the payout | the owner, plus the daily cron | **either party**, plus the cron |

The words matter, so this page uses them consistently. **Vendor** = the gig owner, who fills
orders. **Participant** = the member who places and pays for one. The stored role tokens are
still `client` and `gigworker` — they are never migrated — but their *meaning* is fixed on the
economics, not on the deed:

```
"client"     the paying, approving side     outbound: the gig owner
                                            order:    the participant
"gigworker"  the side doing the work        outbound: the mailbox holder
                                            order:    the vendor (the gig owner)
```

If your integration maps those tokens to labels, map them through the gig's `distribution` or
you will render every byline backwards.

**Membership is the whole credential.** A participant reaches a shop by holding an **active**
mailbox in it — the vendor's invite link is what buys that. There is no browse-all and no public
shopfront; `GET /gigs` still answers `410`. A `pending_approval` mailbox cannot order.

## Routes

Nothing below is a new path except `GET /orders` and `.../withdraw`. The rest are routes you
already know that **change hands** in this mode.

| Method | Path | Auth here | Description |
|--------|------|-----------|-------------|
| POST | `/gigs` | Yes | Open a shop: `distribution: "inbound_order"`, `list_price`, no `price` |
| GET | `/gigs/:id` | Optional | Adds `list_price`, `fee_bps`, and `escrowed_funds` (owner + members) |
| PATCH | `/gigs/:id` | Vendor | `list_price` moves; `price` does not |
| POST | `/inbound/webhook/:gig_id?token=…&draft=true` | Token **+ participant session** | Save a draft order |
| GET | `/gigs/:id/drafts` | Vendor **or participant** | Your own unsent orders |
| PATCH | `/gigs/:id/tasks/:msgId/payload` | **Participant** | Rewrite an unfunded draft's body |
| PATCH | `/gigs/:id/tasks/:msgId/private-details` | **Participant** | The private half of the brief |
| POST | `/gigs/:id/tasks/:msgId/publish` | **Participant** | **Deposit and send.** `{ amount, wallet_alias_id }` |
| GET | `/orders?state=&gig_id=` | Yes | Your deposit ledger across every shop |
| GET | `/gigs/:id/proofs` | Vendor, submitter **or participant** | The delivery |
| GET | `/gigs/:id/proofs/:proof_id` | Vendor, submitter **or participant** | One delivery; poll it for the released note |
| PATCH | `/gigs/:id/proofs/:proof_id` | **Participant** | Approve or reject |
| POST | `/gigs/:id/proofs/:proof_id/report` | **Either party** | Dispute a `timeout_approved` delivery |
| POST | `/gigs/:id/rollups` | Vendor **or participant** | Settle. A participant settles only their own orders |
| POST | `/gigs/:id/tasks/:msgId/withdraw` | **Either party** | Undo the order and return the deposit |

`GET /orders` is account-wide and not scoped to a gig — it is the buyer's own ledger, read from
their own partition. Filter with `?state=pending|open|settled|withdrawn` and `?gig_id=`.

## Open a shop — the vendor's side

```json
POST /gigs
{
  "title": "Haiku on demand",
  "terms": "One haiku, any subject, delivered within a day.",
  "distribution": "inbound_order",
  "list_price": 0.50,
  "review_timeout": 86400
}
→ { "gig": { "id": "GIG_01HX...", "list_price": 0.5, "price": 0, ... },
    "vendor_mailbox_id": "MBX_01HX..." }
```

Four things differ from every other `POST /gigs`:

- **`price` is not asked for and is pinned to 0.** Do not send one.
- **`list_price` is required**, and is the shopfront number.
- **`review_timeout: -1` is refused, and so is anything under 3600 seconds.** `-1` means manual
  review with no auto-approval, which on an outbound gig is the owner's own risk to take. Here
  the reviewer is the buyer and the payee is the vendor, so no auto-approval means a participant
  could simply never rule and the vendor would never be paid. The one-hour floor closes the
  opposite abuse — a window so short it approves the vendor's own submission before the buyer
  can read it.
- **You get a mailbox in your own shop.** `vendor_mailbox_id` is where orders land and where you
  submit deliveries from. You are the owner *and* the worker; that is the mode.

Then hand out two things: an **invite link** (`POST /gigs/:id/invites`) so a buyer can join, and
the gig's **webhook URL with its security token**, which is the door their draft order goes
through. `POST /gigs/:id/rotate-token` closes that door for every participant at once, so in
this mode it demands `{ "confirm": true }` and answers `409 requires_confirmation` without it.

**Do not fund the gig.** `POST /gigs/:id/deposit` answers `409` here, correctly: it credits the
shared pot, and money in the pot is unreachable by the deposit-naming payout this mode uses.
The Treasury's reserved-balance guard would refuse to release it. Your buyers fund each order.

## `list_price`, and why `gig.price` is pinned to 0

`gig.price` is the fallback the whole codebase reaches for whenever a task price cannot be
resolved — `resolveTaskPrice`, the review timeout, the rollup's unpriced-proof path. In this
mode every one of those fallbacks is **wrong by construction**, because the only correct number
comes from the deposit. Pinning it to 0 makes each of them pay nothing rather than pay a figure
nobody agreed to, and the funding guards below make sure that zero is never reachable on a real
order.

`list_price` has a hard floor of **$0.02**, and the reason is arithmetic rather than taste. The
vendor receives the deposit floored to whole cents *after* the fee, and `floorToCent($0.01)` is
`$0.00`. A one-cent order would publish as a real, funded, open deposit whose payout is zero —
and a `$0` rollup is written straight to `paid` with **no transaction**, so the deposit is never
settled on chain, `paid_out_at` is never stamped, the withheld deliverable is never released,
and the undo is already closed by the approval. The buyer pays, the vendor works, and the escrow
is stranded with no exit. $0.02 is the smallest deposit that leaves the vendor $0.01.

**`list_price: 0` is the one value below that floor which is allowed, and it is a different shop
rather than a cheaper one.** See [Free shops](#free-shops-list_price-0). Anything strictly between
`0` and `$0.02` is refused.

`PATCH /gigs/:id` moves `list_price` and refuses `price` (`409`, `reason: "inbound_order"`).
Repricing the shop **does not touch orders already funded** — their amounts were fixed by the
money that was actually sent, and are bounded on chain by it. A PATCH that would cross the
free/paid line in either direction is refused with `409 list_price_mode_change`.

## Free shops (`list_price: 0`)

A free order machine takes orders and **touches no contract at all**. No deposit, no Treasury, no
`DEPOSIT_ID_SECRET`, no gas, and the buyer needs no wallet.

That is the point of it rather than a side effect: a paid shop needs a Treasury carrying the
escrow surface, and a deployment whose Treasury has been drained cannot run one. A free shop runs
anywhere.

Everything the trap above describes is a property of escrow. A free order has no deposit, so
there is no escrow to strand, no rollup to settle, no `paid_out_at` to lose and no undo to close.

**Open one** exactly as above, with `list_price: 0`.

**Place an order** with the same two calls as a paid one — draft through the webhook, then
`POST /gigs/:id/tasks/:msgId/publish` — with **no body**. An `amount` other than `0` is refused
(`409`, `reason: "free_order"`), never silently dropped, so a buyer is never told they tipped when
they did not. The response carries `free: true`, `price: 0`, and no `deposit_id` or `tx_hash`.

What changes downstream:

| | Paid shop | Free shop |
|---|---|---|
| Marker on the sent order | `deposit_id` | `order_free: true` |
| Ledger row `state` | `pending` → `open` → `settled` | `free` (terminal) |
| Releases the withheld `private_note` | `paid_out_at`, after the payout | the buyer's **approval** |
| Rollups | one per settled order | none, ever (`409 free_order`) |
| Approval | revisable until a rollup carries it | **final** (`409 free_order_approved`) |
| Undo | returns the deposit on chain | closes the order, returns nothing |
| Closing the shop | refused (`409 inbound_order`) | allowed |

The approval being final is the one asymmetry worth reading twice. On a paid order the payout
stamp is what closes the verdict; a free order never gets one, and approval is what *releases the
deliverable* — so if approval stayed revisable, a buyer could read a withheld licence key and then
withdraw the verdict that unlocked it, repeatedly. Rejections stay revisable in both modes,
because a rejection releases nothing.

A free cancel writes **no reputation event**. Those are keyed on the depositor's wallet address,
and a free order has no depositor.

**A free shop can be closed** — `PATCH /gigs/:id {"status":"closed"}`, or `DELETE /gigs/:id`,
which writes the same field. A **paid** shop still refuses both (`409`), because closing takes a
gig out of `scanActiveGigs()` and therefore out of the cron payout sweep — the backstop that pays
a vendor whose buyer never pressed the button. A free shop has no sweep to leave.

Closing stops **new** orders only (`409`, "This shop is not open"). Orders already in flight run to
their end: auto-approval is a query over proof rows and never reads a gig, and delivery, the
verdict and the undo have no gig-status check either. Reopen with `{"status":"active"}`.

## Fee-INCLUSIVE pricing: what the deposit actually buys

This is the single most common place to get the arithmetic wrong, because it is the opposite of
[payouts.md](https://dollarplatoon.com/skill/payouts.md).

> **Outbound:** the worker receives `P` and the gig pays `P × 1.1`. Budget 110%.
> **Order:** the buyer deposits `D` and the vendor receives `floorToCent(D ÷ 1.1)`. The fee comes
> out of the deposit. A vending machine charges what the sticker says.

```
D = $0.50   →  vendor receives $0.45   (fee $0.05, residue 0.5¢)
D = $3.30   →  vendor receives $3.00
D = $5.00   →  vendor receives $4.54   (NOT $4.55 — see below)
```

**Three rules if you compute this yourself.**

1. **Floor, never round.** `$5.00 ÷ 1.1` is `$4.5454…`. Rounding to `$4.55` overdraws:
   `4.55 × 1.1 = $5.005`, which is more than the deposit, and the payout reverts permanently
   with the money already taken. Flooring to `$4.54` leaves 0.6¢ of residue in the pot as
   unreserved float, which is the intended outcome.
2. **Work in integer cents.** `Math.floor(3.30 / 1.1 * 100) / 100` is **2.99**, because
   `3.3/1.1` is `2.9999999999999996` in IEEE-754. That shorts the vendor a cent on exactly the
   round numbers a buyer is most likely to type — $3.30, $6.60, $12.10. The integer form gives
   3.00, 6.00, 11.00.
3. **Never hardcode the `1.1`.** The Treasury's fee rate is admin-mutable up to 5000 bps. Read
   it from `fee_bps` on `GET /gigs/:id` and derive the divisor as `1 + fee_bps/10000`. The key
   is present on every order machine including when its value is `null` — `null` means *"the
   Treasury could not be read"*, which is a different statement from *"this shop charges no
   fee"*. **Treat `null` as unknown and say so in your UI. Do not substitute 10%.** Every
   *settled* path uses the rate snapshotted onto the deposit, so only the pre-purchase quote is
   at risk; a `setFee` would otherwise make every quote silently wrong.

**Anything above `list_price` is a tip, and it goes to the vendor too.** Deposit more than the
sticker and the whole surplus flows through the same `D ÷ 1.1`. The tip is
`funded_total − funded_minimum` — derived, never stored. `funded_minimum` exists for exactly this
reason: it snapshots the price *at publish*, so the tip stays correct after the vendor reprices
the shop. Subtracting today's `list_price` instead gives a wrong answer, and a negative one when
the price went up.

Deposits must be a **whole number of cents**. A fractional cent inflates the residue past the
contract's bound and the payout reverts forever, so it is refused before any money moves.

`GET /gigs/:id` also carries **`escrowed_funds`** for the owner and members. On an order machine
`available_funds` is synced raw from the on-chain pot, and that pot holds every open deposit —
so without the split a vendor's dashboard shows their customers' escrow as their own takings.
`escrowed_funds` is a display mirror and never an authority; every decision reads the chain.
`null` means "not synced yet", which is not the same as `0`.

## The order lifecycle

```
draft            saved through the webhook (?draft=true), names the participant, no money
  │
  └─ publish ────▶  DEPOSIT (state: pending → open)  ── the deposit and the send are ONE call
                    │
                    ├─ either party undoes ──────────▶  withdrawn   (terminal, money returned)
                    │
                    └─ vendor delivers a proof
                          │
                          └─ participant approves ───▶  UNDO CLOSES FOR BOTH SIDES
                               │   (or review_timeout auto-approves)
                               │
                               └─ either party settles ─▶  settled  (terminal)
                                    (daily cron is the backstop)
                                        │
                                        └─ paid_out_at stamped ─▶ private_note revealed
```

**Undo closes at approval, not at payment.** Say this to yourself once more, because it is the
one rule integrators guess wrong. If undo survived approval, a participant could approve, watch
the payout transaction fail, and withdraw — having already read the delivery. That is a free
option that needs no timing luck. Approval is the point of no return; payment is merely what
happens next.

## Place an order — the participant's side, end to end

**1. Join the shop.** Open the vendor's invite link, or `POST /gigs/:id/mailboxes` with the
token. You need an **active** mailbox before anything below works.

**2. Save a draft.** The gig webhook, with `draft=true`, **and your `x-api-key` on the same
request**:

```bash
curl -X POST "https://staging.dollarplatoon.com/api/inbound/webhook/GIG_01HX...?token=TOK&draft=true" \
  -H "x-api-key: $KEY" \
  -H "content-type: application/json" \
  -d '{"order":"Please write me a haiku about escrow."}'
```

The token alone is not enough here, and that is deliberate. The webhook's whole credential is a
**shared** secret; on an outbound gig that is right, because every task through it belongs to the
owner. Here many different buyers use the same door, and a draft with no identity is a row nobody
can approve, fund, undo, or even read back — every authority check in this mode is a comparison
against `client_user_id`. So the token still gates the door and your session says who you are.
Both are required: `401` with no session, `403 not_a_member` without an active mailbox.

On an order machine this door **only saves drafts**. `?draft=true` is mandatory (`409` without
it), and `?price=` and `?assign_to=` are refused rather than ignored — a price here would be a
number nobody paid, and an assignee would route a funded order past the vendor.

Read your drafts back with `GET /gigs/:id/drafts`; you get rows naming you and nothing else, and
so does the vendor for theirs. Edit an **unfunded** draft with
`PATCH /gigs/:id/tasks/:msgId/payload` and `.../private-details`. Both refuse the vendor
outright (`403`) and refuse everybody once the draft is funded (`409`): at that point the brief
*is* the record of what was bought.

**3. Publish, which is where you pay.**

```json
POST /gigs/:id/tasks/:msgId/publish
{ "amount": 0.50, "wallet_alias_id": "..." }

→ { "status": "assigned", "message_id": "TASK_01KX...", "published": true,
    "deposit_id": "0x…", "tx_hash": "0x…",
    "funded_total": 0.5, "funded_minimum": 0.5, "tip": 0,
    "price": 0.45, "fee_bps": 1000 }
```

Publishing and funding are **one operation**, because an order becomes real by being paid for and
the price written on the delivered task is derived from the money that just moved. Split them and
there would be a window where the vendor holds a claimable task that nothing funds — priced, by
every fallback in the codebase, at `gig.price`, which is 0.

Delivery mints a **new task id**. `message_id` is the live order; `draft_id` was the draft, and it
is deleted. Use `message_id` from here on.

**4. Watch it.** `GET /orders` is your ledger. `GET /gigs/:id/proofs` shows the delivery when it
arrives.

**5. Rule on it.** `PATCH /gigs/:id/proofs/:proof_id` with `{ "action": "approve" }` or a reject
with a `rejection_tag`. **You are the only party who may.** The vendor is refused on their own
gig — they are the one being judged.

**6. Settle.** `POST /gigs/:id/rollups`. You may call it for **your own orders only**; the proof
set is narrowed to the tasks your deposits funded, so the button can never move somebody else's
money or reveal somebody else's work. The vendor keeps the unrestricted gig-wide trigger, and
the daily cron is the backstop for both.

### The publish call answers honestly when it half-succeeds

A deposit passes an explicit gas limit, which skips estimation — so a reverting transaction still
reaches the mempool. Every failure path therefore **asks the chain** before concluding anything,
and the `reason` tells you which of these you are in:

| `reason` | Status | What to do |
|---|---|---|
| `insufficient_funds` | 402 | Fund the wallet. Nothing was sent. |
| `balance_unreadable` / `chain_unreadable` / `fee_rate_unreadable` | 503 | Retry shortly. Nothing was sent. |
| `deposit_id_squatted` | 409 | A fresh id is already prepared. **Publish again** — this is not an error state. |
| `deposit_failed` | 502 | The send itself failed. Read `GET /orders` before retrying. |
| `ledger_write_failed` | 409 | **Your money is safe on chain.** Publish again to finish, or undo it. |
| `residue_too_large` | 409 | The amount cannot settle. Nothing was sent. |
| `contract_mismatch` | 409 | The shop points at a different Treasury than this server. Not your problem to fix. |
| `deposit_unreconciled` / `state_mismatch` / `ledger_missing` | 409 | Stop. These need a human; nothing was sent twice. |

A response carrying `draft_kept: true` means your draft still exists and is still the only copy
of what you typed. A response carrying `funded: true` **and** `draft_kept: true` means the money
moved and the delivery did not — publish again to finish, or undo.

Retrying is safe. The deposit id is derived, the ledger row and the draft stamp are written with
conditional puts **before** any chain call, and every step after that lock is idempotent given
the same id. A double-click derives the same id twice and exactly one caller gets past.

## Hand the buyer a payment link — two calls, one URL

You do not have to build the checkout. Write the order through the webhook, then send the buyer
`/fund/:gig_id/:task_id` — a standalone page that does **nothing but wallet management and the
deposit**. It is the funding half of the order form, on its own URL, and it is framable.

**Call 1 — save the draft.** Exactly the call above, with the gig token and the buyer's session:

```bash
curl -X POST "https://staging.dollarplatoon.com/api/inbound/webhook/GIG_01HX...?token=TOK&draft=true" \
  -H "x-api-key: $BUYERS_KEY" \
  -H "content-type: application/json" \
  -d '{"order":"Please write me a haiku about escrow."}'
→ { "message_id": "TASK_01M..." }
```

Optionally follow it with `PATCH /gigs/:id/tasks/:msgId/private-details` for anything the buyer
should not see in the visible order. Both calls are yours to make; the buyer makes neither.

**Call 2 — the buyer, on the page.** Send them the link and they press one button, which is
`POST /gigs/:gig_id/tasks/:task_id/publish` under the hood:

```
https://staging.dollarplatoon.com/fund/GIG_01HX.../TASK_01M...?api_key=BUYERS_KEY
```

```html
<iframe src="https://staging.dollarplatoon.com/fund/GIG_01HX.../TASK_01M...?api_key=BUYERS_KEY&hide_navbar=true&hide_logo=true"
        width="420" height="760" style="border:1px solid #e5e7eb;border-radius:12px"></iframe>
```

**What the page shows, and nothing else:** the order text (unwrapped from its JSON envelope), the
shop and its `list_price`, the exact split — deposit, vendor payout, platform fee, tip — at the
live `fee_bps` when the gig read carries one and at the documented 10% assumption when it does not,
*saying which*; an amount field defaulting to `list_price` with anything above it labelled a tip;
the paying wallet with its USDC balance, its gas balance and its address to copy; and one button
that deposits and publishes.

**It has no navigation, no order form and no way to write anything.** The words were fixed by your
call; this page only pays for them.

### The states it answers honestly

An embed has no address bar and no support page, so every dead end is a sentence rather than a
status code:

| What happened | What the buyer sees |
|---|---|
| Draft missing, or `404` because it names another account | "This order is not here" — the link is spent or was never theirs |
| Read refused `403` | "This order belongs to somebody else" |
| Already funded — including a reload after paying | The receipt, and **no button**. It never offers to pay twice |
| `order_withdrawn_at` set | "This order was cancelled", with what came back |
| Gig is not `inbound_order` | "This machine does not take orders" |
| Shop closed (`status !== "active"`) | "This shop is closed", nothing charged |
| No hot wallet / short of USDC / **no gas** | Each named separately. Gas is a wall, not a warning — the buyer signs their own deposit |
| A `reason` from the publish table above | The server's sentence, plus what to do: retry, add funds, or stop and ask a human |

`ledger_write_failed` and any response carrying `funded: true` put the page into its
funded-but-undelivered state: **Finish this order (already paid)**, or **Cancel and refund me**.
Neither charges twice.

### Three things to know before you build on it

1. **The link is single-use by construction.** Publishing mints a new task id and deletes the
   draft, so the URL stops resolving to a draft the moment it is paid. A reload finds the deposit
   in `GET /orders` and shows the receipt instead. Do not treat a working `/fund/` link as a
   durable handle on an order — `/gigs/:gig_id/order/:task_id` is that.
2. **Membership is still the credential.** A buyer needs an **active** mailbox in the shop before
   the draft call works at all, so the first link you ever hand a stranger is the invite
   (`POST /gigs/:id/invites`), never this one.
3. **`?api_key=` grants the whole account.** It is what makes a pre-authenticated link possible,
   and it is the buyer's own key — never yours, and never one you mint for them out of an admin
   route. Without it the page shows a sign-in that returns to itself.

### Which chain, and where to send gas

A buyer signs their own deposit, so they need native ETH on the **right** network — and "Base"
names two of them. Sending Sepolia ETH to a mainnet address loses it; the reverse loses real money.

`GET /gigs/:gig_id` carries the answer on an order machine, beside `fee_bps`, so a funding page
gets the split and the network in one call:

```json
{ "fee_bps": 1000,
  "chain": { "chain_id": 84532, "chain_name": "Base Sepolia", "is_testnet": true } }
```

`GET /api/health` carries the same block plus `treasury_address` and `usdc_address`, which is the
cheaper call when you only want to know which deployment you are pointed at.

The chain id is read from the node, never inferred from the stage — those two disagreed for two
days once, and production ran against a testnet. **`chain` is `null` when the node is
unreachable, and a client must render that as "unknown" rather than guessing.** Naming the wrong
network is worse than naming none: it tells somebody to send real money to a testnet address.

## Undo: either party, until approval

```json
POST /gigs/:id/tasks/:msgId/withdraw
→ { "success": true, "state": "withdrawn", "tx_hash": "0x…", ... }
```

`:msgId` accepts the **live task** or the **draft** that was stamped with a deposit id by a
publish that never finished. Both are real ways to end up holding an open deposit.

| `reason` | Meaning |
|---|---|
| `not_inbound_order` | Wrong mode. There is no per-task escrow on an outbound gig. |
| `not_a_party` | You are neither the buyer nor the vendor. |
| `not_funded` | Nothing was ever deposited against this order. |
| `already_approved` | The delivery was approved. **Undo is closed.** Payout is the next step. |
| `already_settled` | The vendor has been paid. |
| `never_funded` | Answered `200` — the deposit was never opened, so there is nothing to return. |
| `state_mismatch` / `ledger_missing` | Stop. Nothing was sent; these need a human. |

Calling it twice on an already-withdrawn deposit returns success rather than an error, because
the point of the route is the money's final position, not the caller's bookkeeping.

The refusal is tested on *payability*, not on the literal string `"approved"` —
`timeout_approved` is a distinct status, and a buyer must not be able to withdraw after
auto-approval while the cron's payout is in flight.

**An undone order keeps its draft.** That is deliberate: publishing it again funds a fresh
deposit under the next salt, so a buyer can re-send the same words without retyping them. The
order row itself stays too, closed rather than deleted — it is the buyer's record of what they
asked for and the vendor's record of what was pulled.

## `private_note` is the vendor's only protection

**The participant reads `proofs[]` at review time, and can undo before approving.** That is the
whole shape of the risk, and it is designed in rather than overlooked: submit, read the evidence,
undo, walk away with the work.

So on an order machine:

> **Put the deliverable in `private_note`. Put evidence in `proofs[]`.**

`private_note` is released only when `paid_out_at` is stamped — money on chain, for that proof.
Approving does not open it. The buyer sees `private_note: null, private_note_locked: true` until
then, and polls `GET /gigs/:id/proofs/:proof_id` after the rollup settles. `proofs[]` should
carry a watermarked preview, a hash, a word count, a description — enough to rule on, not enough
to use. See [proofs.md](https://dollarplatoon.com/skill/proofs.md) for the full field contract.

`POST /gigs/:id/proofs/:proof_id/withdraw` — taking a submission back — is **refused** in this
mode (`409`). The buyer has already paid and may already have read it; un-delivering would leave
them holding a funded order with nothing against it and an approval they can no longer give. The
withheld note is the protection, not the ability to retract.

**Repeat extraction is an accepted, open risk.** Order, read, undo, re-order. It is bounded, not
closed, by: every withdrawal writing an `order_withdrawn` event against the participant, so a
serial extractor is visible in the ledger, and rate limits on the publish/withdraw pair. There is
no join gate to refuse them at the door — a vendor who wants somebody gone revokes the invite. If
you are building a vendor agent, withhold by default.

## Approve, then settle, then reveal

Approval is a **cheap database write**. It sends no transaction. Payment is a **separate call**
that either party may make and that the daily cron makes on its own.

This is deliberate and not an oversight. An approve that also paid would not fit API Gateway's
29-second cut — waiting on one confirmation alone leaves about nine seconds for a dozen reads and
writes — and there is no queue in the stack to defer to. Separating them removes the need for
one.

So an integrator must not treat `approved` as done:

1. `PATCH .../proofs/:proof_id` → `approved`. Undo is now closed for both sides.
2. `POST /gigs/:id/rollups` (either party) or wait for the cron.
3. `proof.paid_out_at` is stamped. **This is the only proof that money moved.**
4. `private_note` opens on the next read.

Rollups in this mode are grouped by **(mailbox, task)** rather than by mailbox alone. An order
machine has exactly one mailbox, so mailbox-only grouping would mint a single rollup naming every
buyer's order — which reverts *in full* if any one of those deposits was withdrawn, taking every
other buyer's paid-for work down with it. They are also chunked at the contract's batch ceiling,
so a vendor with a long backlog cannot mint a rollup that reverts on count and never recovers.

## Report and comments both work in both directions

**`report`.** `POST /gigs/:id/proofs/:proof_id/report` works only on `timeout_approved` proofs —
the ones the clock approved because nobody looked. In this mode **either party** may call it, and
the row records which (`reported_by: "client" | "gigworker"`). It matters: a buyer reporting is
"I never got to review this and it is not what I paid for", which is their only remedy when the
timeout ruled for them; a vendor reporting is something else entirely about the same wallet.
A reported proof is excluded from payouts.

**Comments.** `GET`/`POST /gigs/:id/tasks/:msgId/comments` work for both the buyer and the vendor
on an order — the buyer passes the membership gate because they hold a mailbox. Private replies
work the same way. Read the role note at the top of this page before you render a byline: the
buyer's comments are stored `client` and the vendor's are stored `gigworker`, which is the
inverse of what "owns the gig" would tell you.

## What this mode refuses, and why a `409` is an answer

Roughly twenty owner-only levers are closed on an order machine. Every one of them answers
**`409` with `reason: "inbound_order"`** and a sentence saying what to do instead. That is a
documented response, not a bug and not a permissions failure — the caller *is* the gig owner and
their credentials are fine; the operation is wrong for this mode.

| Refused | Why |
|---|---|
| `POST /gigs/:id/deposit` | Owner float is unreachable by the deposit-naming payout. |
| `DELETE /gigs/:id` | A soft close drops the gig from the payout sweep, stranding open deposits. |
| `DELETE .../tasks/:id`, `.../queue/:msgId`, `/queue/all`, `/inbound/all` | A deleted order still has real money behind it and nothing left that knows whose. |
| `DELETE /gigs/:id/proofs/all` | A delivery is the record a deposit settles against. |
| `POST .../tasks/:msgId/recycle`, `.../assign` | Taking paid work off its holder, or handing it to a wallet the buyer never chose. |
| `POST .../tasks/:msgId/extend` | There is no clock. `task_timeout` is pinned `null`; orders never expire. |
| `PATCH .../tasks/:msgId/availability` | An order is delivered, never queued. `view_only` would freeze paid work. |
| `POST .../queue/:msgId/violation`, `.../decline` | There is nobody else to give it to. Undo instead. |
| `PATCH .../tasks/:msgId/price` (once funded) | The amount was fixed by the deposit and is bounded on chain by it. Reported per task in `skipped` on the bulk route. |
| `PATCH .../tasks/:msgId/draft` | A draft order belongs to the buyer who wrote it. |
| `PATCH .../payload`, `.../private-details` | Scoped to the buyer, then closed once funded. |
| `POST .../proofs/:id/withdraw` | Un-delivering work the buyer has already read. |
| `POST /inbound/webhook` without `?draft=true` | An order is placed by the buyer, in the call that funds it. |
| `POST /inbound/email` | Answers `{"status":"ignored"}`. An order names its buyer; email cannot. |
| `price_tbd`, `allow_price_offers`, `task_timeout`, non-zero `min_payout` | Refused at gig create and on PATCH. Each is a way to pay the wrong number or to pay nobody. |
| `POST /gigs/:id/rotate-token` | Not refused — gated on `{ "confirm": true }`, because it locks out every buyer at once. |

## Fields you will see on an order

`GET /gigs/:id/tasks/:msgId` and the proof routes project these **only to the two parties** — the
vendor and the buyer. A member merely browsing the shop gets *no key at all*, not a null, so an
order's value is not readable by everyone who joined.

| Field | Meaning |
|---|---|
| `funded_total` | What the buyer deposited. `null` on an unfunded draft. |
| `funded_minimum` | The shop's price **at publish**. The tip is the difference. |
| `tip` | Derived: `funded_total − funded_minimum`. |
| `price` | What the vendor is owed — the deposit after the fee, floored to a cent. |
| `fee_bps` | The rate snapshotted onto **this** deposit. Not the live rate. |
| `deposit_id` | The on-chain record. Present from the moment funding is *attempted*. |
| `client_email` | Visible to both parties — they are already transacting. |
| `client_user_id` | **The buyer alone.** An account id is an authority identifier; nobody else's is handed out. |
| `is_my_order` | Answers the same question without guessing. |
| `accepted_at` | When the vendor took delivery. There is no claim step in this mode. |
| `order_withdrawn_at` | Set on an undone order. Absent everywhere else. |

On `GET /orders`, `state` is one of `pending | open | settled | withdrawn`, `vendor_payout` is
derived from the deposit's own `fee_bps`, and `tip_estimate` is explicitly nullable and
best-effort — it is measured against the shop's price **today**. The exact figure is on the task
row. Do not treat the two as interchangeable.

`GET /orders` sorts on `created_at` **and you must not assume the natural order**. Unlike every
ULID-keyed list on this platform, the ledger's sort key is a keccak digest, so the partition's own
order is arbitrary — "first" does not mean "oldest", and a re-derived id under a stepped salt
lands somewhere unrelated.

## What is still missing

Honest gaps, so you do not design around something that is not there.

- **On production, only gigs created since 2026-08-29 can be shops.** `inbound_order` shipped to
  production that day on a new Treasury. Gigs older than that are on the retired contract, which
  has no escrow in its bytecode, and a publish-with-deposit there is refused `contract_no_escrow`.
  This cannot be configured away and escrow cannot be moved between contracts — create a new gig.
  See [staging.md](https://dollarplatoon.com/skill/staging.md).
- **No route lists a gig's deposits.** A vendor cannot enumerate who owes what; they see the
  aggregate `escrowed_funds` and their own order inbox.
- **No vendor display name on the gig read.** `GET /gigs/:id` gives `owner_wallet_alias` only, so
  a shopfront has to resolve the person through `GET /profiles/:alias_id`.
- **A shop with live escrow is pinned to its Treasury.** Escrow leaves only by settling or by
  undo, and nothing migrates a deposit between contracts. This constrains future platform
  migrations, not your integration.
- **`GET /public/read-url` signs any S3 key for any share-token holder.** Unfixed. In this mode
  an offloaded order payload is the buyer's brief, which is a new exposure class. Do not put
  secrets in an order body; that is what `private_details` is for.
- **No refunds after payout, no partial undo, no post-approval tips.** Out of scope by decision.
