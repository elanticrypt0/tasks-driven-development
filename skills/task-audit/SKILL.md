---
name: task-audit
description: 'Audit existing code and turn verified findings into planned fix tasks. Reviews correctness, test coverage, security, input sanitization, performance, error handling and dependencies. Use when the user asks to audit, review, or harden code ("audit this module", "review the generated code", "find bugs and improvements", "auditar el código", "revisar seguridad", "plan de fix"). Also invoked by `task-impl` as its post-implementation audit over a task diff.'
---

# Task Audit

Audit mode of the Tasks-Driven-Development workflow. Inspect code that already exists, produce **verified** findings, and turn them into fix tasks that `task-impl` can execute. **Never fix production code in this mode** — the output is an audit report plus planned fix tasks. The only files this skill may write are tests, on its own branch (see `## Tests`).

Runs standalone (audit any scope, any time) or as the post-implementation audit requested inside `task-impl` (see `## Invoked from task-impl`).

## Scope selection (first step, always)

An audit without a scope produces noise. Establish the scope before reading anything, and state it in the first reply:

| Scope | Meaning | Typical trigger |
|-------|---------|-----------------|
| `diff` | `git diff <base>...<branch>` — only what a task changed | post-implementation audit |
| `module` | one directory, package, or feature | "audit the payments module" |
| `repo` | the whole codebase | "audit the project" |
| `axis` | one dimension across the codebase (e.g. security only) | "review input validation everywhere" |

For `repo` scope, first map the codebase and propose a prioritized order (entry points, auth, data access, external I/O first), then audit in that order. Do not attempt a uniform sweep of everything.

## Model requirement

Auditing is analysis: it MUST run on the most powerful model available from the current provider, same rule and resolution chain as `task-plan` (see `task-plan/references/model-selection.md`). If the session is on a weaker model, say so and ask whether to continue or switch.

Exception: when `task-impl` passes a user-named model for the audit, run on **that exact model**. If it is unavailable, report it and ask — never substitute silently, never skip the audit on your own.

## Principles

- **Evidence or nothing.** Every finding cites `path:line` and a concrete failure scenario (input or state → wrong output, crash, or leak). No evidence, no finding.
- Read the real code before claiming anything. Never assume names, paths, or behavior.
- Report what is broken, not what is different from your taste. Style preferences are not findings.
- Cite `path:line` instead of dumping file contents.
- Batch all questions into a single message.
- Rank by impact. A ranked list of 8 real problems beats 40 observations.

## Audit flow

1. **Scope** — establish and state it (see above). Confirm the base branch when scope is `diff`.
2. **Recon** — read the project's own docs first (`docs/AGENTS.md`, `CLAUDE.md`, README, `.tasks/plans/`) for stack, build/test commands, and conventions. Trust manifests and config over prose.
3. **Collect candidates** — sweep the scope against `references/audit-checklist.md`. Candidates are suspicions, not findings yet.
4. **Verify each candidate** against the real code (see `## Verification`). Discard whatever does not survive.
5. **Prove with tests** (optional, see `## Tests`) — write characterization or regression tests for the findings worth proving.
6. **Write the report** at `.tasks/audits/audit_<scope>_<YYYY-MM-DD>.md`.
7. **Generate fix tasks** — one planned task per accepted finding, per `## Fix tasks`.
8. **Close** per `## Closing`.

## Verification (mandatory before reporting)

Every candidate ends in one of three states. Only the first two are reported:

- **CONFIRMED** — traced end-to-end in the code, or reproduced by a failing test or a command that was actually run. Say which.
- **PLAUSIBLE** — the code path is real and the risk is real, but it was not reproduced. Say what would confirm it.
- **DISCARDED** — could not be traced, already handled elsewhere, or unreachable in practice. Not reported individually; report the count.

Additional bars, non-negotiable:

- A **performance** finding needs a measurement, or a complexity argument tied to a real data volume in this system ("this list is loaded per request and grows with orders"). "This could be slow" is discarded.
- A **security** finding names the attacker-controlled input and the path it reaches. "Unsanitized input" without a source and a sink is discarded.
- A **test coverage** finding names the specific untested behavior and why it matters. "Coverage is low" is discarded.

State the discarded count in the report — it is how the reader knows the list was filtered.

## Severity and risk

Two different scales, both required on every finding:

**Severity** — how bad it is if unaddressed:

| Severity | Meaning |
|----------|---------|
| `critical` | Exploitable security hole, data loss/corruption, or the feature is broken in normal use. |
| `high` | Wrong results in realistic conditions, unhandled failure mode, or a serious performance cliff. |
| `medium` | Edge case, missing validation with limited reach, avoidable inefficiency, missing test on real logic. |
| `low` | Maintainability, duplication, dead code, hardening with no known exploit path. |

**Risk (R1–R5)** — how dangerous the *fix* is, using the scale in `task-plan` (`## Risk scale`). It governs the fix task, not the finding.

They are independent: a `critical` finding can have an `R1` fix (add a missing check), and a `low` finding can be `R4` (touches a migration). Never collapse them into one number.

## Finding format

Each finding in the report:

```md
### F<n> — <short title>
- **Severity**: critical | high | medium | low
- **Status**: CONFIRMED | PLAUSIBLE
- **Location**: `path/to/file.ext:120-134`
- **Problem**: what is wrong, in one or two sentences.
- **Failure scenario**: concrete input or state → wrong outcome.
- **Evidence**: what was traced, run, or the test that fails (`path/to/test:12`).
- **Proposed fix**: the direction, not the patch.
- **Fix risk**: R<n> — <why> · revert: <how>
```

## Report format (`.tasks/audits/`)

`.tasks/audits/audit_<scope>_<YYYY-MM-DD>.md` (create the directory if missing):

