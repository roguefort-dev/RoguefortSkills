---
name: webgpu-optimization
description: WebGPU optimization reference. Use when making a WebGPU renderer or compute workload faster, porting WebGL/WebGL2 to WebGPU, deciding what WebGPU can do that WebGL2 cannot (compute, storage buffers and atomics, indirect draws, render bundles, attachment load/store), building GPU-driven rendering or culling in the browser, or adopting optional features such as shader-f16, subgroups, timestamp queries and transient attachments.
---

# WebGPU optimization

WebGPU's large wins over WebGL2 are **compute with workgroup memory**, **writable storage with atomics** (fragment shaders included) and **GPU-driven work**: indirect draws and dispatches whose counts a compute pass writes. Together they keep whole algorithms on the GPU (culling, binning, sorting, simulation, software rasterization) that WebGL2 either cannot express or splits into many passes and CPU round trips.

Second come cheaper or cleaner versions of what WebGL2 already does: pre-validated pipelines and bind groups, no synchronous calls in the frame, explicit uploads and async readback, explicit attachment load/store (decisive on tile-based GPUs), one device for many canvases. Third, optional features: `shader-f16`, subgroups, dual-source blending, 11-bit float targets, transient attachments, HDR canvas.

**Restructure, don't translate.** A shader moved over one-to-one keeps all its work, adds validation and translation, and can run slower than on WebGL2 (one fill-bound renderer measured 8–24% slower on Apple silicon). The gain is the work WebGL2 forces and WebGPU lets you skip.

## Method

1. **Baseline.** Measure the workload on the target devices: GPU time per pass, CPU frame time, memory, and an output-quality contract (for example SSIM or a per-pixel tolerance). Keep an *optimized* WebGL2 baseline when comparing APIs. Done when the numbers and the correctness check are recorded. If the `perf-aio` skill is installed, its `benchmarking.md` covers A/B validity.
2. **Find the bound** with [porting-and-measurement.md](porting-and-measurement.md): CPU submission, invisible or inactive work, repeated full-screen passes, bandwidth, or fragment ALU. Done when a controlled experiment (disable, halve, or isolate the suspect) moves the number.
3. **Pick the capability** that removes that cost from [capabilities.md](capabilities.md), and the matching design from [techniques.md](techniques.md). Prefer removing work (cull, compact, classify, fuse) over making each operation cheaper.
4. **Gate it** per [feature-detection.md](feature-detection.md): every optional feature, WGSL language feature, raised limit and newer API surface is detected, requested, and paired with a fallback path.
5. **Validate, then re-measure** with the same scenario. Done when the output passes the quality contract (empty and full inputs, partial workgroups, nondeterministic atomic order, reduced precision) and the gain exceeds run-to-run noise. Keep the change only then.

## Router

| Task | Load |
|---|---|
| What WebGPU offers over WebGL2, per area: compute, storage and atomics, indirect work, CPU cost per draw, uploads and readback, render passes, WGSL, formats, debugging | [capabilities.md](capabilities.md) |
| Designs these enable, with tradeoffs and code: GPU culling + indirect draws, tile-classified passes, atomic accumulation, hardware depth test instead of depth reads, fused post chains, sorting, software rasterization, OIT, histograms, mip chains | [techniques.md](techniques.md) |
| Requesting a device, optional features, WGSL language features, limits, fallbacks, browser and Chrome versions | [feature-detection.md](feature-detection.md) |
| Bottleneck → first experiment, porting from WebGL2, where WebGL2 is equal or ahead, GPU timing, validation checklist | [porting-and-measurement.md](porting-and-measurement.md) |
| Compute kernel tuning (workgroup size, tiling, occupancy, reductions), shader cost, scan/compaction/sort primitives | the `perf-aio` skill, if installed: `compute-kernels.md`, `shader-optimization.md`, `parallel-primitives.md` |

## Boundaries

- The browser API plus negotiated features is the boundary. Native Dawn or wgpu extensions, mesh shaders, hardware ray tracing, bindless resources, multi-draw-indirect-count and device-wide barriers are not web WebGPU.
- Fragment-bound work costs the same pixels on the same hardware in either API; it gets faster only when the work itself shrinks.
- Version numbers in these files are Chrome releases. They show when a feature shipped there, not universal support.

Provenance and license: [SOURCES.md](SOURCES.md).
