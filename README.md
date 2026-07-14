# Tasks-Driven-Development

*Leé esto en español: [README.es.md](README.es.md)*

Two agent skills that split feature work into an explicit **analyze → plan → implement** cycle, with a risk scale, model-selection policy, and orchestration guardrails. Provider-agnostic: they work with any model family.

## The two skills

| Skill | Mode | Output |
|-------|------|--------|
| [`task-analysis`](skills/task-analysis/SKILL.md) | Investigate a task, assign risk, produce a plan. Never writes feature code. | Plan in `.tasks/plans/` + task marked `planned` |
| [`task-implementation`](skills/task-implementation/SKILL.md) | Execute a planned task (or a direct R1–R2 request) with quality and security guardrails. | Working change + task marked `review`/`done` |

### Handoff contract

- Analysis ends when the plan exists at `.tasks/plans/plan_<task>.md` and the task header in `.tasks/tasks.md` says `planned` with a risk level.
- Implementation **requires that handoff for any R3+ task**. Only R1–R2 requests may skip analysis.
- On completion, implementation updates the task state and archives the plan to `.tasks/plans/archive/`.

## Task registry (`.tasks/`)

```
.tasks/
  tasks.md            # task index
  plans/
    plan_<task>.md    # one plan per task
    archive/          # plans of finished tasks
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

Roles are defined by capability, not brand (`primary`, `agent`, `review`, `fast`). One hard rule: **analysis and plan creation always run on the most powerful model the provider offers** — Anthropic: Fable, else Opus; OpenAI: the highest SOL variant; anything else: ask the user. Details in [`model-selection.md`](skills/task-analysis/references/model-selection.md).

## Project-specific configuration

These skills are project-agnostic on purpose. Stack, build/test commands, and artifact-language conventions belong in the consuming project's docs (`docs/AGENTS.md`, `CLAUDE.md`, or equivalent), where the implementation skill reads them.

## Installation

This repo is the canonical source. Symlink each skill into the directories your agents read:

```sh
ln -s "$(pwd)/skills/task-analysis"       ~/.agents/skills/task-analysis
ln -s "$(pwd)/skills/task-analysis"       ~/.claude/skills/task-analysis
ln -s "$(pwd)/skills/task-implementation" ~/.agents/skills/task-implementation
ln -s "$(pwd)/skills/task-implementation" ~/.claude/skills/task-implementation
```

(Claude Code reads `~/.claude/skills`; opencode reads both. Verified 2026-07-07.)

## Repo layout

```
skills/
  task-analysis/
    SKILL.md
    references/model-selection.md
  task-implementation/
    SKILL.md
    references/code-quality.md
    references/orchestration.md
directive.md        # original (Spanish) directive these skills were split from
```
