# Specification Tracking Matrix — `taskq-api`

> Human-readable requirements view derived from the approved SRS. The canonical specification source is root-level `SPEC.md`.

## Project Info

- Project Name: `taskq-api`
- Version: v1.0.0
- Created: 2026-09-09
- Requirements baseline: `01-requirements/SRS.md`
- Canonical specification: `SPEC.md`
- Ownership convention: Agent A (Requirements) owns the requirement definition; implementation ownership is assigned in later phases.

## Specification Status

> **The Status column is machine-refreshed** — `advance-phase` overwrites each
> FR's Status from `build_traceability`'s live code/test scan (`IN_PROGRESS` once
> code/module exists, `VERIFIED` once code and test evidence exist). The authoritative
> status is that scan and `quality_manifest.json`, not a hand-filled cell. Semantic
> columns and ownership notes remain human-maintained.

| FR ID | Spec Description | Intent Class | Decision Framework | Status | Notes |
|-------|-----------------|--------------|-------------------|--------|-------|
| FR-01 | Task resource CRUD API with scope authorization, request validation, cursor pagination, and consistent errors. | API capability | SRS AC-1.1–AC-1.8: HTTP contract and CRUD integration evidence. | DRAFT | Owner: Agent A (Requirements); Source: `SPEC.md:78-90`. |
| FR-02 | Asynchronous task execution endpoint and newest-first run-history retrieval. | Execution capability | SRS AC-2.1–AC-2.5: runner lifecycle and HTTP integration evidence. | DRAFT | Owner: Agent A (Requirements); Source: `SPEC.md:92-98`. |
| FR-03 | X-API-Key authentication with hashed storage, constant-time comparison, creation, and revocation. | Security | SRS AC-3.1–AC-3.5: authentication, CLI, and persisted-key evidence. | DRAFT | Owner: Agent A (Requirements); Source: `SPEC.md:100-106`. |
| FR-04 | Hierarchical read, write, and admin authorization enforced through one shared dependency. | Security | SRS AC-4.1–AC-4.3: scope matrix and common route-dependency evidence. | DRAFT | Owner: Agent A (Requirements); Source: `SPEC.md:108-112`. |
| FR-05 | Persistent per-token token-bucket rate limiting with transactional updates and health-route exemptions. | Traffic control | SRS AC-5.1–AC-5.4: limit, recovery, exemption, and transaction evidence. | DRAFT | Owner: Agent A (Requirements); Source: `SPEC.md:114-119`. |
| FR-06 | Repository-only persistence with request-scoped transactions, parameterized access, eager loading, and pool configuration. | Data integrity | SRS AC-6.1–AC-6.5: architecture, transaction, query-count, and pool evidence. | DRAFT | Owner: Agent A (Requirements); Source: `SPEC.md:121-127`. |
| FR-07 | Reversible Alembic v1, v2, and v3 schema evolution, including lossless v3 data migration. | Schema evolution | SRS AC-7.1–AC-7.9: real-SQLite round trip, downgrade, schema, and offline-SQL evidence. | DRAFT | Owner: Agent A (Requirements); Source: `SPEC.md:129-142`. |
| FR-08 | Bounded async runner with graceful drain, cancellation propagation, timeout cleanup, and no orphan process. | Reliability | SRS AC-8.1–AC-8.4: concurrency, shutdown, cancellation, and subprocess-lifecycle evidence. | DRAFT | Owner: Agent A (Requirements); Source: `SPEC.md:144-149`. |
| FR-09 | Unauthenticated liveness and readiness endpoints plus admin-scoped operational metrics. | Operability | SRS AC-9.1–AC-9.3: health, database and migration readiness, and metrics HTTP evidence. | DRAFT | Owner: Agent A (Requirements); Source: `SPEC.md:151-159`. |
| FR-10 | RFC 7807 problem responses with correlation identifiers and suppression of internal details. | Error contract | SRS AC-10.1–AC-10.13: media type, field, status mapping, correlation, and disclosure evidence. | DRAFT | Owner: Agent A (Requirements); Source: `SPEC.md:161-167`. |

## Completeness Validation

- Expected functional requirements from `01-requirements/SRS.md` §3: 10 (`FR-01` through `FR-10`).
- Tracked functional requirements: 10 (`FR-01` through `FR-10`).
- Missing IDs: none.
- Duplicate IDs: none.
- Initial status: all rows are `DRAFT`; status remains subject to the machine refresh described above.
- Ownership: every row names Agent A (Requirements).

## Update log

| Date | Change | By |
|------|--------|----|
| 2026-09-09 | Replaced initialization scaffolding with the complete FR-01–FR-10 tracking matrix. | Agent A |
