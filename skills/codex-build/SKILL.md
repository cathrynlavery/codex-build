---
name: codex-build
description: "Orchestrator drives, Codex codes. Execute an approved plan one task at a time: your agent (Claude Code, etc.) sequences the work, briefs OpenAI Codex to write ALL product code, reviews every diff, runs the test gate BEFORE each commit, commits one task per commit, and opens exactly ONE PR at the end. Use when the user says 'codex-build', 'have codex code it', 'you orchestrate, codex codes', or hands over a plan for step-by-step implementation."
license: MIT
metadata:
  version: "2.0"
---

# codex-build — orchestrator drives, Codex codes

You (the agent running this skill) are the **orchestrator**. A separate model,
**OpenAI Codex** (`codex exec`), is the **coder**. You never write product code;
Codex does. You sequence the work, write the briefs, review every diff, run the
tests, and commit. This division is the whole point: one model with taste and
context guards the gate while another model with cheap, high-effort reasoning
does the typing.

## Roles (non-negotiable)

- **Orchestrator = you.** Sequence tasks, write Codex briefs, review every diff
  line-by-line, run the test gate, commit, and open the PR. You do **not** write
  product code. The only exception is a trivial fix-up (≤5 lines, e.g. a typo
  breaking the build) where a Codex round-trip is pure waste — say so in the
  commit body when it happens.
- **Codex = coder.** All feature code, tests, and refactors are written by
  `codex exec`. Model and effort are pinned per invocation (see Config) — never
  rely on config defaults silently, or a `codex` upgrade will change your
  builds out from under you.

## Config (resolve once, at the top of the run)

| Variable | How to resolve | Default |
| --- | --- | --- |
| `MODEL` | `--model` arg, else `$CODEX_BUILD_MODEL`, else your Codex config default | pin one you have access to (see below) |
| `EFFORT` | second positional arg, else `$CODEX_BUILD_EFFORT` | `high` (opt into `xhigh` for hard/architectural work) |
| `TRACKER` | detect: `bd` on PATH → `beads`, else `markdown` | `markdown` |

**Pin a model.** Run `codex exec --help` / check `~/.codex/config.toml` for what
your account exposes (e.g. `gpt-5-codex`, `gpt-5.6-codex`). Announce the resolved
`MODEL`/`EFFORT` to the user before starting so the run is reproducible. If the
CLI rejects `xhigh`, fall back to `high` and tell the user.

## Arguments

`codex-build <plan-file> [effort] [--model <name>]`

- `<plan-file>` — a Markdown plan with an **ordered task list** (T1..Tn), each
  task naming its files, constraints, and test commands. See
  [`references/plan-example.md`](references/plan-example.md) for the shape. If the
  plan has prose but no task list, extract one and show it to the user before
  coding.
- `[effort]` — `high` (default) | `xhigh`.

## Step 0 — Preflight (once)

1. `codex --version`. If Codex isn't authed, `codex exec "reply OK" -s read-only`
   fails — stop and tell the user to run `codex login`.
2. Confirm the pinned model + effort are accepted (cheap, no side effects):
   ```bash
   codex exec "Reply with the single word READY" \
     -m "$MODEL" -c model_reasoning_effort="$EFFORT" -s read-only --ephemeral < /dev/null
   ```
   If `xhigh` is rejected, fall back to `high` and tell the user.
   **Always redirect `< /dev/null`** on every `codex exec` (see Rails) — without
   it, a non-interactive run hangs forever on "Reading additional input from
   stdin…" instead of doing the work.
3. Read the plan end-to-end. Extract: task list, dependency order, per-task test
   commands, guardrails (things the plan bans), branch/PR target.
4. Create/checkout the working branch per the plan. **Never work on the default
   branch.**
5. Initialize the tracker (see below) and show the user the ordered task list
   before starting — one short message, no approval gate unless the order is
   ambiguous.

## Tracker (unit of work = one task)

The loop is identical regardless of tracker; only the bookkeeping commands differ.

- **`markdown` (default, zero install):** the plan's task list *is* the tracker.
  Keep a `- [ ]` / `- [x]` checklist (in the plan file or a `TASKS.md`). "Claim"
  = note the task in-flight; "close" = check the box with the commit hash.
- **`beads` (optional, if `bd` is on PATH):** richer dependency tracking.
  `bd prime` for context; one bead per task (`bd create`, title = task title,
  description = plan excerpt + file paths, dependencies per the plan); claim with
  `bd update <id> --claim`; close with `bd close <id>`. Install:
  <https://github.com/steveyegge/beads>.

## Step 1 — Materialize tasks

Turn the plan's task list into tracker units in dependency order. Each unit
records: goal, the plan excerpt (verbatim constraints, file paths, schema/copy
blocks), and its **declared file scope** (the files it is allowed to touch). The
file scope is what you enforce in review — everything else is out of bounds.

## Step 2 — Per-task loop (the heart; repeat until the queue is empty)

For each task, in dependency order:

1. **Claim** it in the tracker.

