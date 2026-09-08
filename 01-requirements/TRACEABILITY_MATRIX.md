# Traceability Matrix — `taskq-api`

> Requirements baseline: `01-requirements/SRS.md`  
> Requirement registry: `01-requirements/SPEC_TRACKING.md`  
> Canonical specification: `SPEC.md` v1.0.0  
> Matrix version: v1.0 · 2026-09-09

## 1. Purpose and lifecycle state

This matrix establishes bidirectional links from every SRS requirement and acceptance criterion to a planned design element and a planned test case. Phase 2 must reconcile the logical design IDs with `02-architecture/SAD.md` and refine the test definitions in `02-architecture/TEST_SPEC.md`; `01-requirements/TEST_INVENTORY.yaml` is the naming authority for the concrete test functions. Phase 4 execution evidence belongs in `04-testing/TEST_RESULTS.md`, and final verification evidence belongs in `05-verification/VERIFICATION_REPORT.md`.

No implementation or test-execution evidence exists at this Phase 1 baseline. Every link below is therefore `PLANNED`; none is `VERIFIED`. Per SRS AC-N9.6, a row may become `VERIFIED` only after its named test cases have actually run and passed. The `DRAFT` requirement status in `01-requirements/SPEC_TRACKING.md` is not upgraded by this document.

## 2. Trace conventions

| Item | Convention |
|---|---|
| Requirement source | `FR-01`–`FR-10` and `NFR-01`–`NFR-12` are defined in SRS §§3–4. |
| Acceptance source | Each `AC-*` identifier is the stable acceptance identifier in the SRS. |
| Design link | `DE-*` is a logical responsibility allocation, not a claim that a code file already exists. Phase 2 owns its physical module allocation. |
| Test link | `TC-*` is an independently executable planned case. Concrete Python function names must be assigned one-to-one in `01-requirements/TEST_INVENTORY.yaml`. |
| Range notation | `TC-FR01-01a..01c` expands to three distinct IDs: `TC-FR01-01a`, `TC-FR01-01b`, and `TC-FR01-01c`; downstream inventories must not collapse them into one loop or one record. |
| Forward direction | Requirement/AC → design element(s) → test case(s), in §§4–5. |
| Reverse direction | Design element → requirement/AC/test family in §6; test family → requirement/AC/design in §7. |
| Link state | `PLANNED` = linked but not implemented or executed; `VERIFIED` requires passing execution evidence. |
| Dagger (`†`) | A stable testable core exists, but one part of the final oracle depends on a canonical ambiguity recorded in SRS §7 and §9 below. |

## 3. Planned design-element catalogue

| Design ID | Logical responsibility |
|---|---|
| DE-HTTP-TASKS | FastAPI task CRUD routes, query parsing, response status, and route metadata. |
| DE-HTTP-RUNS | Task-run submission and run-history routes. |
| DE-HTTP-OPS | Liveness, readiness, and metrics routes. |
| DE-DTO | Pydantic v2 request/response and problem-detail models. |
| DE-AUTHN | API-key extraction, hashing, constant-time comparison, revocation, and principal creation. |
| DE-AUTHZ | Single shared hierarchical-scope dependency applied to every protected route. |
| DE-RATE | Persistent per-token token-bucket policy and rejection accounting. |
| DE-RUNNER | Bounded async subprocess execution, state transitions, timeout cleanup, cancellation, and drain. |
| DE-REPO-TASKS | Repository operations for tasks, runs, tags, eager loading, filtering, and pagination. |
| DE-REPO-KEYS | Repository operations for API keys. |
| DE-REPO-RATE | Repository operations and concurrency control for rate buckets. |
| DE-SESSION | Engine/pool setup and request-scoped transaction context manager. |
| DE-DATA-MODEL | Shared SQLAlchemy declarative model for SQLite and PostgreSQL. |
| DE-MIG-V1 | Alembic v1 schema upgrade/downgrade. |
| DE-MIG-V2 | Alembic v2 tags/name-index upgrade/downgrade. |
| DE-MIG-V3 | Alembic v3 reversible result-data migration. |
| DE-PROBLEM | RFC 7807 handlers, status/type mapping, and correlation propagation. |
| DE-REDACTION | Whole-line sensitive-data redaction before persistence or emission. |
| DE-CONFIG | Typed loading and validation of the 12 canonical `TASKQ_*` settings. |
| DE-OBSERVABILITY | Structured logging and operational counters/latency summaries. |
| DE-LIFECYCLE | ASGI startup/shutdown coordination and runner drain. |
| DE-CLI | `migrate`, `seed`, `healthcheck`, and key-creation command surface. |
| DE-ARCH-GUARD | Import-linter layers, independence, and SQLAlchemy forbidden contracts. |
| DE-DEPENDENCIES | Direct/transitive pinning, license allowlist evidence, and SBOM generation. |
| DE-DOC | FR/NFR-cited public docstrings and complete OpenAPI descriptions. |
| DE-TEST-GOV | Collection, assertion, no-skip, coverage, mutation, and truthful-status controls. |
| DE-QUALITY | MI, complexity, file/directory, and handler-size controls. |
| DE-SYSTEM-VERIFY | Ordered `verify-system` orchestration and PASS marker. |

## 4. Functional requirement → design → test

