---
name: task-analysis
description: 'Analyze a task and produce a verifiable implementation plan without writing feature code. Use when the user asks to analyze, plan, or register a task ("analyze this task", "create a plan for", "new task", "analizar tarea", "planificar", "nueva tarea"). Includes an "init" submode: when the user passes an unstructured document (README, spec, PRD, feature description) and wants it decomposed into the task registry, run the init flow in `## Init submode (project / document decomposition)` below.'
---

# Task Analysis

Analysis mode of the Tasks-Driven-Development workflow. Investigate a task and produce a plan. **Never write feature code in this mode** — the output is a plan, not an implementation. Implementation is handled by the `task-implementation` skill.

## Submode selection

Two submodes share this skill — both end in `planned` tasks with plans at `.tasks/plans/`:

- **Single-task analysis** (default): the request names ONE already-scoped task. Run `## Pre-analysis steps` → `## Plan format`.
- **Init submode** (project / document decomposition): the user passes an unstructured document (or asks to "start from this README", "break this into tasks", "plan the project"). Run `## Init submode (project / document decomposition)` instead of `## Pre-analysis steps`. Both submodes converge on the same `## Plan format` and `## Handoff contract`.

## Model requirement

Analysis and plan creation MUST run on the most powerful model available from the current provider. If the session is running on a weaker model, say so and ask the user whether to continue anyway or switch.

Resolution chain (details and verification dates in `references/model-selection.md`):

| Provider | First choice | Fallback | If unavailable |
|----------|--------------|----------|----------------|
| Anthropic | Fable | Opus | ask the user |
| OpenAI | SOL (highest variant) | — | ask the user |
| Other | — | — | ask the user |

## Principles

- Inspect the real code before stating or proposing anything. Never assume names or paths; never invent data. If something is missing, verify it in the code or ask.
- Batch all questions into a single message. If there are more than 5, create a markdown file with each question followed by `A:` for the user to fill in.
- Goal over process: resolve the request with the minimum number of iterations.
- Cite `path:line` instead of dumping file contents.

## Task registry

Tasks live in `.tasks/tasks.md`. Task header format:

```
** <title> — <state> · R<n> · plan: .tasks/plans/plan_<task>.md **
```

- `<state>`: `idea` (not analyzed) | `planned` (analyzed) | `review` (needs review) | `done`.
- A task in `planned`, `review`, or `done` MUST have both `R<n>` and `plan:`. If either is missing, treat it as `idea`.

### Registering a new task

1. Think the request through; detect ambiguities and suggest improvements the user may not have considered.
2. Batch any questions into one message (see Principles).
3. Register the task in `.tasks/tasks.md` as `idea`.

## Risk scale (mandatory when analyzing)

- **R1** — Cosmetic / local UI. Reversible. No data or security involved.
- **R2** — Local logic. Reversible. No migrations or external contracts.
- **R3** — Touches multiple modules, API contracts, or shared state.
- **R4** — DB migrations, auth/permissions, concurrency, or hard-to-revert data.
- **R5** — Security, mass write/delete, irreversible or infrastructure changes.

Behavior by risk:
- R1–R2: proceed without blocking if the request is clear.
- R3: proceed; confirm only if there is real ambiguity.
- R4–R5: confirm the plan with the user BEFORE any execution. Never delegable to a worker.

## Pre-analysis steps

1. Read the request and inspect the real code involved.
2. Detect the current state of the task.
3. Assign risk R1–R5.
4. Point out concrete improvements or problems. If critique is requested, be firm and clear without being disrespectful.
5. Suggest alternatives or ideas the user may not have considered.
6. Ask only if critical, irreplaceable information is missing (all in one message).
7. Evaluate whether the task is orchestrable: identify independent steps that workers could run in parallel and mark them in the plan.

## Init submode (project / document decomposition)

Use when the user provides a document (README, spec, PRD, feature description, pasted content) and wants it decomposed into the task registry before any implementation. Differs from single-task analysis in that the task list itself is the output of analysis — answers to clarifying questions shape which tasks exist.

### Init flow

1. **Read the provided document end-to-end.** Then verify against the real codebase: root manifests, lockfiles, build/test/lint config, existing instruction files (`AGENTS.md`, `CLAUDE.md`, `.cursor/rules/`, `opencode.json`). Trust executable sources over prose.
2. **Detect and state current state**: greenfield / partial / extending an existing feature. State it explicitly in the first reply.
3. **Model check** (see `## Model requirement`): if the session is on a weaker model than the provider's most powerful available, say so and ask whether to continue or switch.
4. **Assess risk of the whole effort** (R1–R5 from `## Risk scale`). State it up front; the risk level tells the user whether plans need confirmation before execution.
5. **Generate one batch of clarifying questions** (≈10 max, prefer fewer). Ask only questions whose answers change which tasks exist or their scope — not things the repo already answers. Number them `P1`, `P2`, ... Present as a single message.
6. **WAIT for answers.** Do not register tasks or write plans yet. The user answers in any order; some may be skipped — note the default you'll assume for each skipped answer.
7. **Pre-register the task list in `.tasks/tasks.md`**:
   - One bullet per task as `idea` with title, provisional risk (Rn), short description.
   - Include a `## Dependencias` section with an ASCII graph showing which tasks block which.
   - Present the list inline to the user for review BEFORE writing plans.
8. **Get explicit confirmation** of the task list. Adjust on feedback before proceeding to plans.
9. **Write plans**: one file per task at `.tasks/plans/plan_<id>_<slug>.md` following the common `## Plan format` (affected files, action, success criterion, risk + revert, `parallelizable: yes|no`). Then update each task header in `.tasks/tasks.md` to `planned` with `R<n>` and the plan path.
10. **Closing** per `## Closing`.

### Init hard rules

- Never write feature code in this submode either. Output is plans, not implementation.
- Cite `path:line` instead of dumping file content.
- Batch all questions into one message (per `## Principles`).
- Confirm with the user before executing anything for R3+ tasks once implementation starts.
- Do NOT trigger real external API calls (OpenRouter, LLM providers, payment gateways, etc.) without explicit user approval — spikes may consume credits or hit rate limits. Include `--dry-run` flags where applicable and let the user run the real invocation.

## Plan format (`.tasks/plans/`)

Create `.tasks/plans/plan_<task>.md` (create the directory if it does not exist). Each step includes:

- affected files (real paths);
- concrete action;
- verifiable success criterion;
- step risk and how to mitigate or revert it;
- `parallelizable: yes|no` — yes only if the step shares no files with, and does not depend on the result of, any other pending step.

Record the plan path in the task header.

## Handoff contract (output of this skill)

Analysis is complete when:
1. The plan exists at `.tasks/plans/plan_<task>.md`.
2. The task header in `.tasks/tasks.md` is updated to `planned` with `R<n>` and the plan path.

The `task-implementation` skill requires this handoff for any R3+ task.

## Closing (max 2 lines)

```md
Changed: <what changed and where>.
Next: <next step or "nothing">.
```
