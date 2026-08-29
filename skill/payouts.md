# Payouts, wallets, and the event ledger

How an approved proof becomes USDC in a wallet, and how to tell whether it actually did.

## Contents

- Routes
- The fee
- Trigger a rollup
- The rollup flow, step by step
- **How to tell whether a proof was really paid**
- Failed rollups repair themselves
- Wallets
- The event ledger
- Profiles

---

## Routes

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/gigs/:id/rollups` | Owner | Pay out approved proofs now |
| GET | `/gigs/:id/rollups` | Yes | Rollups for one gig |
| GET | `/rollups/mine` | Yes | Rollups across all your mailboxes |
| POST | `/wallets` | Yes | Create a wallet alias |
| GET | `/wallets` | Yes | List your wallet aliases |
| GET | `/wallets/:alias_id/balances` | Yes | On-chain ETH + USDC |
| POST | `/wallets/:alias_id/transfer` | Yes | Send USDC from your hot wallet |
| DELETE | `/wallets/:alias_id` | Yes | Delete an alias |
| GET | `/reputation/:wallet/events` | No | Raw event ledger for one wallet |
| PATCH | `/profiles/me` | Yes | Update your profile |
| GET | `/profiles/:identifier` | No | Public profile |

## The fee

| Event | Fee | Detail |
|-------|-----|--------|
| Client deposits USDC | 0% | No deposit fee |
| Gigworker payout | 10% **on top** | The worker receives the full gross; the fee is charged additionally from the gig balance |

A worker earning $10 costs the gig $11. **Budget 110% of expected payouts when funding.**

> **An order machine charges the fee the other way round, and this is the easiest thing on the
> platform to get wrong.** On an `inbound_order` gig the buyer deposits `D` and the vendor
> receives `floorToCent(D ÷ 1.1)` — the fee comes **out of** the deposit, because a vending
> machine charges what the sticker says. $0.50 pays the vendor $0.45. Never round instead of
> flooring (it overdraws and the payout reverts), never compute it in floating dollars
> (`3.30/1.1` shorts a cent), and never hardcode the `1.1` — read `fee_bps` from
> `GET /gigs/:id`. Full arithmetic in
> [orders.md](https://dollarplatoon.com/skill/orders.md).

Other money rules that follow from the contract:

- **Fund isolation** — each gig has its own on-chain balance. One gig's funds can never pay
  another's workers.
- **No withdrawal** — once deposited, USDC leaves a gig only as a worker payout.
- **No debt** — a rollup pre-checks `available_funds >= gross + fee` and fails entirely otherwise.
- **Price lock** — the amount is fixed when the proof is submitted, so a mid-gig price change
  cannot reduce work already done.
- **Minimum payout** — `min_payout` per gig (default `$0`). Anything below it accumulates.

## Trigger a rollup

```json
POST /gigs/:id/rollups
```

```json
{
  "rollups": [
    { "id": "ROLLUP_...", "mailbox_id": "MBX_...", "wallet_address": "0x...",
      "proof_ids": ["PROOF_a", "PROOF_b"],
      "paid_proof_ids": ["PROOF_a"],
      "gross_amount": 5.00, "platform_fee": 0.50, "net_amount": 5.00,
      "tx_hash": "0x...", "status": "paid", "settled_by_reconciliation": false }
  ],
  "available_funds": 44.50,
  "retried_stuck": 1,
  "skipped_below_minimum": [ { "mailbox_id": "MBX_...", "amount": 0.50 } ]
}
```

Groups approved and `timeout_approved` proofs by mailbox. A daily cron does the same thing
automatically — the manual call is for paying immediately.

Returns `400` if the gig cannot cover the total including the fee.

**Before creating anything, this route settles any rollup left unsettled by an earlier run.**
`retried_stuck` counts those, and they appear in `rollups` alongside new ones. A response with
`retried_stuck` set and an otherwise empty `rollups` list means old payouts were repaired and
there was no new work to pay.

**Read `paid_proof_ids`, not `proof_ids`, to learn what a rollup paid for.** `proof_ids` also
carries the mailbox's *rejected* proofs, swept in only so they stop being reconsidered on every
future run. Nobody was paid for those. Rollups created before this distinction existed have no
`paid_proof_ids` — on those, check each proof's own `status`.

## The rollup flow, step by step

1. The client triggers `POST /gigs/:id/rollups`, or the daily cron runs.
2. Approved proofs are grouped by mailbox and `locked_price` is summed. An approved proof that
   somehow arrives still unpriced is summed at the **gig price**, never at `$0`.
3. Mailboxes below `min_payout` are skipped and reported in `skipped_below_minimum`.
4. Pre-check: `gross_amount + platform_fee <= available_funds`. Underfunded fails with `400`.
5. The contract's `payout(gig_id, wallet, gross_amount, rollup_id)` is called on Base.
6. On success: `tx_hash` stored, status `paid`, **`paid_out_at` stamped on every proof in
   `paid_proof_ids`**, and a reputation event created.
7. On failure: status `failed`. The next run asks the **chain** whether that payout actually
   landed. If it did, the rollup is corrected to `paid` without sending anything. If it did not,
   it is sent again. This repeats until it settles.

On an **order machine** three things in that sequence differ. **Either party** may trigger step 1
— a participant settles only their own orders, because the proof set is narrowed to the tasks
their own deposits funded. Proofs are grouped by **(mailbox, task)** rather than by mailbox, since
one machine has one mailbox and a single rollup naming every buyer's order would revert in full if
any one deposit had been withdrawn. And the payout names its deposits rather than drawing on the
shared pot, so it is chunked at the contract's batch ceiling. Everything from step 6 onwards —
`paid_out_at`, the released `private_note` — is identical.

## How to tell whether a proof was really paid

**Check `proof.paid_out_at`.** It is set only when money actually moved on chain for that proof.
If you are automating "have I been paid", build on this field and nothing else.

Three things that look like payment and are not:

- **`rollup.status === "paid"` is not sufficient.** A rollup with `gross_amount: 0` is written
  straight to `paid` with no transaction at all. Those exist only to clear rejected proofs off the
  queue. "Paid" there means "settled", not "the worker was paid".
- **A missing `tx_hash` does not mean unpaid.** When a payout confirms after the process that sent
  it has died, the hash is lost with it. The next run detects the payment on chain and records the
  rollup as `paid` with `tx_hash: null` and `settled_by_reconciliation: true`. The worker holds
  the USDC; only the receipt is missing. **Do not resend against these.**
- **`proof.status === "approved"` only means the client accepted the work.** Payment follows
  separately, on the next rollup, and can wait on `min_payout` or on the gig being funded.

Proofs approved before `paid_out_at` existed were backfilled, so historical proofs carry it too. A
proof with no `paid_out_at` and an `approved` status is simply awaiting its rollup.

## Failed rollups repair themselves

**Never re-create a failed rollup by hand.** The daily cron retries it, reusing the *same* rollup,
until it settles — and it checks the chain first, so nobody is paid twice and nobody is skipped.

**A slow confirmation looks exactly like a failure.** A payout that takes longer than 20 seconds
to confirm is recorded `failed` and the owner is emailed, even though the transaction may land
moments later. The next run detects it and corrects the record to `paid`.

Treat a single `failed` rollup as "not settled yet", never as "lost". Triggering another payout
for the same proofs is how you pay twice. Nothing is required from the worker either — an approved
proof is never dropped.

## Wallets

Every account has a wallet on Base L2 holding USDC (for payments) and a little ETH (for gas).

```json
POST /wallets   { "label": "My Hot Wallet", "is_hot_wallet": true }
POST /wallets   { "label": "My MetaMask", "is_hot_wallet": false, "evm_address": "0x..." }

