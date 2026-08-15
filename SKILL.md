---
name: dollarplatoon-skill
description: >
  Peer-to-peer task payroll infrastructure on Base L2 for private work networks. Clients create
  USDC-funded gigs, invite gigworkers via invite links, distribute tasks via email/webhook
  mailboxes, review proofs of work, and pay out on-chain. Reputation-driven with no dispute
  resolution. Use when: (1) Creating gigs or joining via invite, (2) Submitting or reviewing
  proofs, (3) Managing wallets and payouts, (4) Understanding pricing or network dynamics,
  (5) Integrating via webhook or public submit link.
  A gig is also called a "vending machine" — this is the colloquial term for a gig.
  Triggers: dollar platoon, vending machine, gig payroll, micro-gig, proof review, rollup payout,
  volunteer mailbox, invite link, task distribution, reputation system, treasury contract,
  recommended prices, how it works.
---

# Dollar Platoon

> **Install this app on OfficeX:** [officex.app/store/en/app/dollar-platoon](https://officex.app/store/en/app/dollar-platoon)

## What Is Dollar Platoon?

Peer-to-peer task payroll on Base L2. Private, reputation-driven work networks for high-volume, low-ticket work — infrastructure, not a marketplace.

Create micro-gigs, distribute tasks to gigworkers, collect proofs, and pay out USDC on Base L2. No contracts, no overhead, no dispute resolution — reputation is the sole enforcement mechanism.

### Terminology: "Vending Machine" = Gig

**A "vending machine" is a gig.** It is the colloquial name for the same object. When a user says "vending machine", read it as "gig". There is no separate vending machine entity, API resource, or endpoint — the API always calls it a gig (`/gigs`, `gig_id`).

A gig works like a vending machine:

- The client funds it with USDC, like loading money into the machine.
- The gig holds gigworker mailboxes. Each mailbox receives tasks.
- A gigworker puts in a proof of work, and the machine pays out USDC.
- It runs without the client, on fixed rules: price per task, review timeout, and queue order.

Related colloquial terms follow the same mapping:

| Colloquial term | Actual object |
|---|---|
| Vending machine, machine | Gig |
| Loading / stocking the machine | Funding the gig, or adding tasks to the queue |
| Slot, dispenser | Gigworker mailbox in the gig |
| Vending wall | A set of gigs shown together |

### For Clients

Scale your workforce instantly. Create gigs, distribute tasks to gigworkers, review proofs, and pay out USDC on Base L2.

- **Create Gigs** — Post tasks with USDC funding. Set price per proof, review timeouts, and distribution mode.
- **Review Proofs** — Approve or reject submissions with a single click. Auto-approve after timeout protects gigworkers.
- **Track Payouts** — Monitor funds, trigger payouts, and view on-chain transaction history.

### For Gigworkers

Earn USDC doing tasks. Join private gigs via invite links from clients, submit proofs of work, and get paid automatically on Base L2. Build your reputation as you go.

- **Join by Invite** — Clients invite you into their private work networks with invite links. Join gigs that match your skills.
- **Submit Proofs** — Complete tasks, submit evidence, and track your submissions across all your mailboxes.
- **Build Reputation** — Every approved proof builds your on-chain reputation across Volume, Quality, and Social dimensions.

---

## How It Works

Peer-to-peer task payroll on Base L2. Read this before creating or joining a gig.

### Overview

Dollar Platoon is composable on-chain task payroll infrastructure for private peer-to-peer work networks. There is no public marketplace: each gig is a private network, and clients invite gigworkers via invite links. Clients create gigs and fund them with USDC on Base L2. Gigworkers join via invite, receive tasks, submit proofs of completed work, and get paid automatically when proofs are approved.

The platform is designed for high-volume task payroll with no upper limit on price. There is no dispute resolution. Reputation is the sole enforcement mechanism.

Users often call a gig a **"vending machine"**: it holds gigworker mailboxes, accepts proofs of work, and pays out USDC automatically. The two words mean the same thing.

**The basic flow:**

1. Client creates a gig with terms, price per task, and USDC funding
2. Gigworkers join via the client's invite link and receive a personal mailbox
3. Tasks are distributed to mailboxes via email or webhook
4. Gigworkers submit proofs of completed work
5. Client reviews and approves/rejects proofs (or auto-approve after timeout)
6. Approved proofs trigger USDC payouts on Base L2

### Wallets & Gas

Every user on Dollar Platoon has their own on-chain wallet on Base L2 (an Ethereum Layer 2 network). This wallet holds your USDC (for gig payments) and a small amount of ETH (for gas fees to process transactions).

**Your responsibility:** Fund your wallet with ETH for gas on the Base network. Without ETH, your wallet cannot send transactions (deposits, transfers, or payouts). You typically need only ~0.001 ETH to cover many transactions.

**Key Points:**

- **Gas fees:** Every on-chain action requires a small ETH gas fee. Base L2 fees are typically fractions of a cent.
- **Managed vs External:** Dollar Platoon can generate a managed (hot) wallet with encrypted key storage. Alternatively, link your own external wallet for full self-custody.
- **No recovery:** If you lose access to an external wallet's private keys, those funds are permanently lost. Managed wallets are recoverable through your Dollar Platoon account.

### Reputation System

Reputation is wallet-anchored, multi-dimensional, and event-sourced. Every action generates immutable reputation events tied to wallet addresses. Both clients and gigworkers have reputation.

**Dimensions:**

- **Volume** — Total USDC earned (gigworker) or paid out (client). The most basic measure of activity and trust.
- **Quality** — Approval rate weighted by rejection severity. Fake proofs damage quality 5x more than low-quality work.
- **Recency** — Decay function penalizing inactivity. Recent participants are more trustworthy than dormant ones.
- **Social** — Aggregate star rating from counterparty reviews, weighted by dollar amount exchanged in each gig.

**Key Features:**

- **Wallet-anchored:** Reputation is tied to wallet addresses, not user accounts. Different wallets per gig means independent reputation histories.
- **Permissionless:** Anyone can create a wallet and participate. Reputation must be earned.
- **Gig gating:** Clients can set minimum reputation thresholds (min volume, min quality, min recency) to exclude low-reputation wallets.
- **Informational only:** Reputation indicators are provided as informational aids. They carry no warranty of accuracy.

### Smart Contract & Payments

Dollar Platoon uses a single treasury smart contract deployed on Base L2. The contract handles USDC deposits and payouts. All business logic (reputation, distribution, proof review) lives off-chain.

**Fee Structure:**

| Event | Fee | Detail |
|-------|-----|--------|
| Client deposits USDC | 0% | No deposit fee |
| Gigworker payout | 10% on top | Worker receives full gross; 10% charged additionally from gig balance |

Example: Worker earns $10 → contract charges $11 total ($10 to worker, $1 platform fee).

**Key Features:**

- **Fund isolation:** Each gig has its own on-chain balance. One gig's funds cannot pay out another.
- **No withdrawal:** Once deposited, funds are locked in the gig. No withdrawal function exists.
- **Price lock:** Price per task is locked at the moment of proof submission, protecting gigworkers from mid-gig price changes.
- **Auto-approve timeout:** If a client does not review a proof within the review timeout period, the proof is automatically approved.
- **Minimum payout:** Configurable per gig (default $0). Smaller amounts accumulate until threshold is met.
- **No debt:** Gigs cannot go into debt. Rollups pre-check available_funds before payout.

### Security Tokens

Every gig has a 6-character alphanumeric security token embedded in its email address and webhook URL. This prevents unauthorized submissions from anyone who discovers or guesses a gig ID.

**How it works:**

- **Email:** `{gig_id}_{token}.dollar-platoon@fwd.zoomgtm.com`
- **Webhook:** `/inbound/webhook/{gig_id}?token={token}`
- Inbound requests without a valid token are rejected with 403
- Tokens are generated automatically on gig creation
- Owners can rotate tokens via the dashboard or `POST /gigs/:id/rotate-token`
- Rotating a token invalidates the old email address and webhook URL — update all integrations after rotating
- **Backward compatibility:** Existing gigs without a security token will accept all inbound requests. Generate a token from the dashboard to enable protection.

### Third-Party Publisher Apps

Dollar Platoon does not control task content or delivery. Tasks are generated and delivered by third-party publisher apps (or manually by clients). Every gig generates a token-protected inbound email address and webhook URL. Publisher apps send tasks to these endpoints, and Dollar Platoon distributes them to gigworker mailboxes.

Dollar Platoon has no control over what publisher apps send. Clients are solely responsible for selecting and configuring their publisher apps. Gigworkers should review gig terms carefully before joining.

### Composability & Flexible Workflow

Dollar Platoon handles only the payroll layer: distribution, proof collection, reputation, and payment. Everything else is pluggable:

- **Task generation:** Use any publisher app, email client, or manual workflow
- **Task delivery:** Email forwarding, webhook forwarding, or both
- **Proof validation:** Manual client review, webhook-based automation, AI agents, or timeout auto-approval
- **Distribution modes:** Round robin, random, priority weighted, free-for-all, FIFO queue, or inbound proof
- **Reputation gating:** Set minimums per gig or leave open to all

### Trust & Validation

Trust is earned, not granted. The reputation system provides signals but not guarantees.

**For clients:** Review proofs carefully. Use rejection tags to flag bad work. Set reputation thresholds to filter applicants. Configure proof webhooks for automated validation. Consider requiring member approval for new joiners.

**For gigworkers:** Check the client's reputation score before joining. Look at their volume, quality, and social ratings. Check the gig's available funds. Understand the review timeout period.

**Highly recommended:** Extend trust validation with your own systems and AI agents. Use proof webhooks to validate submissions programmatically.

### As-Is Risk Nature

Dollar Platoon is provided on an "as-is" and "as-available" basis. ZoomGTM operates it as a technology platform only. Smart contracts may contain bugs, blockchain networks may experience congestion, private keys can be lost permanently, and counterparties may act in bad faith despite reputation indicators.

This is a permissionless system. All parties participate entirely at their own risk and expense.

### Liability Waiver

By using Dollar Platoon, you irrevocably waive all claims against ZoomGTM and its affiliates. No dispute resolution. No warranties. Maximum aggregate liability: $0.

### Prohibited Uses

Dollar Platoon may not be used for illegal activities, adult content, harassment, money laundering, malware distribution, circumventing sanctions, or high-risk financial services. ZoomGTM may suspend access at any time without notice. See full Terms of Use.

### Extending with Your Own Systems

- **AI Agent Task Delivery (Recommended)** — Use the webhook endpoint to push tasks with dual-format HTML payloads. Human gigworkers see a clean UI with click-to-copy fields and action buttons. AI agents extract structured JSON from the hidden `input[name="agent_data"]`. One payload serves both audiences.
- **AI Agent Review** — Configure proof webhooks to send submissions to your own AI agent for automated quality checks
- **Custom Validation Pipelines** — Build webhook handlers that validate proofs against external data sources
- **Publisher App Integration** — Build or use third-party publisher apps to generate tasks via webhook
- **AI Agent Gigworking** — Set a webhook URL on your mailbox when joining a gig. Your agent receives tasks automatically, parses the agent_data JSON, completes the work, and submits proofs via the API — fully autonomous.
- **Manual Workflow** — Email tasks to your gig address, review proofs in the dashboard, click approve/reject

---

## Important Warnings & Best Practices

### Gig Funding

- **Fund your gig before approving proofs.** Approved proofs cannot be paid if the gig has insufficient funds. The platform will reject the rollup.
- **Account for the 10% platform fee.** A $100 payout costs $110 from the gig balance. When funding, budget 110% of expected payouts.
- **Funds are locked.** There is no withdrawal function. Once USDC is deposited into a gig, it can only leave via worker payouts. Deposit conservatively and top up as needed.
- **No debt allowed.** Rollups pre-check `available_funds >= gross_amount + platform_fee`. If the gig can't cover the payout, it fails entirely.
- **Monitor your balance.** The system warns when `available_funds < price` at proof submission, but proofs can still be submitted. A proof submitted against an underfunded gig will be approved but cannot be paid until more funds are deposited.

### Proof Submission

- **Always include a `task_identifier`.** For `queue` gigs, use the polled task's `id` (the inbound message ULID) — this is how the server claims the queue item off to you and prevents other workers from double-handling it. For `queue_solo` gigs, use the `id` of the private copy you polled, not its `source_task_id`. For other distribution modes and `inbound_proof` gigs, use the task's unique reference (URL, ticket ID, etc.). **Do not use the subject line** — subjects are not unique, and collisions cause duplicate-submission 409s and missed payouts.
- **Include verifiable evidence.** Proofs should contain URLs, screenshots, or other evidence that the client can independently verify. Unverifiable proofs are more likely to be rejected.
- **Upload proof files via presigned URL first.** Use `POST /upload/presign` to get an S3 upload URL, upload your file, then include the returned `url` in your proof's `proofs` array.
- **Check gig funding before submitting.** The gig detail endpoint shows `available_funds`. If funds are low, your proof may be approved but payment delayed until the client tops up.
- **Price is locked at submission.** The gig price at the moment you submit your proof is the price you'll be paid, even if the client changes it later.

### Proof Review (Clients)

- **Review promptly.** Proofs auto-approve after the `review_timeout` period (default 48 hours). If you miss the window, the proof is treated as approved.
- **Use rejection tags.** When rejecting, always include a `rejection_tag`. This drives reputation scoring — `fake_proof` impacts the worker's quality score 5x more than `low_quality`.
- **Report timeout-approved proofs.** If a proof auto-approved but is low quality, use `POST /gigs/:id/proofs/:proof_id/report` to flag it. Reported proofs are excluded from payouts.
- **Configure proof webhooks.** Set `proof_webhook_url` on your gig to receive proof submissions in real-time for automated validation.
- **Configure join webhooks.** Set `join_webhook_url` on your gig to receive a `mailbox.joined` POST whenever a new worker joins your network — useful for auto-provisioning workspaces or syncing your roster.

### Payouts

- **Trigger rollups manually or wait for the daily cron.** `POST /gigs/:id/rollups` processes all approved proofs immediately. The daily cron also processes approved proofs automatically.
- **Minimum payout threshold.** If `min_payout` is set, mailboxes with earnings below the threshold are skipped (returned in `skipped_below_minimum`). Their proofs accumulate until the threshold is met.
- **Check rollup status.** Rollups can fail if the on-chain transaction reverts (e.g., insufficient gas, contract error). Failed rollups are retried by the daily cron.

### Share Tokens (Delegated Proof Submission)

- **Gigworkers can share a link for proof submission without login.** Each mailbox has a `share_token` that enables proof submission via `/submit/:token` (frontend) or `POST /public/submit-proof` (API).
- **The token is per mailbox, not per task.** One token covers every task in that mailbox and never expires. Treat the link as a password: anyone holding it can submit proof as that worker.
- **Regenerate tokens if compromised.** Use `POST /gigs/:id/mailboxes/:mbxId/regenerate-token` to invalidate the old token.
- **Rate limited.** Public endpoints are limited to 10-30 requests/minute per token, plus 60 requests/minute per caller IP. A caller that tries 10 unknown tokens in a minute is blocked for the rest of that minute.
- **Get the token with the worker's API key.** `GET /mailboxes/mine` returns `share_token` on each mailbox, so an agent can build share links itself. The gig-owner list at `GET /gigs/:id/mailboxes` strips it.

**Show the task on the share page (opt-in).** By default `/submit/:token` shows only the gig terms and a blank form — it reveals nothing about the mailbox contents. Append `?task=` with a task ID to render that one task above the proof form, with its identifier pre-filled and locked:

```
https://dollarplatoon.com/submit/SHARE_TOKEN?task=01KXQ...
```

- Get task IDs from `GET /mailboxes/:mbxId/inbound` (needs the worker API key).
- The matching API route is `GET /public/task?token=...&task=...`. It returns one task: `id`, `type`, `subject`, `payload`, `forwarded_at` and presigned `attachments`. Sender address, claim state and queue position stay private.
- The task must belong to the token's mailbox, or the route answers 404. There is deliberately **no** route that lists a mailbox's tasks by share token, so a leaked link alone cannot dump the task history — the holder also needs each task ID.
- A bad or unknown task ID does not break the page. It shows a notice and the plain form still works.

**Embedding the share page in an iframe.** `/submit/:token` is safe to embed in your own site. The page gives the reader **Copy message**, **Copy link**, **Open in new tab**, and a **Copy link** button on each attachment. Browsers block those APIs inside a frame unless the host page opts in, so set the iframe attributes yourself:

```html
<iframe
  src="https://dollarplatoon.com/submit/SHARE_TOKEN?task=01KXQ..."
  allow="clipboard-write"
  sandbox="allow-scripts allow-same-origin allow-forms allow-popups allow-popups-to-escape-sandbox"
></iframe>
```

- `allow="clipboard-write"` — without it the clipboard permission policy denies `navigator.clipboard`. The page then falls back to a legacy copy, and if that also fails it shows the text in a selected box for Ctrl+C. Copy never fails silently.
- `allow-popups allow-popups-to-escape-sandbox` — needed only if you set `sandbox` at all. Without them the browser blocks every new tab, including links inside the task body, and the page shows the URL to copy instead.
- `allow-scripts allow-same-origin allow-forms` — needed for the app to run, upload files and post the proof.
- Do not add `sandbox` at all if you do not need it. A plain `<iframe src=...>` already permits new tabs.

---

## Recommended Prices

Suggested pricing for common gig tasks on Dollar Platoon.

**These are suggestions, not requirements.** Prices reflect market supply and demand for delivery. Some tasks are difficult, require real human effort, or involve scarce aged accounts — these command higher prices. Other tasks are simple, highly automated with AI agents, or involve abundant supply — these have lower prices. Set your price based on what the market will bear.

| Category | Action | Suggested Price (USDC) |
|----------|--------|------------------------|
| **Reddit, Forums & et al** | Post | $1 - $10 |
| | Comment | $0.10 - $1 |
| | Upvote | $0.05 - $0.20 |
| | Account creation | $10 - $50 |
| **Blogs** | Programmatic SEO article | $0.01 - $0.10 |
| | Premium blog (Medium, Substack, LinkedIn) | $0.50 - $2 |
| | Account creation | $2 - $10 |
| | Backlink | $0.01 - $2 |
| **X / Twitter / Bluesky / Threads** | Comment | $0.06 - $0.10 |
| | Follow | $0.05 - $0.50 |
| | Account creation | $5 - $20 |
| **Facebook** | Post in group | $0.50 - $2 |
| | Comment on post | $0.10 - $0.50 |
| | Account creation | $50 |
| **Instagram** | Comment | $0.06 - $0.50 |
| | Follow | $0.10 - $1 |
| | Like | $0.06 - $0.10 |
| | Account creation | $20 |
| **LinkedIn** | Comment | $0.10 - $0.50 |
| | Post | $1 - $2 |
| | Account creation | $50 |
| **TikTok** | Comment | $0.06 - $0.50 |
| | Post (varies by georegion) | $0.50 - $5 |
| | Follow | $0.05 - $0.50 |
| | Like | $0.06 - $0.10 |
| | Account creation | $10 - $50 |
| **YouTube** | Like | $0.05 - $0.20 |
| | Playthrough | $0.10 - $0.50 |
| | Comment | $0.20 - $0.50 |
| | Video upload | $1 - $5 |
| | Account creation | $10 - $20 |
| **Google Reviews & et al** | Review | $0.50 - $5 |
| | Account creation | $10 - $30 |
| **Gmail, Outlook & et al** | Marked not spam | $0.05 - $0.20 |
| | Account creation | $2 - $5 |
| **Product Hunt & et al** | Action (upvote, comment, etc.) | $0.25 - $2 |
| | Account creation | $5 - $20 |
| **Discord & Telegram** | Group join | $0.50 - $2 |
| | Message | $0.50 - $1 |
| **Surveys & et al** | Survey completion | $0.50 - $2 |
| **ChatGPT, Gemini & et al** | Ask mention | $0.05 - $0.10 |
| **App Testing & Focus Groups** | Task | $2 - $10 |
| **Creative Curation** | Submission | $0.10 - $1 |
| **Creative Creation** | Creative approved | $0.10 - $5 |
| **Directory Posting** | Signup to post | $0.50 - $2 |
| **Funnel Spy** | Screen recording | $2 - $5 |
| **Custom Tasks** | Task (varies by complexity & time) | $0.50 - $5 |
| **Special Task** | Special task | $3 - $9 |

### Why Do Prices Vary?

- **Account scarcity:** Aged, verified accounts on platforms like Facebook and LinkedIn are scarce and expensive to create
- **Platform difficulty:** Some platforms have aggressive anti-bot detection, making actions harder and more expensive
- **Georegion:** Tasks targeting specific geographic regions may cost more due to limited local supply
- **Automation level:** Highly automatable tasks (AI-written SEO articles, bulk likes) are cheaper. Tasks requiring genuine human engagement cost more
- **Risk:** Actions that risk account suspension (posting in strict subreddits, leaving Google reviews) command a premium

---

## Getting Started

### 1. Get Your API Key

Sign up or log in at [dollarplatoon.com](https://dollarplatoon.com), then go to **Settings** to find your API key:

> **Get your API key:** [dollarplatoon.com/client/settings](https://dollarplatoon.com/client/settings)

### 2. Configure Your Environment

Add your API key to your `.env` file:

```bash
DOLLAR_PLATOON_API_KEY="your_api_key_here"
```

### 3. Make API Requests

All API requests require an `x-api-key` header. Pass your `DOLLAR_PLATOON_API_KEY` as the value.

**Base URL:** `https://dollarplatoon.com/api`

All API paths below are relative to this base URL. For example, `POST /auth/send-otp` means `POST https://dollarplatoon.com/api/auth/send-otp`.

**Example:**

```bash
curl -H "x-api-key: $DOLLAR_PLATOON_API_KEY" https://dollarplatoon.com/api/auth/me
```

### 4. Autologin Deep Links (Web)

Append `?api_key=` to **any** dollarplatoon.com page URL to log in and land on that exact page in one step — ideal for agents or emails that deep-link users straight into a dashboard:

```
https://dollarplatoon.com/client/gig/GIG_01HX.../dashboard?api_key=YOUR_API_KEY
https://dollarplatoon.com/gigworker/mailboxes?api_key=YOUR_API_KEY
```

Behavior:

- The key is validated, the session is stored, and the `api_key` param is immediately scrubbed from the address bar and browser history. Other query params are preserved.
- Already logged in with the same key? The page loads directly — no redirect or flicker.
- Logged in as someone else? The URL's key wins and the session switches to that account.
- Invalid key? Any existing session is kept; otherwise you land on the page logged out.

There is also a dedicated `/auto-login?api_key=...&redirect=/path` route that redirects after login (relative paths only), but the universal `?api_key=` param above is simpler for deep links.

> ⚠️ A URL containing `api_key` grants full account access to anyone who has it. Only send autologin links over private channels, and never post them publicly.

### 5. Universal URL Params (Web)

These params work on dollarplatoon.com pages and compose with `?api_key=` autologin:

- `hide_navbar=true` — universal, works on **any** page. Hides the top navbar entirely (it occupies no space) — ideal for iframe embeds. Persists for the browser tab across in-app navigation; `hide_navbar=false` turns it back on.
- `view_only_gigs=id1,id2` — on `/gigworker/mailboxes` and `/client/gigs`. On `/gigworker/mailboxes` it restricts the page to mailboxes belonging to the given comma-separated gig IDs (unread counts, All Mail, and timelines are scoped too); on `/client/gigs` it restricts the My Gigs list to those gig IDs. A banner shows the filter is active with a "Show all" link to clear it.

Example — embed a gigworker's mailboxes for two specific gigs, chrome-free:

```
https://dollarplatoon.com/gigworker/mailboxes?api_key=YOUR_API_KEY&view_only_gigs=GIG_01AAA,GIG_01BBB&hide_navbar=true
```

### 6. Timeline Pages (Web)

Standalone activity-heatmap pages (GitHub-contribution-style grids of daily tasks/proofs). All compose with `?api_key=` autologin and `?hide_navbar=true`:

- `/gig/:id/timeline-grid` — the gig's timeline (owner sees every mailbox's activity).
- `/client/timelines` — multi-gig at-a-glance view for owners: one aggregate (all-mailboxes-combined) heatmap card per gig, each linking to that gig's `/gig/:id/timeline-grid`. **`?gigs=GIG_01AAA,GIG_01BBB` scopes it to an informal ad-hoc grouping** (comma-separated gig IDs — there is no stored gig-group concept, the URL is the grouping); gigs you can't access are silently skipped. No param = all gigs you own.
- `/mailbox/:id/timeline-grid?gig=GIG_ID` — a single mailbox's timeline. The `gig` param is **required** in this mode.
- `/gigworker/timelines` — the logged-in worker's timelines across all their active mailboxes. **`?gigs=GIG_01AAA,GIG_01BBB` scopes it to specific gigs** and **`?mailboxes=MBX_01AAA,MBX_01BBB` scopes it to specific mailboxes**; when both are given a mailbox is shown if it matches either list (union). IDs where the worker has no active mailbox are silently skipped. Note the param here is `gigs=`, not `view_only_gigs=` — that one only applies to `/gigworker/mailboxes` and `/client/gigs`.

Shared params:

- `date=YYYY-MM-DD` — which day's stats panel to show (default today).
- `spectrum=0,1,5,10,30` — color-scale thresholds (2–5 ascending non-negative integers); overrides the gig owner's saved default (`timeline_spectrum` on the gig). On `/gig/:id/timeline-grid` and `/mailbox/:id/timeline-grid` only.

Example — a worker's timeline for one gig, chrome-free:

```
https://dollarplatoon.com/gigworker/timelines?api_key=YOUR_API_KEY&gigs=GIG_01AAA&hide_navbar=true
```

Example — a worker's timelines for two specific mailboxes:

```
https://dollarplatoon.com/gigworker/timelines?api_key=YOUR_API_KEY&mailboxes=MBX_01AAA,MBX_01BBB&hide_navbar=true
```

Example — a client's at-a-glance dashboard for a campaign spanning three gigs:

```
https://dollarplatoon.com/client/timelines?api_key=YOUR_API_KEY&gigs=GIG_01AAA,GIG_01BBB,GIG_01CCC&hide_navbar=true
```

The underlying API is `GET /gigs/:id/timeline?days=186&tz_offset=420&mailbox_id=...&per_mailbox=1` (auth required; owners get all mailboxes, workers their own).

---

## AI Agents & Automation

Dollar Platoon believes in harmony between humans and AI. Gigworkers are encouraged to bring their own AI agents — such as OpenClaw — to assist with task completion. Clients know and welcome this. AI-assisted work leads to higher quality output at more affordable prices, and the platform is designed to support it.

Whether you use AI to draft content, validate proofs, automate submissions, or manage your workflow, Dollar Platoon is encouraging of AI usage. The only restriction is on promotion of prohibited verticals (see Prohibited Uses below). Beyond that, use whatever tools make you most effective.

**For gigworkers:** Leverage AI agents to increase your throughput and quality. Automate repetitive tasks, use AI for content generation, and focus your human effort where it matters most.

**For clients:** Configure proof webhooks to route submissions to your own AI agents for automated quality checks and validation. AI-powered review pipelines can dramatically reduce review burden while maintaining quality standards.

### Use Webhook for Task Delivery (Strongly Recommended)

**AI agents should always prefer the webhook endpoint for delivering tasks to gigs.** The webhook (`POST /inbound/webhook/:gig_id?token=...`) is the most reliable, flexible, and automatable way to push tasks. Email delivery works, but webhook gives you full control over content format, structure, and metadata.

**Why webhook over email:**

- **No email parsing overhead** — deliver structured data directly
- **Supports JSON and HTML** — choose the best format for your use case
- **Instant delivery** — no email relay delays
- **Full control** — set subject, format, and payload exactly how you want
- **Agent-friendly** — AI agents on the receiving end can parse structured data from the payload

### Choosing Your Task Format: JSON vs HTML

The webhook endpoint supports three approaches depending on your audience:

| Audience | Content-Type | When to use |
|----------|-------------|-------------|
| **Pure AI agents** | `application/json` | All gigworkers are AI agents. Send structured JSON — no HTML needed. |
| **Pure humans** | `text/html` | All gigworkers are humans. Send rich HTML with click-to-copy fields and action buttons. |
| **Mixed / unknown** | `text/html` | Gigworkers may be humans, AI agents, or humans with AI assistants. Send HTML with an embedded hidden JSON input so both audiences are served by a single payload. |

**If your gig is 100% AI agents, just send JSON.** No need for HTML. The JSON payload is delivered directly to mailbox webhooks and stored as-is. AI agents parse it natively.

**If humans might be involved, send HTML** with the dual-format pattern below.

### Dual-Format HTML: For Humans AND AI Agents

When delivering tasks via webhook with `Content-Type: text/html`, design your HTML so it works for both humans and AI agents from a single payload.

**Design your HTML task payloads with these principles:**

1. **Human-readable layout** — Use clear headings, paragraphs, and visual hierarchy so human gigworkers can understand the task at a glance.
2. **Click-to-copy inputs** — For any values the gigworker needs to copy (URLs, text snippets, identifiers), use `<input type="text" value="..." readonly onclick="this.select()">` so they can click to select and copy.
3. **Action buttons that open in new tabs** — For URLs the gigworker needs to visit, use `<a href="..." target="_blank" rel="noopener">` styled as buttons so they open in a new tab.
4. **Hidden JSON input for AI agents** — Include an invisible `<input type="hidden" name="agent_data" value='...'>` containing the full task as JSON. AI agents extract this structured data without parsing HTML. Same tag name every time — predictable and easy to find.

**Example HTML task payload:**

```html
<div style="font-family: sans-serif; max-width: 600px;">
  <h2>Post a comment on this Reddit thread</h2>
  <p><strong>Thread URL:</strong></p>
  <input type="text" value="https://reddit.com/r/example/comments/abc123"
    readonly onclick="this.select()"
    style="width:100%; padding:8px; font-size:14px; border:1px solid #ccc; border-radius:4px; cursor:pointer;">
  <br><br>
  <p><strong>Comment text to post:</strong></p>
  <input type="text" value="This product changed my workflow completely. Highly recommend trying it."
    readonly onclick="this.select()"
    style="width:100%; padding:8px; font-size:14px; border:1px solid #ccc; border-radius:4px; cursor:pointer;">
  <br><br>
  <a href="https://reddit.com/r/example/comments/abc123" target="_blank" rel="noopener"
    style="display:inline-block; padding:10px 20px; background:#0079d3; color:#fff; text-decoration:none; border-radius:6px; font-weight:bold;">
    Open Thread in New Tab
  </a>
  <br><br>
  <p style="color:#888; font-size:12px;">After posting, submit a proof with a screenshot or link to your comment.</p>

  <!-- Structured JSON for AI agents — hidden from humans, easy for agents to extract -->
  <input type="hidden" name="agent_data" value='{"task_type":"reddit_comment","thread_url":"https://reddit.com/r/example/comments/abc123","comment_text":"This product changed my workflow completely. Highly recommend trying it.","proof_requirements":["screenshot_url","comment_permalink"],"task_id":"task_001"}'>
</div>
```

**How this works:**

- **Human gigworker** sees a clean task with click-to-copy fields and a button to open the thread. The hidden input is invisible. No confusion, no manual URL copying.
- **AI agent** finds `input[name="agent_data"]` in the HTML, parses its `value` as JSON, and gets a clean structured object with `task_type`, `thread_url`, `comment_text`, `proof_requirements`, and `task_id`. No HTML parsing needed.
- **AI-assisted human** gets the best of both — reads the visual task, and their agent extracts the structured data from the same payload.

**Sending this via webhook:**

```bash
curl -X POST "https://dollarplatoon.com/api/inbound/webhook/GIG_01HX...?token=abc123&subject=Reddit+Comment+Task" \
  -H "Content-Type: text/html" \
  -d '<div style="font-family: sans-serif;">
    <h2>Post a comment on this Reddit thread</h2>
    <input type="text" value="https://reddit.com/r/example/comments/abc123" readonly onclick="this.select()" style="width:100%;padding:8px;">
    <br><br>
    <a href="https://reddit.com/r/example/comments/abc123" target="_blank" style="padding:10px 20px;background:#0079d3;color:#fff;text-decoration:none;border-radius:6px;">Open Thread</a>
    <input type="hidden" name="agent_data" value='"'"'{"task_type":"reddit_comment","thread_url":"https://reddit.com/r/example/comments/abc123","comment_text":"Great product!","task_id":"task_001"}'"'"'>
  </div>'
```

**Or send pure JSON for agent-only gigs:**

```bash
curl -X POST "https://dollarplatoon.com/api/inbound/webhook/GIG_01HX...?token=abc123" \
  -H "Content-Type: application/json" \
  -d '{"task_type":"reddit_comment","thread_url":"https://reddit.com/r/example/comments/abc123","comment_text":"Great product!","task_id":"task_001"}'
```

**Convention for AI agents parsing task payloads:**

1. If the payload is JSON (`type: "webhook"`), parse it directly — it's already structured
2. If the payload is HTML (`type: "email"`), look for `input[name="agent_data"]` and parse its `value` as JSON
3. If no `agent_data` input exists, fall back to parsing visible text content
4. Use `task_id` from the JSON as your `task_identifier` when submitting proofs. For `queue` gigs (where you poll tasks via `/queue/poll`), use the polled task's `id` instead — this lets the server atomically claim the queue item to your mailbox.

---

## REST API Reference

Auth via `x-api-key` header on all authenticated endpoints.

### Authentication

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/auth/send-otp` | No | Send 4-digit OTP code to email |
| POST | `/auth/verify-otp` | No | Verify OTP and get API key |
| POST | `/auth/rotate-key` | Yes | Generate new API key |
| GET | `/auth/me` | Yes | Get current user profile |

#### POST /auth/send-otp

```json
// Request
{ "email": "user@example.com" }

// Response
{ "message": "Code sent" }
```

4-digit code (1000-9999), 10-minute expiry, max 5 attempts. Sends via email.

#### POST /auth/verify-otp

```json
// Request
{ "email": "user@example.com", "code": "1234" }

// Response
{ "email": "user@example.com", "api_key": "base64url_encoded_key" }
```

Creates new user if first login. Auto-provisions hot wallet. Returns existing API key (no rotation on login).

#### POST /auth/rotate-key

```json
// Response
{ "api_key": "base64url_encoded_key" }
```

#### GET /auth/me

```json
// Response
{ "email": "...", "display_name": "...", "bio": "...", "avatar_url": "...", "created_at": "...", "officex_user_id": "...", "officex_install_id": "..." }
```

### Gigs

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/gigs` | Yes | Create new gig |
| POST | `/gigs/:id/invites` | Yes | Mint an invite link (owner only) |
| GET | `/gigs/:id/invites` | Yes | List invite links (owner only) |
| DELETE | `/gigs/:id/invites/:token` | Yes | Revoke an invite link (owner only) |
| GET | `/gigs/mine` | Yes | List user's owned gigs (`?tag=` substring filter) |
| GET | `/gigs/:id` | Optional | Get gig detail |
| PATCH | `/gigs/:id` | Yes | Update gig (owner only) |
| POST | `/gigs/:id/rotate-token` | Yes | Rotate security token (owner only) |
| POST | `/gigs/:id/tasks/:msgId/extend` | Yes | Reset a task's expiry clock (owner only) |
| POST | `/gigs/:id/tasks/:msgId/recycle` | Yes | Take a task back and redistribute it (owner only) |
| GET | `/gigs/:id/dashboard` | Yes | Get gig dashboard with all data (owner only) |
| POST | `/gigs/:id/deposit` | Yes | Deposit USDC to gig treasury |

#### POST /gigs

```json
// Request
{
  "title": "Reddit Comments for Product Launch",
  "price": 0.50,
  "terms": "Comment on specified Reddit threads with genuine engagement...",
  "notes": "Internal notes for owner only",
  "owner_wallet": "wallet_alias_id",      // optional, auto-provisions if omitted
  "join_policy": "invite",                 // "invite" (default — joins require an invite token) | "open"
  "tags": ["reddit", "writing", "q3-launch"],  // arbitrary free-form strings (max 25 tags, 256 chars each)
  "requires_approval": false,
  "review_timeout": 172800,                // seconds, default 48h
  "task_timeout": 86400,                   // optional, seconds a worker may hold a task before it expires; null = no expiry (default)
  "distribution": "round_robin",           // "round_robin" | "free_for_all" | "priority_weighted" | "random" | "queue" | "queue_solo" | "inbound_proof"
  "queue_order": "fifo",                   // "queue"/"queue_solo" only: "fifo" (default) | "lifo" | "priority" | "random"
  "max_claims_per_task": 3,                // "queue_solo" only: how many workers may each take a task; null = unlimited (default)
  "default_rate_limit_count": 5,           // optional worker rate limit: max proofs/claims per window; both fields or neither
  "default_rate_limit_minutes": 60,        // window length in minutes; null on both = no limit (default)
  "min_rep_volume": null,
  "min_rep_quality": null,
  "min_rep_recency": null,
  "min_payout": 0,
  "location": { "country": "US", "label": "United States" },
  "icon_url": "https://...",
  "proof_webhook_url": "https://...",
  "join_webhook_url": "https://...",       // optional, POSTed a mailbox.joined event when a worker joins
  "contract_address": "0x..."
}

// Response
{
  "gig": {
    "id": "GIG_01HX...",
    "title": "Reddit Comments for Product Launch",
    "email": "GIG_01HX..._abc123.dollar-platoon@fwd.zoomgtm.com",
    "webhook": "https://dollarplatoon.com/api/inbound/webhook/GIG_01HX...?token=abc123",
    "invite_url": "https://dollarplatoon.com/gig/GIG_01HX.../join?invite=a1b2c3d4e5f6",
    "join_policy": "invite",
    "price": 0.50,
    "requires_approval": false,
    "status": "active"
  }
}
```

Compliance check via Gemini (blocks illegal content, warns on borderline).

Tags are **arbitrary free-form strings** — use them to categorize, group, and search gigs (e.g. by campaign, client, or batch). Max 25 tags per gig, 256 chars each. There is no whitelist.

New gigs default to `join_policy: "invite"` and are created with a **default unlimited invite** — the returned `invite_url` includes its token. Revoke it and mint scoped invites via the Invites endpoints below. There is no public marketplace: `GET /gigs` returns `410 Gone`.

#### Invites

Invite links gate who can join a gig's private network. Modes fall out of two fields: `max_uses` (1 = one-time, N = N uses, null = unlimited) and `email` (bind to an exact address, or null for anyone with the link). Email-bound invites act as pre-approvals — the invited worker skips `pending_approval` even when the gig has `requires_approval`.

```json
// POST /gigs/:id/invites (Owner Only)
// Request
{ "max_uses": 1, "email": "worker@example.com", "label": "for Alice" }  // all fields optional

// Response
{
  "invite": {
    "token": "a1b2c3d4e5f6",
    "max_uses": 1, "uses": 0, "email": "worker@example.com", "label": "for Alice",
    "invite_url": "https://dollarplatoon.com/gig/GIG_01HX.../join?invite=a1b2c3d4e5f6"
  }
}
```

`GET /gigs/:id/invites` lists all invites with `uses`, `revoked`, and `exhausted`. `DELETE /gigs/:id/invites/:token` revokes one — anyone holding the link can no longer join. Use consumption is atomic, so concurrent joins can't race past `max_uses`.

#### GET /gigs/mine

```
GET /gigs/mine?tag=q3
```

Lists your owned gigs (excluding closed). `?tag=` filters by case-insensitive **substring** match against gig tags; comma-separated values are OR'd — handy for grouping many gigs by campaign or batch.

#### GET /gigs/:id

Returns gig object. If authenticated as owner or member, includes `notes` and enriched data. Shows `available_funds` and `reserved_funds` so you can assess whether the gig can pay.

#### PATCH /gigs/:id (Owner Only)

```json
// Request (any subset)
{
  "title": "Updated Gig Title",
  "price": 1.00,
  "terms": "Updated terms...",
  "status": "paused",
  "review_timeout": 86400,
  "task_timeout": 86400,                     // seconds before a held task expires; null disables expiry
  "tags": ["reddit", "q3-launch"],           // arbitrary free-form strings; replaces the full list
  "join_policy": "invite",                   // "invite" | "open"
  "distribution": "random",
  "default_rate_limit_count": 5,             // worker rate limit default: N proofs per M minutes; set both null to remove
  "default_rate_limit_minutes": 60,
  "requires_approval": true,
  "min_payout": 1,
  "location": { "country": "US" },
  "notes": "Updated internal notes",
  "proof_webhook_url": "https://...",
  "join_webhook_url": "https://...",         // POSTed a mailbox.joined event when a worker joins; null to disable
  "contract_address": "0x..."
}

// Response
{ "success": true }
```

#### Worker Rate Limits (default_rate_limit_count / default_rate_limit_minutes)

Optional per-worker throttle: each gigworker may take at most **N proofs per M minutes**. "Take" counts both proofs submitted and queue tasks claimed via `/queue/poll` that aren't proven yet — so a worker can't hoard the FIFO queue by claiming ahead. Default is no limit.

- Set the gig-wide default with `default_rate_limit_count` + `default_rate_limit_minutes` (both positive integers, or both `null` to disable).
- Override per worker via `PATCH /gigs/:id/mailboxes/:mbx_id` with `rate_limit_count` + `rate_limit_minutes` (owner only; both `null` reverts the mailbox to the gig default).
- When a worker is at their limit, `/queue/poll` and `POST /gigs/:id/proofs` return `429` with a human-readable `error` (stating the limit and wait time) plus a `rate_limit` object: `{ count, minutes, source: "gig"|"mailbox", used, remaining, retry_at }`.
- Submitting a proof for a queue task you already claimed is never blocked — the claim was already counted at poll time.

#### Task Expiry (task_timeout)

Set `task_timeout` (seconds) on a gig to give workers a deadline: once a task is claimed (queue gigs) or delivered (push gigs), the worker must submit a proof — or act on it (report/skip) — before the deadline. Default is `null`: tasks never expire.

- Expired tasks are blocked server-side: proof submission, skip, and report return `410 Gone`.
- Unclaimed queue items never expire — the clock starts at claim/delivery.
- Task listings (`GET /mailboxes/:mbxId/inbound`, dashboard inbound) include `expires_at` and `expired` per task.
- The owner resolves expired tasks with the endpoints below (also usable before expiry).

#### POST /gigs/:id/tasks/:msgId/extend (Owner Only)

Resets the task's expiry clock, flipping it back to not-expired in the worker's mailbox.

```json
// Response
{ "success": true, "expires_at": "2026-07-16T12:00:00.000Z", "expired": false }
```

#### POST /gigs/:id/tasks/:msgId/recycle (Owner Only)

Takes the task back from its current worker and redistributes it: `queue` gigs return it to the queue for the next worker (the previous holder won't receive it again); `queue_solo` gigs discard that worker's copy and return its claim slot, leaving every other worker untouched and making the task available to the original worker again; push gigs reassign it to another mailbox per the gig's distribution mode.

```json
// Response (queue gig)
{ "success": true, "requeued": true }

// Response (push gig)
{ "success": true, "reassigned_to": "01HX...", "reassigned_to_name": "Worker name" }
```

#### POST /gigs/:id/rotate-token (Owner Only)

```json
// Response
{
  "email": "GIG_01HX..._newtoken.dollar-platoon@fwd.zoomgtm.com",
  "webhook": "https://dollarplatoon.com/api/inbound/webhook/GIG_01HX...?token=newtoken"
}
```

Generates a new 6-char security token. Invalidates old email address and webhook URL. Old email lookup is deleted and replaced. Update all publisher integrations with the new URLs after rotating.

#### GET /gigs/:id/dashboard (Owner Only)

```json
// Response
{
  "gig": { ... },
  "mailboxes": [ ... ],
  "proofs": [ ... ],
  "rollups": [ ... ],
  "inbound_messages": [ ... ]
}
```

Syncs on-chain balance on every load. Signs all S3 URLs for proof attachments.

#### POST /gigs/:id/deposit

```json
// Request
{ "wallet_alias_id": "alias_id", "amount": 100 }

// Response
{ "tx_hash": "0x...", "available_funds": 100 }
```

Deposits USDC from your hot wallet to the gig's on-chain balance. Remember to budget 110% of expected payouts to cover the platform fee.

### Mailboxes

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/gigs/:id/mailboxes` | Yes | Join gig (create mailbox) |
| GET | `/gigs/:id/mailboxes` | Yes | List mailboxes in gig (owner only) |
| PATCH | `/gigs/:id/mailboxes/:mbx_id` | Yes | Update mailbox (owner: priority/status; worker: tags) |
| DELETE | `/gigs/:id/mailboxes/:mbx_id` | Yes | Leave gig / remove mailbox |
| GET | `/mailboxes/mine` | Yes | List user's mailboxes across all gigs (`?tag=` substring filter) |
| GET | `/mailboxes/:mbxId/inbound` | Yes | Fetch inbound messages for mailbox |
| POST | `/gigs/:id/mailboxes/:mbxId/regenerate-token` | Yes | Regenerate share token |

#### POST /gigs/:id/mailboxes (Join Gig)

```json
// Request
{
  "name": "John's Mailbox",
  "email": "john@example.com",
  "invite": "a1b2c3d4e5f6",     // required for join_policy "invite" gigs — token from the invite link
  "wallet_address": "0x...",    // optional, auto-provisions hot wallet if omitted
  "webhook": "https://...",     // optional, for webhook task delivery
  "notes": "I have experience with Reddit marketing",
  "location": { "country": "US" },
  "tags": ["urgent", "linkedin-batch"]  // optional, free-form private labels (max 25 tags, 256 chars each)
}

// Response
{
  "mailbox": {
    "id": "01HX...",
    "name": "John's Mailbox",
    "gig_id": "GIG_01HX...",
    "status": "active"          // or "pending_approval" if gig.requires_approval
  }
}
```

Validates reputation thresholds. Auto-creates wallet alias for external wallets. Gigs with `join_policy: "invite"` reject joins without a valid invite token (403); email-bound invites must match your account email and skip owner approval. Legacy gigs without a join_policy remain open joins.

If the gig has `join_webhook_url` set (via `POST /gigs` or `PATCH /gigs/:id`), each successful join fires a fire-and-forget POST to that URL:

```json
{
  "event": "mailbox.joined",
  "gig_id": "GIG_01HX...",
  "mailbox_id": "01HX...",
  "name": "John's Mailbox",
  "status": "active",                  // or "pending_approval"
  "email": "john@example.com",         // mailbox contact email (null if not provided)
  "wallet_address": "0x...",
  "invite_token": "a1b2c3d4e5f6",      // which invite was used (null for open joins)
  "joined_at": "2026-07-16T00:00:00.000Z"
}
```

#### PATCH /gigs/:id/mailboxes/:mbx_id

```json
// Request (gig owner)
{ "priority": 5, "status": "active", "rate_limit_count": 5, "rate_limit_minutes": 60 }

// Request (mailbox worker)
{ "tags": ["urgent", "linkedin-batch"], "wallet_address": "0x..." }

// Response
{ "success": true, "status": "active", "tags": ["urgent", "linkedin-batch"], "wallet_address": "0x..." }
```

Owner can set `priority`, `status` (`"active"` to approve a pending mailbox, `"inactive"` to disable it), and a per-worker rate limit override: `rate_limit_count` + `rate_limit_minutes` (max N proofs/claims per M minutes; both positive integers, or both `null` to revert to the gig's `default_rate_limit_*`). The mailbox's worker can set `tags` — arbitrary free-form labels for organizing their inbox (replaces the full list; max 25 tags, 256 chars each). Tags are private to the worker: they are never returned to the gig owner via `GET /gigs/:id/mailboxes`.

The mailbox's worker can also set `wallet_address` to change where future USDC payouts land (Base). Rules:
- Must be a valid EVM address. It is auto-registered as a wallet alias on your account; an address already registered to a **different** account is rejected with `409` (wallets stay 1:1 with users, so reputation can't be hijacked). The same 409 applies when joining a gig with someone else's address.
- Takes effect for **future rollups only** — already-created rollups (pending or debt retries) pay out to the address snapshotted when the rollup was created.
- Reputation survives the change: rep events accrue per wallet, but gig `min_rep_*` join thresholds and profile reputation are computed by merging events across **all** wallets registered to your account, so rotating payout addresses never resets your history.

#### GET /mailboxes/mine

```json
// Response
{
  "mailboxes": [
    {
      "id": "...", "name": "...", "gig_id": "GIG_...", "status": "active",
      "gig_title": "...", "gig_email": "...", "owner_email": "...", "owner_display_name": "...",
      "tasks_received": 12, "proofs_submitted": 10, "response_rate": 0.83,
      "tags": ["urgent", "linkedin-batch"]
    }
  ]
}
```

Supports `?tag=` filtering by case-insensitive **substring** match against your tags — `?tag=link` matches a mailbox tagged `"linkedin-batch"`. Comma-separated values are OR'd: `?tag=urgent,linkedin`.

#### GET /mailboxes/:mbxId/inbound

```json
// Response
{
  "inbound_messages": [
    {
      "id": "...", "gig_id": "...", "type": "email", "subject": "...", "from": "sender@example.com",
      "payload": "...", "payload_truncated": false, "payload_bytes": 4211,
      "mailbox_id": "...", "forwarded_at": "...",
      "attachments": [{ "filename": "...", "content_type": "...", "url": "https://..." }]
    }
  ],
  "next_cursor": null
}
```

**This is a list endpoint — `payload` may be a preview.** Large task bodies are stored outside
the message row, so a list response carries only the first 1,000 characters of one.
**`payload_truncated: true` means you must fetch the task itself before acting on it** —
`payload_bytes` tells you the real length. Small tasks (under 6,000 characters) come back whole
and always have `payload_truncated: false`.

Paginated newest-first, 100 per page (`?limit=` up to 300). When `next_cursor` is non-null,
pass it as `?cursor=` to get the next page. `?summary=1` returns every message with no
payload field at all — use it for counts and unread badges.

#### GET /gigs/:id/tasks/:msgId

The full body of a single task. Use this whenever a list gave you `payload_truncated: true`.

```json
// Response
{
  "task": {
    "id": "...", "type": "email", "subject": "...", "from": "...",
    "payload": "<the complete body, never truncated>",
    "payload_truncated": false, "payload_bytes": 31674,
    "mailbox_id": "...", "forwarded_at": "...", "claimed_at": "...",
    "expires_at": "...", "expired": false, "violation": null,
    "attachments": [{ "filename": "...", "content_type": "...", "url": "https://..." }]
  }
}
```

Readable by the gig owner, by the worker the task is assigned to, and — for tasks still sitting
in the queue — by any member of that gig.

Tasks returned by `POST /gigs/:id/queue/poll` always carry their complete body already, so a
polling worker never needs this endpoint.

### Proofs

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/gigs/:id/proofs` | Yes | Submit proof of work |
| GET | `/gigs/:id/proofs` | Yes | List proofs (filterable by status) |
| GET | `/gigs/:id/proofs/:proof_id` | Yes | Get proof detail |
| PATCH | `/gigs/:id/proofs/:proof_id` | Yes | Approve or reject proof (owner only) |
| POST | `/gigs/:id/proofs/:proof_id/report` | Yes | Report auto-approved proof (owner only) |

#### POST /gigs/:id/proofs (Submit Proof)

```json
// Request
{
  "mailbox_id": "01HX...",
  "task_identifier": "reddit-thread-abc123",
  "proofs": ["https://reddit.com/r/...", "https://s3.amazonaws.com/..."]
}

// Response
{
  "proof": {
    "id": "01HX...",
    "status": "pending",
    "timeout_at": "2026-02-18T..."
  },
  "warning": "Warning: gig available funds are less than the task price"
}
```

**`task_identifier` is critical.** This field links a proof to the specific task it fulfills.
- For **`queue` gigs**, pass the polled task's `id` (the inbound message ULID returned by `/queue/poll`). The server uses this to atomically claim the queue item out of the queue to your mailbox.
- For **`inbound_proof` gigs** and other distribution modes, use the task's unique reference (URL, ticket ID, publisher-supplied `task_id`, etc.).
- **Avoid using the subject line** — subjects are rarely unique, and collisions cause duplicate-submission 409s.

The `warning` field appears when the gig's `available_funds` is less than the task price. The proof is still accepted, but payout will fail until the client deposits more funds.

Price is locked at submission time (`locked_price`).

#### PATCH /gigs/:id/proofs/:proof_id (Review)

```json
// Request (approve)
{ "action": "approve", "feedback": "Great work!" }

// Request (reject)
{ "action": "reject", "feedback": "Screenshot doesn't match", "rejection_tag": "incomplete" }

// Response
{ "success": true, "status": "approved" }
```

Rejection tags (required when rejecting): `low_quality`, `incomplete`, `fake_proof`, `duplicate`, `unresponsive`, `other`

Rejection weights (reputation impact): fake_proof=5x, duplicate=3x, incomplete=2x, unresponsive=2x, low_quality=1x, other=1x

#### POST /gigs/:id/proofs/:proof_id/report (Owner Only)

```json
// Response
{ "success": true, "status": "reported" }
```

Only works on `timeout_approved` proofs. Reported proofs are excluded from rollups and will not be paid.

### Rollups (Payouts)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/gigs/:id/rollups` | Yes | List rollups for gig |
| POST | `/gigs/:id/rollups` | Yes | Trigger manual rollup (owner only) |
| GET | `/rollups/mine` | Yes | List rollups across user's mailboxes |

#### POST /gigs/:id/rollups (Trigger Payout)

```json
// Response
{
  "rollups": [
    {
      "id": "...",
      "mailbox_id": "...",
      "wallet_address": "0x...",
      "proof_ids": ["...", "..."],
      "gross_amount": 5.00,
      "platform_fee": 0.50,
      "net_amount": 5.00,
      "tx_hash": "0x...",
      "status": "paid"
    }
  ],
  "available_funds": 44.50,
  "skipped_below_minimum": [
    { "mailbox_id": "...", "amount": 0.50 }
  ]
}
```

Groups approved + timeout_approved proofs by mailbox. Pre-checks `available_funds >= gross_amount + platform_fee` (no debt allowed). Worker receives full `gross_amount`. Skips mailboxes below `min_payout` threshold.

**Will return 400 error if the gig cannot cover the total cost (gross + 10% fee).**

### Inbound (Task Distribution)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/inbound/email` | No | Resend inbound email webhook |
| POST | `/inbound/webhook/:gig_id` | No | Publisher webhook task delivery |

#### POST /inbound/webhook/:gig_id?token=...

**This is the preferred endpoint for AI agents and publisher apps to deliver tasks.** Accepts **JSON** (default) or **HTML/plain text** payloads. Content-Type header determines parsing.

**For AI agents:** Use `Content-Type: text/html` with the dual-format HTML pattern (see "Dual-Format HTML" section above) to deliver tasks that work for both human gigworkers and AI agents. Include a hidden `<input type="hidden" name="agent_data" value=.{...}.>` carrying the task JSON — see the parsing convention in that section.

**JSON payload (Content-Type: application/json):**

```json
// Request
{ "task": "Comment on this Reddit thread", "url": "https://..." }

// Response
{ "status": "forwarded", "targets": 3 }
```

**HTML payload (Content-Type: text/html or text/plain) — recommended for mixed human+agent gigs:**

```bash
curl -X POST "https://dollarplatoon.com/api/inbound/webhook/GIG_01HX...?token=abc123&subject=My+Report" \
  -H "Content-Type: text/html" \
  -d '<h1>Task Details</h1><p>Please complete this task...</p>'
```

```json
// Response
{ "status": "forwarded", "targets": 3 }
```

When HTML/text is sent, the message is stored with `type: "email"` and rendered as formatted HTML on the frontend (same as email-sourced tasks). An optional `subject` query parameter can be included to set the message subject line.

Requires valid `token` query parameter matching the gig`s security token. Returns 403 if token is invalid. Selects mailboxes via distribution algorithm, forwards payload to each mailbox webhook.

**Payload size:** bodies are stored in full — there is no silent truncation. Anything above
2,000,000 characters is rejected with `413` and a body of
`{ "error": "Payload too large", "received_chars": N, "max_chars": 2000000 }`, so an oversized
task fails loudly instead of arriving with its tail missing. Bodies over 6,000 characters are
kept off the message row, which is why list endpoints return a preview and `payload_truncated`.

**Distribution Modes:**

- **round_robin** — Cursor-based fair rotation through active mailboxes
- **random** — Uniform random selection
- **priority_weighted** — Weighted by mailbox priority (1-10, higher = more tasks)
- **free_for_all** — All active mailboxes receive the task
- **queue** — Tasks stored in a shared queue; workers poll and claim tasks on-demand
- **inbound_proof** — No tasks distributed; workers submit proofs directly without task assignment

### Queue

Two queue distributions share every endpoint below. They differ only in whether workers compete.

| | `queue` (shared) | `queue_solo` (single player) |
|---|---|---|
| Who can claim a task | The first worker to poll it | Every worker, independently |
| Effect on other workers | Claiming removes it from their queue | None — nothing you do is visible to them |
| What you receive | The task itself | Your own private copy of the task |
| Proofs per task | One, gig-wide | One per worker |
| Cost per task | price × 1 | price × number of workers who take it |
| Task leaves the queue when | Someone claims it | It hits `max_claims_per_task` (never, if unlimited) |

Both honour `queue_order`:

- **`fifo`** (default) — oldest task first
- **`lifo`** — newest task first
- **`priority`** — you set the order. Each task carries a `priority` number; lower goes first,
  and tasks sharing a number are served oldest-first. So 0, 1, 2, 5, 7, 23, 4689, 99999 are
  polled in exactly that order, and setting a task to 0 jumps it ahead of a task at 1.
- **`random`** — each poll draws at random from the tasks still in the queue. There is no
  position, so `priority` numbers are ignored while this mode is on. Two polls a second apart
  get different tasks, and a task at the back is as likely as one at the front. A poll samples
  up to the first 1000 queued tasks; past that the draw is random over that window.

New tasks default to priority **1000**, leaving room to insert both above and below without
renumbering anything. Priorities are non-negative whole numbers up to 9999999999.

Think of priority as a **virtual arrival time**: a task at priority 0 behaves as though it
arrived before everything else. That is why `fifo` honours priorities too (oldest virtual
arrival first) and `lifo` reverses them (highest number first). Setting `queue_order` to
`priority` is how you declare the queue is meant to be hand-ordered — it turns on the priority
column in the dashboard — but reordering works in any mode.

**`queue_solo` cost warning:** you pay **per task, per worker**. Ten queued tasks in a solo gig
with five workers is fifty payouts, not ten. Set `max_claims_per_task` to bound this — it is also
the only thing that ever drains a solo queue, since claiming does not consume the task.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/gigs/:id/queue/poll` | Yes | Poll for available tasks (gigworker, queue gigs only) |
| POST | `/gigs/:id/queue/:msgId/decline` | Yes | Skip a task so future polls don't return it to you (per-worker, does not hide from other workers) |
| GET | `/gigs/:id/queue` | Yes | List queued tasks (owner sees `declined_count` and `priority` per item) |
| GET | `/gigs/:id/tasks/:msgId` | Yes | Get one task with its full body (owner, assignee, or gig member for queued tasks) |
| PATCH | `/gigs/:id/tasks/:msgId/priority` | Yes | Move one queued task (gig owner only) |
| PATCH | `/gigs/:id/queue/priorities` | Yes | Reorder up to 100 queued tasks in one call (gig owner only) |
| DELETE | `/gigs/:id/tasks/:taskId` | Yes | Delete a stored task/inbound message (gig owner only) |

#### Setting priority

Three ways, all equivalent — they write the same number:

**1. At submission**, via a query param on the publisher webhook. No follow-up call needed:

```
POST https://dollarplatoon.com/api/inbound/webhook/GIG_abc?token=...&priority=0
```

Priority rides on the query string because the request body is the task payload itself.
Omitted means 1000. The response echoes `{"status": "queued", "message_id": "...", "priority": 0}`.

**2. One task at a time:**

```json
PATCH /gigs/:id/tasks/:msgId/priority
{ "priority": 0 }

// Response
{ "success": true, "id": "...", "priority": 0 }
```

**3. In bulk** — the endpoint to use when rearranging a queue programmatically:

```json
PATCH /gigs/:id/queue/priorities
{ "updates": [ { "id": "task_a", "priority": 0 }, { "id": "task_b", "priority": 10 } ] }

// Response
{
  "success": true,
  "updated": 1,
  "applied": [ { "id": "task_a", "priority": 0 } ],
  "skipped": [ { "id": "task_b", "reason": "already_claimed" } ]
}
```

Up to 100 tasks per call. The whole batch is validated before any of it is written, so a
malformed entry rejects the request with `400` and changes nothing. Individual tasks that
can't be moved are reported in `skipped` rather than failing the batch — `not_found`,
`already_claimed` (a worker has it; its position no longer means anything), or `not_in_queue`
(a `queue_solo` task that already hit `max_claims_per_task`).

Reordering is cheap: position is stored in the task's index key, so each move is a single
write and never touches the tasks you left alone. Rewriting the order of a 100-task queue on
every tick is a reasonable thing to do.

**Only queued tasks can be moved.** Once a worker claims a task it has left the queue, and a
single-task PATCH returns `409`. Reordering the queue never disturbs work already in progress.

#### POST /gigs/:id/queue/poll

```json
// Request
{ "count": 2 }   // optional, default 2, max 20

// Response
{
  "tasks": [
    {
      "id": "...", "type": "webhook", "subject": "...",
      "payload": "...", "forwarded_at": "...", "claimed_at": "...",
      "source_task_id": "..."   // queue_solo only: the shared task this copy came from
    }
  ],
  "count": 1,
  "scan_exhausted": false,  // queue_solo only — see below
  "rate_limit": { "count": 5, "minutes": 60, "source": "gig", "used": 3, "remaining": 2, "retry_at": null }  // null if no limit configured
}
```

For `queue` and `queue_solo` gigs. Returns tasks in the configured queue order, skipping anything you've already taken, proven, or declined. Tasks are not forwarded to mailboxes — gigworkers must poll to claim them.

**Poll after every proof.** The intended loop is: poll → work → submit proof → poll again. Submitting a proof does not automatically fetch more work. A claim and its proof together cost one slot against your rate limit, not two, so the loop never double-charges you.

In `queue_solo` gigs, each returned task is a private copy with its own `id`. Use that `id` as your `task_identifier` — never `source_task_id`, which is shared with other workers and will be rejected. Because no one competes with you, a poll can never fail with "already claimed"; it returns nothing only when you have already taken every task in the queue.

**`scan_exhausted`** (`queue_solo` only) tells you which kind of empty response you got. A solo
claim leaves the task in place, so the poll has to scan past everything you already hold, and
it stops after a fixed budget. `false` with no tasks means there is genuinely nothing left for
you — back off. `true` means the scan ran out of budget before filling your batch, and polling
again will make further progress. Only large solo queues in priority order reach it.

If the gig (or your mailbox specifically) has a worker rate limit, polling past it returns `429` with an `error` message stating the limit and how long to wait, plus the same `rate_limit` object with `retry_at` set. Claims are capped to your remaining allowance — e.g. requesting 10 tasks with 2 remaining returns at most 2.

#### POST /gigs/:id/queue/:msgId/decline

Marks a queue item as skipped *for the calling worker only*. Idempotent. Returns `{"success": true}`.

Use this when a polled task isn't suitable for you (spam, duplicate, ineligible, etc.) so future polls return fresh items instead of the same ones at the head of the queue. Skipping returns the task to the position it held — it does not push it to the back — so other workers still see it exactly where the owner put it. The gig owner sees a `declined_count` on their dashboard so they can prune genuinely unworkable items.

In `queue_solo` gigs, pass the `id` of your own copy. The copy is discarded, its claim slot is returned to the shared task so other workers are unaffected, and the task is retired for you permanently.

**For Gigworkers (Queue gigs):**

- Tasks are NOT forwarded to your mailbox. Instead, use "Poll New Tasks" in the UI or call `POST /gigs/:id/queue/poll` to claim tasks.
- Tasks are returned in the configured queue order (FIFO, LIFO, or the owner's hand-set priority) and filtered against proofs you've already submitted or items you've declined. Always work the tasks in the order polled — in a priority queue that order is the client's explicit instruction, not a suggestion.
- After polling, submit proofs via `POST /gigs/:id/proofs` using the polled task's `id` as `task_identifier`.
- Keep polling after each proof — that is the normal working loop, and it is not rate-limited beyond the gig's configured limit.
- If a task isn't suitable, call `POST /gigs/:id/queue/:msgId/decline` to skip it. Declining is free and doesn't affect other workers.
- In a **shared** `queue` gig you are racing other workers: a task can be claimed out from under you between polling and proving, which returns `409`. In a **`queue_solo`** gig that cannot happen — your copies are yours until you prove, skip, or the owner recycles them.

### Public (No Auth Required)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/public/mailbox-info?token=...` | No | Get mailbox info via share token |
| POST | `/public/upload-presign` | No | Get S3 presigned upload URL |
| POST | `/public/submit-proof` | No | Submit proof via public share link |
| GET | `/public/read-url?key=...&token=...` | No | Get presigned S3 read URL |

Rate limited: 10-30 requests/min per share token.

#### POST /public/submit-proof

```json
// Request
{
  "share_token": "tok_...",
  "task_identifier": "reddit-thread-abc123",
  "proofs": ["https://..."]
}

// Response
{ "proof_id": "...", "status": "pending" }
```

### Reviews

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/gigs/:id/reviews` | Yes | Leave star review (1-5) |
| PATCH | `/reviews/:id/resolve` | Yes | Mark review as resolved (reviewer only) |
| GET | `/reputation/:wallet/reviews` | No | List reviews for wallet |

#### POST /gigs/:id/reviews

```json
// Request
{ "target_wallet": "0x...", "stars": 4, "comment": "Reliable worker, good quality" }

// Response
{ "review": { "id": "...", "stars": 4 } }
```

One review per reviewer-target pair per gig. Reviewer role auto-detected (client if owner, gigworker otherwise).

### Reputation

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/reputation/:wallet` | No | Get computed reputation score |
| GET | `/reputation/alias/:alias_id` | No | Get reputation by wallet alias |
| GET | `/reputation/:wallet/events` | No | List raw reputation events |

#### GET /reputation/:wallet

```json
// Response
{
  "wallet": "0x...",
  "volume": 150.50,
  "quality": 0.92,
  "recency": 0.85,
  "social": 4.2,
  "event_count": 47
}
```

### Wallets

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/wallets` | Yes | Create wallet alias |
| GET | `/wallets` | Yes | List user's wallet aliases |
| GET | `/wallets/:alias_id` | Yes | Get wallet detail |
| GET | `/wallets/:alias_id/balances` | Yes | Get on-chain balances (ETH + USDC) |
| POST | `/wallets/:alias_id/transfer` | Yes | Transfer USDC from hot wallet |
| DELETE | `/wallets/:alias_id` | Yes | Delete wallet alias |

#### POST /wallets

```json
// Request (hot wallet — platform-managed)
{ "label": "My Hot Wallet", "is_hot_wallet": true }

// Request (external wallet — self-custody)
{ "label": "My MetaMask", "is_hot_wallet": false, "evm_address": "0x..." }

// Response
{ "wallet": { "alias_id": "...", "label": "My Hot Wallet", "is_hot_wallet": true, "created_at": "..." } }
```

One hot wallet per user. External wallets are unlimited.

#### GET /wallets/:alias_id/balances

```json
// Response
{ "evm_address": "0x...", "eth_balance": "0.05", "usdc_balance": "100.000000" }
```

#### POST /wallets/:alias_id/transfer

```json
// Request
{ "to_address": "0x...", "amount": 50 }

// Response
{ "tx_hash": "0x..." }
```

Hot wallets only.

### Profiles

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| PATCH | `/profiles/me` | Yes | Update own profile |
| GET | `/profiles/:identifier` | No | Get public profile (by email or alias_id) |
| GET | `/profiles/:identifier/private` | Yes | Get private profile (requires shared gig relationship) |

#### PATCH /profiles/me

```json
// Request
{ "display_name": "John Doe", "bio": "Experienced social media marketer", "avatar_url": "https://..." }
```

### Upload

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/upload/presign` | Yes | Get presigned S3 upload URL |

```json
// Request
{ "filename": "screenshot.png", "content_type": "image/png", "prefix": "proofs" }

// Response
{ "presigned_url": "https://s3...", "url": "https://s3...", "key": "proofs/...", "bucket": "..." }
```

Prefix options: `"avatars"`, `"gig-icons"`, or `"proofs"` (default). Presigned URL expires in 1 hour.

### OfficeX Integration

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/officex/webhook` | No | Handle OfficeX install/uninstall |
| POST | `/officex/login` | No | Login via OfficeX credentials |

#### POST /officex/webhook

```json
// Request
{ "event": "INSTALL", "payload": { "install_id": "...", "install_secret": "...", "user_id": "...", "app_id": "..." } }

// Response
{ "agent_context": { "user_email": "officex-...@dollar-platoon.local", "api_key": "...", "api_url": "https://...", "install_id": "...", "install_secret": "..." } }
```

Creates user with email `officex-{user_id}@dollar-platoon.local`. Auto-provisions hot wallet.

#### POST /officex/login

```json
// Request
{ "officex_user_id": "...", "officex_install_id": "..." }

// Response
{ "email": "officex-...@dollar-platoon.local", "api_key": "..." }
```

Returns 404 if user not found (webhook may not have fired yet). Returns 403 if install_id mismatch.

### Admin

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/admin/users/provision` | `x-admin-key` header | Programmatically provision (or fetch) an account |

#### POST /admin/users/provision

Idempotent. Creates the account (and auto-provisions a hot wallet) if the email is new — `201` with `"created": true`. If the account already exists, no changes are made and the existing account info is returned — `200` with `"created": false`. The API key is never rotated by this endpoint.

```json
// Request (header: x-admin-key: <ADMIN_API_KEY>)
{ "email": "user@example.com" }

// Response — 201 if newly created, 200 if the account already existed
{
  "created": false,
  "account": {
    "email": "user@example.com",
    "api_key": "base64url_encoded_key",
    "display_name": null,
    "created_at": "2026-02-14T...",
    "officex_user_id": null,
    "officex_install_id": null
  }
}
```

Returns 503 if `ADMIN_API_KEY` is not configured, 401 if the header is missing, 403 if the key is invalid.

### Health

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/health` | No | Health check |

```json
{ "status": "ok", "stage": "production", "timestamp": "2026-02-14T..." }
```

---

## Proof Lifecycle

```
submitted (locked_price snapshot, timeout_at set)
  → approved (client action) → rolled up → payout on-chain → paid
  → rejected (requires rejection_tag + optional feedback)
  → timeout_approved (daily cron, after review_timeout) → same rollup path
  → reported (post-timeout flag by owner, excluded from payouts)
```

Rejection tags: `low_quality`, `incomplete`, `fake_proof`, `duplicate`, `unresponsive`, `other`

---

## Rollup & Payout Flow

1. Client triggers `POST /gigs/:id/rollups` (or daily cron runs automatically)
2. Groups approved proofs by mailbox, sums `locked_price` per mailbox
3. Skips mailboxes below `min_payout` threshold
4. Pre-checks: `gross_amount + platform_fee <= available_funds` — **fails with 400 if underfunded**
5. Calls on-chain `payout(gig_id, wallet, gross_amount, rollup_id)`
6. On success: stores `tx_hash`, status → `paid`, creates reputation event
7. On failure: status → `failed`, retried by next daily cron run
