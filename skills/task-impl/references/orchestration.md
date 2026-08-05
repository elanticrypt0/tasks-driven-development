# Orchestration

The orchestrator (`primary` role) distributes plan steps among **workers** and verifies every result. A worker can be a **native subagent** (backend A) or an **`opencode` run** with another model (backend B).

Environment-specific details (model tables, flags, known bugs) age: each carries its verification date. Renew dates when re-verified.

## When to orchestrate

Orchestrate only if ALL of these hold:

- The plan has ≥2 steps marked `parallelizable: yes`.
- Every delegable step is R1–R3 (R4–R5 are never delegated).
- Steps share no files: a file belongs to one worker at a time.

Otherwise execute in series with the primary role. Orchestrating a trivial task costs more than it saves.

## Isolation: one worktree per parallel worker (mandatory)

Parallel workers never share a working tree — two agents writing in the same checkout overwrite each other and leave the task branch in an unreviewable state. Every parallel worker gets its own git worktree, branched off the task branch created by the implementation skill.

```sh
# from the main checkout, on task/<slug>
git worktree add ../<repo>-<slug>-w1 -b task/<slug>/w1 task/<slug>
git worktree add ../<repo>-<slug>-w2 -b task/<slug>/w2 task/<slug>
```

- The worker's prompt states its worktree path as the working directory and lists files relative to it. A worker never touches another worktree.
- Backend A: if the agent tool offers worktree isolation, use it instead of the manual command. Backend B: `cd` into the worktree before `opencode run`, or pass it as the run's working directory.
- The orchestrator stays in the main checkout and does not implement while workers are active.
- Integration, after verifying each worker's diff against its success criterion:
  ```sh
  git merge --no-ff task/<slug>/w1
  git worktree remove ../<repo>-<slug>-w1
  ```
- A worker branch that fails verification is not merged: fix it, redo the step, or take it over directly — and report it.
- Leftover worktrees are a bug: `git worktree list` must show only the main checkout when the task is done.

## Subtask contract (applies to both backends)

Every delegation includes, in the worker's prompt:

- the worker's worktree path and branch (see above);
- a precise goal and the exact files to touch (real paths, relative to that worktree);
- the minimum necessary context (do not dump the whole repo);
- a verifiable success criterion for the step;
- restrictions: do not delete content, do not touch files outside the list, do not introduce secrets or hardcoded values;
- expected output format (applied diff + short summary);
- if the step depends on an integration, which skill to use (workers see project and global skills — verified 2026-07-07).

## Backend A — native subagents

Use when the runtime offers subagents (e.g. Task/Agent in Claude Code).

- Launch independent workers in parallel with the `agent` role; search/exploration with the `fast` role.
- Each writing worker runs in its own worktree (isolation option if the tool has one, manual `git worktree add` otherwise). Read-only workers do not need one.
- The orchestrator does not implement while workers are active: it coordinates, verifies, and integrates.
- Final review of the whole with the `review` role if the task is R3+.

## Backend B — `opencode` with other models

Use when there are no subagents, or when it pays to route volume to cheaper external models.

Base invocation (tested 2026-07-06 with parallel workers on both providers):

```sh
timeout 240 opencode run "SUBTASK CONTRACT PROMPT" -m MODEL --auto
```

- `--auto` is required in non-interactive runs so the worker can write files; it is safe only because the contract restricts which files may be touched.
- **Never use angle-bracket placeholders (`<value>`, `<path>`) in the prompt**: `opencode run` hangs without emitting output (verified 2026-07-07 on v1.17.14, A/B tested with the same prompt). Use UPPERCASE markers instead ("reply: TEMP Rosario NUMBER").
- Always wrap in `timeout`: a hung run emits nothing and would block the orchestration. If it expires, check the prompt first (angle brackets?) and only then apply the model fallback.
- `--format json` only if the output will be parsed programmatically; to verify, look at the real diff on disk, not the output.

Useful flags:

| Flag | Short | Description |
|------|-------|-------------|
| `--model` | `-m` | Model to use (`provider/model` format) |
| `--agent` | — | Agent to trigger (e.g. `plan`, `build`) |
| `--format` | — | Output `default` (text) or `json` |
| `--continue` | `-c` | Continue the last active session |
| `--file` | `-f` | Attach local files to the context |
| `--thinking` | — | Show reasoning blocks |
| `--auto` | — | Auto-approve operations not denied in config |

Authenticated providers (verified 2026-07-06 with `opencode models` and `opencode auth list`):

- `opencode-go/` — OpenCode Go account models.
- `opencode/` — **OpenCode Zen** gateway, free models (no cost); same credential.

Available models, in priority order (1 = use first):

