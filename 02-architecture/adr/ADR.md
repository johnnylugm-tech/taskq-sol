# Architecture Decision Records (ADR) — `taskq-api`

This collection records the architecture decisions derived from `02-architecture/SAD.md`. The observed project interpreter is CPython 3.11.15; compatibility remains at the SAD's Python 3.11 minor-version baseline rather than one local patch release. Each decision names the SRS specification it satisfies; the traceability matrix below maps every ADR to the FR and NFR identifiers it serves and to the SAD sections that own the corresponding design.

## Traceability Matrix

The SRS (functional requirements in §3, non-functional requirements in §4) is the specification these decisions satisfy. The matrix maps each ADR to the SRS FR and NFR identifiers it serves and to the SAD sections that own the corresponding design; NFR coverage of this matrix is machine-checked by the harness artifact-consistency gate.

| ADR | FRs served | NFRs served | Specification satisfied |
|-----|-----------|-------------|------------------------|
| ADR-001 | FR-01–FR-10 (runtime and stack foundation) | NFR-07 | SRS §2.1–§2.2; SAD §1.1, §2.3 |
| ADR-002 | FR-06 | NFR-06, NFR-08, NFR-11 | SRS FR-06; SAD §2.1–§2.2, §4.1 |
| ADR-003 | FR-01, FR-02, FR-10 | NFR-06, NFR-10, NFR-11, NFR-12 | SRS FR-01, FR-02, FR-10; SAD §2.3, §3.2 |
| ADR-004 | FR-03, FR-04, FR-05, FR-09 | NFR-02 | SRS FR-03, FR-04, FR-05, FR-09; SAD §3.3 |
| ADR-005 | FR-02, FR-08 | NFR-03 | SRS FR-02, FR-08; SAD §3.4 |
| ADR-006 | FR-05, FR-06 | NFR-03 | SRS FR-05, FR-06; SAD §2.5 |
| ADR-007 | FR-09 | NFR-03, NFR-12 | SRS FR-09; SAD §1.3, §3.5 |
| ADR-008 | FR-07 | NFR-03 | SRS FR-07; SAD §3.5 |
| ADR-009 | FR-03, FR-10 | NFR-02, NFR-04 | SRS FR-03, FR-10, NFR-04; SAD §3.6, §4.1 |
| — (no single owning decision) | — | NFR-01, NFR-05, NFR-09 | SAD §4.1, §4.2 |

NFR-01 (performance and query efficiency), NFR-05 (documentation coverage), and NFR-09 (verification truthfulness) are cross-cutting: no single decision owns them, so the row above points at the SAD sections that do. SAD §4.2 fixes the NFR-01 latency thresholds, and SAD §4.1 names the NFR-05 and NFR-09 verification targets. Whether each decision meets the SRS acceptance criteria is bound by Phase 4 tests, not by this document.

## ADR-001: Use Python 3.11 with the prescribed ASGI and persistence stack

### Status
Accepted

### Context
The service must expose HTTP and management CLI entry points, validate transport models, persist to SQLite and PostgreSQL, and evolve its schema through reversible migrations. The SAD names Python 3.11, FastAPI, Pydantic v2, SQLAlchemy 2.x, Alembic, and Uvicorn for those responsibilities. The project virtual environment reports Python 3.11.15.

### Decision
Target Python 3.11 and use FastAPI/Pydantic for the ASGI boundary, SQLAlchemy 2.x for repositories, Alembic for migrations, and Uvicorn for serving `taskq_api.app:app`. Prefer Python standard-library facilities for capabilities they already provide, including `asyncio`, `hashlib`, and `hmac`, but do not describe the application as Python stdlib-only because the SAD's HTTP, validation, persistence, migration, and serving contracts require the named third-party packages. Pin direct and transitive dependencies in the project requirements and lock files.

### Consequences
- Positive: the implementation matches the documented runtime, entry points, database support, and migration tooling.
- Positive: Python 3.11 provides `asyncio.TaskGroup` for structured worker lifetime management.
- Negative: dependency updates, license checks, an SBOM, and lock-file maintenance are required.
- Negative: the local Python 3.11.15 patch version is observed evidence, not a promise that deployments use that exact patch.

### Alternatives Considered
- Python stdlib-only HTTP, validation, SQL, and migration code: rejected because it conflicts with the explicit SAD stack and would duplicate required framework behavior.
- A separate worker service or broker: rejected because the canonical requirements need neither another network boundary nor distributed execution.
- Pinning the architecture to exactly Python 3.11.15: rejected; the SAD commits to the Python 3.11 line, while patch selection belongs to deployment and lock policy.

