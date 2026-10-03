# Concurrency, ownership and pipelines

Use for jobs/workers, shared state, queues, asset/data pipelines, CPU–GPU overlap, or stalls between otherwise fast stages. Draw the dependency path from request/input to usable output. Measure service time, wait time, queue age, copies and synchronization. Optimize the limiting path within throughput, latency and memory budgets.

## Scheduling and ownership

- Measure the serial/parallel crossover on the complete workload. Sweep batch size and worker count; small tasks add scheduling cost, large batches can reduce balance or increase latency. Include merging and framework overhead.
- Assign disjoint output ranges or partitions when possible. Accumulate locally and merge at defined boundaries. Preserve reduction order/tolerance and required deterministic output.
- Schedule independent work early and wait at its actual consumption point. Use framework task dependencies rather than custom spin synchronization. Work stealing can improve skewed workloads, but locality and scheduler costs need measurement.
- Separate contention from false sharing: unrelated writes sharing a coherence line can cause traffic without sharing logical state. Use supported counters/tools to locate it before changing layout. Sharding/padding costs memory and may reduce locality; cache-line geometry is target-specific.
- Compare fewer shared updates or different ownership before introducing more elaborate lock-free structures. Any concurrent container requires explicit lifetime/reclamation, progress and memory-order contracts.

## Queues and backpressure

Bound outstanding work and retained bytes. Define overload behavior: wait, reject, coalesce, or drop according to the application contract. Preserve required events; stale-work cancellation belongs only where superseded results have no required effect. Avoid a producer waiting while holding a resource needed by its consumer.

Measure queue-age distribution and input-to-result latency alongside throughput. Batching amortizes setup while delaying individual results; an async API can move waits to a queue. Worker messages may copy/serialize data: compare transfer/ownership mechanisms and include those costs. Deadline-aware work budgets should preserve freshness and correctness.

## CPU–GPU overlap and resource lifetime

For GPU synchronization within a dispatch use [compute-kernels.md](compute-kernels.md). Here, track the lifetime across producers, submissions, consumers, and resource reuse.

- Keep useful intermediate data at its consumer; remove redundant uploads, readbacks and repacking. Integrated/shared-memory hardware still requires ordering and can incur copies/cache synchronization.
- Reuse bounded buffer/resource rings so producers can prepare work while consumers use an earlier version. Recycle only after completion is established; more buffered work trades idle time for retained memory and potentially more latency.
- Use async transfers/readbacks where supported and useful, and consume results at an explicit deadline. Choose storage modes and host/API barriers from official backend documentation. A completion callback alone does not prove all required memory visibility unless the API guarantees it.
- Snapshots/double buffering give readers coherent versions. Account for copy/merge costs, version lifetime and stale observations. In-place neighborhood updates require proven independence or separate buffers.
- Async compute or extra queues help only when dependencies and hardware permit useful overlap. Compare whole-frame/whole-request time and bandwidth contention, not the isolated kernel time.

For frame-based applications, inspect presentation pacing and frames in flight. Queue stuffing can increase input latency even while submission looks fast. Test sustained workloads and thermal/power limits, not only short bursts.

## Verification

Check overload, skew, cancellation, slow consumers, shutdown, resource reuse, and dependency ordering. Compare throughput and tail latency with memory bounds. Use race/concurrency tools and deterministic invariants where supported; benchmark representative scaling through [benchmarking.md](benchmarking.md).

Cross-domain transfer: workgroup-local reductions → worker-local counters; streaming backpressure → asset loading and worker messages; ping-pong textures → coherent application snapshots.

## Primary references

- [Microsoft parallelism pitfalls](https://learn.microsoft.com/en-us/dotnet/standard/parallel-programming/potential-pitfalls-in-data-and-task-parallelism): overhead, shared state and over-parallelization.
- [Linux false sharing](https://kernel.org/doc/html/latest/kernel-hacking/false-sharing.html): diagnosis and layout tradeoffs.
- [Bounded channels](https://learn.microsoft.com/en-us/dotnet/core/extensions/channels): capacity and overload policies.
- [Apple CPU/GPU synchronization](https://developer.apple.com/documentation/metal/synchronizing-cpu-and-gpu-work): versioned resources and overlap.
- [Android frame pacing](https://developer.android.com/games/sdk/frame-pacing): presentation queues and latency.
