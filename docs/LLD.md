# FarePerch — Low-Level Design

## Architecture

```
Browser (supabase-js CDN, magic link)
   |
   |  HTTPS
   v
Cloudflare Worker (single worker, TypeScript ESM)
   |-- Workers Assets: serves public/ (index.html, app.js, styles.css)
   |-- /api/watches            -> CRUD on the user's watched routes
   |-- /api/watches/:id/history -> fare snapshot + alert history for one watch
   |-- /api/me
   |-- cron: poll-fares (hourly — picks watches due per their plan-derived cadence)
   |
   |  plain fetch (no supabase-js on the Worker)
   v
Supabase (GoTrue, PostgREST, RPC spend_credit/refund_credit)

   |  Amadeus Self-Service Flight Offers Search API
   v
Amadeus for Developers (Test environment quota)

   |  Resend (transactional email)
   v
Alert delivery
```

Poll flow: the hourly cron selects `watches` whose `next_poll_at <= now`, calls Amadeus Flight Offers Search for each, writes a `fare_snapshots` row, recomputes the rolling baseline for that route+cabin, and — if the drop clears the alert threshold and the watch isn't in its post-alert cooldown — inserts an `alerts` row and emails the user via Resend. There is **no LLM anywhere in this path** — see LLM strategy below.

## Data model

```sql
create table watches (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id),
  origin_iata text not null,
  dest_iata text not null,
  depart_date date not null,          -- v1: a specific date, not a flexible range
  cabin_class text not null default 'ECONOMY',
  active boolean not null default true,
  poll_interval_hours int not null,   -- set from the user's plan at creation time (24 free / 8 paid)
  next_poll_at timestamptz not null default now(),
  created_at timestamptz not null default now()
);
alter table watches enable row level security;
create policy watches_own on watches for select using (auth.uid() = user_id);

create table fare_snapshots (
  id uuid primary key default gen_random_uuid(),
  watch_id uuid not null references watches(id) on delete cascade,
  price_rs numeric not null,
  currency text not null default 'INR',
  source text not null default 'amadeus_test',
  fetched_at timestamptz not null default now()
);
alter table fare_snapshots enable row level security;
create policy fare_snapshots_via_watch on fare_snapshots for select using (
  exists (select 1 from watches w where w.id = watch_id and w.user_id = auth.uid())
);

create table alerts (
  id uuid primary key default gen_random_uuid(),
  watch_id uuid not null references watches(id) on delete cascade,
  user_id uuid not null,              -- denormalized for a simple RLS predicate
  price_rs numeric not null,
  baseline_avg_rs numeric not null,
  drop_pct numeric not null,
  guidance text not null check (guidance in ('book_now', 'wait')),
  sent_at timestamptz not null default now()
);
alter table alerts enable row level security;
create policy alerts_own on alerts for select using (auth.uid() = user_id);
```

No complex state machine — a `watch` is simply `active`/inactive, and `next_poll_at` advances by `poll_interval_hours` after each successful poll (whether or not it produced an alert).

## API routes

| Route | Method | Auth | Behavior | Failure modes |
|---|---|---|---|---|
| `/api/watches` | GET/POST | JWT | List/create watches. First watch spends the user's 1 free credit; a second or later watch requires `plan='active'` | 402 on second watch without subscription, 400 invalid IATA code |
| `/api/watches/:id` | DELETE | JWT | Deactivate a watch | 404 not owned |
| `/api/watches/:id/history` | GET | JWT | Fare snapshots + alerts for one watch | 404 not owned |
| `/api/me` | GET | JWT | Credits, plan, watch count, this-month's poll count (for operator quota visibility) | 401 unauthenticated |

Cron: `poll-fares` (hourly) — see Architecture above.

## LLM strategy