## ADR-002: Implement a layered modular monolith with enforced downward dependencies

### Status
Accepted

### Context
The SAD requires the import direction `api → service → repository → models`, confines SQLAlchemy to persistence code, and defines one ASGI process, one relational database boundary, and OS child processes. Reverse imports would couple transport, business logic, and persistence and could create cycles.

### Decision
Implement one layered modular monolith. Runtime dependencies flow downward only through `api`, `service`, `repository`, and `models`. Keep `config` and `errors` independent of application layers. Keep SQLAlchemy mappings and session ownership in `repository/`; migrations may depend on repository metadata but not runtime API or service modules. Use `bootstrap.py` as the composition root and enforce the dependency graph with Import Linter.

### Consequences
- Positive: transport, use-case, and persistence concerns can be tested independently.
- Positive: SQLAlchemy cannot leak into service inputs or domain models.
- Positive: one deployable process avoids broker and worker-service operations.
- Negative: API handlers cannot bypass services for superficially simpler queries.
- Negative: composition code must adapt repository objects without exposing a raw SQLAlchemy `Session`.

### Alternatives Considered
- Direct route-to-database access: rejected because it violates the binding layer order and transaction ownership.
- A distributed worker and message broker: rejected because it adds an unrequired network and operational boundary.
- A single undivided application module: rejected because it would undermine import enforcement, cohesion, and source-size limits.

## ADR-003: Stabilize transport-neutral service and repository interfaces

### Status
Accepted

### Context
Both HTTP routes and the management CLI invoke the same use cases. Persistence must remain replaceable between SQLite and PostgreSQL without leaking ORM sessions, while HTTP failures must be rendered separately from business decisions.

### Decision
Expose a thin `service.facade` as the public use-case interface for API routes and CLI adapters. Its inputs and outputs are domain/schema values and typed application errors, never HTTP response objects or SQLAlchemy sessions. Expose repositories through `request_scope()` and `worker_scope()` bundles. Preserve the following principal interfaces:

- ASGI application: `taskq_api.app:app`, composed by `bootstrap.create_app()`;
- management CLI: `python -m taskq_api` with `migrate`, `seed`, `healthcheck`, and `key create --scope` commands;
- authorization: `api.dependencies.authorize(required_scope)`;
- runner lifecycle: `start()`, `submit(task_id)`, and `drain(timeout)`;
- repository lifetime: `request_scope()` and `worker_scope()`;
- public HTTP resources: `/v1/tasks`, `/v1/tasks/{id}/run`, `/v1/tasks/{id}/runs`, `/v1/metrics`, `/healthz`, and `/readyz`.

### Consequences
- Positive: HTTP and CLI adapters share the same behavior without coupling the service layer to either transport.
- Positive: transaction and ORM details stay behind repository scopes.
- Positive: internal interfaces provide explicit seams for integration and failure-path tests.
- Negative: adapters are required to map schemas, domain values, and typed failures at each boundary.
- Negative: the façade must remain thin to avoid becoming a god module.

### Alternatives Considered
- Let each route call repository methods directly: rejected because it duplicates orchestration and violates layering.
- Return framework-specific responses from services: rejected because it couples CLI and service behavior to FastAPI.
- Expose SQLAlchemy sessions as service parameters: rejected because transaction ownership would escape `repository/`.

## ADR-004: Centralize protected-request policy inside one transaction scope

### Status
Accepted

### Context
Every protected `/v1` request must authenticate an API key, authorize its hierarchical scope before looking up a target resource, apply a per-key rate token, and then execute the use case. The bound operations have a deterministic order and coherent rollback behavior; the SAD deliberately leaves the order of rate limiting relative to scope authorization and request validation—and whether a later failure consumes a token—unresolved pending canonical clarification. Health and readiness are deliberately public and unthrottled.

### Decision
Open one request repository scope in the shared dependency path. Within it, authenticate the `X-API-Key` using stored SHA-256 hashes and `hmac.compare_digest`, reject revoked keys, and authorize the required scope before any resource lookup. Coordinate rate limiting through the named policy seam in `api.dependencies`: it requests one atomic persisted bucket decision, but this ADR does not select its order relative to scope authorization or request validation, nor whether a later failure consumes a token; those observable behaviors remain blocked on canonical clarification (SAD §1.4, §3.3) and require a follow-up decision before tests may bind them. Commit on successful handler completion and roll back on an escaping exception. Route `/healthz` and `/readyz` through a separate public context; require `admin` for `/v1/metrics`.

