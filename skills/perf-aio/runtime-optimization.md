# Compiler, runtime and specialization

Use when hot loops remain expensive after algorithm/access improvements, or compilation, dispatch, startup and specialization cause latency. Preserve the project's compiler/runtime/backend. Read the official documentation for its actual version before selecting flags or APIs.

## Inspect the result

Use compiler optimization remarks, disassembly, JIT/Burst inspectors and sampled profiles to see what executes. Check inlining/devirtualization, bounds/alias checks, boxing/allocation, vectorization, register spills and dependency chains. Optimize observed costs; source-code appearance is weak evidence. SIMD legality and tails are covered in [cpu-data-and-simd.md](cpu-data-and-simd.md).

For dynamic runtimes, use stable hot shapes/types and consistent representations where they match the domain. Check runtime transitions and deoptimization evidence before restructuring code. Compare cold initialization with warmed representative execution; JIT tiering and dynamic PGO can change code during a run.

## Profile-guided optimization

Where supported, collect representative profiles and build/run the matching configuration. Keep workload coverage, compiler/runtime version and profile identity with the result. Stale/narrow profiles may optimize the wrong paths. Compare startup, steady state, binary size and representative out-of-profile cases. Check whether the runtime already enables PGO/tiering before changing defaults.

## Bounded specialization and precomputation

Specialize stable shapes, dtypes, types or options when removed inner-loop work repays compilation/plan construction. Cache the compiled pipeline or plan object, not just its configuration, when recreating it defeats caching. Keys must include every relevant backend/compiler/device and semantic condition; define invalidation and lifetime.

Cap variant count and retained code/data; compare instruction-cache and startup costs. Keep a general path for unsupported or infrequent cases. A precomputed table/plan also needs [work-avoidance.md](work-avoidance.md)'s validity contract. Reuse library/framework specialization and cache facilities when available.

Inlining, unrolling, AOT, LTO, explicit intrinsics and fast math each trade resources or semantics. Check the actual build capabilities and generated result; enable only choices that satisfy the workload's correctness and end-to-end budget. Avoid changing numerical tolerance to make a candidate pass.

Cross-domain examples: shader variants → specialized serializers/parsers; cached pipeline objects → reusable query or transform plans; grouped material paths → stable hot dispatch. Verify through [benchmarking.md](benchmarking.md), including miss/compile and cache-eviction cases.

## Primary references

- [Clang PGO](https://clang.llvm.org/docs/UsersManual.html#profile-guided-optimization): instrumentation/sample profiles and compiler options.
- [.NET compilation configuration](https://learn.microsoft.com/en-us/dotnet/core/runtime-config/compilation): tiering, startup and PGO.
- [V8 fast properties](https://v8.dev/blog/fast-properties): object representations; verify behavior in the deployed runtime.
- [LLVM vectorizers](https://llvm.org/docs/Vectorizers.html): optimization remarks and cost models.
