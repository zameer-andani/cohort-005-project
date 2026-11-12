# Which third-party service should back Notifications?

Research for [#2](https://github.com/zameer-andani/cohort-005-project/issues/2), part of the map [#1](https://github.com/zameer-andani/cohort-005-project/issues/1). Researched 2026-11-11. Prices and limits change, so check the cited pages again before signing up.

## Summary

- **Shortlist:** **Knock** first, **Novu** second.
- **Recommendation: Knock.** It is the only candidate whose free plan covers everything both kinds of Notification need. That means an Inbox feed, batching on every plan (for Comment Alert digests), delays of up to 365 days and a cancellation key (for Nudges), per-workflow and per-channel preferences, a commercial one-click unsubscribe, and Postmark as the email channel. Its React SDK accepts React 19.
- **Novu** is the fallback. It is open source, its `<Inbox />` works with React 19, and it supports Postmark. But on the free plan, delays and digest windows are capped at 1 day. That cap pushes Nudge timing into our own cron, and self-hosting needs MongoDB and Redis.
- **Ruled out:** Courier (digests and batching are listed only under Enterprise) and MagicBell (1,000 free deliveries, no documented digest step, Postmark setup unclear).
- **Owning the Inbox in SQLite + Postmark** is the cheapest option and works offline. But the map says Notifications must be backed by a real third-party service, and we would have to build batching, delay and cancel, preferences and unsubscribe ourselves.

## Comparison

| | Knock | Novu (Cloud) | Courier | MagicBell | Own SQLite + Postmark |
|---|---|---|---|---|---|
| React Inbox component | `@knocklabs/react` feed components [1][2] | `<Inbox />`, `<Preferences />`, headless hooks [9] | `@trycourier/courier-react` (Courier Inbox) [14] | `@magicbell/magicbell-react`, react-headless [17][18] | Build it |
| React 19 (peer deps) | `react ^17 \|\| ^18 \|\| ^19` (v0.15.1) [2] | `react ^18 \|\| ^19` (v3.19.2) [2] | `react >=18` (v9.3.5) [2] | `react >=18.3.1` (v11.5.1) [2] | n/a |
| React Router 8 | Framework-agnostic components, React Router not a peer dep [2] | Framework-agnostic [9] | Framework-agnostic [2] | Framework-agnostic [2] | n/a |
| Digest / batching | Batch function on all plans, fixed or sliding windows, up to 90 days [1][4] | Digest step, regular or scheduled; max window 1 day Free, 7 days Pro, 30 days Team [11] | Digests listed only under Enterprise [13][15] | No documented digest step [19] | Build it |
| Delays (Nudges) | Delay up to 365 days, fixed, dynamic or relative; cancel through a `cancellation_key` [5] | Fixed, scheduled or dynamic delay; max 1 day Free, 7 days Pro, 30 days Team [10] | Delay and cancel steps in Automations (marked Legacy) and Journeys [16] | `wait`, `pause` and `abort` workflow steps [19] | Own cron/job |
| Preferences | Per workflow, category, channel type or channel; hosted or custom preference center [6] | Per workflow and channel, in `<Preferences />` [9] | User Preferences product [13] | Not confirmed [17] | Build it |
| Email unsubscribe | Commercial 1-click unsubscribe link that keeps transactional mail [6][7] | Not checked | Not checked | Not checked | Build it (incl. `List-Unsubscribe`) |
| Postmark as email provider | Yes: API key, from address, optional webhook, sandbox toggle [3] | Yes: server token, from address [12] | Yes, MessageStream override [15] | Marketing page names Postmark; only SMTP is documented [19] | Already in `app/lib/mailer.server.ts` |
| Free tier | 10,000 messages/mo, no hard cap [1] | 10,000 workflow runs/mo, 5 workflows, 24h activity retention [8] | 10,000 sends/mo [13] | 1,000 deliveries/mo [17] | Postmark's own pricing |
| Next tier | Starter $250/mo with 50k messages, then $0.005 each [1] | Pro from $30/mo, 30k runs, $1.20 per 1k [8] | $0.005 per send from the first one [13] | Startup $249/mo with 50k deliveries [17] | — |
| Self-hosting | No | Community edition exists; needs MongoDB, Redis and object storage; the pricing page lists self-hosting under Enterprise [8][20] | No | No | Yes (it is the app) |
| Local dev and tests without a key | No offline mode. There are dev/prod environments, `sandbox_mode` and `skip_delay` on trigger, and a test runner [21] | No offline mode without self-hosting; dev environment on Free [8] | Test environments count as sends [13] | Not checked | Works fully, like the `.outbox/` fallback |

## Notes by vendor

### Knock (recommended)

- **Nudges:** there are two workable designs. In the first, a Student's activity (re)triggers a Nudge workflow that has a delay step and a per-Student `cancellation_key`, and each new activity cancels the pending run before re-triggering [5]. In the second, our own cron spots inactivity and triggers a workflow with no delay. Either fits inside the 365-day delay limit [5]. This settles the map's "Nudge scheduling" question: the vendor can do it, so we don't need our own job.
- **Comment Alerts:** a batch step keyed per Instructor, with an optional batch key such as the course or lesson, collapses a burst of comments into one message ("8 new comments"). `total_activities` and up to 10 rendered `activities` are available to the template [4]. Batching is not a fixed-time digest. Knock suggests driving recurring digests at fixed times from our own system [4].
- **Postmark** is a first-class email channel, configured per environment, with a sandbox-mode toggle [3]. Our sending domain and reputation stay where they are now.
- **Unsubscribe:** the commercial unsubscribe sets `commercial_subscribed=false` and leaves transactional messages flowing [6][7]. That maps well onto Nudges (re-engagement) versus Comment Alerts (transactional). I did not confirm whether Knock sets the `List-Unsubscribe` header itself. Check this before launch.
- **Cost:** 10,000 messages a month free, and one message is one delivery to one user on one channel [1]. A Nudge sent to the Inbox and to email counts as 2. The jump to the next tier is steep: $250/mo [1].
- **Local dev and tests:** there is no offline mode [21]. Put Knock behind an interface, in the same way `MailTransport` hides Postmark, with a fake that writes to a local folder or an in-memory list when `KNOCK_API_KEY` is unset. Use a Knock development environment with `sandbox_mode` for staging [21].

### Novu (runner-up)

- Open source, with a good `<Inbox />` that has tabs, snooze, preferences and headless hooks [9]. Postmark is supported [12].
- The free plan caps delays and digest windows at **1 day** (Pro 7 days, Team 30 days) [10][11]. Nudges about multi-day inactivity would need our own cron on Free, or Team ($250/mo) for a 30-day window [8].
- The pricing page lists self-hosting as an Enterprise feature [8]. The self-hosting docs describe a community edition that needs MongoDB, Redis and S3-compatible storage [20]. That is a lot of infrastructure next to a single SQLite file ([ADR 0006](../adr/0006-one-sqlite-file-through-drizzle.md)).
- The cheapest paid step-up of the shortlist: Pro from $30/mo [8].

### Courier (ruled out)

- It has an Inbox, preferences, Postmark support and 10k free sends [13][15]. But the pricing page lists "digest and batching" under Enterprise [13], and Comment Alerts need batching. Its Automations product is labelled Legacy in favour of Journeys [16], so the platform is shifting under us.

### MagicBell (ruled out)

- Its free tier is the smallest (1,000 deliveries) [17]. It has no documented digest step [19], and its email channel docs list SES, Mailgun, SMTP, SendGrid and Ping, without a dedicated Postmark integration [18][19].

### Own SQLite + Postmark (baseline)

- It needs no new vendor, works fully offline, and keeps all data in our one SQLite file. But we would build the Inbox UI, unread state, batching, delayed sends with cancellation, preferences and a legally sound unsubscribe ourselves. It also doesn't meet the map's constraint that Notifications are backed by a real third-party service.

## Open questions for later tickets

- Does Knock add `List-Unsubscribe` / `List-Unsubscribe-Post` headers to Postmark email, or do we pass them as overrides? [3][7]
- Data model: which fields stay in SQLite (for example, a Notification log for tests) versus only in Knock?
- Can the volume of Nudges and Comment Alerts stay under 10,000 messages a month, counted per channel? [1]

## Sources

1. Knock pricing — https://knock.app/pricing
2. npm registry (`npm view <pkg> version peerDependencies`, run 2026-11-11): `@knocklabs/react`, `@novu/react`, `@trycourier/courier-react`, `@magicbell/magicbell-react`, `@magicbell/react-headless`
3. Knock — Postmark integration — https://docs.knock.app/integrations/email/postmark
4. Knock — Batch function — https://docs.knock.app/designing-workflows/batch-function
5. Knock — Delay function — https://docs.knock.app/designing-workflows/delay-function
6. Knock — Preferences overview — https://docs.knock.app/preferences/overview
7. Knock — Commercial unsubscribe — https://docs.knock.app/preferences/commercial-unsubscribe
8. Novu pricing — https://novu.co/pricing
9. Novu — Inbox overview — https://docs.novu.co/platform/inbox/overview
10. Novu — Delay step — https://docs.novu.co/platform/workflow/add-and-configure-steps/configure-action-steps/delay
11. Novu — Digest step — https://docs.novu.co/platform/workflow/add-and-configure-steps/configure-action-steps/digest
12. Novu — Postmark integration — https://docs.novu.co/platform/integrations/email/postmark
13. Courier pricing — https://www.courier.com/pricing
14. Courier — Digests — https://www.courier.com/docs/platform/automations/digest
15. Courier — Postmark integration — https://www.courier.com/docs/external-integrations/email/postmark
16. Courier — Automation steps (delay, cancel) — https://www.courier.com/docs/platform/automations/steps ; Automations (Legacy) — https://www.courier.com/docs/platform/automations/automations-overview
17. MagicBell pricing — https://www.magicbell.com/pricing
18. MagicBell docs index — https://www.magicbell.com/docs
19. MagicBell — Workflows — https://www.magicbell.com/workflows ; SMTP channel — https://www.magicbell.com/docs/channels/email/smtp ; Email — https://www.magicbell.com/email
20. Novu — Self-hosting overview — https://docs.novu.co/community/self-hosting-novu/overview
21. Knock — Testing workflows — https://docs.knock.app/send-notifications/testing-workflows ; Environments — https://docs.knock.app/concepts/environments
