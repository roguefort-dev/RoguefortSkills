# Technique library

4308 optimization techniques mined from the commit histories of 21 performance-critical open-source projects.

## Lenses (index/<lens>.md)

| Lens | Techniques | Covers |
|---|---|---|
| algorithms | 422 | Complexity fixes, container choice, hidden quadratics, indexing, hashing, search and sort. |
| laziness | 362 | Not doing work: deferral, dirty flags, early exit, skip-when-unchanged, compute on demand. |
| amortization | 52 | Batching, bulk operations, incremental work spread over time, growth policies. |
| caching | 135 | Memoization, result and object caches, invalidation, eviction, cache keys. |
| lookup-tables | 94 | Precomputed tables, decode tables, dispatch tables, table-driven logic. |
| branching-fast-paths | 159 | Fast paths for the common case, branch removal, specialisation, predictable control flow. |
| cpu-microopt | 249 | Instruction-level savings, inlining, code size, memory-level parallelism, prefetch, arithmetic tricks. |
| compiler-build | 224 | Compiler flags, LTO/PGO, attributes, templates, constant folding, helping the optimiser. |
| simd | 122 | Vectorisation, intrinsics, data-parallel kernels, runtime CPU dispatch. |
| memory-layout | 357 | Struct packing, cache locality, hot/cold splitting, compact encodings, indirection removal. |
| allocation | 311 | Avoiding heap traffic: pools, arenas, reuse, stack and inline storage, sizing. |
| concurrency | 389 | Threads, locks, lock-free and per-thread structures, contention, false sharing, NUMA/topology. |
| scheduling | 86 | Event scheduling, task queues, timers, pacing, latency and fairness. |
| io | 299 | File and device I/O: syscalls, buffering, async and vectored I/O, mmap, read-ahead. |
| networking | 105 | Sockets, protocol handling, connection reuse, buffering, TLS, kernel network features. |
| storage-engine | 67 | Persistence, logs, compaction, LSM trees, on-disk formats, bloom and other filters. |
| jit-interpreters | 331 | Interpreter dispatch, bytecode design, JIT/recompiler code generation, block caches. |
| gpu-rendering | 233 | Draw-call and state batching, texture and shader caches, pipelines, presentation, UI redraw. |
| query-planning | 67 | Cost-based choice of execution strategy, predicate handling, plan rewrites. |
| compression | 11 | Lightweight encodings, per-block codec choice, operating on encoded data. |
| batch-processing | 8 | Vectorised execution: batch-at-a-time loops, per-batch checks, typed arrays over boxed values. |
| platform | 87 | OS- and hardware-specific features, per-platform paths, power and mobile constraints. |
| observability | 75 | Cheap instrumentation, counters, profiling hooks, benchmarks that guard performance. |
| other | 63 | Techniques that fit no other lens. |

## Sources (cards/<prefix>.md)

| Project | Id prefix | Techniques |
|---|---|---|
| ares | ares | 164 |
| curl | curl | 166 |
| Dolphin | dolphin | 339 |
| DuckDB | duckdb | 155 |
| DuckStation | duckstation | 347 |
| Flycast | flycast | 227 |
| HAProxy | haproxy | 179 |
| melonDS | melonds | 174 |
| Mesen2 | mesen2 | 168 |
| mGBA | mgba | 236 |
| Netty | netty | 176 |
| nginx | nginx | 150 |
| PCSX2 | pcsx2 | 298 |
| PostgreSQL | postgresql | 220 |
| PPSSPP | ppsspp | 293 |
| Redis | redis | 197 |
| RocksDB | rocksdb | 190 |
| Seastar | seastar | 157 |
| Snes9x | snes9x | 165 |
| SQLite | sqlite | 187 |
| Tokio | tokio | 120 |
