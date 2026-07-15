---
name: task-implementation
description: Execute a planned task or a direct low-risk request with quality and safety guardrails. Use when the user asks to implement, execute, or build a task ("implement task X", "execute the plan", "implementar tarea", "ejecutar el plan").
---

# Task Implementation

Implementation mode of the Tasks-Driven-Development workflow. Execute a task that was analyzed with the `task-analysis` skill, or a direct low-risk request. **Do not analyze and implement in the same pass** — if the task needs a plan and has none, hand off to `task-analysis` first.

## Preconditions

- **R3+ tasks require an existing plan** at `.tasks/plans/plan_<task>.md` and state `planned` in `.tasks/tasks.md`. If missing, stop and run analysis first.
- Direct requests without a plan are only acceptable for R1–R2 (cosmetic or local, reversible, no data/security impact).
- Implementing a single task does not require the whole task list to be analyzed.

Task states (in `.tasks/tasks.md`): `idea` | `planned` | `review` | `done`.

## Principles

- Make the minimal change that meets the goal; follow the style of the neighboring code.
- Reversible, low-scope changes. One change = one goal.
- No hardcoded values: configuration via environment variables or named constants.
- If something fails, report the exact error and the proposed fix. Never hide failures.
- Independent tool calls in parallel, not in series. Do not re-read files already read in the session.
- Stack, build/test commands, and artifact-language conventions are project-specific: read them from the consuming project's docs (`docs/AGENTS.md`, `CLAUDE.md`, or equivalent), not from this skill.

## Execution steps

1. Load relevant skills/tools before starting.
2. **Create a task branch before making any change** (see below). All implementation happens on this branch, never directly on the base branch.
3. With a task tool available: one task per step; mark `WIP` on start, `DONE` on finish.
4. If the plan has ≥2 steps marked `parallelizable: yes` → apply orchestration (see `references/orchestration.md`). Otherwise execute directly with the primary role.
5. Apply the code quality standards in `references/code-quality.md` to every change.
6. On failure, report the exact error and proposed solution.

## Task branch (before implementing)

Before implementing a task, create a dedicated branch for it so the work stays isolated and reversible.

- **Name**: the simplified task name in `kebab-case`, prefixed with `task/` — e.g. task "Agregar login con Google" → `task/agregar-login-con-google`.
- Simplify the name: lowercase, remove accents and punctuation, collapse spaces to single hyphens, drop filler words, and keep it short (roughly ≤ 6 words). If the task has an id, prefix it: `task/<id>-<slug>` (e.g. `task/T12-agregar-login`).
- **Branch from the up-to-date base branch** (the project's default, usually `main`).
- Commands:
  ```sh
  git checkout main && git pull
  git checkout -b task/<slug>
  ```
- If the branch already exists, switch to it instead of recreating it (`git checkout task/<slug>`).
- Never implement directly on the base branch.

## Security (always check, non-negotiable)

- Never commit secrets; use environment variables. Do not log sensitive data or PII.
- Validate and sanitize all external input; parameterized queries (avoid SQL/command injection).
- Verify authorization/permissions on every new endpoint; respect `root` protection.
- Least privilege: do not widen the attack surface or permissions beyond what is needed.
- For R4–R5, list the security impact explicitly before executing.

## Orchestration

When the plan allows parallel work, the orchestrator (primary role) distributes steps among workers and verifies every result. Full rules — when to orchestrate, subtask contract, backends, guardrails — are in `references/orchestration.md`. Hard limits that always apply:

- R4–R5 steps are never delegated.
- Workers never run destructive commands (`rm`, `DROP`, mass `DELETE`, `--force`).
- Every worker result is verified against the step's success criterion before integration.

## Definition of done

Before declaring the task finished:

- Compiles / lints / passes tests per the project stack (commands in the project's docs).
- Meets the plan's success criteria.
- No secrets, dead code, or critical TODOs introduced by the change.
- **R3+: run a critical review with the `review` role** (best reasoning model available) before declaring done — whether or not orchestration was used.

Report the real result of each verification (what ran and what it returned). If a step was skipped, say so.

## Task closeout

1. Update the task state in `.tasks/tasks.md`: `review` if it needs human review, `done` otherwise.
2. When a task reaches `done`, move its plan to `.tasks/plans/archive/`.
3. **Remind the user that the merge is manual**: the task branch is left as-is; a human reviews and merges it into the base branch. Never merge the task branch yourself.

## Closing (max 2 lines)

```md
Changed: <what changed and where>.
Branch: <task/branch-name> — merge is manual; a human must review and merge it.
Next: <next step or "nothing">.
```