| Requirement / criterion | Testable acceptance oracle | Design element(s) | Planned test case(s) | State |
|---|---|---|---|---|
| FR-01 / AC-1.1 | Valid `POST /v1/tasks` with write authority validates through `TaskCreate`, returns 201, and includes a task id. | DE-HTTP-TASKS, DE-DTO, DE-AUTHZ, DE-REPO-TASKS | TC-FR01-01 | PLANNED |
| FR-01 / AC-1.2 | Authorized `GET /v1/tasks/{id}` returns the full stored task. | DE-HTTP-TASKS, DE-AUTHZ, DE-REPO-TASKS | TC-FR01-02 | PLANNED |
| FR-01 / AC-1.3 | List supports status filtering, limit, and cursor. | DE-HTTP-TASKS, DE-REPO-TASKS | TC-FR01-03a, TC-FR01-03b, TC-FR01-03c | PLANNED |
| FR-01 / AC-1.4 | Pagination is cursor-only; default limit is 50, limit 200 is accepted, and limit 201 returns 422. | DE-HTTP-TASKS, DE-DTO, DE-REPO-TASKS | TC-FR01-04a, TC-FR01-04b, TC-FR01-04c, TC-FR01-04d | PLANNED |
| FR-01 / AC-1.5 | Admin delete removes the task and result rows atomically. | DE-HTTP-TASKS, DE-AUTHZ, DE-REPO-TASKS, DE-SESSION | TC-FR01-05 | PLANNED |
| FR-01 / AC-1.6† | Each specified empty, over-1000, and injection-blacklist violation returns 422 problem details. | DE-DTO, DE-HTTP-TASKS, DE-PROBLEM | TC-FR01-06a, TC-FR01-06b, TC-FR01-06c | PLANNED |
| FR-01 / AC-1.7 | Unknown task id returns 404 problem details with `/errors/not-found`. | DE-HTTP-TASKS, DE-PROBLEM | TC-FR01-07 | PLANNED |
| FR-01 / AC-1.8 | Duplicate task name returns 409 with `/errors/conflict`. | DE-HTTP-TASKS, DE-REPO-TASKS, DE-PROBLEM | TC-FR01-08 | PLANNED |
| FR-02 / AC-2.1 | Authorized run submission returns 202 and a `run_id`. | DE-HTTP-RUNS, DE-AUTHZ, DE-RUNNER | TC-FR02-01 | PLANNED |
| FR-02 / AC-2.2 | Runner passes `shlex.split` arguments to `create_subprocess_exec`, never enables a shell, and applies `TASKQ_TASK_TIMEOUT`. | DE-RUNNER, DE-CONFIG | TC-FR02-02a, TC-FR02-02b | PLANNED |
| FR-02 / AC-2.3 | Execution follows pending → running and terminates as done, failed, or timeout. | DE-RUNNER, DE-REPO-TASKS | TC-FR02-03a, TC-FR02-03b, TC-FR02-03c | PLANNED |
| FR-02 / AC-2.4† | A completed run persists the five named execution-result fields and associates them with the correct task in `task_results`. | DE-RUNNER, DE-REPO-TASKS, DE-DATA-MODEL | TC-FR02-04 | PLANNED |
| FR-02 / AC-2.5 | Authorized run history returns only the task's runs, newest first. | DE-HTTP-RUNS, DE-AUTHZ, DE-REPO-TASKS | TC-FR02-05 | PLANNED |
| FR-03 / AC-3.1 | Missing and invalid API keys each return 401 problem details with `/errors/unauthenticated`. | DE-AUTHN, DE-PROBLEM | TC-FR03-01a, TC-FR03-01b | PLANNED |
| FR-03 / AC-3.2 | Storage contains only a 64-hex SHA-256 hash, and runtime comparison uses `hmac.compare_digest`. | DE-AUTHN, DE-REPO-KEYS | TC-FR03-02a, TC-FR03-02b | PLANNED |
| FR-03 / AC-3.3 | `key create --scope` creates a key and emits plaintext exactly once at creation. | DE-CLI, DE-AUTHN, DE-REPO-KEYS | TC-FR03-03 | PLANNED |
| FR-03 / AC-3.4 | A key with non-null `revoked_at` is rejected. | DE-AUTHN, DE-REPO-KEYS | TC-FR03-04 | PLANNED |
| FR-03 / AC-3.5 | `/healthz` and `/readyz` succeed without API-key authentication. | DE-HTTP-OPS, DE-AUTHN | TC-FR03-05a, TC-FR03-05b | PLANNED |
| FR-04 / AC-4.1 | Read, write, and admin principals obey `read < write < admin` inclusion. | DE-AUTHZ | TC-FR04-01a, TC-FR04-01b, TC-FR04-01c | PLANNED |
| FR-04 / AC-4.2 | Insufficient scope returns 403 `/errors/forbidden`, and existing/non-existing resource responses are indistinguishable. | DE-AUTHZ, DE-PROBLEM | TC-FR04-02a, TC-FR04-02b | PLANNED |
| FR-04 / AC-4.3 | Every `/v1` route declares the same authorization dependency; handlers contain no alternate authorization decisions. | DE-AUTHZ, DE-HTTP-TASKS, DE-HTTP-RUNS, DE-HTTP-OPS | TC-FR04-03 | PLANNED |
| FR-05 / AC-5.1 | Bucket capacity and refill follow configured burst and per-second rate for each token independently. | DE-RATE, DE-CONFIG, DE-REPO-RATE | TC-FR05-01a, TC-FR05-01b | PLANNED |
| FR-05 / AC-5.2 | An over-limit request returns 429 problem details, `/errors/rate-limited`, and numeric-seconds `Retry-After`. | DE-RATE, DE-PROBLEM | TC-FR05-02 | PLANNED |
| FR-05 / AC-5.3† | Bucket state survives worker/request boundaries and concurrent updates do not over-admit; update and lock occur in one transaction. | DE-RATE, DE-REPO-RATE, DE-SESSION | TC-FR05-03a, TC-FR05-03b | PLANNED |
| FR-05 / AC-5.4 | Health and readiness routes remain callable beyond a token's limit. | DE-RATE, DE-HTTP-OPS | TC-FR05-04a, TC-FR05-04b | PLANNED |
| FR-06 / AC-6.1 | Static architecture evidence shows all data access in repositories and no service-held `Session`. | DE-REPO-TASKS, DE-REPO-KEYS, DE-REPO-RATE, DE-ARCH-GUARD | TC-FR06-01 | PLANNED |
| FR-06 / AC-6.2 | One request-scoped session commits on success and rolls back on exception through its context manager. | DE-SESSION | TC-FR06-02a, TC-FR06-02b | PLANNED |
| FR-06 / AC-6.3 | Source scan/review finds no interpolated SQL; repository queries are ORM or parameterized. | DE-REPO-TASKS, DE-REPO-KEYS, DE-REPO-RATE, DE-ARCH-GUARD | TC-FR06-03 | PLANNED |
| FR-06 / AC-6.4 | Relationship loading is explicit and list SQL-statement count remains constant as result count changes. | DE-REPO-TASKS | TC-FR06-04 | PLANNED |
| FR-06 / AC-6.5 | Engine construction applies configured pool size, `pool_pre_ping=True`, and the canonical default URL. | DE-SESSION, DE-CONFIG | TC-FR06-05 | PLANNED |
| FR-07 / AC-7.1 | v1, v2, and v3 each upgrade and downgrade successfully. | DE-MIG-V1, DE-MIG-V2, DE-MIG-V3 | TC-FR07-01a, TC-FR07-01b, TC-FR07-01c | PLANNED |
| FR-07 / AC-7.2† | At the v1 boundary the canonical v1 table/column set exists; downgrade removes that set. | DE-MIG-V1, DE-DATA-MODEL | TC-FR07-02a, TC-FR07-02b | PLANNED |
| FR-07 / AC-7.3 | v2 creates tags, task-tags, and the unique name index; downgrade removes only v2 additions and preserves v1 data. | DE-MIG-V2 | TC-FR07-03 | PLANNED |
| FR-07 / AC-7.4 | v3 moves existing result data into `task_results` and downgrade restores it to `tasks.result_json` without loss. | DE-MIG-V3 | TC-FR07-04a, TC-FR07-04b | PLANNED |
| FR-07 / AC-7.5 | Upgrade to head succeeds; downgrade to base exits 0 and leaves no application tables. | DE-MIG-V1, DE-MIG-V2, DE-MIG-V3 | TC-FR07-05a, TC-FR07-05b | PLANNED |
| FR-07 / AC-7.6 | Real-file SQLite v3 round trip preserves every sample value column-by-column. | DE-MIG-V3 | TC-FR07-06 | PLANNED |
| FR-07 / AC-7.7 | Migration source contains no destructive raw-drop shortcut replacing real downgrade logic. | DE-MIG-V1, DE-MIG-V2, DE-MIG-V3 | TC-FR07-07 | PLANNED |
| FR-07 / AC-7.8 | Offline SQL generation executes and assertions cover upgrade and downgrade migration paths. | DE-MIG-V1, DE-MIG-V2, DE-MIG-V3 | TC-FR07-08a, TC-FR07-08b | PLANNED |
| FR-07 / AC-7.9† | Head schema exactly exposes all six canonical tables and their principal fields. | DE-DATA-MODEL, DE-MIG-V1, DE-MIG-V2, DE-MIG-V3 | TC-FR07-09 | PLANNED |
| FR-08 / AC-8.1† | Shutdown drains work within the configured deadline; overdue work becomes interrupted. | DE-RUNNER, DE-LIFECYCLE, DE-CONFIG | TC-FR08-01a, TC-FR08-01b | PLANNED |
| FR-08 / AC-8.2 | Active jobs never exceed configured concurrency, and excess jobs wait rather than spawning unbounded coroutines. | DE-RUNNER, DE-CONFIG | TC-FR08-02a, TC-FR08-02b | PLANNED |
| FR-08 / AC-8.3 | Timeout invokes kill then waits for process reaping, leaving no orphan process. | DE-RUNNER | TC-FR08-03 | PLANNED |
| FR-08 / AC-8.4 | Cancellation propagates as `CancelledError` and is not converted to HTTP 500. | DE-RUNNER, DE-PROBLEM | TC-FR08-04 | PLANNED |
| FR-09 / AC-9.1 | Unauthenticated liveness returns 200 and exactly `{"status":"ok"}` while the process is alive. | DE-HTTP-OPS | TC-FR09-01 | PLANNED |
| FR-09 / AC-9.2 | Readiness returns 200 only with DB connectivity and migration head; DB failure and revision lag each return 503 with the failed item named. | DE-HTTP-OPS, DE-REPO-TASKS, DE-MIG-V3, DE-PROBLEM | TC-FR09-02a, TC-FR09-02b, TC-FR09-02c | PLANNED |
| FR-09 / AC-9.3† | Admin metrics contain per-status task counts, execution-latency percentiles, and rate-limit rejection count. | DE-HTTP-OPS, DE-AUTHZ, DE-OBSERVABILITY | TC-FR09-03 | PLANNED |
| FR-10 / AC-10.1 | Every produced non-2xx response has `application/problem+json` content type. | DE-PROBLEM | TC-FR10-01 | PLANNED |
| FR-10 / AC-10.2 | Problem bodies contain all six required contract fields with valid value types. | DE-PROBLEM, DE-DTO | TC-FR10-02 | PLANNED |
| FR-10 / AC-10.3 | Problem detail contains no SQL, stack trace, file path, or schema disclosure. | DE-PROBLEM, DE-REDACTION | TC-FR10-03 | PLANNED |
| FR-10 / AC-10.4 | One correlation id is identical in response header, body, and captured server log. | DE-PROBLEM, DE-OBSERVABILITY | TC-FR10-04 | PLANNED |
| FR-10 / AC-10.5 | Request validation maps to 422 `/errors/validation`. | DE-PROBLEM | TC-FR10-05 | PLANNED |
| FR-10 / AC-10.6 | Missing/invalid key maps to 401 `/errors/unauthenticated`. | DE-PROBLEM, DE-AUTHN | TC-FR10-06 | PLANNED |
| FR-10 / AC-10.7 | Insufficient scope maps to non-disclosing 403 `/errors/forbidden`. | DE-PROBLEM, DE-AUTHZ | TC-FR10-07 | PLANNED |
| FR-10 / AC-10.8 | Unknown task maps to 404 `/errors/not-found`. | DE-PROBLEM | TC-FR10-08 | PLANNED |
| FR-10 / AC-10.9 | Name conflict maps to 409 `/errors/conflict`. | DE-PROBLEM | TC-FR10-09 | PLANNED |
| FR-10 / AC-10.10 | Rate rejection maps to 429 `/errors/rate-limited` with `Retry-After`. | DE-PROBLEM, DE-RATE | TC-FR10-10 | PLANNED |
| FR-10 / AC-10.11 | DB failure/revision lag maps to 503 `/errors/not-ready`. | DE-PROBLEM, DE-HTTP-OPS | TC-FR10-11 | PLANNED |
| FR-10 / AC-10.12 | Unexpected exception maps to non-disclosing 500 `/errors/internal`. | DE-PROBLEM, DE-REDACTION | TC-FR10-12 | PLANNED |
| FR-10 / AC-10.13 | Task timeout is represented as status `timeout` in a 200 response, not as a problem type. | DE-RUNNER, DE-HTTP-RUNS | TC-FR10-13 | PLANNED |

