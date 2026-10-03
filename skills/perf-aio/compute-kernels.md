# GPU-agnostic compute and shader optimization

Use for compute shaders and GPU kernels in WGSL, HLSL, GLSL, MSL, or a native compute backend. Preserve the project's backend. Workgroup/threadgroup/block denotes cooperating invocations; subgroup/wave/SIMD group denotes a smaller execution unit whose width is device- and pipeline-dependent.

Apply perf-aio's shared performance loop. This guide supplies compute-specific decisions and benchmark guards; graphics coverage, overdraw, and texture filtering remain in [shader-optimization.md](shader-optimization.md).

## Establish a valid baseline

Use [benchmarking.md](benchmarking.md) for the shared workload/acceptance contract. Record the kernel and launch wrapper, target devices, API/compiler versions, shapes, strides/layouts, dtypes, aliasing, edge semantics, and numerical tolerance. Include representative small, large, and partial-tile inputs. Preserve an existing correctness oracle; establish one before judging a new kernel.

- Measure **completed GPU work**, using timestamp queries/events or the backend's equivalent. CPU dispatch duration measures submission, not execution. If only synchronized wall time is available, label it as end-to-end time, including submission and synchronization. Use the same queue/stream and measurement scope for A/B runs.
- Warm compilation and pipeline caches separately from steady-state timing. Keep allocations, input uploads, readbacks, validation, and tuning outside a kernel-only interval; include them in a separate application-path measurement when they are real costs.
- Repeat and interleave baseline/candidate runs under comparable thermal/power conditions. Report latency distribution or variability with the central estimate. A difference within noise is inconclusive. Profile the selected configuration separately: counter collection/replay can distort timings.
- Match production cache behavior: repeatedly using one tiny buffer measures a hot-cache case. Add a representative working-set case if production streams fresh data.
- Reset mutable inputs and accumulators between trials, outside the timed kernel interval. Preallocate scratch buffers; preserve aliasing and in-place semantics. A split-buffer search still needs final correctness and timing with the production aliasing contract.
- When timestamps are unavailable or too coarse, batch independent work at the intended scale and report the scope. If no target GPU is available, deliver correctness checks and an experiment plan; mark performance unverified.

## Diagnose the bound

Roofline is a hypothesis, not a classification by fixed FLOPs/byte thresholds:

`arithmetic intensity = useful operations / bytes crossing the measured memory level`

`ideal lower bound on time ≈ max(operations / relevant compute throughput, transferred bytes / relevant bandwidth)`

The balance point depends on device, dtype, operation mix, and cache level. Count reads **and** writes; distinguish estimated algorithmic bytes from measured traffic. Low bandwidth and low ALU utilization can mean stalls, poor parallelism, or submission gaps.

| Evidence / controlled diagnostic experiment | Candidate bound | Next experiment |
|---|---|---|
| Significant memory traffic; a reduced-byte/layout variant cuts GPU time | Bandwidth or cache misses | Adjacent-lane access, compact formats, reuse, pass fusion |
| Arithmetic-heavy variant costs more with comparable memory traffic | Compute throughput / dependency chain | Less repeated math, shorter dependency chains, validated precision change |
| Many live values, spills, or resource-limited resident waves | Register/shared-memory pressure | Smaller tile, shorter live ranges, less unrolling |
| Idle lanes, few workgroups, or poor scaling on small shapes | Insufficient parallel work / imbalance | Workgroup geometry, more independent tiles, balanced work distribution |
| Barrier or atomic stalls / contention | Synchronization | Local aggregation, fewer shared phases, partitioned outputs |
| Tiny kernels separated by queue gaps; batching reduces application time | Launch/submission overhead | Batch independent work, reuse pipelines, fuse compatible stages |

Use a diagnostic simplification to locate cost, then implement a semantics-preserving change. Confirm with counters or a second controlled experiment; bandwidth, occupancy, and instruction counts alone do not prove the bottleneck.

## Choose the next optimization

