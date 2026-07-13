# ADR 0001: Add a dual-engine read-only real-time serving path

- **Status:** Proposed
- **Date:** 2026-07-13
- **Decision owners:** pgrust maintainers
- **Scope:** PostgreSQL-compatible analytical serving directly on Delta Lake and Apache Iceberg

## Context

The public pgrust implementation preserves PostgreSQL's process-per-connection and tuple-at-a-time execution architecture. Its PostgreSQL compatibility work is valuable, but the target real-time lakehouse workload requires a different data path: asynchronous high-concurrency scheduling, columnar batches, direct object-store reads, table-format snapshots, stable tail latency, and policy-aware distributed execution.

Changing every existing PostgreSQL executor node and backend global in one migration would place regression compatibility and performance work on the same critical path. A foreign/custom scan adapter is useful for prototyping, but a one-row-at-a-time callback is not an acceptable permanent boundary for a vectorized engine.

## Decision

pgrust will retain its existing engine as the compatibility/fallback path and add a separate read-only `rt` engine.

The pipeline will be:

1. PostgreSQL wire protocol.
2. PostgreSQL parse, analyze, and rewrite.
3. Explicit `RtEligibility` classification.
4. Whole-query lowering to a pgrust-owned `RtLogicalPlan`.
5. Catalog authorization and immutable Delta/Iceberg snapshot binding.
6. RT optimization and `RtPhysicalPlan` construction.
7. Asynchronously scheduled Arrow-compatible batch execution.
8. Batch-aware PostgreSQL result encoding.

Unsupported queries will fail visibly or fall back to the compatibility engine; they will never discover an unsupported seam through a runtime panic.

## Initial eligibility

- Read-only `SELECT`.
- Supported scalar/aggregate/window functions with differential tests.
- No volatile functions, DDL/DML, row locks, sequence mutation, temp-table dependence, unsafe extension callbacks, or unsupported collation behavior.
- One immutable snapshot per referenced lakehouse table for the duration of the query.

Expose routing through `EXPLAIN (ENGINE)` and a session setting equivalent to `pgrust.rt_engine = auto|on|off`.

## Runtime invariants

- New RT code receives an explicit `SessionContext`; it does not read implicit backend TLS.
- No `.await` occurs while a legacy PostgreSQL TLS compatibility guard is installed.
- Network, object-store I/O, CPU, memory, and spill each have bounded concurrency.
- Every operator supports cancellation, deadlines, memory accounting, and metrics.
- The scheduler uses backpressure and admission control instead of unbounded task creation.

## Execution ABI

pgrust owns a stable Arrow-compatible batch interface. DataFusion may implement the first version, but pgrust does not expose DataFusion types as the permanent public architecture boundary beyond Arrow schemas/batches where practical.

The result path accepts batches directly. It must not route every output row back through `ExecProcNode`.

## Storage and catalog

- `arrow-rs` / Parquet for columnar representation and decoding.
- delta-rs for Delta snapshots.
- Apache Iceberg Rust for Iceberg snapshots and REST catalogs.
- Apache OpenDAL or `object_store` for asynchronous storage access.
- A generic catalog/policy API with a Unity Catalog OSS adapter as an initial implementation.

Caches are derived, evictable, and keyed by immutable snapshot and policy versions. Delta/Iceberg remains the source of truth; no mandatory import/CDC pipeline is introduced.

## Consequences

### Positive

- PostgreSQL compatibility remains independently testable.
- The RT path can reach vectorized performance without rewriting every legacy executor node.
- Unsupported SQL has a safe fallback.
- DataFusion accelerates the first vertical slice while pgrust retains freedom to replace hot components.
- New lakehouse and distributed code can live in isolated crates, reducing rebase conflict with upstream pgrust.

### Negative

- Two execution engines require differential testing and explicit routing semantics.
- Some functions/types must have both row and vector implementations.
- Cross-engine transaction semantics are intentionally limited in the first read-only release.
- Planning may temporarily require pinned workers while legacy state remains thread-local.

## Rejected alternatives

### Replace `fork()` with one OS thread per connection

This reduces process overhead but does not provide M:N scheduling, asynchronous object-store I/O, bounded work queues, or batch execution.

### Convert the entire PostgreSQL executor to batches immediately

This maximizes semantic blast radius, blocks incremental delivery, and makes regression failures difficult to isolate.

### Put DataFusion behind a PostgreSQL custom scan forever

The public custom scan ABI is tuple-oriented. It would preserve per-row materialization and prevent whole-query join/exchange optimization.

### Import lakehouse tables into PostgreSQL storage

This creates a second serving copy and synchronization pipeline, violating the target architecture.

## Validation

The decision is accepted only if the project demonstrates:

- direct Delta and Iceberg reads without mandatory data import;
- a batch-native hot path from scan through result encoding;
- deterministic fallback for unsupported SQL;
- 10,000 concurrent sessions without process-per-client;
- explicit p50/p95/p99 and load-slope benchmarks;
- differential correctness for every advertised PostgreSQL-compatible operation;
- policy and snapshot versioning in plan/cache keys.
