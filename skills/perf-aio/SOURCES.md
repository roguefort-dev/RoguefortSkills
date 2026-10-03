# Sources

The original bundled skill packs below are MIT-licensed (fetched 2026-09-30). The compute guide added 2026-10-03 adapts CC BY 4.0 documentation as described below. License texts are in `licenses/`.

| Folder | Source | Commit |
|---|---|---|
| `pixijs/` | [pixijs/pixijs-skills](https://github.com/pixijs/pixijs-skills) — all 27 skills, `SKILL.md` renamed to `GUIDE.md` | df555b8 |
| `shaders/` (except `post-fx-extras.md`) | [minimax-ai/skills](https://github.com/MiniMax-AI/skills) `shader-dev` | 60aaae5 |
| `threejs/` | [emalorenzo/three-agent-skills](https://github.com/emalorenzo/three-agent-skills) `three-best-practices` (MIT per package.json/README; no LICENSE file in repo) | f950f95 |
| `unity/` | [nice-wolf-studio/unity-claude-skills](https://github.com/nice-wolf-studio/unity-claude-skills) `unity-performance` | a149d3e |

Written for this bundle: `SKILL.md`, `shader-optimization.md`, `zero-alloc.md`, `shaders/post-fx-extras.md`.

## Portable optimization expansion — 2026-10-03

Original synthesis: `optimization-patterns.md`, `work-avoidance.md`, `cpu-data-and-simd.md`, `concurrency-and-pipelines.md`, `parallel-primitives.md`, `benchmarking.md`, and `runtime-optimization.md`. Primary references are linked in each guide. These files describe general mechanisms and decision criteria; no upstream code or book passages are copied. Hardware-specific documentation supplies examples rather than fixed portable defaults.

The bundle's original material is MIT-licensed when published in RoguefortSkills. Upstream MIT notices remain in `licenses/`; `compute-kernels.md` retains its CC BY 4.0 license. Repository-wide license boundaries are recorded in the public repository's `LICENSES.md`.

## Compute optimization addition — 2026-10-03

`compute-kernels.md` adapts NVIDIA Corporation & Affiliates / TileGym Team's [tilegym-improve-cutile-kernel-perf](https://github.com/NVIDIA/skills/tree/0e0d506f4eb67204a62586ac5f19df3cb7ad9b1f/skills/tilegym-improve-cutile-kernel-perf) (`SKILL.md`, `references/optimization-playbook.md`, `references/performance-model.md`, `references/perf-knobs-catalog.md`) and [tilegym-cutile-autotuning/references/pitfalls.md](https://github.com/NVIDIA/skills/blob/0e0d506f4eb67204a62586ac5f19df3cb7ad9b1f/skills/tilegym-cutile-autotuning/references/pitfalls.md), commit `0e0d506f4eb67204a62586ac5f19df3cb7ad9b1f`.

The source repository [license notice](https://github.com/NVIDIA/skills/blob/0e0d506f4eb67204a62586ac5f19df3cb7ad9b1f/README.md#license) assigns documentation/skills to CC BY 4.0 and source code to Apache-2.0. Only documentation is adapted here; its license is retained in [licenses/nvidia-CC-BY-4.0.txt](licenses/nvidia-CC-BY-4.0.txt). The adapted guide is CC BY 4.0; existing files retain their licenses. No NVIDIA executable code or separate upstream skill installation is included.

Changes: rewritten for GPU-agnostic shader/compute work, with portable benchmarking, resource tradeoffs, bounded tuning, and correctness guards. Removed fixed arithmetic-intensity thresholds, occupancy targets, speedup promises, cuTile-specific optimizations, TileGym paths, automatic commits/resets, and approval gates. Synchronization and device-limit guidance was checked against official NVIDIA, AMD GPUOpen, Apple, WGSL, and HLSL documentation linked in the guide.

Local corrections to bundled content:
- `threejs/rules/shader-mobile.md`: "avoid discard, use alphaTest" replaced (alphaTest compiles to discard); normal.z reconstruction note fixed (sign loss).
- `threejs/rules/object-pooling.md`: bullet example no longer allocates per frame (`clone()` → preallocated vector + `addScaledVector`).
- `threejs/GUIDE.md`: `shader-avoid-discard` index line updated to match.
