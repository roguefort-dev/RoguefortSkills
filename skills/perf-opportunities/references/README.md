# Technique library

3884 distinct optimization techniques mined from the commit histories of 21 performance-critical open-source projects. When several projects shipped the same technique, it has one card whose "Seen in" line credits every project (424 cross-project duplicates folded).

## Lenses (index/<lens>.md)

| Lens | Techniques | Covers |
|---|---|---|
| algorithms | 378 | Complexity fixes, container choice, hidden quadratics, indexing, hashing, search and sort. |
| laziness | 329 | Not doing work: deferral, dirty flags, early exit, skip-when-unchanged, compute on demand. |
| amortization | 43 | Batching, bulk operations, incremental work spread over time, growth policies. |
| caching | 116 | Memoization, result and object caches, invalidation, eviction, cache keys. |
| lookup-tables | 84 | Precomputed tables, decode tables, dispatch tables, table-driven logic. |
| branching-fast-paths | 152 | Fast paths for the common case, branch removal, specialisation, predictable control flow. |
| cpu-microopt | 217 | Instruction-level savings, inlining, code size, memory-level parallelism, prefetch, arithmetic tricks. |
| compiler-build | 209 | Compiler flags, LTO/PGO, attributes, templates, constant folding, helping the optimiser. |
| simd | 119 | Vectorisation, intrinsics, data-parallel kernels, runtime CPU dispatch. |
| memory-layout | 333 | Struct packing, cache locality, hot/cold splitting, compact encodings, indirection removal. |
| allocation | 270 | Avoiding heap traffic: pools, arenas, reuse, stack and inline storage, sizing. |
| concurrency | 340 | Threads, locks, lock-free and per-thread structures, contention, false sharing, NUMA/topology. |
| scheduling | 80 | Event scheduling, task queues, timers, pacing, latency and fairness. |
| io | 266 | File and device I/O: syscalls, buffering, async and vectored I/O, mmap, read-ahead. |
| networking | 96 | Sockets, protocol handling, connection reuse, buffering, TLS, kernel network features. |
| storage-engine | 62 | Persistence, logs, compaction, LSM trees, on-disk formats, bloom and other filters. |
| jit-interpreters | 290 | Interpreter dispatch, bytecode design, JIT/recompiler code generation, block caches. |
| gpu-rendering | 220 | Draw-call and state batching, texture and shader caches, pipelines, presentation, UI redraw. |
| query-planning | 61 | Cost-based choice of execution strategy, predicate handling, plan rewrites. |
| compression | 11 | Lightweight encodings, per-block codec choice, operating on encoded data. |
| batch-processing | 8 | Vectorised execution: batch-at-a-time loops, per-batch checks, typed arrays over boxed values. |
| platform | 80 | OS- and hardware-specific features, per-platform paths, power and mobile constraints. |
| observability | 62 | Cheap instrumentation, counters, profiling hooks, benchmarks that guard performance. |
| other | 58 | Techniques that fit no other lens. |

## Sources (cards/<prefix>.md)

| Project | Id prefix | Techniques |
|---|---|---|
| ares | ares | 96 |
| curl | curl | 103 |
| Dolphin | dolphin | 339 |
| DuckDB | duckdb | 155 |
| DuckStation | duckstation | 347 |
| Flycast | flycast | 142 |
| HAProxy | haproxy | 179 |
| melonDS | melonds | 174 |
| Mesen2 | mesen2 | 168 |
| mGBA | mgba | 236 |
| Netty | netty | 121 |
| nginx | nginx | 150 |
| PCSX2 | pcsx2 | 298 |
| PostgreSQL | postgresql | 138 |
| PPSSPP | ppsspp | 293 |
| Redis | redis | 197 |
| RocksDB | rocksdb | 190 |
| Seastar | seastar | 118 |
| Snes9x | snes9x | 165 |
| SQLite | sqlite | 187 |
| Tokio | tokio | 88 |
