# RoguefortSkills

Agent skills for measured, practical optimization and codebase cleanup.

| Skill | What it does |
|---|---|
| [perf-aio](skills/perf-aio/SKILL.md) | Performance diagnosis, optimization patterns and graphics guidance |
| [john-wick](skills/john-wick/SKILL.md) | Cleanup sweeps that find every instance of a code smell; first pass: implicit state machines |

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

## Install John Wick

```sh
npx skills add roguefort-dev/RoguefortSkills --skill john-wick
```

Or copy `skills/john-wick/` into your agent's skills directory.

## John Wick

John Wick runs cleanup sweeps over a whole codebase. Each kind of target is a pass with its own guide, and every pass scopes the codebase, sweeps it for signals, confirms each candidate by reading the code, and reports findings ranked by severity with exact locations. It reports by default and changes code only when asked.

The first pass finds implicit state machines: lifecycles and modes encoded in loose booleans, status strings, nullable fields, counters or call-order rules instead of an explicit state type. For each one it reconstructs the states and transitions, flags invalid combinations and gaps, and suggests an explicit model in the project's language.

## Sources and licenses

Original material is MIT-licensed. Upstream MIT notices are retained, and the adapted NVIDIA compute guide is CC BY 4.0. See [LICENSES.md](LICENSES.md) and [pinned sources](skills/perf-aio/SOURCES.md) for file boundaries, attribution, and modifications.

Changes are recorded in the [changelog](skills/perf-aio/docs/CHANGELOG.md) and [plain-language update notes](skills/perf-aio/docs/IN_APP_CHANGELOG.md).
