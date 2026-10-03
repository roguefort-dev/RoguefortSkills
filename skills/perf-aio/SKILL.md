---
name: perf-aio
description: Performance and graphics reference. Use for data-oriented design and optimization patterns; algorithmic work avoidance; CPU locality/SIMD, concurrency, pipelines and runtime tuning; profiling frame time, GPU cost, draw calls, memory or GC; shaders (GLSL/HLSL/WGSL/MSL), compute kernels and post-processing; zero-allocation hot paths; or PixiJS v8, three.js and Unity performance work.
---

# Perf AIO

One skill, many guides. Load only the guide(s) the task needs from the router below.

## Method

Every performance task follows the same loop. Each step ends on its criterion:

1. **Measure** on the target device/build using [benchmarking.md](benchmarking.md): identify the workload's latency/throughput, memory and quality budget. Done when a representative baseline and correctness contract are recorded.
2. **Find the bound** — excess work, CPU execution/cache behavior, submission, GPU execution/bandwidth, dependencies/queueing, or allocation/GC. Done when a controlled experiment confirms a candidate: work volume → [work-avoidance.md](work-avoidance.md); CPU → [cpu-data-and-simd.md](cpu-data-and-simd.md); waits → [concurrency-and-pipelines.md](concurrency-and-pipelines.md); graphics → [shader-optimization.md](shader-optimization.md); compute → [compute-kernels.md](compute-kernels.md); GC → [zero-alloc.md](zero-alloc.md).
3. **Fix the biggest cost** in that bound. Consider reducing work before making each operation cheaper; use [optimization-patterns.md](optimization-patterns.md) when representation or update structure is involved, the relevant stack guide, and cross-domain transfer below.
4. **Validate and re-measure** with the same scenario. Done when correctness passes and the new number is recorded next to the baseline; keep the change only if the gain exceeds measurement noise and meets the workload's regression budget.

### Cross-domain transfer

Actively and creatively pattern-match optimization techniques across domains. Identify the shared cost structure—data movement, repeated work, dependency chains, contention, dispatch overhead, or allocation—and adapt a technique from another domain when its mechanism fits the measured bottleneck. Examples to spark hypotheses:

- Kernel tiling and data reuse → cache-blocked CPU processing, image/audio pipelines, or chunked simulation.
- SIMD-friendly layouts and coherent work → packed data, grouped processing, and fewer per-item decisions in application hot paths.
- Shader pass fusion and local reductions → fused traversals, batched I/O, or worker-local aggregation before shared updates.

Search beyond these examples; consult another bundled guide when it informs a specific hypothesis. Explain the transferable mechanism and re-check its assumptions in the target domain: memory model, ordering, numerical behavior, parallelism, latency, and resource limits. Treat each analogy as a candidate experiment; accept it through the shared correctness and measurement loop.

## Router

| Task | Load |
|---|---|
| Data-oriented design, ECS/chunking tradeoffs, batch interfaces, dense handles, deferred mutations, reusable optimization patterns | [optimization-patterns.md](optimization-patterns.md) |
| Algorithms, indexes, active worklists, caching/invalidation, dirty/versioned or incremental updates | [work-avoidance.md](work-avoidance.md) |
| CPU hot loops, data layout, cache locality, SIMD/vectorization, branch/control-flow experiments | [cpu-data-and-simd.md](cpu-data-and-simd.md) |
| Jobs/workers, ownership, contention/false sharing, queues/backpressure, CPU–GPU overlap, transfers/readbacks, pacing | [concurrency-and-pipelines.md](concurrency-and-pipelines.md) |
| Scan/prefix sum, compaction, sorting/grouping, histograms, segmented reductions | [parallel-primitives.md](parallel-primitives.md), then the CPU or compute guide for its backend |
| Compiler/JIT/Burst inspection, PGO/tiering, precomputed execution plans, bounded specialization | [runtime-optimization.md](runtime-optimization.md) |
| Baseline/A-B validity, noisy results, startup vs steady state, percentiles, regression budgets | [benchmarking.md](benchmarking.md) |
| Shader cost, GPU profiling, precision, branching, overdraw, bandwidth, variants | [shader-optimization.md](shader-optimization.md) |
| Compute shaders / GPU kernels: workgroups, tiling, shared memory, reductions, atomics, synchronization, tuning, benchmark validity (any GPU/backend) | [compute-kernels.md](compute-kernels.md); also [shader-optimization.md](shader-optimization.md) when comparing a graphics pass with compute |
| Post-FX: bloom, blur, tonemapping, grading, vignette, grain, FXAA/TAA, DoF | [shaders/techniques/post-processing.md](shaders/techniques/post-processing.md), then [shaders/post-fx-extras.md](shaders/post-fx-extras.md) for dual-Kawase and CoD-style bloom, AgX/Hable/ACES-Hill/Khronos Neutral tonemappers, LUTs and linear-sampled Gaussian blur |
| Any other shader technique (SDF, ray marching, noise, lighting, water, fluids, particles, AO, shadows, ...) | [shaders/GUIDE.md](shaders/GUIDE.md) — its routing table points into `shaders/techniques/` |
| Zero-allocation / GC pressure / object pooling (any language) | [zero-alloc.md](zero-alloc.md) |
| PixiJS v8 (anything) | [pixijs/pixijs/GUIDE.md](pixijs/pixijs/GUIDE.md) — the PixiJS router |
| PixiJS performance specifically | [pixijs/pixijs-performance/GUIDE.md](pixijs/pixijs-performance/GUIDE.md) |
| PixiJS custom shaders / filters | [pixijs/pixijs-custom-rendering/GUIDE.md](pixijs/pixijs-custom-rendering/GUIDE.md), [pixijs/pixijs-filters/GUIDE.md](pixijs/pixijs-filters/GUIDE.md) |
| three.js (performance, disposal, instancing, WebGPU/TSL, post-processing) | [threejs/GUIDE.md](threejs/GUIDE.md) — rule index; rule bodies live in `threejs/rules/` |
| Unity profiling (Profiler, Memory Profiler, Frame Debugger, IL2CPP, batching) | [unity/GUIDE.md](unity/GUIDE.md) and `unity/references/` |

## Conventions inside the bundled guides

- A backticked PixiJS guide name such as `pixijs-performance` means `pixijs/pixijs-performance/GUIDE.md`.
- `threejs/GUIDE.md` lists some rule IDs that have no file in `threejs/rules/`; the one-line index entry is then the whole rule.
- `shaders/` is ShaderToy-style GLSL (`iResolution`, `iTime`, `fragCoord`, `iChannel0`). Port before use:
  - three.js/WebGL: `iResolution` → a `resolution` uniform, `fragCoord` → `gl_FragCoord.xy` or `vUv * resolution`.
  - PixiJS filters: GLSL ES 3.0 with `finalColor` output and `uTexture`/`vTextureCoord`; see `pixijs-custom-rendering`.
  - Unity HLSL: `vecN`→`floatN`/`halfN`, `mix`→`lerp`, `fract`→`frac`, `atan(y,x)`→`atan2(y,x)`, `texture(s,uv)`→`SAMPLE_TEXTURE2D(tex, sampler_tex, uv)`, `M * v`→`mul(M, v)`. GLSL `mod` floors while HLSL `fmod` truncates, so they differ for negatives: use `x - y * floor(x / y)`.
- `shaders/` templates default to `precision highp float`. That is fine on desktop; on mobile, apply the precision rules in [shader-optimization.md](shader-optimization.md).
- `unity/` targets Unity 6.3 docs; confirm API availability against the project's Unity version and C# language version.

Provenance and licenses: [SOURCES.md](SOURCES.md).
