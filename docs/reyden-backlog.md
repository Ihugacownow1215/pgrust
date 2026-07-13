# Reyden-compatible serving backlog

Every issue must include a correctness or benchmark exit condition and identify whether it affects the compatibility engine, RT engine, or both.

## P0 — Establish truth

1. Reproduce the public pgbench performance gap in release mode with full environment metadata.
2. Add ClickBench, TPC-H, selected TPC-DS, and open-loop dashboard load harnesses.
3. Add a user-visible runtime capability registry; reject/fallback before any uninstalled seam can execute.
4. Add p50/p95/p99, queue time, batch, byte, request, cache, memory, and spill instrumentation.
5. Coordinate with upstream on the unpublished thread-per-connection/analytical branch.

## P1 — Protocol and concurrency foundations

6. Implement PostgreSQL Parse/Bind/Describe/Execute/Sync and pipelining tests.
7. Split synchronous transport from command execution and add bounded async reads/writes.
8. Introduce explicit `SessionContext`, cancellation token, deadline, memory budget, and trace context.
9. Add many-sessions-per-worker runtime; prohibit process-per-client on the RT listener.
10. Add `pgrust.rt_engine = auto|on|off` and `EXPLAIN (ENGINE)`.

## P2 — Batch vertical slice

11. Define pgrust-owned Arrow batch schema/stream and `RtExecutionPlan` traits.
12. Add full-query `RtEligibility` and PostgreSQL Query → `RtLogicalPlan` lowering.
13. Add DataFusion-backed scan/filter/project/limit/aggregate operators.
14. Add a direct batch-to-PostgreSQL result encoder.
15. Add async object-store abstraction with retry, timeout, tracing, and concurrency limits.
16. Differential-test supported expressions and types against PostgreSQL.

## P3 — Open lakehouse tables

17. Add Delta snapshot binding and scan through delta-rs.
18. Add Iceberg snapshot binding, delete handling, and REST catalog through iceberg-rust.
19. Add snapshot-aware metadata, manifest, footer, dictionary, and plan caches.
20. Add partition/file/row-group/page pruning and asynchronous range coalescing/prefetch.
21. Add concurrent-commit freshness and snapshot-isolation tests.

## P4 — Analytical completeness and performance

22. Add vectorized hash/sort aggregates and top-k.
23. Add broadcast, partitioned hash, and sort-merge joins.
24. Add windows, unions, and subquery lowering.
25. Add dynamic filters, bloom filters, late materialization, and dictionary-aware kernels.
26. Add memory accounting, spill, skew handling, and adaptive repartition.
27. Profile DataFusion boundaries and replace only measured hot-path bottlenecks.

## P5 — Serving discipline

28. Add fair admission control, tenant priorities, and bounded work queues.
29. Add work-conserving cooperative scheduling and load-slope regression gates.
30. Add prepared-plan reuse and snapshot/policy-aware result caching where safe.
31. Add object-store and overload fault injection.
32. Demonstrate 10,000 sessions and the accepted-QPS progression: 1k → 5k → 12k.

## P6 — Distributed and governed service

33. Add fragment/exchange protocol and stateless worker service.
34. Add partition-level retry, worker-loss handling, and shuffle backpressure.
35. Add generic catalog/policy/credential/audit interface and Unity Catalog OSS adapter.
36. Add incremental add/remove-one-node autoscaling with warm-capacity controls.
37. Add multi-tenant isolation, rolling upgrade, and chaos test suites.