| Priority | Model | ID for `-m` | Suggested use |
|----------|-------|-------------|---------------|
| 1 | MiniMax M3 | `opencode-go/minimax-m3` | code subtasks (`agent` role) |
| 2 | DeepSeek V4 Flash | `opencode-go/deepseek-v4-flash` | trivial / high-volume tasks (`fast` role) |
| 3 | Kimi K2.7-code | `opencode-go/kimi-k2.7-code` | alternative heavy coding |
| 4 | DeepSeek V4 Flash free | `opencode/deepseek-v4-flash-free` | Zen: `fast` role at no cost |
| 5 | Nemotron 3 Ultra free | `opencode/nemotron-3-ultra-free` | Zen: backup |
| 6 | GLM 5.2 | `opencode-go/glm-5.2` | last resort |

Other Zen models available as replacements: `opencode/big-pickle`, `opencode/mimo-v2.5-free`, `opencode/north-mini-code-free`.

Fallback: if a run fails or returns garbage, retry ONCE with the next model in priority; if it fails again, the orchestrator performs the step directly and reports it.

Note: do not use the company name as provider: `minimax/minimax-m3` returns "Unexpected server error".

## Implementation model and consultations

The worker that writes code is **always a lesser model than the one that analyzed the task**; the provider's top model is reserved for consultations. Brand names age — verified 2026-07-29.

| Provider | Implementation worker | Consultation model |
|----------|----------------------|--------------------|
| Anthropic | Sonnet | Opus or Fable — not the model that spawned the worker |
| OpenAI | Luna/medium or Tierra/medium | SOL/high |
| Other providers | ask the user | ask the user |

- On Anthropic, the consultant is the top model that is **not** the caller: if the orchestrator runs on Fable, consultations go to Opus, and vice versa. If the orchestrator is itself the consultation model, it answers the consultation directly instead of spawning another agent.
- On OpenAI, the implementation worker runs Luna or Tierra at `medium` effort and consults SOL at `high` effort.

### What a consultation is for

- code review of the diff the worker just produced;
- hunting for improvements over a step that already meets its criterion;
- a fix when the worker is blocked or its own attempt fails verification.

### Consultation rules

- **Read-only**: the consultant returns findings, never writes files. Backend A — launch it with the `review` role. Backend B — run it **without `--auto`** so it cannot write to disk:
  ```sh
  timeout 240 opencode run "CONSULTATION PROMPT" -m CONSULTATION_MODEL
  ```
- The consultation prompt carries the same subtask contract plus the actual diff (`git diff`) and the step's success criterion. Never the whole repo.
- The worker (or the orchestrator) applies the accepted findings; the orchestrator still verifies the result against the success criterion.
- One consultation per step by default. If a second one is needed, the orchestrator takes the step over directly and reports it.
- A consultation never lifts a guardrail: R4–R5 stay with the orchestrator, and no secrets go into the consultation prompt.
- Consultations do not replace the R3+ critical review in the Definition of done.

## Post-implementation audit agent

Requested by the user before implementation starts (see `SKILL.md` → `## Post-implementation audit`). It runs once, over the whole task diff, on the **model the user named** — not on the default consultation model. **The procedure belongs to the `task-audit` skill**; this section only covers how to launch it from an orchestration.

- The audit agent runs `task-audit` with scope `diff` and the requested model.
- Backend A: launch it as a subagent with the `review` role, on the requested model, with no write tools beyond the ones `task-audit` allows for tests.
- Backend B: run it **without `--auto`** so it cannot write to disk:
  ```sh
  timeout 240 opencode run "AUDIT PROMPT" -m REQUESTED_MODEL
  ```
- Prompt contents: the task's full diff (`git diff BASE...task/SLUG`), the plan's success criteria, and the code-quality standards. Never the whole repo, never secrets.
- Output is findings only. The orchestrator decides which in-scope findings to apply, applies them (or hands them to a worker), and re-runs the Definition of done. Out-of-scope findings stay as the fix tasks `task-audit` registered.
- If the requested model is unavailable, report it and ask the user — do not substitute a model silently and do not skip the audit on your own.

## Verification and integration (mandatory per worker)

1. When each worker finishes, the orchestrator reviews the real diff against the step's success criterion (do not trust the worker's self-report).
2. Fix or redo whatever does not comply; minor improvements are applied directly by the orchestrator.
3. Once all steps are integrated, run the Definition of done over the whole.

## Orchestration guardrails

- **FORBIDDEN: running content-deleting commands through workers** (`rm`, `DROP`, mass `DELETE`, `--force`). If a step requires one, the orchestrator runs it with user confirmation.
- R4–R5 are never delegated: the orchestrator executes them after user confirmation.
- Never pass secrets or credentials in worker prompts.
- Maximum 3 workers in parallel; a file belongs to one worker at a time; each writing worker has its own worktree.
- Every worker result is verified before integration (see above).
