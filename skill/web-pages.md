# Web pages, deep links, and embeds

Every page you can hand to somebody or frame inside your own product, and the URL params that
control them.

## Contents

- Autologin deep links
- Universal URL params
- The Task page — one task, on a page of its own
- Notifications — the bell in the navbar
- The Share Proof link — one proof, on its own review page
- The Insert Task page — put work in without an account
- The Submit page — hand in work without an account
- Feed reader pages
- Timeline pages
- Iframe rules that actually matter
- Page map

---

## Autologin deep links

Append `?api_key=` to **any** dollarplatoon.com URL to log in and land on that exact page in one
step. This is how an agent, an email, or a partner site sends somebody straight to a dashboard.

```
https://dollarplatoon.com/client/gig/GIG_01HX.../dashboard?api_key=YOUR_API_KEY
https://dollarplatoon.com/gigworker/mailboxes?api_key=YOUR_API_KEY
```

- The key is validated, the session stored, and `api_key` is **immediately scrubbed** from the
  address bar and browser history. Other query params survive.
- Already logged in with the same key? The page loads directly, no redirect or flicker.
- Logged in as somebody else? The URL's key wins and the session switches.
- Invalid key? Any existing session is kept; otherwise you land on the page logged out.

There is also `/auto-login?api_key=...&redirect=/path` (relative paths only), but the universal
param above is simpler.

> **A URL containing `api_key` grants full account access to anyone who sees it.** Send autologin
> links over private channels only. Never post one publicly, never put one in a shared document,
> and never bake one into a public page's HTML.

## Universal URL params

These work on any page and compose with `?api_key=`.

| Param | Effect |
|---|---|
| `hide_navbar=true` | Removes the top navbar entirely (it occupies no space). Ideal for embeds. |
| `hide_logo=true` | Removes every brand mark: navbar logo and wordmark, footer brand block, share-page header, and the "Powered by" line. For whitelabel embeds. |
| `view_only_gigs=id1,id2` | On `/gigworker/mailboxes`, restricts to mailboxes in those gigs (unread counts and timelines scope too). On `/client/gigs`, restricts the gig list. A banner offers "Show all". |

Both `hide_*` params persist for the browser tab across in-app navigation; pass `=false` to undo.
The footer legal text and the Terms/Privacy links always stay.

```
# A worker's mailboxes for two gigs, chrome-free
https://dollarplatoon.com/gigworker/mailboxes?api_key=KEY&view_only_gigs=GIG_01AAA,GIG_01BBB&hide_navbar=true

# Fully whitelabel
https://dollarplatoon.com/gigworker/mailboxes?api_key=KEY&hide_navbar=true&hide_logo=true

# A whitelabel public submit page (no key needed — the share token is the credential)
https://dollarplatoon.com/submit/SHARE_TOKEN?hide_logo=true
```

## The Task page — one task, on a page of its own

`/claim/:gig_id/:task_id` and `/task/:gig_id/:task_id` are the same page. It shows one task, its
comments, and — when the task is claimable — an **Accept this task** button. The client copies the
link from **Share task…** in the task's detail pane in the dashboard.

```
# claimable: the first person to open it takes the task
https://dollarplatoon.com/claim/GIG_01HX.../TASK_01KV...

# with a gig invite, so somebody who has not joined can use it
https://dollarplatoon.com/claim/GIG_01HX.../TASK_01KV...?invite=abc123def456

# view-only: readable and commentable by every member, claimable by nobody
https://dollarplatoon.com/task/GIG_01HX.../TASK_01KW...
```

Unlike `/insert/` and `/submit/`, this page has **no per-task token**. Gig membership is the
credential, so a visitor who has not joined sees the gig's join link instead of the task, and a
signed-out visitor is sent to sign in and returned here afterwards. That makes the URL safe to
paste into a chat of workers who all belong to the gig: the first to open it wins, and everybody
else is told it is taken.

**`?invite=<token>` extends it to people who have not joined.** The token is a gig invite from
`POST /gigs/:id/invites`. The "you have not joined" panel then becomes "you are invited": joining
returns the reader to this task rather than to a mailbox list. Use an unlimited invite for a link
that goes to more than one person. The **Share task…** dialog picks one and turns it on by default,
and offers to mint an unlimited invite when the gig has none.

**A reserved task keeps the button, and nobody is racing you for it.** No poll offers a reserved
task, so the link is the only way in and it can sit unopened for as long as it takes. If the
reservation names one worker, everybody else who opens the link is told so plainly — "reserved
for another worker", not "somebody took it", because nobody did.

