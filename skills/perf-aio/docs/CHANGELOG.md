# Changelog

## 2026-11-18

- Added `realtime-audio.md`: callback deadline measurement, operations to keep off the audio thread, priority inversion, lock-free parameter/state/event handoff with deferred reclamation, block-based DSP, denormals, and AudioWorklet/SharedArrayBuffer specifics.
- Added "Instruction-level costs" to `cpu-data-and-simd.md`: dependency chains and multiple accumulators, expensive operations, lookup tables vs computation, pointer chasing, subnormal floats, hot/cold code layout, and TLB pressure. Named common CPU profilers and added instruction-table references.
- Routed both from `SKILL.md` and cross-linked the audio guide from `zero-alloc.md`.

## 2026-10-03

- Added `compute-kernels.md`, adapting NVIDIA TileGym documentation into GPU-agnostic guidance for WGSL, HLSL, GLSL, MSL, and native compute backends.
- Covers valid GPU timing, bottleneck experiments, memory reuse, tiling, workgroup geometry, resource pressure, reductions, synchronization, aliasing, numerical contracts, and bounded tuning.
- Updated skill discovery and routing; graphics and compute now share correctness and measurement-noise acceptance criteria.
- Recorded pinned upstream provenance and CC BY 4.0 attribution. Existing graphics/engine guides remain available.
- Added active cross-domain pattern matching to the shared method: adapt kernel, SIMD, graphics, and other optimization mechanisms to matching bottlenecks, then validate target assumptions, correctness, and measured benefit.
- Added data-oriented design and optimization patterns, work avoidance/invalidation, CPU layout/cache/SIMD, concurrency/ownership/backpressure, CPU–GPU pipelines, parallel primitives, shared benchmarking, and compiler/runtime specialization guides.
- Routed each topic through the shared method and cross-linked existing allocation and compute material. Allocation goals now include retained-memory and latency budgets; layout guidance has a single detailed owner.
- Prepared public distribution under `skills/perf-aio/` in RoguefortSkills, with install instructions, source attribution and explicit license boundaries.
