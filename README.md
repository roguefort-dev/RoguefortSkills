# RoguefortSkills

Agent skills for measured, practical optimization.

## Install Perf AIO

```sh
npx skills add roguefort-dev/RoguefortSkills --skill perf-aio
```

Choose your agent and installation scope in the installer. Add `-g` for a global installation. To inspect available skills without installing:

```sh
npx skills add roguefort-dev/RoguefortSkills --list
```

You can also copy `skills/perf-aio/` into your agent's skills directory. Start with its [SKILL.md](skills/perf-aio/SKILL.md); it routes tasks to focused references.

## Install WebGPU Optimization

```sh
npx skills add roguefort-dev/RoguefortSkills --skill webgpu-optimization
```

Or copy `skills/webgpu-optimization/` into your agent's skills directory. It pairs with Perf AIO, whose compute and shader guides it points to when both are installed.

## Perf AIO

Perf AIO connects performance diagnosis with programming patterns and graphics guidance:

- Data-oriented design, layouts, batch processing, stable handles, and deferred mutations.
- Work avoidance, indexing, caching, invalidation, and incremental updates.
- CPU locality, SIMD, compiler/runtime tuning, and bounded specialization.
- Parallel jobs, ownership, contention, queues, backpressure, and CPU–GPU pipelines.
- Portable shader/compute optimization, scan, compaction, reductions, and sorting.
- Allocation/GC, shader techniques, post-processing, PixiJS v8, three.js, and Unity.
- Realistic benchmarks, correctness contracts, and latency, memory, and quality budgets.

The shared method actively and creatively transfers optimization mechanisms across domains. Each analogy becomes an experiment with explicit assumptions and correctness checks. GPU advice is vendor agnostic; optional hardware features require capability checks and a portable fallback. Choices are measured on the target workload and device.

Load only the references needed for the task. Confirm API availability against your project's versions, especially in bundled engine guides. This repository contains guidance; benchmark results depend on the implementation and target hardware.

## WebGPU Optimization

WebGPU Optimization covers what WebGPU does faster or more cleanly than WebGL2, and how to turn that into measured gains:

- Compute and workgroup memory, writable storage with atomics, and GPU-driven indirect draws and dispatches.
- Cheaper draws and API state: pipelines, bind groups, dynamic offsets, render bundles, no synchronous calls.
- Uploads, async readback, zero-copy video, explicit attachment load/store and transient attachments for tile-based GPUs.
- WGSL and optional features: `textureGather`, `f16`, subgroups, dual-source blending, 11-bit float targets, timestamps.
- Designs these enable, such as GPU culling, tile-classified passes, atomic accumulation, sorting, software rasterization, visibility buffers and OIT, with code where it helps.
- Feature detection with fallbacks, porting from WebGL2, where WebGL2 is equal or ahead, and validation before accepting a speed-up.

Start with its [SKILL.md](skills/webgpu-optimization/SKILL.md).

## Sources and licenses

Original material is MIT-licensed. Upstream MIT notices are retained, and the adapted NVIDIA compute guide is CC BY 4.0. See [LICENSES.md](LICENSES.md), [Perf AIO sources](skills/perf-aio/SOURCES.md) and [WebGPU Optimization sources](skills/webgpu-optimization/SOURCES.md) for file boundaries, attribution, and modifications.

Changes are recorded in each skill's changelog ([Perf AIO](skills/perf-aio/docs/CHANGELOG.md), [WebGPU Optimization](skills/webgpu-optimization/docs/CHANGELOG.md)) and plain-language update notes ([Perf AIO](skills/perf-aio/docs/IN_APP_CHANGELOG.md), [WebGPU Optimization](skills/webgpu-optimization/docs/IN_APP_CHANGELOG.md)).