1. **Header** — scope, base/branch or paths audited, date, model used, commands actually run.
2. **Summary** — counts by severity, and the discarded count.
3. **Findings** — ordered by severity, then by impact. Format above.
4. **Fix plan** — the ordered list of fix tasks generated, with dependencies between them.
5. **Not audited** — what the scope deliberately left out. Never let the reader assume more was covered than was.

## Fix tasks (handoff to implementation)

Accepted findings become tasks in `.tasks/tasks.md`, planned and ready for `task-impl`:

- **Grouping**: `critical` and `high` get their own task each. `medium`/`low` findings that share a file or module may be grouped into a single hardening task — never group across modules, and never group a security finding with a cleanup.
- **One task = one goal**, same rule as implementation.
- Register the header in `.tasks/tasks.md` in the standard format, with the audit as the origin:
  ```
  ** <title> — planned · R<n> · plan: .tasks/plans/plan_fix_<slug>.md · audit: F3 **
  ```
- Write the plan at `.tasks/plans/plan_fix_<slug>.md` using the **`task-plan` plan format** (affected files, action, verifiable success criterion, step risk + revert, `parallelizable: yes|no`). This skill runs on the analysis-grade model, so it writes the plan directly instead of round-tripping through `task-plan`.
- The success criterion must be checkable. When a test was written for the finding, the criterion is "test `path:line` passes".
- **R4–R5 fix tasks are confirmed with the user before any execution**, per the standard risk rules. Say so explicitly in the closing.
- Do not register a fix task for a `PLAUSIBLE` finding without saying it is unconfirmed in the plan's first line.

## Tests

Tests are the only files this skill may write, and they exist to **prove findings**, not to raise a coverage number.

- **Branch**: `audit/<slug>`, created off the audited branch (scope `diff`) or the up-to-date base branch (any other scope). Never write tests on the base branch, never on someone else's task branch.
  ```sh
  git checkout -b audit/<slug> <base-or-task-branch>
  ```
- **Test files only.** Do not touch production code — not even to make a test pass. If a test cannot be written without changing production code, that is a finding (untestable design), not a license to edit.
- A test that reproduces a bug is **expected to fail**. Leave it failing, reference it from the finding, and say so in the report. Only mark it skipped/expected-failure if the project's convention requires a green suite — and say which you did.
- Tests must be deterministic, offline, and non-destructive: no real external API calls (LLM providers, payment gateways, third-party services), no writes against a live database, no filesystem changes outside temp dirs.
- If the project has no test infrastructure, do not scaffold a framework unprompted. Report it as a finding and propose the setup as its own fix task.
- Follow the project's existing test conventions and the standards in `task-impl/references/code-quality.md`.
- **Merge is manual**: the `audit/<slug>` branch is left for a human to review and merge, exactly like task branches.

## Parallel auditing

An audit may be split across read-only workers when the scope is large:

- Split by **module** or by **axis** (one worker on security, one on performance), never both at once.
- Workers are **read-only**: no writes, no git commands. They return candidates, not findings.
- Maximum 3 workers in parallel.
- **The orchestrator verifies every candidate itself** before it becomes a finding. Never promote a worker's self-reported issue straight into the report.
- Delegation mechanics (backends, subtask contract, invocation) are in `task-impl/references/orchestration.md`.

## Invoked from task-impl

When `task-impl` calls this skill as the post-implementation audit:

- Scope is `diff`: `git diff <base>...task/<slug>`, checked against the plan's success criteria plus the checklist.
- Run on the model the user named in the implementation flow, not on the default.
- **Findings inside the audited task's scope** are returned to `task-impl` to apply immediately — do not register them as separate fix tasks.
- **Findings outside that scope** (pre-existing problems the diff merely revealed) are registered as fix tasks per `## Fix tasks`, so they are not lost and not smuggled into the current task.
- Write the report anyway; a task whose audit left no trace cannot be reviewed later.
- The audit never lifts a guardrail: R4–R5 stay with the orchestrator, no secrets go into the prompt, and it does not replace the R3+ critical review in the implementation's Definition of done.

## Guardrails (non-negotiable)

- **Read-only on production code.** No fixes, no refactors, no "while I was there" edits. Tests on `audit/<slug>` are the only exception.
- No destructive commands (`rm`, `DROP`, mass `DELETE`, `--force`), ever — not to reproduce a finding, not to clean up.
- **Never trigger real external API calls** (LLM providers, payment gateways, paid services) to verify something. Use `--dry-run` where it exists, or leave the invocation to the user.
- Do not execute untrusted or unknown scripts to "see if it works". Prefer reading. If execution is needed, use only the project's declared build/test commands, and report exactly what was run.
- **Redact secrets.** If a credential is found in the code, report its location and type — never its value, not in the report, not in a prompt, not in a fix task.
- Do not weaken a guardrail because a finding would be easier to prove without it.

## Handoff contract (output of this skill)

The audit is complete when:

1. The report exists at `.tasks/audits/audit_<scope>_<date>.md`, with every finding CONFIRMED or PLAUSIBLE and the discarded count stated.
2. Each accepted finding has a fix task in `.tasks/tasks.md` marked `planned` with `R<n>` and a plan path — or is explicitly listed as accepted-without-task, with the reason.
3. If tests were written, they live on `audit/<slug>` and the report says which findings they prove and which of them currently fail.

`task-impl` can consume any of those fix tasks directly; no re-analysis is needed.

## Closing (max 4 lines)

```md
Audit: <scope> — <n> findings (<n> critical, <n> high, <n> medium, <n> low), <n> discarded.
Report: .tasks/audits/audit_<scope>_<date>.md
Tasks: <n> fix tasks planned<, of which N are R4-R5 and need your confirmation before execution>.
Next: <highest-impact next step, or "nothing">.
```
