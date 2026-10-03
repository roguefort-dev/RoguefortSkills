# Data-oriented design and optimization patterns

Use when the bottleneck calls for changing representation, update structure, or ownership. Start from data volume, access order, fields read/written together, mutation frequency, and latency requirements. Keep the project's abstractions where they serve its interfaces; design hot storage and transformations around the measured workload.

## Data-oriented design

1. Map the hot transformations: inputs, outputs, iteration count, bytes touched, dependencies, and useful results. Include layout conversion, queries, updates, and downstream consumers. Done when the expensive data path and its owner are identified.
2. Choose an experiment matching that path: contiguous records, field arrays, blocked records, active subsets, or homogeneous batches. Use [cpu-data-and-simd.md](cpu-data-and-simd.md) for layout/cache/SIMD decisions and [compute-kernels.md](compute-kernels.md) for GPU geometry. Done when a candidate explains which cost it removes.
3. Preserve identity, lifecycle, order, and numerical semantics; compare the complete path through [benchmarking.md](benchmarking.md). Done when correctness and the application budget justify keeping the design.

ECS is one implementation of data-oriented storage and queries. Archetypes group matching component sets; chunked component arrays support bulk iteration. Component composition changes can move entities and invalidate locations, so measure structural churn, chunk utilization, and query/update costs. Plain arrays and batch functions can apply these ideas without adopting ECS. Check the project's framework capabilities before building custom storage.

## Pattern catalog

Use the mechanism to generate hypotheses across domains. The linked guides own the detailed tradeoffs and experiments.

| Pattern | Cost mechanism / useful workload | Confirming experiment and transfer |
|---|---|---|
| Contiguous/blocked storage and hot/cold separation | Fewer irrelevant bytes and pointer traversals in repeated processing | Compare field-wise and whole-record traversal including conversion; GPU coalescing → CPU/audio blocks. [CPU guide](cpu-data-and-simd.md) |
| Homogeneous batch processing | Amortize setup and group similar access/control flow | Compare equivalent ordered work across batch sizes, including grouping cost; shader/material batching → parsers, simulation, packet processing. [CPU guide](cpu-data-and-simd.md) |
| Dirty/versioned data and incremental computation | Reuse valid derived results; update affected regions | Vary changed fraction and rebuild frequency; dirty image tiles → UI layout or derived simulation state. [Work avoidance](work-avoidance.md) |
| Dense storage with stable handles | Compact traversal with identity independent of physical location | Compare iteration plus create/delete/lookup costs; particle slots → application registries. See handles below |
| Partitioned ownership and local aggregation | Fewer shared writes, locks and coherence transfers | Compare disjoint batches and final merge including skew; workgroup-local reductions → worker counters. [Concurrency](concurrency-and-pipelines.md) |
| Deferred command buffers | Consolidate mutations at explicit phase boundaries | Measure collection, playback and visibility delay; ECS structural edits → batched scene/UI changes. See deferred mutations below |
| Snapshots/double buffering | Readers and writers use coherent versions with fewer dependency waits | Measure copying, retained versions and end-to-end latency; ping-pong simulation → application snapshots. [Pipelines](concurrency-and-pipelines.md) |
| Precomputed execution plans / bounded specialization | Resolve stable lookup, configuration and dispatch work outside inner loops | Include plan construction and invalidation; shader variants → parsers, serializers, numerical pipelines. [Runtime guide](runtime-optimization.md) |

### Dense collections and handles

Keep externally visible identity separate from dense array position. A handle can pair a slot with a generation; lookup checks liveness/generation and maps to the current dense index. Swap-removal updates the moved item's mapping. Define generation wrap/reuse behavior, reference lifetime, and synchronization; a stale handle must not silently name a new item.

Choose ordered removal when iteration order matters. Account for lookup indirection, sparse mapping size, mutation frequency, and partially filled storage. Verify create/delete/reuse sequences and externally retained handles. Reuse framework registries or containers when they already own identity.

### Deferred mutations

Collect operations while traversing stable data, then apply them at an explicit boundary. Specify playback order, conflicting commands, create/delete dependencies, visibility of pending changes, and failure behavior. Preserve immediate semantics where callers require them. Batch only compatible operations; replay overhead and delayed visibility belong in the measurement. Use the framework's command buffer or transaction mechanism when suitable.

## Pattern entry contract

For a new reusable optimization, record: measured cost → applicable workload → mechanism → correctness and maintenance tradeoffs → confirming benchmark → cross-domain example. Keep the simplest design that repays its complexity across the intended workload; retain a straightforward baseline for comparison.

## Primary references

- [Unity archetypes/chunks](https://docs.unity3d.com/Packages/com.unity.entities@1.4/manual/concepts-archetypes.html): concrete ECS storage and structural-change costs.
- [Unity structural changes](https://docs.unity3d.com/Packages/com.unity.entities@1.4/manual/concepts-structural-changes.html): movement, synchronization and invalidation.
- [Game Programming Patterns: Data Locality](https://github.com/munificent/game-programming-patterns/blob/master/book/data-locality.markdown): access-driven design.
- [LLVM vectorizers](https://llvm.org/docs/Vectorizers.html): homogeneous loops and compiler evidence.

Original synthesis for this bundle; examples are hypotheses, with no assumed speedup.
