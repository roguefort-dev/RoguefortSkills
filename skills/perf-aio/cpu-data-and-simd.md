# CPU locality and SIMD

Use for hot loops, bulk transforms, simulation, image/audio processing, parsers, or cache-bound data paths. Start with sampled CPU profiles and counters where available: time, cycles, instructions, cache/TLB misses, branch misses and transferred bytes. Interpret counters with the CPU/tool model; compare useful work and latency, not isolated counter percentages.

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

## Control flow and runtime behavior

Group work by stable type/state when sorting/grouping cost is repaid and order permits. Compare branches with selection/masking using actual distributions; branch prediction and eager computation change the tradeoff. Preserve short-circuit semantics and exceptional-value behavior.

For managed/dynamic runtimes, keep hot representations predictable and check allocation, boxing, dispatch and specialization in the actual runtime. Use [runtime-optimization.md](runtime-optimization.md) for code generation and [zero-alloc.md](zero-alloc.md) for allocation mechanics.

Cross-domain transfer: GPU coalescing → contiguous CPU traversal; kernel tiling → cache-blocked transforms; material batching → homogeneous parsing/simulation batches. Verify through [benchmarking.md](benchmarking.md), including conversion and grouping costs.

## Primary references

- [Intel layouts](https://www.intel.com/content/www/us/en/developer/articles/technical/memory-layout-transformations.html): AoS/SoA/AoSoA tradeoffs and conversion overhead; vendor examples are not portable defaults.
- [LLVM vectorizers](https://llvm.org/docs/Vectorizers.html): reports, cost models, dependencies and runtime checks.
- [Data Locality](https://github.com/munificent/game-programming-patterns/blob/master/book/data-locality.markdown): contiguous access and workload-driven layout.
