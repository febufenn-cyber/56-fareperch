# FarePerch — Build Plan

TDD throughout: write the test first, watch it fail for the right reason, then implement. Each task is sized for one focused agent session.

## T1 — Supabase schema and RLS
Files: `supabase/migrations/0001_init.sql`.
Interfaces: none (schema only).
Test first: RLS check — a second user's JWT cannot `select` another user's `watches`/`alerts` rows, and cannot read `fare_snapshots` for a watch they don't own via the join-based policy.
Done when: migration applies cleanly; both RLS checks pass with two test JWTs.

## T2 — Worker skeleton and static shell
Files: `wrangler.jsonc` (hourly `poll-fares` cron), `src/worker.ts`, `src/router.ts`, `public/index.html`, `public/config.js` route.
Interfaces: `router(req: Request, env: Env): Promise<Response>`; `scheduled(event, env)` export for the poll-fares cron.
Test first: `test/router.test.ts` — unknown route 404, `/api/me` without auth 401.
Done when: `wrangler dev` serves the shell and stub routes locally.

## T3 — Supa client
Files: `src/supa.ts`.
Interfaces: `class Supa { getUser(jwt); listWatches(userId); insertWatch(row); deleteWatch(id, userId); getDueWatches(now: Date): Promise<WatchRow[]>; insertFareSnapshot(row); getRecentSnapshots(watchId, limit): Promise<FareSnapshotRow[]>; insertAlert(row); getLastAlert(watchId): Promise<AlertRow|null>; advanceNextPoll(watchId, next: Date); spendCredit(userId); getProfile(userId); }` — injectable `fetchFn`, mirrors the reference `Supa` shape.
Test first: `test/supa.test.ts` with mocked fetch asserting URLs/headers per method.
Done when: all methods pass against mocked fetch.

## T4 — Amadeus client module
Files: `src/amadeus.ts`.
Interfaces: `getFareQuote(opts: {origin: string, dest: string, departDate: string, cabinClass: string, env: AmadeusEnv, fetchFn?: typeof fetch}): Promise<{priceRs: number, currency: string}>` — handles the OAuth2 client-credentials token fetch (cached until near expiry) then the Flight Offers Search call.
Test first: a mocked token endpoint + search endpoint returns a parsed price; a `429` (quota exceeded) response raises a typed `AmadeusQuotaError` distinct from other API errors, so the poll cron can log it without retrying immediately.
Done when: token caching (no re-fetch within the token's validity window) and the quota-error distinction both pass.

## T5 — Watch CRUD handlers with the credit/plan gate
Files: `src/handlers.ts` (partial), `src/validate.ts`.
Interfaces: `validateWatchInput(body): {ok: true, watch: WatchInput} | {ok: false, reason: string}` — IATA code format, depart date not in the past.
Test first: creating a second watch without `plan='active'` returns 402; an invalid IATA code (not 3 letters) is rejected before any credit spend; `poll_interval_hours` is set to 24 on the free tier and 8 on the paid tier at creation time.
Done when: all three cases pass.

## T6 — Heuristic module: rolling baseline, alert threshold, book-now/wait guidance
Files: `src/heuristic.ts`, `config/fare-thresholds.json`.
Interfaces: `evaluateFare(current: {priceRs: number, fetchedAt: Date}, recentSnapshots: FareSnapshotRow[], lastAlert: AlertRow|null, thresholds: FareThresholds): {shouldAlert: boolean, guidance?: 'book_now'|'wait', baselineAvgRs?: number, dropPct?: number}` — `thresholds` loaded from the versioned config file, not hardcoded.
Test first: a price 15% below the 10-snapshot rolling average with no prior alert triggers `shouldAlert: true`; the same drop within 24 hours of a previous alert on the same watch does not re-fire; a price within 5% of the historical minimum yields `guidance: "book_now"`, a real-but-smaller drop yields `"wait"`.
Done when: all threshold and cooldown boundary cases pass — this is the core business logic and needs the widest test coverage in the repo.

## T7 — Poll-fares cron: dispatch, snapshot, alert
Files: `src/cron/poll-fares.ts`.
Interfaces: `pollDueWatches(supa: Pick<Supa, 'getDueWatches'|'insertFareSnapshot'|'getRecentSnapshots'|'getLastAlert'|'insertAlert'|'advanceNextPoll'>, quoteFn: typeof getFareQuote, evaluateFn: typeof evaluateFare, now: Date): Promise<{polled: number, alerted: number, quotaErrors: number}>`.
Test first: a due watch gets a snapshot and its `next_poll_at` advances by its `poll_interval_hours` even when the poll doesn't trigger an alert; an `AmadeusQuotaError` on one watch doesn't stop the cron from processing the remaining due watches.
Done when: both cases pass against mocked Amadeus and Supa.

## T8 — Alert email via Resend
Files: `src/email.ts`.
Interfaces: `sendFareAlertEmail(opts: {to: string, watch: WatchRow, alert: AlertRow, env: {RESEND_API_KEY: string}, fetchFn?: typeof fetch}): Promise<void>` — email body states "fares as of last check" and labels guidance "heuristic".
Test first: the composed email body contains the not-live-pricing disclaimer and never uses the word "prediction" or "forecast".
Done when: the disclaimer/wording test passes against a mocked Resend call.

## T9 — Frontend: watch management and price history
Files: `public/app.js`, `public/watch.js`, `public/styles.css`.
Interfaces: none new (calls `/api/watches*` from T5-T7).
Test first: none (manual QA).
Done when: a user can add a watch, see its price history and any past alerts, and the guidance badge always reads "heuristic guidance", never a percentage confidence.

## T10 — Deploy, live smoke test, launch checklist
Files: `scripts/smoke.ts`, deployment config.
Interfaces: `smoke.ts` runs against the deployed URL: creates a watch, manually triggers one poll cycle (via a test-only endpoint or by invoking the cron handler directly), asserts a `fare_snapshots` row was written.
Done when: `wrangler deploy` succeeds; `scripts/smoke.ts` passes against production; pricing page shows Rs199/mo + 1 free watch; secrets (`AMADEUS_CLIENT_ID`, `AMADEUS_CLIENT_SECRET`, `RESEND_API_KEY`, `SUPABASE_SERVICE_ROLE_KEY`) set via `wrangler secret put`; the Amadeus usage/quota dashboard is bookmarked for the operator and the polling-budget arithmetic from the LLD is re-checked against actual signup numbers before raising any default polling frequency.
