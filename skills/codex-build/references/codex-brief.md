# Codex brief skeleton

`codex exec` is **stateless across invocations** and sees **only the prompt you
give it** (plus the repo on disk). Everything Codex needs to get the task right
the first time has to be in the brief. A good brief is the difference between one
clean round-trip and three failed iterations.

Fill this in per task. Keep it tight — it's a spec, not an essay.

---

## GOAL

One or two sentences: what this task must produce, in plain terms.

## CONTEXT / INTERFACES YOU BUILD ON

The public surface earlier tasks created that this task depends on — paste the
real signatures, don't describe them:

```
# e.g.
export function getSession(req: Request): Session | null   // src/auth/session.ts
POST /api/checkout  { items: LineItem[] } -> { orderId: string }
type LineItem = { sku: string; qty: number }               // src/types/cart.ts
```

If this is the first task, say "greenfield — no prior interfaces."

## PLAN EXCERPT (verbatim — do not paraphrase)

Paste the exact slice of the approved plan for this task: constraints, schema,
copy blocks, acceptance criteria. Verbatim, because the wording is the contract.

## FILES YOU MAY TOUCH

List them explicitly. This is the scope boundary — the orchestrator enforces it
in review.

```
src/foo/bar.ts
src/foo/bar.test.ts
```

## CONVENTIONS TO MATCH

Point at 1–2 existing sibling files to copy patterns from (naming, error
handling, test style, imports). "Match the style of `src/foo/baz.ts`." Cheaper
and more reliable than describing the conventions in prose.

## OUT OF SCOPE

Name the neighboring tasks/files this one must NOT touch. Be explicit — Codex
will happily "helpfully" refactor adjacent code if you don't fence it off.

## DEFINITION OF DONE

- Code implements the plan excerpt above.
- Tests written per the plan and they assert real behavior (not just that the
  code runs).
- Only the files listed under "FILES YOU MAY TOUCH" are changed.
- `<the exact test/build command(s) the orchestrator will run as the gate>` pass.

---

## Resume / correction briefs

When iterating on the same task, use `codex exec resume --last "<corrections>" < /dev/null`.
Corrections must be **specific and evidence-backed** — paste the failing test
output or name the exact line/behavior that's wrong. Vague notes ("this isn't
right", "make it better") waste a round-trip.

Good:
> `auth.test.ts` fails: `getSession` returns `undefined` for an expired token but
> the plan says it must return `null`. Fix the return, keep the rest.

Bad:
> the session handling seems off, please fix