For scan/prefix sum, compaction, radix sorting, histograms, or segmented processing, load [parallel-primitives.md](parallel-primitives.md). For host resource rings, transfers/readbacks, queueing, or CPU–GPU overlap, load [concurrency-and-pipelines.md](concurrency-and-pipelines.md).

### Memory access and reuse

- Map neighboring lanes to neighboring addresses in the contiguous dimension. Respect actual strides, alignment, storage-layout rules, and tails. Vector operations help only when the compiler emits useful transactions; a `vec4` declaration alone does not ensure this.
- Remove redundant loads/stores and keep intermediates on the GPU. Fuse compatible stages when saved traffic/submissions outweigh larger live state and lost parallelism. Dependencies requiring all workgroups to finish remain separate dispatches.
- Stage a tile plus its halo/apron in shared memory when several invocations reuse it. Compare against cached direct reads: staging adds instructions, barriers, and occupancy cost. For tiled filters, compare separable passes and direct texture sampling too.
- Padding or changing a shared-memory layout may reduce bank conflicts on a measured target. Bank geometry is implementation-dependent; retest memory use and occupancy after padding.
- Match format/precision to the numerical contract and supported storage formats. Smaller arithmetic types may still occupy full-width storage/registers; verify the generated code and traffic.

### Workgroup geometry and occupancy

- Treat **workgroup size**, **tile size**, and **elements per invocation** as separate knobs. One invocation can load several tile elements; a tile need not have one thread per element.
- Query backend/device limits and pipeline-specific resource limits. On Metal, use `threadExecutionWidth` and `maxTotalThreadsPerThreadgroup`. Where subgroup width is exposed, account for it; otherwise compare portable candidate sizes. Neither 32 nor 64 is a universal width.
- Sweep a small legal set suited to the access pattern, such as 64/128/256 total invocations or 8×8/16×8/16×16 tiles **when the limits permit**. These are starting experiments, not defaults or promises. Include partial tiles and skinny shapes.
- Balance reuse against registers and shared memory. More resident waves can hide latency; maximizing occupancy is not the objective. Large groups or aggressive unrolling can reduce residency or spill, while very small groups can waste execution lanes.
- Compare work distribution when tails or variable-length tasks dominate. Grid-stride/persistent scheduling is an experiment for suitable workloads; a normal dispatch with many queued workgroups is valid and often efficient. Preserve fair progress and avoid cross-workgroup dependencies.

### Arithmetic, reductions, and atomics

- Hoist invariant address/math work, shorten dependency chains, and reduce live temporaries. Unrolling trades loop overhead for code size and registers; inspect the result before extending it.
- Use FP16, fast math, reassociation, or denormal flushing only within the established numerical contract and supported capabilities. Keep accumulation precision where needed. Validate error and long-running behavior; preserve tolerances when comparing candidates.
- Aggregate reductions/histograms locally before contended global atomics when it pays. A multi-dispatch reduction trades launch/temporary-buffer costs for lower contention. Floating-point reduction order can change results; test the permitted error and determinism.
- Subgroup operations can reduce shared traffic/barriers when supported. Query feature and width constraints, handle partially active subgroups, and retain a portable fallback. Treat matrix units and asynchronous-copy extensions similarly: optional, capability-gated experiments with measured benefit.

## Synchronization and bounds are part of correctness

Workgroup barriers synchronize participating invocations in that workgroup, not the entire dispatch. Memory visibility and execution synchronization are distinct; use the primitive whose memory scope covers the accessed resource. Atomics on a counter do not automatically publish unrelated writes.

- Keep workgroup barriers in uniform control flow. For tails, mask reads/writes while all participating invocations reach each barrier; use the algorithm's neutral/padding value. Zero suits sums, but max reductions and image edge policies may require different values.
- An out-of-range invocation may return early only when it cannot skip a barrier required by its group. Initialize shared elements that other invocations will read, including halo and tail elements. Barrier loops must have compatible trip counts across the group.
- Put dispatch-wide dependencies into ordered dispatches with the API's required resource transitions/barriers. Global spin barriers can deadlock when waiting groups prevent other groups from being scheduled.
- In-place neighborhood operations can race across workgroups even with local barriers. Use ping-pong buffers unless independence or an API-supported synchronization scheme proves the operation safe.

