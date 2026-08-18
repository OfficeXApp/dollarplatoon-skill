# Payouts, wallets, and reputation

How an approved proof becomes USDC in a wallet, and how to tell whether it actually did.

## Contents

- Routes
- The fee
- Trigger a rollup
- The rollup flow, step by step
- **How to tell whether a proof was really paid**
- Failed rollups repair themselves
- Wallets
- Reputation
- Reviews
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
| GET | `/reputation/:wallet` | No | Computed reputation |
| GET | `/reputation/:wallet/events` | No | Raw reputation events |
| GET | `/reputation/:wallet/reviews` | No | Reviews for a wallet |
| POST | `/gigs/:id/reviews` | Yes | Leave a star review |
| PATCH | `/profiles/me` | Yes | Update your profile |
| GET | `/profiles/:identifier` | No | Public profile |

## The fee

| Event | Fee | Detail |
|-------|-----|--------|
| Client deposits USDC | 0% | No deposit fee |
| Gigworker payout | 10% **on top** | The worker receives the full gross; the fee is charged additionally from the gig balance |

A worker earning $10 costs the gig $11. **Budget 110% of expected payouts when funding.**

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

## Reputation

Reputation is wallet-anchored, multi-dimensional, and event-sourced. Every settled action writes
an immutable event. Both clients and gigworkers have it.

```json
GET /reputation/:wallet
→ { "wallet": "0x...", "volume": 150.50, "quality": 0.92, "recency": 0.85,
    "social": 4.2, "event_count": 47 }
```

| Dimension | What it measures |
|---|---|
| **Volume** | Total USDC earned (worker) or paid out (client) |
| **Quality** | Approval rate weighted by rejection severity — `fake_proof` hurts 5× what `low_quality` does |
| **Recency** | A decay function penalising inactivity |
| **Social** | Aggregate star rating from counterparties, weighted by the dollar amount exchanged |

- **Wallet-anchored, not account-anchored.** Different wallets mean independent histories — but
  join thresholds and profile reputation merge events across **all** wallets registered to one
  account, so rotating a payout address never resets your history.
- **Permissionless.** Anyone can create a wallet and participate. Reputation must be earned.
- **Gig gating.** A client can set `min_rep_volume`, `min_rep_quality`, `min_rep_recency` to
  exclude low-reputation wallets at join time.
- **Informational only.** These are signals, not guarantees, and carry no warranty of accuracy.

**Before joining a gig**, check the owner's volume, quality and social scores, and check the gig's
`available_funds`. **Before approving work**, check the worker's quality score. Reputation is the
only enforcement mechanism here — there is no dispute resolution.

## Reviews

```json
POST /gigs/:id/reviews
{ "target_wallet": "0x...", "stars": 4, "comment": "Reliable worker, good quality" }
→ { "review": { "id": "REVIEW_...", "stars": 4 } }
```

One review per reviewer-target pair per gig. Your role is detected automatically — client if you
own the gig, gigworker otherwise. Reviews feed the `social` dimension, weighted by how much money
moved in that gig, so a review from a large engagement counts for more than one from a $1 job.

`PATCH /reviews/:id/resolve` marks one resolved (reviewer only).

## Profiles

```json
PATCH /profiles/me
{ "display_name": "John Doe", "bio": "Experienced social media marketer", "avatar_url": "https://..." }
```

`GET /profiles/:identifier` is public, by email or wallet alias id.
`GET /profiles/:identifier/private` needs a shared gig relationship.
