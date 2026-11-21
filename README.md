# RoguefortSkills

Agent skills for practical optimization, codebase cleanup, and search performance analysis.

| Skill | What it does |
|---|---|
| [perf-aio](skills/perf-aio/SKILL.md) | Performance diagnosis, optimization patterns and graphics guidance |
| [john-wick](skills/john-wick/SKILL.md) | Cleanup sweeps that find every instance of a code smell; first pass: implicit state machines |
| [search-console-opportunities](skills/search-console-opportunities/SKILL.md) | Prioritize SEO actions using Google Search Console exports |
| [perf-opportunities](skills/perf-opportunities/SKILL.md) | Full performance review of a codebase against 4,300+ techniques mined from 21 performance-critical projects; writes an opportunities report |

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
- CPU locality, SIMD, instruction-level costs, compiler/runtime tuning, and bounded specialization.
- Real-time audio/DSP: callback deadlines, lock-free handoff, denormals, and AudioWorklet.
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

## Install Search Console Opportunities

```sh
npx skills add roguefort-dev/RoguefortSkills --skill search-console-opportunities
```

Or copy `skills/search-console-opportunities/` into your agent's skills directory. Give the agent your Search Console property, brand, and a few facts about your site. It can work from supplied exports or download them with a connected browser extension and access to the Downloads folder.

## Install Perf Opportunities

```sh
npx skills add roguefort-dev/RoguefortSkills --skill perf-opportunities
```

Or copy `skills/perf-opportunities/` into your agent's skills directory.

## Perf Opportunities

Perf Opportunities reviews a codebase, directory, file or diff for ways to make it faster or lighter, and writes every opportunity it finds to a plain `performance-opportunities.html` report. It reports by default and changes code only when asked.

Each opportunity cites a technique a mature project actually shipped. The library holds 4,308 domain-generic principles mined from the commit histories of Redis, nginx, SQLite, RocksDB, HAProxy, DuckDB, PostgreSQL, Seastar, curl, Netty, Tokio and ten console emulators. Each technique records why it is faster, where else it applies, when it does not pay off, and how the source project used it, with commit references. Techniques are indexed by lens (algorithms, laziness, caching, memory layout, allocation, concurrency, I/O, SIMD and more), so the agent loads only the cards it needs.

## Sources and licenses

Original material is MIT-licensed. Upstream MIT notices are retained, and the adapted NVIDIA compute guide is CC BY 4.0. See [LICENSES.md](LICENSES.md) and [pinned sources](skills/perf-aio/SOURCES.md) for file boundaries, attribution, and modifications.

Changes are recorded in the [changelog](skills/perf-aio/docs/CHANGELOG.md) and [plain-language update notes](skills/perf-aio/docs/IN_APP_CHANGELOG.md).