### Consequences
- Positive: unauthorized callers cannot infer whether a resource exists.
- Positive: authentication, throttling, and the use case share one explicit request transaction boundary.
- Positive: all protected routes apply the same bound ordering through the shared dependency.
- Negative: a request can hold a database transaction while service work completes.
- Negative: dependency wiring must prevent accidental use of the public path by protected routes.
- Negative: rate-limit ordering and failed-request token accounting remain unresolved, so tests cannot bind those observable behaviors until canonical clarification arrives.

### Alternatives Considered
- Per-handler authentication and authorization: rejected because policy order could drift between endpoints.
- Resource lookup before authorization: rejected because it permits resource-existence disclosure.
- Rate-limit health endpoints: rejected because operators need unthrottled liveness and readiness probes.
- Fixing the rate-limit order and failed-request accounting in this ADR: rejected because the SAD blocks selecting either observable policy until canonical clarification is recorded.

## ADR-005: Use a bounded asyncio worker queue and structured concurrency

### Status
Accepted

### Context
Run submission returns `202` before a command completes, excess work must queue without unbounded active tasks, child processes require timeout and cancellation cleanup, and shutdown must drain within a configured budget. Database sessions cannot cross the queue boundary.

### Decision
Own one `asyncio.Queue` and a fixed number of workers inside one `asyncio.TaskGroup`. `submit(task_id)` assigns a unique `run_id` and enqueues work; each worker opens its own `worker_scope()`. Execute stored commands with `asyncio.create_subprocess_exec(*argv)` and never `shell=True`. On timeout or cancellation, kill any live child, await `process.wait()` to reap it, and re-raise `CancelledError`. Stop new submissions and invoke `drain(TASKQ_DRAIN_TIMEOUT)` during ASGI shutdown.

### Consequences
- Positive: active command count is bounded by the worker count while queued submissions remain cheap.
- Positive: TaskGroup gives workers a common lifetime and propagates cancellation.
- Positive: direct argv execution prevents shell metacharacter interpretation.
- Negative: execution remains local to one service process and is not distributed.
- Negative: the final persisted representation of work interrupted at shutdown remains unresolved by FR-08 and must not be invented during implementation.

### Alternatives Considered
- `concurrent.futures.ThreadPoolExecutor`: rejected for command execution because subprocesses and queue operations already have native asynchronous APIs, and threads would complicate cancellation and child reaping.
- One `asyncio.create_task` per submission: rejected because active coroutine count would grow with request volume and structured shutdown would be harder.
- `subprocess` with `shell=True`: rejected because command text could be interpreted by a shell.
- An external job broker: rejected because it is outside the specified system boundary.

## ADR-006: Make database transactions the atomic-write boundary

### Status
Accepted

### Context
Durable tasks, results, API-key hashes, rate buckets, and schema state live in the relational database. Request failures must not leave partial changes; concurrent rate-limit consumption must not overdraw a bucket; task deletion must include its results; and migration failures must preserve the prior revision.

### Decision
Use repository context managers as the atomic-write unit: commit on success and roll back on exception. Perform refill and decrement of one rate bucket as one atomic persisted consume operation inside the active transaction. Delete a task and its results in one request transaction. Let Alembic own migration transactions. Do not use filesystem state as an authoritative persistence mechanism, and therefore do not apply the temp-file-plus-rename atomic write pattern to domain data.

The repository interface guarantees atomic rate-bucket consumption across supported dialects, but the SAD intentionally leaves the SQLite-versus-row-lock serialization mechanism unresolved. That mechanism requires a separate verified decision before implementation is finalized.

### Consequences
- Positive: callers observe complete state transitions rather than partially persisted operations.
- Positive: transaction behavior is centralized and can be tested for both commit and rollback.
- Positive: no local file can diverge from database state.
- Negative: dialect-specific concurrency behavior for rate buckets remains Requires Verification.
- Negative: transaction duration and lock contention must be measured under concurrent requests.

### Alternatives Considered
- Atomic filesystem write via temporary file, flush/fsync, and rename/replace: rejected for domain state because the architecture assigns durability to SQLite/PostgreSQL, not files.
- Several independently committed repository calls per use case: rejected because failures could expose partial state.
- An in-memory token bucket: rejected because the rate limit must persist and remain per key.

