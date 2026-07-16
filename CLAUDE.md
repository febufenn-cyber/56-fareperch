# CLAUDE.md — FarePerch (50-SaaS #56)

This repo is product #56 of Febin's 50-SaaS challenge. Nothing is built yet — `docs/` is the source of truth.

Read `docs/LLD.md` then `docs/PLAN.md` and execute tasks in order with TDD.

Stack conventions and the shipped reference implementation live in the private repo `febufenn-cyber/50-saas` (`contract-reviewer/`) — same Worker + Supabase + credits patterns, adapted here to a no-LLM heuristic and a subscription gate.

The no-LLM/heuristic-only design, the Amadeus Test-tier quota arithmetic (2,000 calls/month, shared across all users, not per user), the licensed-API-not-scraping constraint, and the subscription-over-credits monetization deviation in the LLD are verified decisions — do not re-litigate without evidence.