### 4.1 Functional test subcase expansion

Each suffixed ID in §4 is a separate test case with one primary behavior. This table is normative for downstream one-to-one inventory expansion.

| Test case | Independent behavior |
|---|---|
| TC-FR01-03a | Filter the task list by status. |
| TC-FR01-03b | Restrict the task list with `limit`. |
| TC-FR01-03c | Continue the task list from a cursor. |
| TC-FR01-04a | Prove pagination exposes cursor semantics and no offset parameter. |
| TC-FR01-04b | Omitted limit returns at most the default 50 records. |
| TC-FR01-04c | Limit 200 is accepted. |
| TC-FR01-04d | Limit 201 returns 422. |
| TC-FR01-06a† | A canonical empty-value violation returns 422 problem details. |
| TC-FR01-06b† | A canonical value longer than 1000 returns 422 problem details. |
| TC-FR01-06c† | A canonical blacklisted-injection value returns 422 problem details. |
| TC-FR02-02a | Subprocess receives `shlex.split` arguments and no shell mode. |
| TC-FR02-02b | Subprocess execution honors `TASKQ_TASK_TIMEOUT`. |
| TC-FR02-03a | Successful process transitions pending → running → done. |
| TC-FR02-03b | Non-zero process transitions pending → running → failed. |
| TC-FR02-03c | Overdue process transitions pending → running → timeout. |
| TC-FR03-01a | Missing API key returns the canonical 401 problem. |
| TC-FR03-01b | Invalid API key returns the canonical 401 problem. |
| TC-FR03-02a | Persistent key value is a 64-hex SHA-256 hash with no plaintext. |
| TC-FR03-02b | Credential comparison calls `hmac.compare_digest`. |
| TC-FR03-05a | Liveness requires no key. |
| TC-FR03-05b | Readiness requires no key. |
| TC-FR04-01a | Read key permits read operations but not write/admin operations. |
| TC-FR04-01b | Write key permits read/write operations but not admin operations. |
| TC-FR04-01c | Admin key permits read/write/admin operations. |
| TC-FR04-02a | An under-scoped request returns canonical 403 problem details. |
| TC-FR04-02b | Under-scoped probes of existing and missing ids have indistinguishable bodies. |
| TC-FR05-01a | Each token receives the configured independent burst capacity. |
| TC-FR05-01b | Each bucket refills at the configured per-second rate. |
| TC-FR05-03a† | Bucket state persists across independent requests/workers. |
| TC-FR05-03b† | Concurrent atomic updates do not over-admit and use the resolved locking rule. |
| TC-FR05-04a | Liveness remains exempt after rate exhaustion. |
| TC-FR05-04b | Readiness remains exempt after rate exhaustion. |
| TC-FR06-02a | Successful request commits exactly once. |
| TC-FR06-02b | Exceptional request rolls back and does not commit. |
| TC-FR07-01a | v1 upgrade and downgrade both execute. |
| TC-FR07-01b | v2 upgrade and downgrade both execute. |
| TC-FR07-01c | v3 upgrade and downgrade both execute. |
| TC-FR07-02a† | v1 upgrade exposes the stakeholder-resolved canonical v1 schema. |
| TC-FR07-02b† | v1 downgrade removes exactly that resolved schema. |
| TC-FR07-04a | v3 upgrade migrates existing `result_json` values into result rows. |
| TC-FR07-04b | v3 downgrade restores those values before removing result rows. |
| TC-FR07-05a | Upgrade from base to head exits 0. |
| TC-FR07-05b | Downgrade from head to base exits 0 with no application tables. |
| TC-FR07-08a | Offline upgrade SQL generation succeeds and contains expected operations. |
| TC-FR07-08b | Offline downgrade SQL generation succeeds and contains expected operations. |
| TC-FR08-01a | Shutdown waits for in-flight work that completes within drain timeout. |
| TC-FR08-01b† | Shutdown marks work exceeding drain timeout as `interrupted`. |
| TC-FR08-02a | Observed active-process count never exceeds `TASKQ_MAX_CONCURRENT`. |
| TC-FR08-02b | Excess submissions wait for worker capacity rather than spawning unbounded coroutines. |
| TC-FR09-02a | Connected DB at Alembic head returns ready 200. |
| TC-FR09-02b | DB connection failure returns canonical 503 with DB detail. |
| TC-FR09-02c | Revision behind head returns canonical 503 with migration detail. |