**N/A — the book-now/wait call is an explicit heuristic, not an LLM call.** Guidance is computed from a rolling price baseline: `baseline_avg_rs` is the mean of the last 10 snapshots for that route+cabin; an alert fires when the current price is at least 10% below that baseline and the watch isn't within a 24-hour post-alert cooldown (avoids re-alerting on noise around the same price). Guidance is `"book_now"` when the current price is within 5% of the lowest price ever observed for that route+cabin, else `"wait"` (the drop is real but history suggests further movement is plausible). This is a deliberate divergence from the reference architecture's "LLM layer when the product needs one" — a threshold-and-rolling-average rule is exactly reproducible, needs no API cost, and is more trustworthy for a "should I spend money now" decision than an LLM's prose guess. The heuristic thresholds (10% alert, 5% book-now, 24h cooldown, 10-snapshot window) are versioned config, not hardcoded, so they can be tuned without a redeploy — and the UI must always label this guidance "heuristic", never "prediction" or "forecast".

## Frontend pages

- `/` — landing + pricing (Rs199/mo, 1 free watch).
- `/app` — watch list, add a watch (origin, destination, date, cabin).
- `/app/watches/:id` — price history chart, past alerts, current guidance if an alert is active.
- `/account` — credits/plan, subscription upgrade instructions.

## Error handling and credits/subscription flow

Watch creation spends 1 credit for the first watch only (free trial primitive); a second watch requires `plan='active'`, checked before insert, no spend/refund cycle needed since creation is synchronous and either succeeds or is rejected up front. Poll failures (Amadeus API error, quota exceeded, malformed response) are logged per-watch and simply retried on the next scheduled poll — a missed poll degrades freshness, not correctness, so there is no credit to refund and no user-facing error beyond a "last checked" timestamp going stale. **Monetization deviates from the reference's per-op credit model** for the same reason as the other subscription products in this batch: Rs199/mo is recurring access, not a per-call charge, so `profiles.plan`/`subscription_expires_at` (manual UPI top-up day 1, operator sets `plan='active'`) gates additional watches and faster polling, while the credits RPCs remain only the free-trial unlock for watch #1.

## Integrations and launch gates

**Amadeus Self-Service is the hard constraint, and it is a quota problem, not an approval-wait problem** — the Test environment activates immediately on signup (no approval delay like WhatsApp), but it comes with two limits that must be designed around from day one:

1. **Cached, not live, data.** Test-environment fare data is not guaranteed real-time — the UI and email alerts must say "fares as of last check, may not reflect live pricing" rather than implying a live quote.
2. **A fixed monthly call quota shared across the whole application, not per user.** Flight Offers Search allows 2,000 free calls/month total. Polling budget arithmetic: a free-tier watch at 24-hour polling costs 30 calls/user/month; a paid-tier watch at 8-hour polling costs ~90 calls/user/month. At 2,000 calls/month, that ceiling is roughly **66 free-tier-only users, or as few as ~22 users if most are on a single paid watch each** (fewer still once a paid user has multiple watches). This means the product will need a paid/Production Amadeus tier well before it has a meaningful number of subscribers — budget for that upgrade cost explicitly in the pricing model, and have the operator monitor Amadeus's usage dashboard (`My Self-Service Workspace > API usage and quota`) rather than discovering the cap via a `429` in production. Do not raise default polling frequency beyond the 24h/8h defaults above without re-running this arithmetic.

No other integration is required for v1 — no OAuth, no WhatsApp. Alert delivery is transactional email via Resend, chosen specifically because it needs no approval process (unlike the WhatsApp-first products elsewhere in this batch).

## Security notes

- RLS scopes `watches` and `alerts` directly by `user_id`; `fare_snapshots` is scoped via a join back to the owning `watches` row since it has no user column of its own.
- Amadeus API credentials and the Resend API key are secrets (`wrangler secret put`), never exposed to the client.
- IATA codes and dates from user input are validated server-side (3-letter code format, date not in the past) before ever reaching the Amadeus call.

## Out of scope for v1

- Flexible date-range watches (v1 is one specific depart date per watch) or round-trip/multi-city.
- Any scraping of airline or OTA sites — this is a permanent boundary, not a v1 gap, per the licensed-API caveat.
- Moving to Amadeus Production tier — v1 ships on Test tier and documents the quota ceiling; the Production upgrade is a follow-up decision once the user base approaches the arithmetic above.
- Push notifications or WhatsApp alerts — v1 is email only.
- Razorpay-automated billing — v1 is manual UPI + admin-set `plan`.
