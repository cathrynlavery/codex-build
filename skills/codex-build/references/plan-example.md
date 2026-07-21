# Plan: add rate limiting to the public API

> This is the **input** to codex-build — an approved plan with an ordered task
> list. codex-build turns each task into a tracker unit, briefs Codex to build
> it, gates it, and commits it. Copy this shape for your own plans.

## Context

The public API (`/api/*`) has no rate limiting. We want a token-bucket limiter,
60 req/min per API key, returning `429` with a `Retry-After` header when
exceeded. Redis-backed so it works across instances.

## Branch & PR

- Branch: `feat/api-rate-limit`
- PR target: `main`

## Guardrails (things this plan bans)

- No new heavyweight dependencies — use the `ioredis` client already in the repo.
- Do not change the response shape of any existing successful (`2xx`) endpoint.
- No in-memory fallback that silently disables limiting in prod.

## Test commands (the gate)

- Unit: `npm test -- src/middleware`
- Build: `npm run build`

## Tasks (ordered)

### T1 — token-bucket core

- **Files:** `src/ratelimit/bucket.ts`, `src/ratelimit/bucket.test.ts`
- **Do:** pure token-bucket implementation (capacity, refill rate, `tryConsume`).
  No Redis, no HTTP — just the algorithm and its tests.
- **Done when:** unit tests cover refill-over-time, burst, and exhaustion.

### T2 — Redis-backed store

- **Files:** `src/ratelimit/store.ts`, `src/ratelimit/store.test.ts`
- **Depends on:** T1 (`Bucket` type/interface).
- **Do:** persist/restore bucket state in Redis keyed by API key, using the
  existing `ioredis` client from `src/redis.ts`.
- **Done when:** tests (against a mock/redis-memory) verify get/set round-trip
  and TTL expiry.

### T3 — middleware + wiring

- **Files:** `src/middleware/rate-limit.ts`, `src/middleware/rate-limit.test.ts`,
  `src/server.ts`
- **Depends on:** T1, T2.
- **Do:** Express middleware that consumes a token per request, returns `429` +
  `Retry-After` when empty, and mounts on `/api/*` in `src/server.ts`.
- **Done when:** tests cover under-limit pass-through and over-limit `429`; the
  build is green.