## 5. Non-functional requirement → design → test

| Requirement / criterion | Testable acceptance oracle | Design element(s) | Planned test case(s) | State |
|---|---|---|---|---|
| NFR-01 / AC-N1.1 | With 10,000 rows over ASGI transport, single-task GET p95 is below 30 ms. | DE-HTTP-TASKS, DE-REPO-TASKS | TC-NFR01-01 | PLANNED |
| NFR-01 / AC-N1.2 | With 10,000 rows, 50-task list GET p95 is below 80 ms. | DE-HTTP-TASKS, DE-REPO-TASKS | TC-NFR01-02 | PLANNED |
| NFR-01 / AC-N1.3 | SQLAlchemy event counts stay constant as returned list size changes. | DE-REPO-TASKS | TC-NFR01-03 | PLANNED |
| NFR-02 / AC-N2.1† | Canonical source scan reports zero `shell=True`, `eval(`, and `exec(` hits. | DE-RUNNER, DE-ARCH-GUARD | TC-NFR02-01 | PLANNED |
| NFR-02 / AC-N2.2 | Grep plus code review report zero f-string, percent, or concatenated SQL construction. | DE-REPO-TASKS, DE-REPO-KEYS, DE-REPO-RATE, DE-ARCH-GUARD | TC-NFR02-02 | PLANNED |
| NFR-02 / AC-N2.3 | Runtime/storage evidence proves hashed keys and `compare_digest` comparison. | DE-AUTHN, DE-REPO-KEYS | TC-NFR02-03 | PLANNED |
| NFR-02 / AC-N2.4 | Existing and missing resource probes produce indistinguishable 403 bodies. | DE-AUTHZ, DE-PROBLEM | TC-NFR02-04 | PLANNED |
| NFR-02 / AC-N2.5 | Triggered failures expose no stack, SQL, or path in response bodies. | DE-PROBLEM, DE-REDACTION | TC-NFR02-05 | PLANNED |
| NFR-02 / AC-N2.6 | Default CORS rejects origins; only explicitly configured origins are allowed. | DE-CONFIG, DE-LIFECYCLE | TC-NFR02-06 | PLANNED |
| NFR-02 / AC-N2.7 | Canonical Bandit command reports zero HIGH and zero MEDIUM findings. | DE-ARCH-GUARD | TC-NFR02-07 | PLANNED |
| NFR-03 / AC-N3.1 | Success commits and injected failure rolls back within the request context. | DE-SESSION | TC-NFR03-01 | PLANNED |
| NFR-03 / AC-N3.2 | AST/static scan reports no bare `except:` and no `except Exception: pass`. | DE-RUNNER, DE-ARCH-GUARD | TC-NFR03-02 | PLANNED |
| NFR-03 / AC-N3.3 | Cancellation injection propagates `CancelledError`. | DE-RUNNER | TC-NFR03-03 | PLANNED |
| NFR-03 / AC-N3.4 | DB failure makes readiness return bounded-time 503 with explicit DB detail. | DE-HTTP-OPS, DE-PROBLEM | TC-NFR03-04 | PLANNED |
| NFR-03 / AC-N3.5 | Timed-out subprocess is killed, reaped, and absent from process inspection. | DE-RUNNER | TC-NFR03-05 | PLANNED |
| NFR-03 / AC-N3.6 | Injected migration failure rolls back and leaves the prior Alembic revision/data intact. | DE-MIG-V1, DE-MIG-V2, DE-MIG-V3 | TC-NFR03-06 | PLANNED |
| NFR-04 / AC-N4.1 | Every canonical secret-pattern line becomes exactly `[REDACTED]` in tails, logs, and error output before persistence/emission. | DE-REDACTION, DE-RUNNER, DE-PROBLEM, DE-OBSERVABILITY | TC-NFR04-01 | PLANNED |
| NFR-04 / AC-N4.2 | Captured logs, errors, and metrics contain no configured DB password fragment. | DE-REDACTION, DE-PROBLEM, DE-OBSERVABILITY | TC-NFR04-02 | PLANNED |
| NFR-04 / AC-N4.3 | Created key plaintext appears once on CLI stdout and nowhere in persisted state or later output. | DE-CLI, DE-AUTHN, DE-REPO-KEYS | TC-NFR04-03 | PLANNED |
| NFR-05 / AC-N5.1 | AST scan finds a docstring with an FR/NFR citation on 100% of public functions/classes. | DE-DOC | TC-NFR05-01 | PLANNED |
| NFR-05 / AC-N5.2 | Every API operation in `/openapi.json` has non-empty summary and description. | DE-DOC, DE-HTTP-TASKS, DE-HTTP-RUNS, DE-HTTP-OPS | TC-NFR05-02 | PLANNED |
| NFR-06 / AC-N6.1 | `.importlinter` declares the exact four-layer order and config/errors independence. | DE-ARCH-GUARD | TC-NFR06-01 | PLANNED |
| NFR-06 / AC-N6.2 | Forbidden contract rejects SQLAlchemy imports outside repository. | DE-ARCH-GUARD | TC-NFR06-02 | PLANNED |
| NFR-06 / AC-N6.3 | `lint-imports` exits 0 on production code and a controlled service/API SQLAlchemy import is rejected. | DE-ARCH-GUARD | TC-NFR06-03 | PLANNED |
| NFR-06 / AC-N6.4 | Configuration review finds no missing contract, wildcard ignore, or weakened boundary. | DE-ARCH-GUARD | TC-NFR06-04 | PLANNED |
| NFR-07 / AC-N7.1 | Every runtime direct dependency is `==` pinned and every resolved transitive dependency is locked. | DE-DEPENDENCIES | TC-NFR07-01 | PLANNED |
| NFR-07 / AC-N7.2 | Complete resolved tree contains only MIT, BSD-2-Clause, BSD-3-Clause, Apache-2.0, or PSF licenses. | DE-DEPENDENCIES | TC-NFR07-02 | PLANNED |
| NFR-07 / AC-N7.3 | Canonical `pip-licenses --format=json --with-system` evidence covers direct and transitive dependencies. | DE-DEPENDENCIES | TC-NFR07-03 | PLANNED |
| NFR-07 / AC-N7.4 | `08-config/SBOM.json` lists name, version, license, and direct/transitive class for the complete tree. | DE-DEPENDENCIES | TC-NFR07-04 | PLANNED |
| NFR-08 / AC-N8.1 | Harness configuration has `features.mutation_testing: true`. | DE-TEST-GOV | TC-NFR08-01 | PLANNED |
| NFR-08 / AC-N8.2 | Mutmut result computes a mutation score of at least 70. | DE-TEST-GOV | TC-NFR08-02 | PLANNED |
| NFR-08 / AC-N8.3 | Mutation paths are exactly service/repository and configuration records the execution-budget rationale. | DE-TEST-GOV | TC-NFR08-03 | PLANNED |
| NFR-09 / AC-N9.1 | Static/collection scan finds no skip, skipif, xfail, or assertion-free requirement test. | DE-TEST-GOV | TC-NFR09-01 | PLANNED |
| NFR-09 / AC-N9.2 | Canonical full pytest command passes and reports zero skipped tests. | DE-TEST-GOV | TC-NFR09-02 | PLANNED |
| NFR-09 / AC-N9.3 | Exact-policy AST scan reports `zero_assert == 0`. | DE-TEST-GOV | TC-NFR09-03 | PLANNED |
| NFR-09 / AC-N9.4 | Test config/commands contain none of the five prohibited collection exclusions. | DE-TEST-GOV | TC-NFR09-04 | PLANNED |
| NFR-09 / AC-N9.5 | FR-07 runs on a real SQLite file and compares round-trip values without skip. | DE-TEST-GOV, DE-MIG-V3 | TC-NFR09-05 | PLANNED |
| NFR-09 / AC-N9.6 | Automated trace audit rejects any `VERIFIED` row lacking a passing named result. | DE-TEST-GOV | TC-NFR09-06 | PLANNED |
| NFR-09 / AC-N9.7 | Canonical full coverage command reports 100% TOTAL source line coverage. | DE-TEST-GOV | TC-NFR09-07 | PLANNED |
| NFR-10 / AC-N10.1 | Integration-only suite reports at least 80% TOTAL source line coverage. | DE-TEST-GOV | TC-NFR10-01 | PLANNED |
| NFR-10 / AC-N10.2 | Integration tests construct `httpx.AsyncClient` with `ASGITransport(app)` and do not call handlers directly. | DE-TEST-GOV | TC-NFR10-02 | PLANNED |
| NFR-10 / AC-N10.3 | The twelve independently named scenarios in §5.1 execute the required CRUD, error, migration, rate, and drain behavior. | DE-TEST-GOV and the feature design element named by each scenario | TC-NFR10-03a, TC-NFR10-03b, TC-NFR10-03c, TC-NFR10-03d, TC-NFR10-03e, TC-NFR10-03f, TC-NFR10-03g, TC-NFR10-03h, TC-NFR10-03i, TC-NFR10-03j, TC-NFR10-03k, TC-NFR10-03l | PLANNED |
| NFR-11 / AC-N11.1 | LLOC-weighted project MI is at least 80. | DE-QUALITY | TC-NFR11-01 | PLANNED |
| NFR-11 / AC-N11.2 | Maximum per-function cyclomatic complexity is 10. | DE-QUALITY | TC-NFR11-02 | PLANNED |
| NFR-11 / AC-N11.3 | Every file is at most 400 lines and every directory at most 15 files. | DE-QUALITY | TC-NFR11-03 | PLANNED |
| NFR-11 / AC-N11.4 | Every API handler is at most 40 lines and business logic resides in service. | DE-QUALITY, DE-ARCH-GUARD | TC-NFR11-04 | PLANNED |
| NFR-12 / AC-N12.1 | `verify-system` contains and orders upgrade, full tests, service health/readiness smoke, downgrade-base, and re-upgrade. | DE-SYSTEM-VERIFY | TC-NFR12-01 | PLANNED |
| NFR-12 / AC-N12.2 | `make verify-system` exits 0 and emits `verify-system: PASS`. | DE-SYSTEM-VERIFY | TC-NFR12-02 | PLANNED |
| NFR-12 / AC-N12.3 | `.env.example` declares exactly the 12 named variables, each with a comment. | DE-CONFIG | TC-NFR12-03 | PLANNED |
| NFR-12 / AC-N12.4 | Module CLI exposes functioning migrate, seed, and healthcheck commands. | DE-CLI | TC-NFR12-04 | PLANNED |
| NFR-12 / AC-N12.5 | Canonical uvicorn target starts; host/port defaults and log-level/log-format domains match the SRS. | DE-CONFIG, DE-LIFECYCLE | TC-NFR12-05 | PLANNED |

