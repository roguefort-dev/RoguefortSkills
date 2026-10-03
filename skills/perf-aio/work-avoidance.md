# Work avoidance and algorithmic improvement

Use when cost grows with items, queries, updates, or repeated recomputation. Before reducing per-operation cost, ask which operations contribute useful results. Compare a simpler algorithm and a better candidate over realistic sizes/distributions, including setup and maintenance. Consult [optimization-patterns.md](optimization-patterns.md) when representation or lifecycle needs redesign.

## Choose a mechanism

| Evidence | Candidate | Cost to include |
|---|---|---|
| Repeated full scans or all-pairs checks; selective results | Index, spatial grid/BVH, sorted data, broad-phase filtering | Construction, updates, candidate checks, skew and dense worst cases |
| Expensive derived data used repeatedly while sources rarely change | Cache, memoization, precomputed table | Keys, invalidation, retained memory, miss/rebuild cost |
| Many source edits before derived results are consumed | Dirty/versioned data, coalesced updates | Dependency propagation, deferred-work spikes, update ordering |
| Small changed region; full recomputation each tick | Incremental update, running aggregate, dependency graph | Bookkeeping, numerical drift, deletion and full-rebuild behavior |
| Most items inactive/invisible or irrelevant | Active worklist, early filtering, compacted subset | Membership maintenance, reactivation, ordering and compaction |
| Only a prefix, first match, or small top set is needed | Early termination, partial selection | Required ordering, completeness and adversarial inputs |

Consider bitsets for compact membership/set operations and sparse/dense representations for different selectivity. Compare conversion and update cost with query savings. Choose by actual density, access shape and supported operations, not a universal size threshold.

## Invalidation is part of the algorithm

Identify the canonical source and all writes that can invalidate a derived value. Include dependency changes, configuration/version changes, deletion, external mutation and concurrent updates. Cache keys must represent the full computation contract. Define who marks dirty, who recomputes, and which version readers observe.

Prefer a running/incremental result when updates are cheap and reliable. Use deferred dirty recomputation when coalescing saves work and access deadlines tolerate it. Schedule expensive rebuilds before their deadline or budget them across updates; deferring everything until use can create a spike.

For floating aggregates, repeated add/remove updates may accumulate error. Compare against a full-rebuild oracle and use a justified recalibration policy where needed. Coalescing must preserve required intermediate events and side effects.

## Acceptance

Test unchanged, sparse-change, full-change, cold-cache, deletion/reuse, dependency-change and forced-rebuild cases. Include index/cache memory, maintenance time and maximum update latency. An index that accelerates queries can still lose when writes dominate. Record both saved useful work and bookkeeping cost through [benchmarking.md](benchmarking.md).

Cross-domain examples: database covering indexes → compact simulation lookup tables; dirty render tiles → changed UI/asset regions; sparse kernel active sets → event-driven application worklists. Existing engine culling/caching and three.js BVH guidance remains in its engine guides.

## Primary references

- [SQLite query planner](https://www.sqlite.org/queryplanner.html): indexed/covering queries and ordering tradeoffs.
- [Dirty Flag](https://github.com/munificent/game-programming-patterns/blob/master/book/dirty-flag.markdown): deferred work and incremental alternatives.
- [Spatial Partition](https://github.com/munificent/game-programming-patterns/blob/master/book/spatial-partition.markdown): query/update tradeoffs.
- [Unity query filters](https://docs.unity3d.com/Packages/com.unity.entities@1.4/manual/systems-entityquery-filters.html): change-filter granularity; write access can mark a chunk potentially changed.
