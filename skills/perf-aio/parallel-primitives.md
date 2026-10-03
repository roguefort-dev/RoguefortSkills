# Parallel algorithm building blocks

Use for bulk filtering, output construction, sorting/grouping, histograms and reductions across CPU/GPU domains. Prefer an existing tested library/backend primitive where available. Compare setup, scratch storage, synchronization, dispatches and complete-path latency with a serial/simple baseline; parallelism needs enough useful work to repay overhead.

## Select the primitive

| Problem | Mechanism | Contract and costs |
|---|---|---|
| Aggregate values | Tree/local reductions, segmented reductions for groups | Identity, operation order, associative behavior, partial groups; scratch and merge cost |
| Assign disjoint offsets to variable-length outputs | Exclusive prefix scan of counts | Capacity and integer overflow; global phases and tail handling |
| Remove inactive items while retaining a dense output | Predicate → scan → scatter (stream compaction) | Stable order if required, unique output offsets, membership maintenance |
| Group/order many fixed-width keys | Radix sort or suitable library sort | Signed/float key encoding, stability, passes and traffic; compare comparison sorting for workload |
| Count categories | Local/sharded histograms then merge | Skew, contention, bin count/storage; counters must not overflow |
| Process sparse groups | Segmented operations / grouped batches | Segment boundaries, empty segments, skew; grouping and index maintenance |

For a scan, define inclusive/exclusive semantics and the identity. The parallel grouping must be legal for the operation: floating-point sums can change with order, and noncommutative operations must preserve operand order. Keep established numerical and determinism requirements.

## Hierarchical execution

A portable multi-group scan can compute local scans and group totals, scan those totals, then add each group's prefix to its local outputs. Each global phase needs the backend's ordered dispatch/task dependencies and memory visibility. GPU workgroup barriers do not synchronize separate workgroups; use [compute-kernels.md](compute-kernels.md) for scopes/tails and [concurrency-and-pipelines.md](concurrency-and-pipelines.md) for host ownership.

Use scratch sized for the workload and account for hierarchy depth, initialization and passes. Compare subgroup/wave implementations only behind verified capability/width requirements, with portable paths. Avoid device-wide spin waits unless the API explicitly supplies the required cooperative execution and progress guarantees.

Compaction can avoid downstream work, but adds predicate/scan/scatter traffic. Vary selectivity and downstream cost to find its crossover. Histograms with highly skewed keys may need different partitioning from uniform keys. A general reduction/scan primitive is valuable only when its semantic and resource contract fits its callers.

## Verification and transfer

Test empty/singleton inputs, lengths around tile/segment boundaries, all/none selected, repeated keys, skewed groups, output capacity, large totals and aliasing. Compare with a straightforward oracle and verify stable order when required. Benchmark cold and reused scratch through [benchmarking.md](benchmarking.md).

Cross-domain examples: scan allocates disjoint ranges for variable-size serialization; compaction builds active simulation/UI worklists; sorting groups work for coherent kernel or CPU batches; local histograms aggregate telemetry. Preserve externally visible order and side effects.

## Primary references

- [GPU Gems 3: Parallel Prefix Sum](https://developer.nvidia.com/gpugems/gpugems3/part-vi-gpu-computing/chapter-39-parallel-prefix-sum-scan-cuda): scan, compaction and radix-sort algorithms. Historical CUDA hardware constants and implementation recipes need adaptation to current backends.
- [LLVM vectorizers](https://llvm.org/docs/Vectorizers.html): reduction transformations and compiler legality.