| Backend | Shared storage and workgroup synchronization | Check when porting |
|---|---|---|
| WGSL / WebGPU | `var<workgroup>`; `workgroupBarrier()` | `storageBarrier()` concerns storage memory within a workgroup; uniformity, device limits, optional features |
| HLSL | `groupshared`; `GroupMemoryBarrierWithGroupSync()` | Device-memory visibility needs its corresponding primitive; wave support/width, host resource barriers |
| GLSL compute | `shared`; `barrier()` for shared-memory exchange | Buffer/image memory needs matching memory barriers; host-side visibility between dispatches |
| Metal / MSL | `threadgroup`; `threadgroup_barrier(mem_flags::mem_threadgroup)` | Choose memory flags for the resource; pipeline limits, partial/nonuniform groups, encoder dependencies |

These are scope reminders, not interchangeable APIs. Read the project's API/version specification before writing or removing synchronization.

## Bounded tuning and acceptance

Change one hypothesis per experiment; sweep its related parameters together when needed. Reject illegal or incorrect configurations before ranking latency. Use the project's benchmark harness rather than changing inputs, tolerance, or timing scope to favor a candidate.

Keep an experiment record: device/workload, hypothesis/configuration, correctness result, baseline and candidate latency with variability, application-path impact, and keep/revert decision. Keep only gains beyond noise that meet the supported workload/device regression budget; leave the portable baseline when evidence is inconclusive.

For reusable tuning, prune by feature/resource limits and search only plausible configurations. Key cached choices by device/backend, pipeline/compiler identity, and relevant shape/layout/dtype/aliasing conditions. Reuse compiled pipelines; separate tuning/compilation startup cost from the steady-state gain. Stop when the target is met, a bounded search plateaus, or missing hardware/evidence prevents a supported decision.

## Official references and attribution

Check current documentation for the target stack; hardware guidance describes that vendor's implementations:

- [NVIDIA CUDA Best Practices](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/index.html): timing, coalescing, shared memory, resource/occupancy tradeoffs.
- [AMD GPUOpen: Occupancy explained](https://gpuopen.com/learn/occupancy-explained/): residency limits and latency hiding.
- [Apple: Calculating threadgroup and grid sizes](https://developer.apple.com/documentation/metal/calculating-threadgroup-and-grid-sizes): pipeline limits and dispatch geometry.
- [WGSL specification](https://www.w3.org/TR/WGSL/): uniformity, synchronization, and memory scopes.
- [HLSL group synchronization](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/groupmemorybarrierwithgroupsync): shared-memory ordering and divergent-flow restrictions.
- [Khronos GLSL specification](https://registry.khronos.org/OpenGL/specs/gl/GLSLangSpec.4.60.html): compute shared-memory exchange and memory barriers.
- [Apple: Porting Metal code to Apple silicon](https://developer.apple.com/documentation/apple-silicon/porting-your-metal-code-to-apple-silicon): threadgroup/SIMD synchronization and memory flags.

Adapted from NVIDIA Corporation & Affiliates / TileGym Team, [tilegym-improve-cutile-kernel-perf](https://github.com/NVIDIA/skills/tree/0e0d506f4eb67204a62586ac5f19df3cb7ad9b1f/skills/tilegym-improve-cutile-kernel-perf) and the [autotuning pitfalls](https://github.com/NVIDIA/skills/blob/0e0d506f4eb67204a62586ac5f19df3cb7ad9b1f/skills/tilegym-cutile-autotuning/references/pitfalls.md), reviewed 2026-10-03. Changes: rewritten for multiple vendors/backends; replaced fixed thresholds and speedup claims with measured hypotheses; removed TileGym paths, automatic commits/resets, cuTile-only recipes, and setup approval gates; added portable synchronization and aliasing guards. This adapted guide is under [CC BY 4.0](licenses/nvidia-CC-BY-4.0.txt). No upstream executable code is bundled. See [SOURCES.md](SOURCES.md).
