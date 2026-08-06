# Tasks-Driven-Development

*Leé esto en español: [README.es.md](README.es.md)*

Three agent skills that split feature work into an explicit **analyze → plan → implement → audit** cycle, with a risk scale, model-selection policy, and orchestration guardrails. Provider-agnostic: they work with any model family.

## The three skills

| Skill | Mode | Output |
|-------|------|--------|
| [`task-plan`](skills/task-plan/SKILL.md) | Investigate a task, assign risk, produce a plan. Never writes feature code. | Plan in `.tasks/plans/` + task marked `planned` |
| [`task-impl`](skills/task-impl/SKILL.md) | Execute a planned task (or a direct R1–R2 request) with quality and security guardrails. | Working change + task marked `review`/`done` |
| [`task-audit`](skills/task-audit/SKILL.md) | Audit existing code — correctness, tests, security, sanitization, performance. Never fixes production code. | Report in `.tasks/audits/` + planned fix tasks |

The audit closes the loop: its fix tasks are already `planned`, so `task-impl` consumes them with no re-analysis.

### Handoff contract

- Analysis ends when the plan exists at `.tasks/plans/plan_<task>.md` and the task header in `.tasks/tasks.md` says `planned` with a risk level.
- Implementation **requires that handoff for any R3+ task**. Only R1–R2 requests may skip analysis.
- On completion, implementation updates the task state and archives the plan to `.tasks/plans/archive/`.
- Audit ends when the report exists, every finding is CONFIRMED or PLAUSIBLE, and each accepted finding has a `planned` fix task (or is explicitly listed as accepted-without-task).

### Implementation git flow

- **One branch per task** (`task/<slug>`), created before touching code; never implement on the base branch.
- **Parallel work shares that same branch**: no worktrees and no per-worker branches. Isolation is by file ownership (one file, one worker), workers only write files, and the orchestrator owns git — it verifies each scoped diff before committing it.
- **Optional audit**: before implementing, the user is asked whether an audit agent should review the produced code. Answer with the model name, or `No`; empty = No. `task-impl` only asks — the review itself runs `task-audit` over the task diff.
- Merging into the base branch is **always manual**, and the closeout suggests a human or agent review.

## Audit

`task-audit` runs standalone over any scope (`diff`, `module`, `repo`, or a single axis), or is invoked by `task-impl` as the post-implementation review.

- **Evidence or nothing**: every finding cites `path:line` plus a concrete failure scenario. Candidates are collected, then verified; whatever does not survive is discarded and only counted.
- **Two scales**: `severity` (critical/high/medium/low) for the finding, `R1–R5` for the fix. They are independent.
- **Read-only on production code.** The one exception is tests: it may write characterization/regression tests on an `audit/<slug>` branch to prove findings. A test that reproduces a bug is left failing on purpose.
- Accepted findings become `planned` fix tasks — `critical`/`high` get their own task, `medium`/`low` may be grouped within a module.

## Task registry (`.tasks/`)

```
.tasks/
  tasks.md            # task index
  plans/
    plan_<task>.md    # one plan per task
    archive/          # plans of finished tasks
  audits/
    audit_<scope>_<date>.md
```

Task header format in `tasks.md`:

```
** <title> — <state> · R<n> · plan: .tasks/plans/plan_<task>.md **
```

States: `idea` → `planned` → `review` → `done`.

## Risk scale

| Level | Meaning |
|-------|---------|
| R1 | Cosmetic / local UI. Reversible. |
| R2 | Local logic. Reversible. No migrations or external contracts. |
| R3 | Multiple modules, API contracts, or shared state. |
| R4 | DB migrations, auth, concurrency, hard-to-revert data. |
| R5 | Security, mass write/delete, irreversible or infrastructure changes. |

R4–R5 always require user confirmation before execution and are never delegated to workers.

## Model policy

Roles are defined by capability, not brand (`primary`, `agent`, `review`, `fast`). One hard rule: **analysis and plan creation always run on the most powerful model the provider offers** — Anthropic: Fable, else Opus; OpenAI: the highest SOL variant; anything else: ask the user. Auditing is analysis, so it follows the same rule — unless the user named a model for it, in which case that one is used exactly. Details in [`model-selection.md`](skills/task-plan/references/model-selection.md).

## Project-specific configuration

These skills are project-agnostic on purpose. Stack, build/test commands, and artifact-language conventions belong in the consuming project's docs (`docs/AGENTS.md`, `CLAUDE.md`, or equivalent), where the implementation skill reads them.

## Installation

This repo is the canonical source. Symlink each skill into the directories your agents read:

```sh
ln -s "$(pwd)/skills/task-plan"  ~/.agents/skills/task-plan
ln -s "$(pwd)/skills/task-plan"  ~/.claude/skills/task-plan
ln -s "$(pwd)/skills/task-impl"  ~/.agents/skills/task-impl
ln -s "$(pwd)/skills/task-impl"  ~/.claude/skills/task-impl
ln -s "$(pwd)/skills/task-audit" ~/.agents/skills/task-audit
ln -s "$(pwd)/skills/task-audit" ~/.claude/skills/task-audit
```

(Claude Code reads `~/.claude/skills`; opencode reads both. Verified 2026-07-07.)

## Repo layout

```
skills/
  task-plan/
    SKILL.md
    references/model-selection.md
  task-impl/
    SKILL.md
    references/code-quality.md
    references/orchestration.md
  task-audit/
    SKILL.md
    references/audit-checklist.md
```
