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

## Sources and licenses

Original material is MIT-licensed. Upstream MIT notices are retained, and the adapted NVIDIA compute guide is CC BY 4.0. See [LICENSES.md](LICENSES.md) and [pinned sources](skills/perf-aio/SOURCES.md) for file boundaries, attribution, and modifications.

Changes are recorded in the [changelog](skills/perf-aio/docs/CHANGELOG.md) and [plain-language update notes](skills/perf-aio/docs/IN_APP_CHANGELOG.md).