### 5.1 AC-N10.3 one-to-one scenario expansion

| Test case | Independent scenario | Feature trace |
|---|---|---|
| TC-NFR10-03a | CRUD full chain | FR-01 / DE-HTTP-TASKS |
| TC-NFR10-03b | HTTP 401 | FR-03, FR-10 / DE-AUTHN, DE-PROBLEM |
| TC-NFR10-03c | HTTP 403 with no existence disclosure | FR-04, FR-10 / DE-AUTHZ, DE-PROBLEM |
| TC-NFR10-03d | HTTP 404 | FR-01, FR-10 / DE-HTTP-TASKS, DE-PROBLEM |
| TC-NFR10-03e | HTTP 409 | FR-01, FR-10 / DE-HTTP-TASKS, DE-PROBLEM |
| TC-NFR10-03f | HTTP 422 | FR-01, FR-10 / DE-DTO, DE-PROBLEM |
| TC-NFR10-03g | HTTP 429 with `Retry-After` | FR-05, FR-10 / DE-RATE, DE-PROBLEM |
| TC-NFR10-03h | HTTP 503 readiness failure | FR-09, FR-10 / DE-HTTP-OPS, DE-PROBLEM |
| TC-NFR10-03i | Real migration round trip | FR-07 / DE-MIG-V3 |
| TC-NFR10-03j | Rate-limit trigger | FR-05 / DE-RATE |
| TC-NFR10-03k | Rate-limit recovery | FR-05 / DE-RATE |
| TC-NFR10-03l | Graceful drain | FR-08 / DE-RUNNER, DE-LIFECYCLE |

