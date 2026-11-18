# CPU locality and SIMD

Use for hot loops, bulk transforms, simulation, image/audio processing, parsers, or cache-bound data paths. Start with sampled CPU profiles and counters where available: time, cycles, instructions, cache/TLB misses, branch misses and transferred bytes. Interpret counters with the CPU/tool model; compare useful work and latency, not isolated counter percentages. Use the profiler that fits the platform (for example Linux `perf`, Intel VTune, AMD uProf, Apple Instruments, Windows Performance Analyzer, Superluminal or Tracy).

## Match layout to access

- AoS keeps fields of a record together; SoA keeps one field across records contiguous; AoSoA blocks field arrays to balance both access shapes. Choose from fields touched together, traversal shape, mutation and downstream consumers. Include repacking and boundary conversion in end-to-end measurements.
- Split rarely used fields from hot processing data when saved traffic repays lookup/lifecycle complexity. Compact representations, integer handles and bitsets can reduce working sets; check decoding, range limits and identity semantics.
- Arrange traversal to follow the contiguous dimension. Loop interchange and cache blocking can improve reuse when dependencies permit; sweep plausible block sizes on target data rather than encoding assumed cache sizes.
- Fuse compatible traversals when saved reads/writes exceed increased live state and register pressure. Preserve dependency order and numerical results. Keep hot interface boundaries batch-shaped so a caller can process ranges without repeated setup.

For storage/identity patterns load [optimization-patterns.md](optimization-patterns.md). For false sharing and parallel ownership load [concurrency-and-pipelines.md](concurrency-and-pipelines.md).

## Vectorization workflow

1. Read the generated code or optimization reports for the actual build/runtime. LLVM provides loop-vectorization success/missed/analysis remarks; use the equivalent in the project's compiler, Burst inspector, or JIT disassembly. Done when vectorization or its blocker is observed.
2. Test a specific blocker: loop-carried dependence, alias uncertainty, unsuitable call, irregular access, mixed work, or numerical restrictions. Hoist invariant work and compare homogeneous batches when grouping semantics permit. Done when generated-code evidence and a measured result explain the change.
3. Use explicit SIMD/intrinsics only when compiler/library capabilities leave a measured gap. Preserve supported-ISA dispatch and a valid fallback. Done when all paths agree within the contract and the complete workload improves.

An alias/alignment assertion is a correctness promise; make it only when the caller's contract proves it. Test overlapping inputs if allowed, unaligned buffers, empty/small arrays, and lengths around vector/block boundaries. Handle tails without invalid reads. Floating-point reassociation, FMA and approximate operations can change results; validate against the established tolerance and determinism requirements.

SIMD width, alignment requirements, register availability and instruction throughput vary. Wider vectors can increase register pressure or change CPU operating behavior. Compare them on the supported devices. Software prefetch is a targeted experiment after access/working-set improvements; extra instructions or pollution can erase its benefit.

## Instruction-level costs

Use after layout and vectorization when profiles show a hot loop that is limited by its own instructions or by dependent memory accesses. Instruction latency and throughput vary by microarchitecture; look them up for the supported CPUs rather than assuming numbers.

- **Dependency chains:** a loop-carried chain (`sum += x[i]`) runs at the latency of each operation, not at the CPU's throughput. Several independent accumulators combined at the end expose instruction-level parallelism. For floating point this reassociates the sum, and compilers do it only with explicit permission, so validate the result against the tolerance.
- **Expensive operations:** division, square root, transcendental functions and modulo by a runtime value cost far more than multiply/add. Hoist reciprocals out of loops, use shifts/masks for power-of-two divisors, and replace functions with approximations or recurrences only within the established error budget.
- **Lookup tables vs computation:** a table trades arithmetic for cache footprint and load latency. Measure with the real working set; a table that fits a microbenchmark's cache can lose once the full workload competes for it.
- **Dependent loads (pointer chasing):** linked lists, trees and per-node indirection serialize on memory latency because each address depends on the previous load. Flatten into arrays or indices, or interleave several independent traversals so multiple misses are in flight.
- **Subnormal floats:** values near zero can take a slow microcode path on many CPUs, which shows up in decaying feedback such as filters, damping and accumulators. Flush-to-zero/denormals-are-zero modes (per thread) or explicit flushing remove it where the numerical contract allows; [realtime-audio.md](realtime-audio.md) covers the audio case.
- **Code layout:** keep hot paths compact and move error handling and rare cases out of line (cold/noinline attributes, `[[likely]]`/`[[unlikely]]` hints, PGO-driven function ordering). Check front-end/instruction-cache counters first; aggressive inlining and unrolling grow code and can cost more than they save.
- **TLB pressure:** large working sets with scattered access can be limited by address-translation misses rather than data caches. Improve locality first; large/huge pages are a platform-specific experiment with memory and deployment costs.

## Control flow and runtime behavior

Group work by stable type/state when sorting/grouping cost is repaid and order permits. Compare branches with selection/masking using actual distributions; branch prediction and eager computation change the tradeoff. Preserve short-circuit semantics and exceptional-value behavior.

For managed/dynamic runtimes, keep hot representations predictable and check allocation, boxing, dispatch and specialization in the actual runtime. Use [runtime-optimization.md](runtime-optimization.md) for code generation and [zero-alloc.md](zero-alloc.md) for allocation mechanics.

Cross-domain transfer: GPU coalescing → contiguous CPU traversal; kernel tiling → cache-blocked transforms; material batching → homogeneous parsing/simulation batches. Verify through [benchmarking.md](benchmarking.md), including conversion and grouping costs.

## Primary references

- [Intel layouts](https://www.intel.com/content/www/us/en/developer/articles/technical/memory-layout-transformations.html): AoS/SoA/AoSoA tradeoffs and conversion overhead; vendor examples are not portable defaults.
- [LLVM vectorizers](https://llvm.org/docs/Vectorizers.html): reports, cost models, dependencies and runtime checks.
- [Data Locality](https://github.com/munificent/game-programming-patterns/blob/master/book/data-locality.markdown): contiguous access and workload-driven layout.
- [Agner Fog's optimization resources](https://www.agner.org/optimize/): microarchitecture behavior and instruction latency/throughput tables.
- [uops.info](https://uops.info/): measured instruction latency, throughput and port usage across x86 microarchitectures.