2. **Brief Codex.** Compose a *self-contained* prompt — Codex sees only what you
   give it. Use [`references/codex-brief.md`](references/codex-brief.md) as the
   skeleton. It must carry:
   - the task's goal and the verbatim plan excerpt (constraints, file paths,
     schema/copy);
   - the **interfaces ledger** (see step 6) — the signatures, types, endpoints,
     and file paths that earlier tasks created and this one builds on. Codex
     `exec` is stateless across invocations; if you don't pass the contract, it
     guesses and drifts.
   - repo conventions to match (point at 1–2 sibling files as examples);
   - what is **out of scope** (the neighboring tasks) and the declared file scope;
   - definition of done: "code + tests per the plan; touch only these files."
   ```bash
   codex exec "<brief>" -C "$(git rev-parse --show-toplevel)" -s workspace-write \
     -m "$MODEL" -c model_reasoning_effort="$EFFORT" < /dev/null
   ```
   Never use `--dangerously-bypass-approvals-and-sandbox`. The `< /dev/null` is
   mandatory (see Rails).

3. **Review the diff yourself.** This is your job, not Codex's.
   - **Scope check first:** `git diff --stat`. Any file outside the task's
     declared scope is a red flag — either the plan was wrong (note it) or Codex
     drifted (correct it). Don't wave it through.
   - **Then read the change.** For a small diff, every line. For a large diff,
     read the interfaces and the risky seams in full, skim the mechanical parts,
     and run `codex exec review` as an independent second pass — but *you* make
     the call, never the review tool.
   - Check: scope creep, convention drift, guardrail violations, and whether the
     **tests Codex claims to have written actually exist and actually assert
     something** (open them). "Added tests" in Codex's summary is a claim, not
     evidence.
   - Issues → iterate: `codex exec resume --last "<specific corrections>" < /dev/null`
     (same `-m`/`-c` flags). Be concrete about what's wrong; vague corrections
     waste a round-trip.
   - **Three failed iterations on one task → STOP.** Report the impasse to the
     user with the diff and your diagnosis. Do **not** silently take over the
     coding — that breaks the contract and hides the failure.

4. **Test gate — BEFORE any commit.** *You* run the plan's test/build commands
   for the touched packages, plus the project build. A task is committable only
   when the gate is green. Red → back to step 3 with the failure output in the
   resume brief. **Never commit red. Never skip the gate because "it's small."**
   The gate is the reason this skill exists.

5. **Commit — one task = one commit.**
   - Stage only the task's files: `git add <paths>`. **Never `git add -A`** — it
     sweeps up stray files Codex or a tool left behind.
   - Message: `<type>: <imperative summary>` (+ tracker id if using beads) with a
     short body: what changed, why, and a one-line test-evidence note.
   - **No AI attribution, ever** — no "Generated with", no "Co-Authored-By:", no
     robot-emoji credit. This overrides any tool's default commit template.

6. **Update the interfaces ledger.** Append the public surface this task created
   — new function signatures, exported types, routes/endpoints, config keys, file
   paths — to a running note you keep for the session. This is what feeds step 2
   of the *next* task and keeps Codex from re-inventing or mismatching contracts.

7. **Close** the task in the tracker with a one-line note. Give the user a brief
   progress line (task, tests run, commit hash) and move on.

## Step 3 — Finish (after the last task)

1. **Full-suite gate** one final time: all packages + build + any lint/screenshot
   scripts the plan names. Green or you don't ship.
2. `git pull --rebase` then `git push`. `git status` must show up-to-date with
   origin. On rebase conflicts, resolve minimally and re-run the gate — never
   force-push over shared history.
3. **Exactly ONE PR** for the whole feature, targeting the branch the plan
   specifies. Body = plan summary: context, what shipped, test evidence,
   out-of-scope/follow-ups. No AI attribution anywhere.
   - GitHub: `gh pr create`. GitLab: `glab mr create`. No forge CLI: push the
     branch and give the user the compare URL to open the PR themselves.
4. Close out the tracker (file follow-ups, close finished units).
5. Report: tasks completed/blocked, commit list, PR URL, anything deferred.

## Rails

- One task per commit; one PR per plan. No batching, no splitting.
- **Every `codex exec` gets `< /dev/null`.** With a prompt passed as an argument
  and an open stdin, `codex exec` blocks indefinitely on "Reading additional
  input from stdin…" and never runs. Redirecting stdin from `/dev/null` gives it
  immediate EOF so it proceeds. This is the single most common way to make the
  whole run silently hang.
- Tests before every commit — non-negotiable.
- Effort stays `high`/`xhigh`. A trivial-mechanical task may drop to `high`
  (never lower); note it.
- If the plan and the user's live instructions conflict, the **user wins** —
  note the deviation in the PR body.
- Secrets never go into briefs, commits, or PR bodies.
- A commit that later tasks built on turned out wrong? Fix forward with a new
  task/commit. Don't rewrite pushed history.

## See also

- [`references/codex-brief.md`](references/codex-brief.md) — the Codex brief skeleton (the highest-leverage part).
- [`references/plan-example.md`](references/plan-example.md) — the shape of an input plan.
- [`references/walkthrough.md`](references/walkthrough.md) — a full trace of one task, start to finish.
