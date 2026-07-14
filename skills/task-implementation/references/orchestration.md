# Orchestration

The orchestrator (`primary` role) distributes plan steps among **workers** and verifies every result. A worker can be a **native subagent** (backend A) or an **`opencode` run** with another model (backend B).

Environment-specific details (model tables, flags, known bugs) age: each carries its verification date. Renew dates when re-verified.

## When to orchestrate

Orchestrate only if ALL of these hold:

- The plan has ≥2 steps marked `parallelizable: yes`.
- Every delegable step is R1–R3 (R4–R5 are never delegated).
- Steps share no files: a file belongs to one worker at a time.

Otherwise execute in series with the primary role. Orchestrating a trivial task costs more than it saves.

## Subtask contract (applies to both backends)

Every delegation includes, in the worker's prompt:

- a precise goal and the exact files to touch (real paths);
- the minimum necessary context (do not dump the whole repo);
- a verifiable success criterion for the step;
- restrictions: do not delete content, do not touch files outside the list, do not introduce secrets or hardcoded values;
- expected output format (applied diff + short summary);
- if the step depends on an integration, which skill to use (workers see project and global skills — verified 2026-07-07).

## Backend A — native subagents

Use when the runtime offers subagents (e.g. Task/Agent in Claude Code).

- Launch independent workers in parallel with the `agent` role; search/exploration with the `fast` role.
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

## Verification and integration (mandatory per worker)

1. When each worker finishes, the orchestrator reviews the real diff against the step's success criterion (do not trust the worker's self-report).
2. Fix or redo whatever does not comply; minor improvements are applied directly by the orchestrator.
3. Once all steps are integrated, run the Definition of done over the whole.

## Orchestration guardrails

- **FORBIDDEN: running content-deleting commands through workers** (`rm`, `DROP`, mass `DELETE`, `--force`). If a step requires one, the orchestrator runs it with user confirmation.
- R4–R5 are never delegated: the orchestrator executes them after user confirmation.
- Never pass secrets or credentials in worker prompts.
- Maximum 3 workers in parallel; a file belongs to one worker at a time.
- Every worker result is verified before integration (see above).