## 6. Design → requirement/test reverse index

| Design element | Requirement/criterion consumers | Test families returning evidence |
|---|---|---|
| DE-HTTP-TASKS | FR-01; FR-04 AC-4.3; NFR-01; NFR-05 AC-N5.2; NFR-10 AC-N10.3 | TC-FR01-*, TC-FR04-03, TC-NFR01-*, TC-NFR05-02, TC-NFR10-03a/d/e |
| DE-HTTP-RUNS | FR-02; FR-04 AC-4.3; FR-10 AC-10.13; NFR-05 AC-N5.2 | TC-FR02-*, TC-FR04-03, TC-FR10-13, TC-NFR05-02 |
| DE-HTTP-OPS | FR-03 AC-3.5; FR-04 AC-4.3; FR-05 AC-5.4; FR-09; FR-10 AC-10.11; NFR-03 AC-N3.4; NFR-05 AC-N5.2; NFR-10 AC-N10.3 | TC-FR03-05*, TC-FR04-03, TC-FR05-04*, TC-FR09-*, TC-FR10-11, TC-NFR03-04, TC-NFR05-02, TC-NFR10-03h |
| DE-DTO | FR-01 AC-1.1/1.4/1.6; FR-10 AC-10.2; NFR-10 AC-N10.3 | TC-FR01-01/04*/06*, TC-FR10-02, TC-NFR10-03f |
| DE-AUTHN | FR-03; FR-10 AC-10.6; NFR-02 AC-N2.3; NFR-04 AC-N4.3; NFR-10 | TC-FR03-*, TC-FR10-06, TC-NFR02-03, TC-NFR04-03, TC-NFR10-03b |
| DE-AUTHZ | FR-01/02 route scopes; FR-04; FR-09 AC-9.3; FR-10 AC-10.7; NFR-02 AC-N2.4; NFR-10 | TC-FR01-*, TC-FR02-*, TC-FR04-*, TC-FR09-03, TC-FR10-07, TC-NFR02-04, TC-NFR10-03c |
| DE-RATE | FR-05; FR-10 AC-10.10; NFR-10 | TC-FR05-*, TC-FR10-10, TC-NFR10-03g/j/k |
| DE-RUNNER | FR-02; FR-08; FR-10 AC-10.13; NFR-02 AC-N2.1; NFR-03 AC-N3.2/3.3/3.5; NFR-04 AC-N4.1; NFR-10 | TC-FR02-*, TC-FR08-*, TC-FR10-13, TC-NFR02-01, TC-NFR03-02/03/05, TC-NFR04-01, TC-NFR10-03l |
| DE-REPO-TASKS | FR-01/02/06; FR-09 AC-9.2; NFR-01; NFR-02 AC-N2.2 | TC-FR01-*, TC-FR02-*, TC-FR06-*, TC-FR09-02*, TC-NFR01-*, TC-NFR02-02 |
| DE-REPO-KEYS | FR-03; FR-06; NFR-02 AC-N2.2/2.3; NFR-04 AC-N4.3 | TC-FR03-*, TC-FR06-01/03, TC-NFR02-02/03, TC-NFR04-03 |
| DE-REPO-RATE | FR-05; FR-06; NFR-02 AC-N2.2 | TC-FR05-*, TC-FR06-01/03, TC-NFR02-02 |
| DE-SESSION | FR-01 AC-1.5; FR-05 AC-5.3; FR-06 AC-6.2/6.5; NFR-03 AC-N3.1 | TC-FR01-05, TC-FR05-03*, TC-FR06-02*/05, TC-NFR03-01 |
| DE-DATA-MODEL | FR-02 AC-2.4; FR-07 AC-7.2/7.9 | TC-FR02-04, TC-FR07-02*/09 |
| DE-MIG-V1 | FR-07; NFR-03 AC-N3.6 | TC-FR07-01a/02*/05*/07/08*/09, TC-NFR03-06 |
| DE-MIG-V2 | FR-07; NFR-03 AC-N3.6 | TC-FR07-01b/03/05*/07/08*/09, TC-NFR03-06 |
| DE-MIG-V3 | FR-07; FR-09 AC-9.2; NFR-03 AC-N3.6; NFR-09 AC-N9.5; NFR-10 | TC-FR07-01c/04*/05*/06/07/08*/09, TC-FR09-02c, TC-NFR03-06, TC-NFR09-05, TC-NFR10-03i |
| DE-PROBLEM | FR-01/03/04/05/08/09/10; NFR-02/03/04/10 | TC-FR01-06*/07/08, TC-FR03-01*, TC-FR04-02*, TC-FR05-02, TC-FR08-04, TC-FR09-02*, TC-FR10-*, TC-NFR02-04/05, TC-NFR03-04, TC-NFR04-01/02, TC-NFR10-03b..03h |
| DE-REDACTION | FR-10 AC-10.3/10.12; NFR-02 AC-N2.5; NFR-04 AC-N4.1/4.2 | TC-FR10-03/12, TC-NFR02-05, TC-NFR04-01/02 |
| DE-CONFIG | FR-02/05/06/08; NFR-02 AC-N2.6; NFR-12 AC-N12.3/12.5 | TC-FR02-02b, TC-FR05-01*, TC-FR06-05, TC-FR08-01*/02*, TC-NFR02-06, TC-NFR12-03/05 |
| DE-OBSERVABILITY | FR-09 AC-9.3; FR-10 AC-10.4; NFR-04 AC-N4.1/4.2 | TC-FR09-03, TC-FR10-04, TC-NFR04-01/02 |
| DE-LIFECYCLE | FR-08 AC-8.1; NFR-02 AC-N2.6; NFR-10 AC-N10.3; NFR-12 AC-N12.5 | TC-FR08-01*, TC-NFR02-06, TC-NFR10-03l, TC-NFR12-05 |
| DE-CLI | FR-03 AC-3.3; NFR-04 AC-N4.3; NFR-12 AC-N12.4 | TC-FR03-03, TC-NFR04-03, TC-NFR12-04 |
| DE-ARCH-GUARD | FR-06 AC-6.1/6.3; NFR-02 AC-N2.1/2.2/2.7; NFR-03 AC-N3.2; NFR-06; NFR-11 AC-N11.4 | TC-FR06-01/03, TC-NFR02-01/02/07, TC-NFR03-02, TC-NFR06-*, TC-NFR11-04 |
| DE-DEPENDENCIES | NFR-07 | TC-NFR07-* |
| DE-DOC | NFR-05 | TC-NFR05-* |
| DE-TEST-GOV | NFR-08/09/10 | TC-NFR08-*, TC-NFR09-*, TC-NFR10-* |
| DE-QUALITY | NFR-11 | TC-NFR11-* |
| DE-SYSTEM-VERIFY | NFR-12 AC-N12.1/12.2 | TC-NFR12-01/02 |

