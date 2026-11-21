# Technique library

5307 distinct optimization techniques mined from the commit histories of 34 performance-critical open-source projects. When several projects shipped the same technique, it has one card whose "Seen in" line credits every project (1070 cross-project duplicates folded).

## Lenses (index/<lens>.md)

| Lens | Techniques | Covers |
|---|---|---|
| algorithms | 510 | Complexity fixes, container choice, hidden quadratics, indexing, hashing, search and sort. |
| laziness | 418 | Not doing work: deferral, dirty flags, early exit, skip-when-unchanged, compute on demand. |
| amortization | 117 | Batching, bulk operations, incremental work spread over time, growth policies. |
| caching | 211 | Memoization, result and object caches, invalidation, eviction, cache keys. |
| lookup-tables | 84 | Precomputed tables, decode tables, dispatch tables, table-driven logic. |
| branching-fast-paths | 152 | Fast paths for the common case, branch removal, specialisation, predictable control flow. |
| cpu-microopt | 343 | Instruction-level savings, inlining, code size, memory-level parallelism, prefetch, arithmetic tricks. |
| compiler-build | 209 | Compiler flags, LTO/PGO, attributes, templates, constant folding, helping the optimiser. |
| simd | 131 | Vectorisation, intrinsics, data-parallel kernels, runtime CPU dispatch. |
| memory-layout | 438 | Struct packing, cache locality, hot/cold splitting, compact encodings, indirection removal. |
| allocation | 402 | Avoiding heap traffic: pools, arenas, reuse, stack and inline storage, sizing. |
| concurrency | 466 | Threads, locks, lock-free and per-thread structures, contention, false sharing, NUMA/topology. |
| scheduling | 125 | Event scheduling, task queues, timers, pacing, latency and fairness. |
| io | 353 | File and device I/O: syscalls, buffering, async and vectored I/O, mmap, read-ahead. |
| networking | 217 | Sockets, protocol handling, connection reuse, buffering, TLS, kernel network features. |
| storage-engine | 192 | Persistence, logs, compaction, LSM trees, on-disk formats, filters, replication and recovery. |
| jit-interpreters | 290 | Interpreter dispatch, bytecode design, JIT/recompiler code generation, block caches. |
| gpu-rendering | 220 | Draw-call and state batching, texture and shader caches, pipelines, presentation, UI redraw. |
| query-planning | 61 | Cost-based choice of execution strategy, predicate handling, plan rewrites. |
| compression | 37 | Lightweight encodings, per-block codec choice, operating on encoded data. |
| batch-processing | 8 | Vectorised execution: batch-at-a-time loops, per-batch checks, typed arrays over boxed values. |
| platform | 105 | OS- and hardware-specific features, per-platform paths, power and mobile constraints. |
| observability | 160 | Cheap instrumentation, counters, profiling hooks, benchmarks that guard performance. |
| other | 58 | Techniques that fit no other lens. |

## Sources (cards/<prefix>.md)

| Project | Id prefix | Techniques |
|---|---|---|
| Aeron | aeron | 106 |
| ares | ares | 96 |
| curl | curl | 103 |
| Dolphin | dolphin | 339 |
| DuckDB | duckdb | 155 |
| DuckStation | duckstation | 347 |
| Envoy | envoy | 121 |
| Flycast | flycast | 142 |
| H2O | h2o | 92 |
| HAProxy | haproxy | 179 |
| jemalloc | jemalloc | 105 |
| Kafka | kafka | 130 |
| libuv | libuv | 70 |
| melonDS | melonds | 174 |
| memcached | memcached | 95 |
| Mesen2 | mesen2 | 168 |
| mGBA | mgba | 236 |
| NATS | nats | 131 |
| Netty | netty | 121 |
| nginx | nginx | 150 |
| PCSX2 | pcsx2 | 298 |
| Pebble | pebble | 113 |
| PostgreSQL | postgresql | 138 |
| PPSSPP | ppsspp | 293 |
| Redis | redis | 197 |
| RocksDB | rocksdb | 190 |
| Seastar | seastar | 118 |
| simdjson | simdjson | 80 |
| Snes9x | snes9x | 165 |
| SQLite | sqlite | 187 |
| TigerBeetle | tigerbeetle | 132 |
| Tokio | tokio | 88 |
| Varnish | varnish | 123 |
| zstd | zstd | 125 |
