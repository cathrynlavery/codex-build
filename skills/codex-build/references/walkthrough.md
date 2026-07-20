# Walkthrough — one task, start to finish

A concrete trace of the per-task loop from `SKILL.md`, using task **T1** of
[`plan-example.md`](plan-example.md). This is illustrative — command output is
elided.

---

**Preflight (once).** Model pinned, auth confirmed, branch created:

```
$ codex exec "Reply with the single word READY" \
    -m gpt-5.6-codex -c model_reasoning_effort="high" -s read-only --ephemeral < /dev/null
READY
$ git checkout -b feat/api-rate-limit
```

Orchestrator announces: `MODEL=gpt-5.6-codex EFFORT=high TRACKER=markdown`, shows
the ordered task list (T1 → T2 → T3), and starts.

**T1 — claim.** `TASKS.md` gets `- [ ] T1 token-bucket core` marked in-flight.

**T1 — brief Codex.** The orchestrator fills the brief skeleton: goal (pure
token-bucket), interfaces (greenfield — none yet), the verbatim T1 excerpt, files
allowed (`src/ratelimit/bucket.ts` + its test), a sibling file to match for style,
out-of-scope (T2/T3), and done-criteria including the gate command.

```
$ codex exec "<brief>" -C "$(git rev-parse --show-toplevel)" -s workspace-write \
    -m gpt-5.6-codex -c model_reasoning_effort="high" < /dev/null
```

**T1 — review.** Scope check first:

```
$ git diff --stat
 src/ratelimit/bucket.ts      | 41 +++++++++
 src/ratelimit/bucket.test.ts | 38 ++++++++
```

Both files are in scope — good. The orchestrator reads `bucket.ts` in full,
opens `bucket.test.ts`, and confirms the tests actually assert refill/burst/
exhaustion (not just that the constructor runs). One gap: no test for a zero
refill rate. Correction:

```
$ codex exec resume --last "Add a test: a bucket with refillRate=0 never refills \
    after exhaustion. Keep everything else." < /dev/null
```

**T1 — gate.** The orchestrator runs it (not Codex):

```
$ npm test -- src/middleware   # per the plan
$ npm run build
```

Green.

**T1 — commit.** Stage only T1's files, no attribution:

```
$ git add src/ratelimit/bucket.ts src/ratelimit/bucket.test.ts
$ git commit -m "feat: token-bucket rate-limit core

Pure capacity/refill/tryConsume implementation with unit tests covering
refill-over-time, burst, exhaustion, and zero-refill. No Redis/HTTP yet (T2/T3).
Tests: npm test -- src/middleware (green)."
```

**T1 — ledger.** The orchestrator records the interface for the next task's brief:

```
class Bucket { constructor(capacity: number, refillRate: number); tryConsume(n=1): boolean }
// src/ratelimit/bucket.ts
```

**T1 — close.** `TASKS.md` → `- [x] T1 (commit a1b2c3d)`. Progress line to user.
On to T2 — whose brief now pastes the `Bucket` interface above under CONTEXT.

---

After T3, the finish step runs the full suite once more, pushes, and opens **one**
PR for `feat/api-rate-limit` → `main`.
