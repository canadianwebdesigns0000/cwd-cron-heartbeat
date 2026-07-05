# cwd-cron-heartbeat

External heartbeat for **team.canadianwebdesigns.ca** scheduled jobs.

## Why this exists

The portal's scheduled jobs (renewal invoices, QBO paid-sync, PnL, SEO reports, etc.)
are supposed to run on **Vercel cron**. Vercel's scheduler stopped firing them —
`lastRunAt: never` on every cron, despite the app, endpoints, plan, and auth all being
healthy. It's a platform-level scheduler stall.

The app is fine. The **trigger** is broken. This repo is a replacement trigger: a
GitHub Actions workflow that, on each schedule, `curl`s the matching portal endpoint
with the `x-vercel-cron: 1` header (the same header Vercel's own crons send, which the
portal middleware accepts). Nothing in the app changes — this just pokes it on time.

Endpoints are idempotent (Mongo watermarks + per-step dedup + processed-payment guards),
so an early or duplicate poke is safe.

## Schedules (mirror of the portal's `vercel.json`, all UTC)

| Endpoint | Schedule | Purpose |
|---|---|---|
| `/api/cron/leads-digest` | `0 3 * * *` | leads digest email |
| `/api/clients/renewal-alerts` | `0 9 * * 1` | weekly renewal alerts (Mon) |
| `/api/cron/ads-daily-report` | `0 10 * * *` | Google Ads report |
| `/api/cron/cwd-seo-report` | `0 11 * * *` | CWD SEO report |
| `/api/cron/tvtc-seo` | `0 12 * * *` | TVTC SEO automation |
| `/api/cron/pnl` | `0 13 * * *` | PnL calc + email |
| `/api/cron/qbo-paid-sync` | `0 13 * * *` | paid invoice → client services |
| `/api/cron/renewal-invoices` | `0 14 * * *` | create & send renewal invoices |
| `/api/cron/cloud-pharmacy-invoice-guard` | `0 15 * * *` | CP invoice safety net |
| `/api/cron/cloud-pharmacy-invoice` | `0 13 1 * *` | monthly CP invoice |
| `/api/cron/qbo-payment-discord` | `*/5 * * * *` | payment → Discord (Vercel was `*/3`; GitHub min is 5m) |

## Manual test

Actions tab → **CWD Cron Heartbeat** → **Run workflow**. Optional `target` input =
exact path to hit, e.g. `/api/cron/renewal-invoices?dry=true`. Blank target = safe
dry connectivity check.

## Caveats

- **GitHub scheduled runs can be delayed** 5–20 min under load (occasionally skipped).
  Fine for these daily/idempotent jobs; not millisecond-precise.
- **`qbo-payment-discord`** was `*/3` on Vercel; GitHub's minimum is `*/5` and it's the
  least reliable of the set at high frequency. If near-real-time matters, host that one
  on a real 1-minute cron (a VPS/cPanel box).
- **60-day inactivity:** GitHub disables scheduled workflows after 60 days with no
  repo commits. Push any trivial commit every ~2 months to keep it alive.
