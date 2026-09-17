---
description: Execute an approved plan with Codex as the coder — one task per commit, one PR at the end
argument-hint: <plan-file> [high|xhigh|max] [--model <name>]
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - AskUserQuestion
---

Execute the approved plan at `$1` using the codex-build loop, following the procedure documented in [`skills/codex-build/SKILL.md`](../skills/codex-build/SKILL.md). Treat that skill as the source of truth — don't reimplement the loop here.

Full argument string: `$ARGUMENTS`

## Defaults

- Effort defaults to `high`; pass `xhigh` or `max` for hard/architectural work. Never `ultra` (auto-delegation breaks the one-task/one-commit contract).
- Model resolves from `--model`, else `$CODEX_BUILD_MODEL`, else your Codex config default. Announce the resolved model/effort before starting.

## Required behaviour

1. **No plan file provided** → ask the user which plan to execute. Don't guess.
2. **Plan has prose but no ordered task list** → extract one and show it to the user before coding.
3. **Codex not installed / not authed** → stop and tell the user to install and `codex login`; don't proceed.
4. **Every `codex exec` gets `< /dev/null`** → otherwise a non-interactive run hangs on "Reading additional input from stdin…". This is a hard rail in the skill.
5. **Test gate before every commit** → never commit red, never skip the gate. This is the point of the skill.
6. **One task per commit, exactly one PR** → no batching, no splitting; no AI attribution anywhere.
7. **Persist and enforce run state** → write task allowlists and the interfaces
   ledger under the worktree's private Git directory, reload them on resume, and
   run the bundled scope checker after Codex and before every commit.

Report progress per task (task, tests run, commit hash) and finish with the PR URL and anything deferred.
