# Reyden-inspired real-time lakehouse benchmark protocol

This benchmark suite defines reproducible pgrust engineering gates inspired by the public Lakehouse//RT/Reyden product claims. It is **not** a reconstruction of Databricks' unpublished benchmark configuration, query mix, hardware, or percentile definition.

## Principles

1. Report distributions, not averages.
2. Use open-loop load generation to avoid coordinated omission.
3. Separate queue time from execution/service time.
4. Run cold-metadata, warm-metadata, and warm-data-cache modes.
5. Test Delta Lake and Apache Iceberg from the same logical dataset.
6. Keep the lakehouse table as source of truth; local caches must be evictable.
7. Record raw machine-readable output and exact environment metadata.
8. Include correctness, overload, cancellation, and fault tests—not only steady-state speed.

## Required environment metadata

- pgrust commit and dirty state
- dependency lockfile hash
- `rustc` and LLVM versions
- build profile and relevant `RUSTFLAGS`
- kernel, libc, container/runtime, and filesystem
- CPU model, sockets, NUMA topology, cores, SMT, frequency policy
- RAM and configured query/cache limits
- object-store provider, region, endpoint distance, and connection limits
- dataset generator seed, scale factor, table-format versions, file-size distribution
- cold/warm cache procedure
- query text and parameter distribution

## Workloads

### A. Protocol and session scale

- 1, 100, 1,000, and 10,000 concurrent PostgreSQL sessions.
- Prepared Parse/Bind/Execute/Sync cycles and pipelined requests.
- Idle, lightly active, and cancellation-heavy modes.
- Track memory per idle session, protocol errors, cancellation latency, and event-loop stalls.

### B. Hot selective serving

- Point/range filters with small projections and limits.
- Metadata and data-cache warm and explicitly reported.
- Parameter cardinality high enough to prevent a trivial one-result cache from dominating.

### C. Dashboard mix

- Concurrent filters, group-bys, top-k, and time-window aggregations.
- Zipfian and uniform parameter distributions.
- Open-loop ramps from 10% to 100% of accepted QPS.

### D. TPC-H-shaped scale test

- Large scans, multi-table joins, and aggregations.
- At minimum 10 GB, 100 GB, and 1 TB scale points where infrastructure permits.
- Report bytes read after pruning and exchange/shuffle volume.

### E. TPC-DS-shaped complexity test

- Deep joins, subqueries, window functions, skew, and spill pressure.
- Differential result validation against a trusted engine.

### F. Freshness and snapshot isolation

- Commit new Delta/Iceberg snapshots while read traffic continues.
- Verify each query binds one snapshot and never mixes file sets.
- Measure time from committed snapshot to query visibility.

### G. Fault and overload

- Object-store timeout, throttling, partial read, and credential-expiry injection.
- Worker loss for distributed mode.
- CPU, memory, network, and queue saturation.
- Verify bounded deadlines, cancellation, and predictable rejection instead of latency collapse.

## Metrics

- accepted/offered QPS and concurrency
- p50, p90, p95, p99, and max end-to-end latency
- queue, planning, catalog, scan I/O, compute, exchange, and serialization time
- error/rejection/cancellation rates
- CPU, RSS, allocator statistics, and context switches
- bytes and request count by object-store operation
- partition/file/row-group/page pruning ratios
- metadata/footer/data/plan/result cache hit rates
- batches, rows per batch, operator input/output rows
- spill bytes and partitions
- load-slope ratio: p95 at max accepted QPS divided by p95 at 10% QPS

## Default proposed gates

The machine-readable defaults live in `acceptance.yaml`. They are aspirational project gates and must be calibrated per published benchmark environment without weakening result transparency.

## Run record layout

Each result directory should contain:

```text
results/<date>/<commit>/<environment>/<workload>/
  environment.json
  dataset.json
  queries/
  raw.ndjson
  summary.json
  flamegraph.svg          # when profiling
  README.md               # operator notes and anomalies
```

A failed run remains in the results archive. Do not discard overload, crash, timeout, or correctness-failure data.
