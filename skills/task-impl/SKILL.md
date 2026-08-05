---
name: task-impl
description: Execute a planned task or a direct low-risk request with quality and safety guardrails. Use when the user asks to implement, execute, or build a task ("implement task X", "execute the plan", "implementar tarea", "ejecutar el plan").
---

# Task Implementation

Implementation mode of the Tasks-Driven-Development workflow. Execute a task that was analyzed with the `task-plan` skill, or a direct low-risk request. **Do not analyze and implement in the same pass** — if the task needs a plan and has none, hand off to `task-plan` first.

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
2. **Setup before any change** — in a single message: create the task branch (see below) and ask the audit question (see `## Post-implementation audit`). Do not start implementing until both are settled.
3. All implementation happens on the task branch, never directly on the base branch.
4. With a task tool available: one task per step; mark `WIP` on start, `DONE` on finish.
5. If the plan has ≥2 steps marked `parallelizable: yes` → apply orchestration (see `references/orchestration.md`); each parallel worker gets its own **git worktree** (see below). Otherwise execute directly with the primary role.
6. Apply the code quality standards in `references/code-quality.md` to every change.
7. Run the post-implementation audit if the user requested one, apply the accepted findings, and re-verify.
8. On failure, report the exact error and proposed solution.

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
- If the working tree has uncommitted changes that do not belong to this task, say so and ask before switching branches. Never discard them.
- Never implement directly on the base branch.

## Parallel work uses worktrees

Parallel implementation by agents is allowed only when orchestration applies (see `references/orchestration.md`). When it does, **every parallel worker runs in its own git worktree**, never in the shared checkout — concurrent agents in one working tree corrupt each other's changes.

- One worktree per worker, branched off the task branch:
  ```sh
  git worktree add ../<repo>-<slug>-w1 -b task/<slug>/w1 task/<slug>
  ```
- The worker's file list (subtask contract) is expressed as paths inside its own worktree; a file still belongs to one worker at a time.
- If the runtime creates worktrees itself for subagents (e.g. an isolation option in the agent tool), use that instead of the manual command — it is the same guarantee.
- Integration is the orchestrator's job, in the main checkout on the task branch: merge each worker branch after verifying its diff against the step's success criterion, then clean up:
  ```sh
  git merge --no-ff task/<slug>/w1
  git worktree remove ../<repo>-<slug>-w1
  ```
- If a worker branch fails verification, do not merge it: fix it or redo the step, and report what happened.
- Serial execution needs no worktree — work directly on the task branch.

## Post-implementation audit (ask before implementing)

Ask the user, together with the branch setup and before writing code, whether an **audit agent** should review the produced code once the task is implemented:

```md
Post-implementation audit: should an audit agent review the produced code?
Answer with the model name for the agent (e.g. `opus`, `sonnet`, `opencode-go/minimax-m3`) or `No`. Empty = No.
```

- **Empty answer, `No`, or anything not identifiable as a model → no audit.** Say which reading you applied and move on; do not re-ask.
- A model name → after the implementation passes the Definition of done, launch an audit agent **on that exact model**. If the model is unavailable, report it and ask instead of silently substituting one.
- The audit agent is **read-only**: it reviews the task's full diff (`git diff <base>...task/<slug>`) against the plan's success criteria and returns findings only. The implementer (or the orchestrator) applies the accepted findings, then re-runs the Definition of done.
- Ask once per task. Do not re-ask per step.
- The audit never lifts a guardrail: R4–R5 stay with the orchestrator, no secrets go into the audit prompt, and it does not replace the R3+ critical review in the Definition of done.
- It is distinct from a **consultation** (per-step advice during orchestration, see below): the audit covers the whole task diff and is requested by the user.

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
- Every worker result is verified against the step's success criterion before integration

### Implementation model and consultations

Analysis uses the provider's top model (see `task-plan/references/model-selection.md`); **implementation is delegated to a lesser model**, and the top model is kept for consultations. Verified 2026-07-29.

| Provider | Implementation worker | Consultation model |
|----------|----------------------|--------------------|
| Anthropic | Sonnet | Opus or Fable — not the model that spawned the worker |
| OpenAI | Luna/medium or Tierra/medium | SOL/high |
| Other providers | ask the user | ask the user |

- Deploy the implementation worker as a subagent (backend A) or an `opencode` run (backend B) on the model above, never on the analysis model.
- A **consultation** is a read-only advice request: code review of the produced diff, hunting for improvements, or a fix for a blocked step. The consultant returns findings only; the worker (or the orchestrator) applies the changes.
- If the orchestrator is already the consultation model, it answers the consultation itself instead of spawning another agent for it.
- Consultations do not replace the R3+ critical review in the Definition of done, and they never lift the R4–R5 no-delegation limit.

Invocation details and the read-only guarantee are in `references/orchestration.md`.

## Definition of done

Before declaring the task finished:

- Compiles / lints / passes tests per the project stack (commands in the project's docs).
- Meets the plan's success criteria.
- No secrets, dead code, or critical TODOs introduced by the change.
- **R3+: run a critical review with the `review` role** (best reasoning model available) before declaring done — whether or not orchestration was used.
- If the user requested an audit agent: the checks above passed first, then the audit ran on the requested model and its accepted findings were applied and re-verified.
- All worker worktrees are merged and removed; nothing is left half-integrated.

Report the real result of each verification (what ran and what it returned). If a step was skipped, say so.

## Task closeout

1. Update the task state in `.tasks/tasks.md`: `review` if it needs human review, `done` otherwise.
2. When a task reaches `done`, move its plan to `.tasks/plans/archive/`.
3. **Remind the user that the merge is manual**: the task branch is left as-is; a human reviews and merges it into the base branch. Never merge the task branch yourself.
4. **Always suggest a final review before merging** — a human review, or an agent review if none was run. If the audit agent already ran, say what it found and still recommend the human pass.

## Closing (max 4 lines)

```md
Changed: <what changed and where>.
Branch: <task/branch-name> — merge is manual; a human must review and merge it.
Review: <audit agent result, or "no audit was run"> — suggest a human review or an agent review before merging.
Next: <next step or "nothing">.
```