## ADR-007: Use bounded fail-closed readiness checks without a circuit breaker

### Status
Accepted

### Context
Liveness reports whether the process is running, whereas readiness must verify that the database is usable and its Alembic revision is at configured head. The architecture has no outbound remote service other than its required database boundary. Readiness failures must not expose connection details.

### Decision
Keep `/healthz` as a process-level liveness response. For `/readyz`, perform bounded database connectivity and migration-head queries through `repository.health`; return `200` only when both facts pass and sanitized `503` problem details otherwise. Do not add a stateful circuit breaker: a probe must report current readiness rather than remain open or half-open based on historical failures.

### Consequences
- Positive: deployment systems receive current, fail-closed readiness information.
- Positive: migration drift prevents traffic before runtime queries encounter a mismatched schema.
- Positive: avoiding a circuit breaker removes state transitions and recovery timing that the requirements do not specify.
- Negative: each readiness request performs bounded database work.
- Negative: suitable query time bounds still require configuration and verification.

### Alternatives Considered
- Circuit breaker around database probes: rejected because cached breaker state can outlive recovery and the SAD calls for direct DB/head facts, not historical availability policy.
- Always-ready response after process startup: rejected because it ignores database and migration failures.
- Expose raw database exceptions: rejected because operational details and credentials must remain sanitized.

## ADR-008: Evolve the schema through three reversible Alembic revisions

### Status
Accepted, with the v1 schema gap retained for stakeholder resolution

### Context
The required schema history is exactly v1 initial, v2 tags, and v3 result splitting. The v3 transition moves existing `result_json` data into `task_results` and must survive downgrade and re-upgrade without field loss. The canonical v1 description conflicts with rows that mention `rate_buckets` and `result_json`.

### Decision
Maintain an immutable `v1_initial → v2_tags → v3_split_results` revision chain. Give every revision executable `upgrade()` and `downgrade()` paths and support offline SQL generation. Make v3 copy data before removing `result_json`; its downgrade restores that data before removing the split representation. Bind Alembic to repository ORM metadata through `migrations.env`, while keeping revision files self-contained. Do not finalize disputed v1 table contents until the recorded requirements conflict is resolved.

### Consequences
- Positive: deployments can advance and reverse schema state through a deterministic history.
- Positive: the v3 migration preserves existing execution results rather than only changing empty schemas.
- Positive: readiness can compare the current revision with a defined head.
- Negative: reversible data movement increases migration and test complexity.
- Negative: the unresolved v1 contents block a truthful final initial-schema decision.

### Alternatives Considered
- One squashed final-schema migration: rejected because the requirement calls for three distinct revisions and a reversible v3 data move.
- Runtime auto-create tables: rejected because it bypasses migration history and readiness head checks.
- Finalize v1 by guessing which conflicting requirement wins: rejected as an unverified architectural assumption.

## ADR-009: Sanitize every observable failure and secret-bearing output

### Status
Accepted

### Context
Untrusted request data, command output, database errors, and generated API keys cross externally observable boundaries. Every non-2xx HTTP response requires an RFC 7807 representation, and a single correlation identity must connect response body, response header, and server log without exposing internal exception text.

### Decision
Represent known failures as typed, transport-neutral application errors. Map them in `api.problems` to `application/problem+json` with `type`, `title`, `status`, `detail`, `instance`, and `correlation_id`; map unexpected failures to a fixed safe internal detail. Use the same correlation id in the body, `X-Correlation-Id`, and structured log. Apply `service.redaction.redact_lines()` before command output or sensitive diagnostics are persisted or emitted, replacing each canonical matching line wholesale with `[REDACTED]`. Emit a newly generated plaintext API key only once through the CLI and persist only its hash.

### Consequences
- Positive: clients receive one predictable failure contract across routes.
- Positive: incidents remain traceable without serializing raw exceptions or secret-like lines.
- Positive: hash-only API-key persistence limits disclosure from database access.
- Negative: whole-line redaction can remove non-secret context from a matching line.
- Negative: every output sink must consistently pass through the redaction and problem-mapping boundaries.

### Alternatives Considered
- Serialize exception strings in HTTP responses: rejected because messages can contain credentials or implementation details.
- Redact only HTTP output: rejected because logs, persistence, metrics, and command output are also observable sinks.
- Store encrypted or plaintext API keys for later display: rejected because the contract requires one-time plaintext output and hash-only storage.