## 7. Test → requirement/design reverse index

The wildcard families below are closed sets defined by the explicit IDs in §§4–5; a wildcard does not authorize an unnamed test.

| Test family | Returns evidence to | Design lookup |
|---|---|---|
| TC-FR01-* | FR-01 / AC-1.1–AC-1.8 | FR-01 rows in §4 |
| TC-FR02-* | FR-02 / AC-2.1–AC-2.5 | FR-02 rows in §4 |
| TC-FR03-* | FR-03 / AC-3.1–AC-3.5 | FR-03 rows in §4 |
| TC-FR04-* | FR-04 / AC-4.1–AC-4.3 | FR-04 rows in §4 |
| TC-FR05-* | FR-05 / AC-5.1–AC-5.4 | FR-05 rows in §4 |
| TC-FR06-* | FR-06 / AC-6.1–AC-6.5 | FR-06 rows in §4 |
| TC-FR07-* | FR-07 / AC-7.1–AC-7.9 | FR-07 rows in §4 |
| TC-FR08-* | FR-08 / AC-8.1–AC-8.4 | FR-08 rows in §4 |
| TC-FR09-* | FR-09 / AC-9.1–AC-9.3 | FR-09 rows in §4 |
| TC-FR10-* | FR-10 / AC-10.1–AC-10.13 | FR-10 rows in §4 |
| TC-NFR01-* | NFR-01 / AC-N1.1–AC-N1.3 | NFR-01 rows in §5 |
| TC-NFR02-* | NFR-02 / AC-N2.1–AC-N2.7 | NFR-02 rows in §5 |
| TC-NFR03-* | NFR-03 / AC-N3.1–AC-N3.6 | NFR-03 rows in §5 |
| TC-NFR04-* | NFR-04 / AC-N4.1–AC-N4.3 | NFR-04 rows in §5 |
| TC-NFR05-* | NFR-05 / AC-N5.1–AC-N5.2 | NFR-05 rows in §5 |
| TC-NFR06-* | NFR-06 / AC-N6.1–AC-N6.4 | NFR-06 rows in §5 |
| TC-NFR07-* | NFR-07 / AC-N7.1–AC-N7.4 | NFR-07 rows in §5 |
| TC-NFR08-* | NFR-08 / AC-N8.1–AC-N8.3 | NFR-08 rows in §5 |
| TC-NFR09-* | NFR-09 / AC-N9.1–AC-N9.7 | NFR-09 rows in §5 |
| TC-NFR10-* | NFR-10 / AC-N10.1–AC-N10.3 | NFR-10 rows and §5.1 |
| TC-NFR11-* | NFR-11 / AC-N11.1–AC-N11.4 | NFR-11 rows in §5 |
| TC-NFR12-* | NFR-12 / AC-N12.1–AC-N12.5 | NFR-12 rows in §5 |

