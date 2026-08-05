# Audit checklist

Sweep the scope against these axes. Each item is a **candidate generator**, not a finding: everything found here still has to pass `SKILL.md` → `## Verification` before it is reported.

Order matters. Run the axes top to bottom — correctness and security findings change what is worth looking at further down.

## 1. Correctness and failure modes

- Does the code do what its plan / name / docstring says? Compare against `.tasks/plans/` when the scope maps to a task.
- Boundary conditions: empty collection, single element, zero, negative, max value, unicode, very long input.
- Null / undefined / `nil` / `None` handling on every value that crosses a boundary (params, API responses, DB rows, config).
- Off-by-one in slices, ranges, pagination, retries.
- Type coercion and silent conversions (string↔number, truthiness, implicit casts).
- Return-value contracts: does every branch return? Are errors returned or swallowed?
- Time and locale: timezone assumptions, DST, date parsing without an explicit format, `now()` captured once vs per-iteration.
- Money and precision: floating point on currency, rounding direction, unit mismatches (cents vs units).
- State mutation: shared/global state, mutable default arguments, aliasing a caller's structure.

## 2. Error handling

- Empty catch blocks, `catch { }`, ignored error returns, `_ = err`.
- Errors caught and re-thrown with the original context lost.
- Failure paths that leave the system half-written (partial writes, no transaction, no compensation).
- Resources not released on the error path: files, connections, locks, cursors, subscriptions.
- Retries without a cap, without backoff, or on non-idempotent operations.
- Timeouts absent on every network / external call.
- User-facing error messages leaking internals (stack traces, SQL, file paths, hostnames).

## 3. Security

Name the **source** (attacker-controlled input) and the **sink** it reaches, or the candidate is discarded.

- **Injection**: SQL/NoSQL built by string concatenation, shell commands from input, `eval`/dynamic import, template injection, path traversal in file operations.
- **Authorization**: every new or modified endpoint/handler — is authn checked, and is authz checked *for the specific resource*? Look for IDOR (an id from the request used without an ownership check).
- **Privilege**: role checks that can be bypassed by ordering, missing `root`/admin protection, permissions widened beyond need.
- **Secrets**: credentials, tokens, keys hardcoded or committed; secrets in logs, error messages, or client-side bundles. Report location and type, **never the value**.
- **Crypto**: home-made crypto, weak/legacy algorithms, static IVs, unsalted or fast password hashing, predictable randomness for security purposes.
- **Sessions and tokens**: missing expiry, no rotation on privilege change, tokens in URLs, cookies without `HttpOnly`/`Secure`/`SameSite`.
- **Web**: XSS via unescaped output or `innerHTML`/`dangerouslySetInnerHTML`, missing CSRF protection on state-changing requests, over-permissive CORS (`*` with credentials), open redirects.
- **Deserialization** of untrusted data into objects; unbounded payload sizes; zip/archive extraction without path checks.
- **Rate limiting** absent on auth, password reset, and expensive endpoints.
- **Mass assignment**: request body bound directly to a model, letting a client set fields it should not.

## 4. Input validation and sanitization

- Is every external input validated at the boundary — HTTP params, body, headers, CLI args, env vars, file contents, queue messages, third-party API responses?
- Validation on **type, range, length, format, and allowed set** — not just presence.
- Allowlist over denylist. Denylists on input are a finding by default.
- Validation duplicated in the client only, or in the client and never on the server.
- Sanitization applied at the right place: **encode on output for the target context** (HTML, SQL, shell, URL, log), validate on input. Sanitizing on input alone is a finding.
- Normalization before comparison (unicode, case, trailing whitespace, path normalization) — especially before a security decision.
- Size limits: unbounded request bodies, uploads, array lengths, recursion depth.

## 5. Performance

Every candidate needs a measurement or a complexity argument tied to real data volume in this system.

- **N+1 queries** and queries inside loops; missing eager loading.
- I/O inside loops: HTTP calls, file reads, disk writes that could be batched.
- Missing indexes for the columns actually filtered/sorted on; full scans on growing tables.
- `SELECT *` on wide tables, unbounded queries with no `LIMIT`, pagination absent on list endpoints.
- Algorithmic complexity: nested loops over collections that grow, repeated linear lookups where a map would do, sorting inside a loop.
- Loading a whole dataset into memory when streaming or chunking would do.
- Repeated recomputation of a value that does not change; missing memoization on a hot path.
- Blocking calls on an async/event loop; sync I/O in a request handler.
- Concurrency: unnecessary serialization of independent work; conversely, unbounded parallelism with no pool.
- Caching: absent where it obviously pays, or present with no invalidation story.
- Frontend: bundle bloat, render loops, unkeyed lists, unmemoized expensive renders, layout thrash.

## 6. Concurrency and data integrity

- Race conditions on shared state; check-then-act without atomicity.
- Missing transactions around multi-step writes; transaction scope too wide (long locks) or too narrow.
- Lost updates: read-modify-write with no optimistic locking or version column.
- Idempotency absent on operations that can be retried (webhooks, payments, queue consumers).
- Deadlock-prone lock ordering; locks held across I/O.

## 7. Tests

- Which behavior added or changed by this scope has **no test at all**? Name the behavior, not the percentage.
- Tests that assert nothing, or assert only that no exception was thrown.
- Tests coupled to implementation details, so a valid refactor breaks them.
- Missing tests on the paths that matter most: error branches, boundaries, authorization, money, concurrency.
- Flaky patterns: real time (`sleep`, `now()`), real network, shared mutable fixtures, order-dependent tests.
- Tests hitting real external services or a live database.

## 8. Dependencies

- Known-vulnerable versions (check the project's own audit tooling if it exists; report the tool and its output).
- Unpinned or floating versions in a lockfile-less project.
- Dependencies pulled in for trivial functionality; duplicated libraries doing the same job.
- Abandoned packages (no releases, archived upstream) on a critical path.
- License incompatibility, when the project declares a license.

## 9. Observability

- Logging of sensitive data: passwords, tokens, PII, full request bodies, card data.
- No logging at all on failure paths — errors that vanish silently.
- Log levels misused (everything at `info`, or errors at `debug`).
- No correlation id / request id across a multi-service flow.
- Metrics or health checks absent on a service that needs them.

## 10. Maintainability

Lowest priority. Only report when it is concrete and has real cost — never as a style preference.

- Duplicated logic that has already drifted between copies (show both).
- Dead code: unreachable branches, unused exports, commented-out blocks.
- Hardcoded values that should be config or named constants (per `task-impl/references/code-quality.md`).
- Functions doing several unrelated things, deep nesting that hides a branch.
- Critical `TODO`/`FIXME` left in the audited scope.
- Public API drift: docs, types, or callers out of sync with the implementation.
