# Codex Build

**One model orchestrates. Another writes the code. The tests run before every commit.**

Your agent (Claude Code, or any Skills-capable harness) plays *orchestrator*: it sequences an approved plan, briefs [OpenAI Codex](https://github.com/openai/codex) to write all the product code, reviews every diff, runs the test gate, and commits — **one task per commit, one PR per plan**. Codex does the typing; the orchestrator guards the gate.

No mega-diffs. No "trust me, it works." No silent takeover when the coder gets stuck.

---

## Why I built it

I write at [littlemight.com](https://littlemight.com?utm_source=codex-build&utm_medium=readme&utm_campaign=github&utm_content=intro) (and run [BestSelf.co](https://bestself.co?utm_source=codex-build&utm_medium=readme&utm_campaign=github&utm_content=intro) on the side). When I hand a plan to a coding agent, the two things that go wrong are always the same: it writes a giant unreviewable diff, and it commits code whose tests it never actually ran.

So I split the job. The model with my context and taste never gets to skip the review or the gate — and it's a *different* model than the one that wrote the code, so the check is real. The coder model runs cheap, high-effort reasoning and does the mechanical work. The result is a reviewable history: one task, one commit, tests green before each one, and exactly one PR at the end.

> *The gate is the point.* A task is committable only when its tests pass — run by the orchestrator, not taken on faith from the coder.

---

## What it does

```
preflight → persist run state → materialize tasks → for each task:
    claim → brief Codex → enforce allowlist → review diff + verify tests
          → test gate (BEFORE commit) → commit (no AI attribution)
          → persist interfaces ledger → close
finish → full-suite gate → push → ONE PR
```

- **Tests run before every commit.** Not "usually." Every one. Red gate → no commit.
- **One task, one commit; one plan, one PR.** Reviewable history, no mega-diffs.
- **Scope is executable.** Each task gets an on-disk file allowlist. A bundled script fails on tracked, staged, deleted, renamed, or non-ignored untracked paths outside it—after Codex runs and again before commit.
- **Runs survive compaction.** Task status, allowlists, test/commit evidence, and interfaces live under the worktree's private Git directory and are reloaded on resume.
- **No silent takeover.** If Codex can't land a task in three tries, the run *stops* and hands you the diff — it never quietly rewrites the code itself and pretends it passed.
- **Reproducible.** Model and reasoning effort are pinned per invocation, not inherited from whatever your Codex config happens to be today.

Two ideas do most of the work:

- **The Codex brief** ([`references/codex-brief.md`](skills/codex-build/references/codex-brief.md)). `codex exec` is stateless and sees only the prompt, so the brief carries the goal, the verbatim plan excerpt, the allowed files, and the conventions to match. Brief quality is the main driver of output quality.
- **The interfaces ledger.** After each green commit the orchestrator writes the verified public surface (signatures, types, endpoints, paths, source commit) to a durable file and feeds the relevant slice into the next task's brief—so Codex builds on real contracts instead of guessing across stateless calls.

---

## Requirements

| Tool | Required? | Purpose |
| --- | --- | --- |
| A Skills-capable agent | yes | runs the skill (built for [Claude Code](https://docs.claude.com/en/docs/claude-code)) |
| [`codex`](https://github.com/openai/codex) CLI, authed | yes | the coder (`codex login`) |
| `git` | yes | commits, branch, diff |
| Python 3 | yes | runs the bundled exact-path scope gate |
| [`gh`](https://cli.github.com/) or [`glab`](https://gitlab.com/gitlab-org/cli) | optional | opens the PR/MR (falls back to a compare URL) |
| [`bd`](https://github.com/steveyegge/beads) (beads) | optional | richer task tracking; defaults to a Markdown checklist |

---

## Install

```bash
# Clone the repo somewhere, then symlink the inner skill into Claude Code's skills dir
git clone git@github.com:cathrynlavery/codex-build.git ~/code/codex-build
ln -s ~/code/codex-build/skills/codex-build ~/.claude/skills/codex-build
```

The real skill lives at `skills/codex-build/` inside the repo (so the same tree works as a Claude Code plugin, a Codex plugin, and a standalone skill). The symlink points Claude Code at that inner directory.

Restart Claude Code. The skill registers as `codex-build` and activates when you hand over a plan or say "codex-build this."

### Alternative: install as a plugin

**Claude Code:**

```
/plugin marketplace add cathrynlavery/codex-build
/plugin install codex-build@codex-build
```

**Claude Cowork:** Customize → Directory → Plugins → **+** → paste `cathrynlavery/codex-build` → Sync, then install from the Personal list.

**Codex:**

```
npx skills add https://github.com/cathrynlavery/codex-build --skill codex-build
```

---

## Quickstart

1. Write (or have your agent write) an **approved plan** with an ordered task list. See [`references/plan-example.md`](skills/codex-build/references/plan-example.md) for the shape.
2. Hand it over — in chat:

   > "codex-build this plan: `plan.md`"
   > "You orchestrate, Codex codes — execute the approved plan."

   or via the slash command:

   ```
   /codex-build:build plan.md            # effort defaults to `high`
   /codex-build:build plan.md xhigh      # harder / architectural work
   /codex-build:build plan.md max        # hardest problems (falls back to xhigh if rejected)
   /codex-build:build plan.md --model gpt-6-astra
   ```

The orchestrator confirms the model/effort, shows you the ordered tasks, then runs the per-task loop until the queue is empty and opens one PR. See [`references/walkthrough.md`](skills/codex-build/references/walkthrough.md) for a full trace of a single task.

### Configuration

| Variable | Default | Notes |
| --- | --- | --- |
| `CODEX_BUILD_MODEL` | your Codex config default | pin a model you have access to (e.g. `gpt-6-astra`, `gpt-5.6-sol`); list them from `~/.codex/models_cache.json` |
| `CODEX_BUILD_EFFORT` | `high` | `high`, `xhigh`, or `max` (`ultra` is refused) |

`--model` / the effort argument override the env vars for a single run.

---

## Architecture

Progressive disclosure. `SKILL.md` is the whole loop; the references are pulled in only when needed.

```
codex-build/
├── .claude-plugin/                  — Claude Code plugin + marketplace manifest
├── .codex-plugin/                   — Codex plugin manifest
├── commands/
│   └── build.md                     — /codex-build:build slash command
└── skills/codex-build/
    ├── SKILL.md                     — the loop: roles, config, per-task steps, rails
    ├── scripts/check_scope.py       — exact-path allowlist enforcement
    ├── tests/test_check_scope.py
    └── references/
        ├── codex-brief.md           — the Codex brief skeleton (highest-leverage part)
        ├── plan-example.md          — the shape of an input plan
        └── walkthrough.md           — a full trace of one task, start to finish
```

---

## When *not* to use this

- **A one-file, five-line change** → just make the edit; the ceremony isn't worth it.
- **Exploratory / throwaway spikes** → no plan, no gate, no PR discipline needed.
- **Work with no tests and no intention of writing any** → the gate is the whole point; without tests this skill has nothing to enforce.
- **You want the orchestrator to write the code** → that's a normal coding session, not this. Here the orchestrator reviews; Codex writes.

---

## About

Made by **Cathryn Lavery** — founder of [BestSelf.co](https://bestself.co?utm_source=codex-build&utm_medium=readme&utm_campaign=github&utm_content=bio). I write about AI, agents, and building things at [littlemight.com](https://littlemight.com?utm_source=codex-build&utm_medium=readme&utm_campaign=github&utm_content=bio) — blog + newsletter.

If this is useful, **star the repo** and come [say hi on X](https://x.com/cathrynlavery).