**A view-only task drops the button.** Nobody can claim it, so the page is the brief plus the
comment thread — which is how one link can go to a dozen people at once. See
[tasks.md](https://dollarplatoon.com/skill/tasks.md).

The claimable page is the pull half of `?assign_to=`. Use `assign_to` to push a task at a named
worker; use this link when you want a worker you chose to take it themselves. Every poll limit
still applies — see [queue.md](https://dollarplatoon.com/skill/queue.md) for
`POST /gigs/:id/queue/:msgId/claim` and its `reason` codes.

## Notifications — the bell in the navbar

`/notifications` lists what has happened on this account, newest first: comments on your tasks,
proofs waiting for review, verdicts on your work, and payouts. Each row opens in a new tab at the
exact task or proof.

```
https://dollarplatoon.com/notifications
```

The bell shows a **red dot**, never a count: the dot is on when something arrived after the last
time you opened the page **and** within the last 24 hours. Opening the page puts it out.

| Route | Auth | Description |
|---|---|---|
| `GET /notifications?cursor=` | Account | The list, newest first |
| `GET /notifications/summary` | Account | `{ dot, latest_at, latest_title, seen_at }` |
| `POST /notifications/seen` | Account | Mark everything seen — the dot goes out |

Notifications are written by the events themselves; nothing posts one directly. A comment reaches
the gig owner, and the client's reply reaches the people in that thread — never the whole gig, and
never anybody who is not allowed to read the comment. A **private reply** notifies exactly its one
addressee. Rows expire after 90 days.

## The Share Proof link — one proof, on its own review page

`/client/gig/:gig_id/proofs/:proof_id` opens a single proof on the client's review page, with the
approve and reject controls on it. The worker copies the link from the **Share Proof** button in
the proof detail pane of `/gigworker/mailboxes`.

```
https://dollarplatoon.com/client/gig/GIG_01HX.../proofs/PROOF_01HX...
```

**The gig owner is the only reader.** There is no token: the page needs a signed-in account that
owns the gig, so a leaked link shows nothing to anybody else. Add `?api_key=` only if the link
goes to the owner over a private channel.

Use it to chase one review — a proof sent by email, chat, or a support thread — instead of asking
the client to find the proof in the dashboard.

## The Insert Task page — put work in without an account

`/insert/:gig_id` is the client's Insert Task form as a standalone page. It posts to exactly where
the API does — the gig's inbound webhook — and needs **no login**: the gig security token in the
URL is the credential, precisely as it is for `POST /api/inbound/webhook/:gig_id?token=...`.

Use it to let a teammate, a partner, or your own tool insert tasks into one gig without an
account, or to embed a task box inside another product.

**The URL is deterministic — build it, never look it up:**

```
https://dollarplatoon.com/insert/{GIG_ID}?token={GIG_SECURITY_TOKEN}
```

Both values come from `GET /gigs/mine` or the dashboard: `id` and `security_token`. The rule is
one-to-one with the webhook — if `POST /api/inbound/webhook/GIG_X?token=T` works, then
`/insert/GIG_X?token=T` shows the form for it.

**Every optional param pre-fills a field, so a link can carry the shape of the work:**

| Param | Effect |
|---|---|
| `token` | The gig security token. Required when the gig has one; the form `403`s without it. |
| `body` | Pre-fills the task body. |
| `subject` | Pre-fills the subject. |
| `tags` | Comma-separated task tags, e.g. `tags=shortform,urgent`. Defaults to the gig's `default_task_tags`. |
| `price` | A number (`price=2.50`) or `price=tbd`. Defaults to the gig price. |
| `assign_to` | Mailbox id or the worker's account email. |
| `priority` | Queue position, lower polls sooner. Queue gigs only, never with `assign_to`. |
| `json=true` | Sends the body as JSON instead of text/HTML. |
| `hide_logo=true` | Whitelabel. |

```
https://dollarplatoon.com/insert/GIG_01HX...?token=abc123&tags=shortform,urgent&price=2.50&hide_logo=true
```

```html
<iframe src="https://dollarplatoon.com/insert/GIG_01HX...?token=abc123&hide_logo=true"
        width="100%" height="720" style="border:0"></iframe>
```

**Notes.**

- **The link is a credential.** Anyone holding it can insert tasks into that gig. To revoke, rotate
  the token (`POST /gigs/:id/rotate-token`) — every old link stops working at once, **including
  your publisher integrations**.
- **Private notes are hidden** on this page, because they are a separate owner-authenticated
  PATCH. Adding `&api_key=` reveals the field, but that key grants full account access — never do
  it on a link you share.
- **Assign to is free text** here (mailbox id or account email), because a page with no session
  cannot list the gig's mailboxes.
- A gig with **no** security token accepts the link without `token=`, which means anyone who knows
  the gig id can insert tasks. Generate a token first.

## The Submit page — hand in work without an account

`/submit/:token` lets a share-token holder submit a proof with no login. Covered in full,
including the `?task=` opt-in and the skip/report actions, in
[proofs.md](https://dollarplatoon.com/skill/proofs.md).

## Feed reader pages

| Page | URL |
|------|-----|
| Registry | `/feed/<feed_id>/registry` |
| Notifications | `/feed/<feed_id>/notifications` |
| Settings, members, invites (owner only) | `/client/feed/<feed_id>` |
| Accept an invite | `/feed/<feed_id>/join?invite=<token>` |

These render **inside the app**, with the navbar and footer, like every other signed-in page —
feeds are members-only, so there is no reason to strip the chrome by default. To embed one, drop
the chrome yourself with `?hide_navbar=true&hide_logo=true`.

**You do not need `?api_key=` if the browser is already signed in.** The session lives in the
browser, so a member who is logged in just opens the URL.

It is needed when the context has no session of its own, and the common case is a **cross-site
iframe**: browsers partition storage per embedding site, so an embed on your domain cannot see a
login made on dollarplatoon.com. Add `?api_key=` there. `hide_navbar` hides chrome; it does not
authenticate.

The join page is the exception either way — it signs somebody in itself, because an invite link
gets opened cold.

```
https://dollarplatoon.com/feed/FEED_01HX.../registry?api_key=KEY&hide_navbar=true&hide_logo=true
```

## Timeline pages

Standalone activity heatmaps — contribution-graph grids of daily tasks and proofs. All compose
with `?api_key=` and `?hide_navbar=true`.

| Page | What it shows |
|---|---|
| `/gig/:id/timeline-grid` | One gig; the owner sees every mailbox's activity. |
| `/mailbox/:id/timeline-grid?gig=GIG_ID` | One mailbox. The `gig` param is **required** here. |
| `/client/timelines` | One aggregate card per gig you own, each linking to that gig's grid. |
| `/gigworker/timelines` | Your timelines across all your active mailboxes. |

Scoping:

- `/client/timelines?gigs=GIG_01AAA,GIG_01BBB` — an **ad-hoc grouping**. There is no stored
  gig-group concept; the URL *is* the grouping. Gigs you cannot access are silently skipped. No
  param means every gig you own.
- `/gigworker/timelines?gigs=...` and/or `?mailboxes=...` — when both are given, a mailbox shows
  if it matches **either** list. Note the param is `gigs=`, not `view_only_gigs=`, which applies
  only to `/gigworker/mailboxes` and `/client/gigs`.

Shared params:

- `date=YYYY-MM-DD` — which day's stats panel to show. Default today.
- `spectrum=0,1,5,10,30` — colour-scale thresholds (2–5 ascending non-negative integers).
  Overrides the gig owner's saved `timeline_spectrum`. On the two `-grid` pages only.

```
https://dollarplatoon.com/client/timelines?api_key=KEY&gigs=GIG_01AAA,GIG_01BBB&hide_navbar=true
```

The underlying API is
`GET /gigs/:id/timeline?days=186&tz_offset=420&mailbox_id=...&per_mailbox=1` (auth required;
owners get every mailbox, workers get their own).

## Iframe rules that actually matter

Browsers block clipboard and popups inside a frame unless the host page opts in. The submit page
offers **Copy message**, **Copy link**, **Open in new tab**, and a copy button per attachment, so
set the attributes yourself:

```html
<iframe
  src="https://dollarplatoon.com/submit/SHARE_TOKEN?task=TASK_01KXQ..."
  allow="clipboard-write"
  sandbox="allow-scripts allow-same-origin allow-forms allow-popups allow-popups-to-escape-sandbox"
></iframe>
```

- `allow="clipboard-write"` — without it the permission policy denies `navigator.clipboard`. The
  page falls back to a legacy copy, and failing that shows the text selected for Ctrl+C. Copy
  never fails silently.
- `allow-popups allow-popups-to-escape-sandbox` — needed **only if you set `sandbox` at all**.
  Without them the browser blocks every new tab, including links inside the task body.
- `allow-scripts allow-same-origin allow-forms` — needed for the app to run, upload files, and
  post the proof.
- **If you do not need `sandbox`, do not add it.** A plain `<iframe src=...>` already permits new
  tabs.

The app also avoids browser modal dialogs everywhere, because a frame suppresses them — confirms
and prompts are rendered in-page instead.

## Page map

| Path | Who it is for |
|---|---|
| `/client/gigs` | Client — your gigs. Create Gig sits at the top right. |
| `/client/gig/:id/dashboard` | Client — one gig: tasks, proofs, mailboxes, payouts. |
| `/client/feeds` | Client — your feeds. |
| `/client/feed/:id` | Feed owner — one feed: details, members and their scopes, invite links. |
| `/gigworker/mailboxes` | Worker — your mailboxes and their tasks. |
| `/gigworker/feeds` | Worker — feeds you have joined. |
| `/gigworker/earnings` | Worker — what you have been paid. |
| `/gig/:id/join?invite=` | Anyone — accept a gig invite. |
| `/feed/:id/join?invite=` | Anyone — accept a feed invite. |
| `/submit/:token` | Share-token holder — hand in work, no account. |
| `/insert/:gig_id?token=` | Token holder — put work in, no account. |
| `/claim/:gig_id/:task_id?invite=` | Joined worker — accept one exact task. `?invite=` lets a stranger join on the way in. |
| `/task/:gig_id/:task_id` | Any member — read and comment on one task. The same page; the name that fits a view-only task. |
| `/notifications` | Any account — comments, proofs, verdicts and payouts, newest first. |
| `/client/gig/:id/proofs/:proof_id` | Gig owner — review one proof on its own ("Share Proof"). |
