# FarePerch
Watch flight routes, get fare-drop alerts with book-now/wait guidance.

**Status: planned — not yet built (50-SaaS challenge #56)**

## The problem
Fares on a route a traveler cares about drift constantly, and checking manually every day is tedious. Fare-alert tools exist, but most either scrape airline sites (fragile, legally shaky) or only tell you a price changed without any sense of whether it's worth booking now or waiting.

## Target buyer
Indian frequent travelers (leisure and work) who fly a handful of known routes repeatedly and want a nudge when the price on a route they're watching genuinely drops, not just any fluctuation.

## Pricing hypothesis
Rs199/month subscription. Free tier: 1 watched route polled at a slower cadence; paid unlocks more watches and more frequent polling.

## Stack
- Cloudflare Worker (TypeScript, Workers Assets) serving the frontend and `/api/*`, plus a cron trigger for fare polling.
- Supabase: magic-link auth, Postgres + RLS, fare snapshot history per watch.
- Amadeus for Developers Self-Service Flight Offers Search API for fare data — no LLM in the alert path; book-now/wait guidance is an explicit rolling-average-and-threshold heuristic.

## How to continue this build
Read `docs/LLD.md` for architecture, data model, the Amadeus quota economics, and the heuristic design, then `docs/PLAN.md` for the ordered TDD task list. `CLAUDE.md` points back to the reference implementation.

## Risks / constraints
- **Fare data comes from a licensed API (Amadeus Self-Service), not scraping.** This is a deliberate, non-negotiable choice — scraping airline or OTA sites for fares is both fragile and against most carriers' terms of use.
- **Amadeus Self-Service starts in the Test environment**: a fixed, low monthly call quota per API (Flight Offers Search is 2,000 free calls/month) and cached, not guaranteed real-time, fare data. Production access requires a contracted, billed upgrade. Polling frequency and the number of watches per user must be budgeted against this quota — see `docs/LLD.md` for the arithmetic — or the product silently runs out of API calls before the month ends.
- **Book-now/wait guidance is an explicit heuristic (a rolling-average-and-threshold rule), not a prediction model**, and must always be labeled as such in the UI — never presented as a forecast or a guarantee.