GET /wallets/:alias_id/balances
→ { "evm_address": "0x...", "eth_balance": "0.05", "usdc_balance": "100.000000" }

POST /wallets/:alias_id/transfer   { "to_address": "0x...", "amount": 50 }   // hot wallets only
→ { "tx_hash": "0x..." }
```

One hot wallet per account; external wallets are unlimited.

- **Gas is your responsibility.** Without ETH your wallet cannot send transactions. Base fees are
  fractions of a cent, so ~0.001 ETH covers many.
- **Managed versus external.** A hot wallet is generated and its key stored encrypted, and it is
  recoverable through your account. An external wallet is full self-custody — **lose the keys and
  the funds are gone permanently.**
- Changing the payout address on a mailbox affects **future rollups only**. See
  [gigs.md](https://dollarplatoon.com/skill/gigs.md).

## The event ledger

Dollar Platoon does not score participants. There are no ratings, no star reviews, and no
reputation thresholds. It settles payments and records what happened, and the record is what it
publishes.

```json
GET /reputation/:wallet/events
→ { "events": [ { "id": "REPEVENT_...", "event_type": "proof_approved", "gig_id": "GIG_...",
                  "proof_id": "PROOF_...", "wallet_address": "0x...",
                  "metadata": { "amount": 2.50, "rejection_tag": null },
                  "timestamp": "2026-08-28T10:04:11.238Z" } ] }
```

| Event type | Written when |
|---|---|
| `proof_submitted` | A worker sends a proof |
| `proof_approved` / `proof_rejected` | A client rules on one. `metadata.rejection_tag` carries the reason |
| `proof_reported` | A client reports a proof |
| `payout_completed` | A rollup paid, with the amount |
| `gig_created` / `gig_closed` | A gig opened or closed |
| `order_withdrawn` | An `inbound_order` deposit was undone, and by which side |

- **Facts, not opinions.** The ledger says what settled. Whether that makes somebody worth hiring
  is a judgement the platform does not make for you.
- **Append-only.** A changed verdict writes a NEW event rather than overwriting the old one, so a
  correction stays visible as a correction. A reader that wants one verdict per proof takes the
  latest by timestamp.
- **Wallet-anchored, and scoped to one wallet.** This route never merges addresses. An account may
  hold several wallets, and deciding which belong together is an identity question the route has
  no authority to answer — read each address and merge them yourself if that is what you want.
- **Permissionless to read.** No authentication.

**Before joining a gig**, read the owner's events and check the gig's `available_funds`. There is
no dispute resolution and no score standing in for one: judge the history yourself.

### Building a rating on top

Ratings, rankings and review systems are welcome — as third-party apps over this ledger, not as
platform features. The settlement record is the honest input to any of them, and it is the thing
only this platform can publish.

There was previously a star-review system (`POST /gigs/:id/reviews`, `GET /reputation/:wallet/reviews`,
`PATCH /reviews/:id/resolve`) and a computed score (`GET /reputation/:wallet`, returning volume,
quality, recency and social). **All of those endpoints are gone and now answer 404.** So are the
`min_rep_volume` / `min_rep_quality` / `min_rep_recency` join gates on a gig.

## Profiles

```json
PATCH /profiles/me
{ "display_name": "John Doe", "bio": "Experienced social media marketer", "avatar_url": "https://..." }
```

`GET /profiles/:identifier` is public, by email or wallet alias id.
`GET /profiles/:identifier/private` needs a shared gig relationship.
