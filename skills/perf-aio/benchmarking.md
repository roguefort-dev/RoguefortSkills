# Shared benchmark and acceptance contract

Use for baseline, A/B acceptance, ambiguous performance results, or regression coverage across CPU/GPU/application paths. GPU-specific timing and cache/aliasing guards live in [compute-kernels.md](compute-kernels.md); pipelines/queueing live in [concurrency-and-pipelines.md](concurrency-and-pipelines.md).

## Define the workload

Record target hardware/build/runtime, data sizes and distributions, input/output semantics, aliasing, numerical tolerance, and the user-facing budget. Include typical, small/large, worst-relevant, and sustained scenarios. Separate startup, warm steady state and rebuild paths when they differ. Benchmark the deployed optimization/configuration level; instrumentation and debug builds can alter results.

Consume/verify outputs so the compiler cannot discard useful work. Use representative unpredictable inputs when constant folding is not part of production. Reset mutable state consistently, preallocate benchmark scaffolding when measuring a kernel/hot loop, and include real allocation/copy/setup cost in a separate end-to-end case. Scope timing identically in A/B runs.

## Measure and explain variability

- Warm caches/JIT/pipelines according to the intended workload; report cold-start and warm behavior separately. A hot tiny buffer is a different workload from streaming fresh data.
- Repeat enough to see variability. Randomize or interleave baseline and candidate trials to reduce drift; use comparable power/thermal and background-load conditions.
- Report units, sample count, central estimate, spread and relevant latency percentiles. A frame-time spike or queued request can matter more than average throughput. Capture queue age and input-to-result time where appropriate.
- For arrival-driven systems, test representative arrival patterns and overload. A closed-loop client that waits for each response can conceal backlog; report offered/completed work and missed/delayed requests.
- Separate profiler/counter collection from acceptance timing. Use counters and generated code to explain a measured gain; neither instruction count nor a counter percentage alone proves success.

## Acceptance and regressions

Keep a candidate when correctness passes, gain exceeds noise, and the supported workloads meet latency/throughput, memory and quality budgets. Include conversion, bookkeeping, compile/tuning and maintenance costs where incurred. Allocation count, vector width, GPU occupancy and thread activity are diagnostic quantities; optimize the complete budget.

Record workload/build identity, hypothesis/configuration, correctness, before/after values with variability, full-path impact, and keep/revert decision. When evidence is inconclusive, retain the baseline and identify the next discriminating experiment. When target hardware is unavailable, mark performance unverified.

For a bug/regression fix, add focused correctness regression coverage. Performance regression checks need stable representative workloads and noise-aware budgets; use existing harnesses/contracts so new implementations inherit applicable coverage. Run appropriate checks without converting every optimization hypothesis into brittle timing assertions.

## Primary references

- [BenchmarkDotNet good practices](https://benchmarkdotnet.org/articles/guides/good-practices.html): optimized builds, benchmark harness and dead-code elimination.
- [Google Benchmark user guide](https://github.com/google/benchmark/blob/main/docs/user_guide.md): consumed results, repetitions and benchmark scope.
- [Android frame pacing](https://developer.android.com/games/sdk/frame-pacing): latency and presentation queues.
