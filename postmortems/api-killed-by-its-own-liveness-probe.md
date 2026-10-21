# The API that its own liveness probe killed

**Date:** 16 October 2026 · **Impact:** API terminated mid-session (exit 137), dashboard dead

## Summary

The API container was killed by the kubelet during a trading session. The log held
thousands of `No free DB connection within 30s` errors, which pointed straight at the
database — and was the wrong place to look. Those errors are a *consequence* of the
SIGTERM, not its cause. The real chain started in the browser.

## The chain

1. The frontend opens a burst of ~30–40 `GET /api/bars/{symbol}` in a single tick, one
   per ticker in the watchlist.
2. That handler is **synchronous** (`def` with a `Depends(get_db_connection)`), so each
   request occupies a thread from the pool and then blocks on a semaphore with
   `DB_POOL_MAX` slots for up to `DB_POOL_ACQUIRE_TIMEOUT_S`.
3. With the defaults in the code — 10 slots, 30 seconds — dozens of threads sit blocked,
   and GIL contention slows the event loop itself.
4. `/api/status` is `async` and **does not touch the database at all**, but it now misses
   the 5-second readiness and 10-second liveness deadlines. The kubelet kills the
   container.
5. Everything in flight is torn down, and *that* is what writes the flood of connection
   errors into the log.

## Why the log misleads

Reading the log bottom-up gives you a database problem. It is not one: PostgreSQL allows
100 connections and the whole stack was using about five. The bottleneck is the
application's own pool, and the trigger is client behaviour.

The dangerous part is what the misreading suggests you do. A slow `/api/status` tempts
you to relax the probe timeouts, which hides the symptom and leaves a saturated API
serving a dashboard that mostly hangs.

## Resolution

Two changes, one on each side:

- **Server:** `DB_POOL_MAX` 10 → 25, `DB_POOL_ACQUIRE_TIMEOUT_S` 30 → 5. A short acquire
  timeout matters as much as the larger pool: it converts a thread parked for half a
  minute into a fast, visible error.
- **Client:** the fallback path in the batch loader was firing `Promise.all` over *every*
  symbol whenever the grouped request failed. It now walks the same fixed-size groups
  with the same 500 ms spacing as the happy path.

Verified by reproducing the burst: 60 concurrent requests all return 200 in ≤0.78 s, and
`/api/status` stays at 77 ms.

## What I changed afterwards

- **A health endpoint that doesn't touch the database can still be starved by it** —
  through the thread pool and the GIL. "The probe doesn't query Postgres" is not an
  argument that the probe is safe.
- **Read a log flood for its first line, not its loudest one.** The volume of an error
  says how far the failure propagated, not where it started.
- **A retry fallback that drops the rate limiting is an outage amplifier.** The degraded
  path has to be gentler than the normal path, not more aggressive.