For every concrete `TC-<requirement>-<criterion><optional-subcase>` ID, removing `TC-` and decoding the numeric criterion returns exactly one source AC row; that row names every responsible `DE-*`. Sections 4.1 and 5.1 explicitly resolve every suffixed functional case and the multi-scenario NFR criterion. This makes reverse lookup deterministic without treating a family wildcard as execution evidence.

## 8. Coverage validation

Coverage here means planned trace-link completeness, not source-code line coverage and not passing tests.

| Check | Denominator | Linked | Coverage | Current verdict |
|---|---:|---:|---:|---|
| Functional requirements present in both SRS and matrix | 10 | 10 | 100% | PASS |
| Functional requirements present in `SPEC_TRACKING.md` and matrix | 10 | 10 | 100% | PASS |
| FR acceptance criteria → at least one design element | 59 | 59 | 100% | PASS |
| FR acceptance criteria → at least one explicit test case | 59 | 59 | 100% | PASS |
| Non-functional requirements present in SRS and matrix | 12 | 12 | 100% | PASS |
| NFR acceptance criteria → at least one design element | 51 | 51 | 100% | PASS |
| NFR acceptance criteria → at least one explicit test case | 51 | 51 | 100% | PASS |
| All acceptance criteria with forward and reverse lookup | 110 | 110 | 100% | PASS |
| Requirement IDs marked `VERIFIED` with passing evidence | 22 | 0 | 0% | NOT STARTED — correct for Phase 1 |
| Source line coverage | — | — | Unknown | NOT MEASURED — Phase 4 evidence required |

**Count basis:** SRS §3 defines 59 FR acceptance criteria (`8+5+5+3+4+5+9+4+3+13`); SRS §4 defines 51 NFR acceptance criteria (`3+7+6+3+2+4+4+3+7+3+4+5`). There are 110 acceptance criteria in total. Duplicate cross-cutting tests are allowed, but no criterion relies solely on an implicit or unnamed test.

## 9. Deferred decision propagation

These are inherited canonical ambiguities from SRS §7. They do not erase the trace links above: each affected row has a stable core test, while the unresolved part of its oracle must be finalized only after stakeholder resolution. No matrix row silently selects an interpretation.

| Open issue | Affected trace | Stable evidence now planned | Decision-dependent oracle |
|---|---|---|---|
| FR-01-deferred / NFR-99 | AC-1.6; TC-FR01-06a/b/c | Invalid requests must produce 422 problem details. | Which fields receive non-empty/length/blacklist rules and the exact blacklist. |
| FR-02-deferred / NFR-99 | AC-2.4; TC-FR02-04 | All named result fields must persist. | Tail truncation length and unit. |
| FR-05-deferred / NFR-99 | AC-5.3; TC-FR05-03a/b | Persistence, atomicity, and no over-admission under concurrency. | PostgreSQL-only row lock assertion versus SQLite-equivalent serialization. |
| FR-07-deferred / NFR-99 | AC-7.2, AC-7.9; TC-FR07-02a/b, TC-FR07-09 | All named head-schema objects and reversible migrations are inspected. | Reconcile the v1 two-table statement with v1 `rate_buckets` and `result_json`. |
| FR-08-deferred / NFR-99 | AC-8.1; TC-FR08-01b | Drain timeout marks overdue work `interrupted`. | Whether `interrupted` extends the formal state machine or is external metadata. |
| FR-09-deferred / NFR-99 | AC-9.3; TC-FR09-03 | Metrics must expose execution-latency percentile data. | Exact percentile set and field names. |
| NFR-02-deferred / NFR-99 | AC-N2.1; TC-NFR02-01 | The canonical source command must have zero prohibited-call hits. | Whether the controlling scan scope is all repository code or only `03-development/src/`. |

## 10. Consistency and update rules

1. `SPEC.md` remains the single source of truth; an approved change there must update the SRS, this matrix, and `01-requirements/TEST_INVENTORY.yaml` in that order.
2. Phase 2 must preserve each `DE-*` ID or record an explicit replacement mapping in `02-architecture/SAD.md`; deleting an allocation without replacing its requirement links is a traceability defect.
3. Every concrete `TC-*` above must have exactly one inventory record and one unique test function name before implementation. Subcases are separate cases.
4. A test may cover multiple requirements, but its execution result must report all linked IDs; passing one cross-cutting test does not verify an unexecuted sibling case.
5. A requirement can move from `PLANNED` to `VERIFIED` only when every acceptance criterion's required cases have passing evidence and none is blocked by an unresolved decision.
6. Any code added later without at least one requirement and test link is an orphan; any acceptance criterion without a live design and test link is a coverage gap.
