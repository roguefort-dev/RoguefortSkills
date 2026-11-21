# ares — technique cards (96)

Each card is exactly 7 lines. Fetch one with: grep -A 6 "^### ares-NNN " ares.md

### ares-002 — Test whether a trigger falls inside the step you just took
Lens: algorithms · Source: ares
Principle: Don't advance time one unit at a time just to see whether a trigger point has passed. Advance by the whole step and check whether the trigger lies in the half-open interval `[now, now+step)`. You get the same answer from one comparison instead of one per unit.
Why it is faster: —
Applies elsewhere: collision detection with swept intervals, rate limiters that check whether a deadline passed between two samples, animation keyframe lookups between frames.
Watch out: —
Seen in ares: The fast CPU core detects H-IRQs with `hcounter() <= htime*4 && hcounter() + clocks > htime*4` inside `add_clocks`, instead of checking every clock. A follow-up extended this range testing across scanline boundaries, which the changelog credits for part of a 12–15% gain. 9000bb408 2010-08 “Update to 20100810-2 release.” · fa0f1c1e9 2010-08 “Update to 20100811 release.”

### ares-004 — Poll idle waits at a coarse interval
Lens: laziness · Source: ares
Principle: While a component waits for a wake-up condition, check the condition every N units instead of every unit. The waiting work drops by a factor of N, at the cost of up to N units of wake-up latency.
Why it is faster: —
Applies elsewhere: spin-then-sleep backoff in locks, polling loops in embedded firmware, health checks with coarse intervals.
Watch out: —
Seen in ares: The GBA HALT state advances `step(16)` between IRQ checks instead of 1 cycle. The changelog calls this "probably less precise, but provides a massive speedup", and the same WIP took Mr. Driller 2 from 88 to 172 fps. Later, `CPU::step` still idles in 16-clock steps while halted. 01b4cb991 2012-04 “Update to v087r20 release.” · ba384a7c4 2017-08 “Update to v104 release.”

### ares-005 — Cooperative threads that sync only when one side gets ahead
Lens: concurrency · Source: ares
Principle: Run each concurrent component as a coroutine with its own clock. Switch to another component only when you are about to observe its state and it is behind. Most of the time each component runs a long stretch without a context switch, and you still observe correct ordering where it matters.
Why it is faster: —
Applies elsewhere: discrete-event simulators, lazily synchronized actors, optimistic replication that reconciles only on reads.
Watch out: —
Seen in ares: The CPU core synchronizes lazily: `synchronize_ppu() { if(ppu.clock < 0) co_switch(ppu.thread); }`, and coprocessors likewise. The cost of the alternative shows up when the Game Boy core was temporarily switched to "CPU/PPU single-step now instead of sync just-in-time (~30% slower)". 9000bb408 2010-08 “Update to 20100810-2 release.” · 72b6a8b32 2016-01 “Update to v096r04 release.”

### ares-006 — Run small subordinate tasks as plain calls, not coroutines
Lens: concurrency · Source: ares
Principle: A context switch saves and restores registers and moves to a different stack, which costs cache footprint. If a subordinate task can be written to return between steps, call it in a loop from its owner instead of giving it its own coroutine.
Why it is faster: —
Applies elsewhere: inlining a generator into a state machine, stackless coroutines (C++20, Rust async) instead of fibers, polling a sub-parser directly.
Watch out: —
Seen in ares: The fast core runs the sound CPU as a function: `synchronize_smp() { while(smp.clock < 0) smp.run(); }`, while PPU and coprocessors still use `co_switch`. Later, when designing a pixel PPU, the changelog notes that a full libco thread for the object renderer would be "horrendously slow". It asks for a stackless yielding coroutine instead. 9000bb408 2010-08 “Update to 20100810-2 release.” · 2461293ff 2017-06 “Update to v102r19 release.”

### ares-007 — Absolute timestamps with a precomputed per-thread step scale
Lens: algorithms · Source: ares
Principle: Give each clock domain an absolute time in a common fine unit, and precompute the cost of one tick in that unit. Advancing becomes one multiply-add, and syncing becomes one compare. Every so often, subtract the minimum from all clocks so they don't overflow.
Why it is faster: —
Applies elsewhere: multi-rate audio graphs, simulation with mixed sample rates, distributed tracing timestamps in ns.
Watch out: —
Seen in ares: `Thread::setFrequency` stores `_scalar = 1.0L / frequency * Attosecond + 0.5L`. `step(clocks)` is `_clock += _scalar * clocks`, `synchronize(t)` is `if(_clock > t._clock) co_switch(...)`, and the scheduler subtracts the minimum clock on exit. This replaced pairwise `clock -= clocks * other.frequency` bookkeeping and gave an 8–12% speedup in most cores. An earlier step in the same direction hard-coded the DMG frequency so the core did `clock++` instead of `clock += 1 * cpu.frequency`. ca277cd5e 2016-07 “Update to v100r14 release.” · abe639ea9 2012-04 “Update to v088r01 release.”

### ares-008 — Synchronize at the coarsest grain that preserves correctness
Lens: laziness · Source: ares
Principle: Each synchronization point costs a switch. Raise the sync granularity, for example per output sample or per line instead of per cycle, for components whose effects only become visible at that grain.
Why it is faster: —
Applies elsewhere: batching DB writes per request, flushing logs per N lines, syncing game physics at fixed ticks rather than per frame update.
Watch out: —
Seen in ares: For Super Game Boy audio the CPU "sync[s] DSP and Coprocessor once per scanline". In the fast profile, the S-DSP syncs per sample instead of per clock (`#if defined(PROFILE_FAST) step(32*3*8)`), described as "a fairly significant speedup" that only hurts one game. b433838e9 2011-02 “Update to v075r06 release.” · 5d29700fa 2018-05 “Update to v106r33 release.”

### ares-010 — Generate fast and exact variants from one description
Lens: compiler-build · Source: ares
Principle: Describe the behavior once in a small DSL and generate both a coarse version and a fine-grained version from it. You get two speed tiers without maintaining two hand-written copies that drift apart.
Why it is faster: —
Applies elsewhere: protobuf/flatbuffers code generators, SQL query compilation to specialized loops, generating SIMD and scalar kernels from one template.
Watch out: —
Seen in ares: The performance-profile S-SMP core added `core/generate.cpp`, which parses cycle-based `op_*.b` markup into either an opcode-based core (one switch) or a cycle-based core (two switches), chosen with `#if defined(CYCLE_ACCURATE)`. It replaced a hand-written fast SMP from the prior WIPs, which had been 3–5% faster than the old one. 9a3650c6a 2011-05 “Update to v078r01 release.” · 67e6a6e74 2011-05 “Update to v078r02 release.” · 7ffaeb2ac 2011-05 “Update to v078r03 release.”

### ares-011 — Defer independent work to a batch, then run it in parallel
Lens: concurrency · Source: ares
Principle: If per-item work only reads a snapshot of shared state, record the inputs as you go and process all items together at a barrier. The items are independent, so the batch spreads across cores, and the main sequential loop sheds that work.
Why it is faster: —
Applies elsewhere: deferred rendering command lists, collecting log records and compressing them on a worker pool, map-reduce stages.
Watch out: —
Seen in ares: The new fast PPU latches each scanline's state during emulation. At V=241 it runs `#pragma omp parallel for` over `lines[y].render()` for all visible lines. With the skeleton in place bsnes already ran at about 220 fps, against 230 fps with no PPU and 120 fps with the accuracy PPU. 6c8e3c885 2018-05 “Update to v106r28 release.”

### ares-016 — Build each line's working set once before the per-pixel pass
Lens: laziness · Source: ares
Principle: Scan a large table once per batch to extract the few entries that apply, decoding their attributes at the same time. The inner loop then iterates over a short decoded list instead of re-reading and re-decoding the whole table.
Why it is faster: —
Applies elsewhere: spatial culling to a visible list before drawing, prefiltering rules applicable to a request, building per-frame active-entity lists.
Watch out: —
Seen in ares: The WonderSwan PPU added `renderSpriteDecode()`, which per scanline reads sprite attributes from IRAM into a `sprites` vector of at most 32 entries. The changelog calls it an "absolutely massive speed-up". Earlier, the SNES scanline PPU had used a global dirty flag and then sprite-list updates on OAM writes to avoid rebuilding sprite lists. c33065fbd 2016-03 “Update to v097r24 release.” · 70429285b 2010-08 “Updated to v067r23 release.”

### ares-018 — Store comparison operands in the form the hot test needs
Lens: branching-fast-paths · Source: ares
Principle: If a value is written rarely but compared often, apply the transform on write. The frequent comparison then needs no arithmetic.
Why it is faster: —
Applies elsewhere: storing deadlines as absolute timestamps rather than durations, keeping lowercase keys for case-insensitive lookups.
Watch out: —
Seen in ares: The S-CPU stores `io.htime = (htime + 1) << 2` on register writes and un-transforms it only on partial updates. The IRQ test becomes `hcounter(10) == io.htime`. 35ff15f83 2018-07 “Update to v106r50 release.”

### ares-019 — Flatten address decoding into direct-indexed tables
Lens: lookup-tables · Source: ares
Principle: Replace a search over mapped ranges with a precomputed table indexed by the full address, storing a handler id and a translated offset. Each access then costs two loads and an indirect call, and you pay for it in memory.
Why it is faster: —
Applies elsewhere: page tables, URL router tries compiled to arrays, opcode dispatch tables, IP lookup tables.
Watch out: —
Seen in ares: `Bus::read` used to walk `rdpage[addr >> 13]` testing `addr - lo < hi` for each mapping. It became `return reader[lookup[addr]](target[addr]);`, with 16 MB of `lookup` and 64 MB of `target` filled at map time for the 24-bit bus. The changelog raised the minimum RAM to 128 MB. This followed a WIP that moved to bound function callbacks and measured them as slower than virtual calls (−4.5% to −15%). 2d7308656 2011-01 “Update to v074r03 release.” · 1a065bafb 2011-01 “Update to v074r06 release.”

### ares-020 — Range checks as a compile-time mask compare
Lens: branching-fast-paths · Source: ares
Principle: For power-of-two aligned ranges, "lo ≤ x ≤ hi" reduces to `(x & mask) == lo`, with the mask computed at compile time. That is one AND and one compare, and it inlines everywhere.
Why it is faster: —
Applies elsewhere: MMIO decoding in firmware, IP subnet membership tests, page-aligned pointer checks.
Watch out: —
Seen in ares: An `alwaysinline` helper `template<unsigned lo, unsigned hi> bool within(addr) { static const unsigned mask = ~(hi ^ lo); return (addr & mask) == lo; }` replaced hand-written range tests, e.g. `if(within<0x4000, 0x7fff>(addr))` in cartridge mappers. 054bdd409 2011-01 “Update to v074r10 release.”

### ares-021 — Remove type-erased callbacks from hot paths
Lens: compiler-build · Source: ares
Principle: A `std::function`-style member is checked and then called indirectly on every use, and the compiler can't inline it. If there are only a couple of special cases, test for them directly in the hot code.
Why it is faster: —
Applies elsewhere: replacing per-field observer callbacks with explicit dirty checks, avoiding `std::function` in tight loops, replacing event emitters with direct calls in hot code.
Watch out: —
Seen in ares: The GSU register type had `function<void (uint16)> modify`, invoked on every assignment (`if(modify) modify(i)`), and was bound only for r14/r15. Removing it gave a 3–4% speedup in SuperFX games. 9b452c9f5 2016-06 “Update to v098r17 release.”

### ares-022 — Cumulative cases as a fall-through switch
Lens: branching-fast-paths · Source: ares
Principle: When higher modes do everything lower modes do plus more, a chain of `if(mode >= X)` tests runs every comparison. A descending `switch` with fall-through makes one jump and then runs straight-line code.
Why it is faster: —
Applies elsewhere: Duff's-device style unrolling, protocol version upgrades applying migrations in sequence, tiered feature initialization.
Watch out: —
Seen in ares: `Background::get_tile()` and `get_tile_color()` replaced `if(mode >= BPP2)… if(mode >= BPP4)… if(mode >= BPP8)` with `switch(regs.mode) { case BPP8: … case BPP4: … case BPP2: … }`. This took 94.5 fps back to 97.0 after an accuracy fix had cost speed. 7a96321e7 2012-02 “Update to v086r03 release.”

### ares-023 — Index registers by array instead of switching on the number
Lens: memory-layout · Source: ares
Principle: Put same-typed fields in an array so selection by index is one addressed load or store. A switch over sixteen named fields becomes a jump table and an indirect branch.
Why it is faster: —
Applies elsewhere: struct-of-arrays for entity components, bytecode VMs with register arrays, lookup by enum into arrays rather than switch.
Watch out: —
Seen in ares: The M68K `read<Size>(DataRegister/AddressRegister)` used a 16-case `switch` over `r.d0…r.a6, ssp/usp`. It became `return clip<Size>(r.da[reg.number]);`. The changelog says register access became "about eight times faster than the old system". 4b897ba79 2016-07 “Update to v100r10 release.”

### ares-025 — Tune the toolchain: optimization level, target CPU, PGO
Lens: compiler-build · Source: ares
Principle: Compiler flags and versions can be worth double-digit percentages. Use aggressive levels, target the host ISA, omit frame pointers, and use profile-guided optimization so the compiler knows which branches are hot.
Why it is faster: —
Applies elsewhere: PGO builds of browsers and Python, `-march=native` for local HPC builds, release/debug/profile build matrices in CI.
Watch out: —
Seen in ares: The Makefile gained `make pgo=instrument` / `pgo=optimize`. An earlier note says unprofiled binaries were about 15% slower. Flags became `-march=native -O3 -fomit-frame-pointer` (97→99 fps). The changelog reports gcc-4.7 binaries "up to 20% faster" than gcc-4.6. Later, `build=release` became `-O2`, with `build=optimize` for `-O3`. b28c54770 2011-08 “Update to v081r03 release.” · 1a32ed7cf 2010-10 “Updated to 20100813 release.” · 7a96321e7 2012-02 “Update to v086r03 release.” · bb4db22a7 2012-04 “Update to v088r07 release.” · 1ff315838 2017-09 “Update to v104r13 release.”

### ares-031 — Use a recursive IIR instead of a long FIR where phase allows
Lens: algorithms · Source: ares
Principle: A windowed-sinc FIR filter needs many taps per output sample and often a decimation stage. A cascade of second-order IIR sections gets a comparable cutoff from a handful of multiply-adds and two state variables per section.
Why it is faster: —
Applies elsewhere: audio EQ in DAWs and embedded DSP, sensor smoothing on MCUs, anti-aliasing before resampling.
Watch out: —
Seen in ares: The audio path replaced a Blackman-windowed sinc FIR plus Hermite resampler with a sixth-order Butterworth built from `struct Biquad { double a0, a1, a2, b1, b2; double z1, z2; }` sections in transposed direct form II. The changelog calls it "faster, no need for decimation, less code". Earlier, the sinc filter had dropped the frame rate from about 240 to 150 fps. 839813d0f 2016-06 “Update to v098r13 release.”

### ares-035 — Keep coroutine stacks small for cache locality
Lens: memory-layout · Source: ares
Principle: Every context switch touches the top of another stack. Oversized stacks spread those hot regions over more memory and cache sets, so size stacks to what the code needs.
Why it is faster: —
Applies elsewhere: goroutine/fiber stack sizing, thread-pool stack limits, green-thread runtimes.
Watch out: —
Seen in ares: When `co_create` stacks were raised from `65536 * sizeof(void*)` to `262'144 * sizeof(void*)` to chase a driver crash, the changelog noted "a slight speed penalty due to cache locality going down between threads". The later `Thread::create` uses `64 * 1024 * sizeof(void*)`. 55e507d5d 2016-04 “Update to v098r05 release.” · ca277cd5e 2016-07 “Update to v100r14 release.”

### ares-036 — Spend precision only where you need it
Lens: algorithms · Source: ares
Principle: Accept a few units of jitter in a rare event's timing when exact handling would force fine-grained stepping everywhere. Then check whether the jitter is actually harmless, and back out where it isn't.
Why it is faster: —
Applies elsewhere: approximate timers (timer slack in Linux), sampled profiling, probabilistic counters.
Watch out: —
Seen in ares: Range-based Super Scope/Justifier latching in the fast core allowed "the trigger point may vary by 1-2 pixels", which the changelog accepts as avoiding "a massive speed penalty". A later attempt to let the CPU run ahead and sync only on controller-port accesses failed for light guns, so that speed hack was disabled (92→82 fps for those games only). 70429285b 2010-08 “Updated to v067r23 release.” · 6227974bf 2011-12 “Update to v084r05 release.”

### ares-037 — Pre-decode a run of instructions into a chain of direct calls
Lens: jit-interpreters · Source: ares
Principle: An interpreter pays to fetch, decode and dispatch through a switch on every step. A "cached interpreter" does the decoding once per basic block and emits machine code that is only a straight list of calls to the existing handlers, with the decoded operands baked in as immediates. The handlers stay the same, so accuracy is unchanged, and the cost of decoding and of the indirect dispatch branch goes away.
Why it is faster: —
Applies elsewhere: compiling a regex or bytecode program into an array of handler pointers with their arguments (direct-threaded code), template engines that compile to closures, a SQL engine that turns a plan into a chain of operator callbacks.
Watch out: —
Seen in ares: The v118 notes say the N64 and PS1 cores use cached interpreters, "a middle-ground" between interpreters and recompilers. For the 32X, `sh2.cpp` switched from `recompiler.cpp` to `cached.cpp`, which was faster than pure interpretation but more compatible than the full recompiler. Its `emitInstruction` expands `decoder.hpp` so each opcode becomes `call(&SH2::name, &self, args...)`. A later commit made `emitInstruction` `alwaysinline` so the decoder switch folds into the emit loop. 44bdc531b 2021-02 “Update to ares v118 release.” · 27d0826e3 2021-07 “sh2: use cached interpreter instead of recompiler” · 934202c24 2021-07 “sh2: further optimize code gen for the cached interpreter”

### ares-041 — Synchronise only with the party that already orders everyone else
Lens: concurrency · Source: ares
Principle: When one component already synchronises with all the others at fine granularity, the other components don't need to sync pairwise among themselves. Point each of them at that hub only. This removes redundant scheduler work without changing the ordering guarantees.
Why it is faster: —
Applies elsewhere: clients that talk to a central sequencer instead of to every peer, a star topology replacing all-pairs heartbeats, letting the main thread act as the single barrier.
Watch out: —
Seen in ares: In the Mega Drive core, the APU, OPN2, VDP and PSG each called `Thread::synchronize(cpu, vdp, vdp.psg, opn2)` or `(cpu, apu)` in `step()`. Because the CPU syncs the whole system on every clock, each of them now calls `Thread::synchronize(cpu)` only. 27bc5bc2b 2022-12 “md: reduce sync between components”

### ares-042 — In a fast mode, skip eager sync for components that rarely talk
Lens: laziness · Source: ares
Principle: Checking every optional component after every step is expensive when most of them rarely interact. In a build or mode that favours speed, drop the eager per-step sync for those components and let them catch up at coarser natural boundaries that already exist.
Why it is faster: —
Applies elsewhere: updating secondary caches or search indexes on a timer instead of on every write. Polling slow sensors once per frame instead of once per control-loop tick.
Watch out: —
Seen in ares: `sfc::CPU::step()` synchronised every coprocessor on every step. Under `PROFILE_PERFORMANCE`, that loop is compiled out, and coprocessors are still synced once per scanline in `CPU::scanline()`. The commit reports a significant gain on low-power machines. e88e2cf03 2022-05 “sfc: lazily sync co-processors”

### ares-043 — Fold lock-step tasks into one thread of control
Lens: concurrency · Source: ares
Principle: Running two components as separate coroutines that always advance at the same rate only buys context switches. Merge them into one loop that calls each component's step function in turn. The switches and the scheduler bookkeeping disappear, and the order of execution stays the same.
Why it is faster: —
Applies elsewhere: merging two goroutines that ping-pong on a channel into one loop, running audio and MIDI processing in the same callback, fusing pipeline stages that always process the same item together.
Watch out: —
Seen in ares: On the Famicom, `APU` and the cartridge board `Interface` stop being `Thread`s. Their `tick()` functions, which did `Thread::step(rate()); Thread::synchronize(cpu);`, were deleted, and the CPU now drives `apu.main()` and `cartridge.main()` directly. The commit notes this is only valid because they share one base clock. 265ba002e 2022-12 “fc: merge apu/cpu/cartridge into one cothread”

### ares-046 — Drive subordinate components with catch-up loops instead of threads
Lens: concurrency · Source: ares
Principle: When the main component owns the timeline, the others can be plain objects with a time debt. After the main component runs, subtract the elapsed time from each one and call its `main()` in a `while(clock < 0)` loop. There are no coroutine switches, and each component runs in a burst, which is friendly to the cache.
Why it is faster: —
Applies elsewhere: fixed-timestep game loops (`while(accumulator >= dt) update()`), deferred-work queues drained after each request, audio mixers rendering whatever a stream is owed.
Watch out: —
Seen in ares: In `n64::CPU::synchronize()`, `vi`, `ai`, `rsp` and `rdp` each have a `clock` that is reduced by the CPU's elapsed clocks, followed by `while(rsp.clock < 0) rsp.main();` and so on. v120r03 also scaled everything to a common `clocks = Thread::clock * 2`, so the separate `* 3` scaling for the RSP is gone. 8c7bad148 2021-05 “Update to ares v120r03 release.”

### ares-048 — Key a compilation cache by content hash so swapped-in code is reused
Lens: jit-interpreters · Source: ares
Principle: If a small code region is repeatedly overwritten with a handful of recurring programs, a cache keyed by address keeps throwing away compiled code. Key compiled artifacts by a hash of the source bytes instead. When a previously seen program is loaded again, the lookup hits and recompilation is skipped. Store only a small {hash, pointer} record in the index, not the payload.
Why it is faster: —
Applies elsewhere: shader caches keyed by source hash, ccache and Bazel action caches, content-addressed storage in git or Docker layers, memoising WebAssembly module compilation by module hash.
Watch out: —
Seen in ares: `RSP::Recompiler::pool()` hashes IMEM and looks the hash up in a `set`. A refactor split `Pool` (only `Block* blocks[1024]`) from a small `PoolHashPair{pool, hashcode}`. That removed a heap copy of a mostly zeroed pool and some wasted code-cache space. A collision found by a test led to replacing the ad-hoc `h*33+b` hash with CRC32. f9e4070b1 2021-12 “n64: improve rsp recompiler pool allocation” · c2735c5c1 2022-05 “n64: use a stronger hash in RSP recompiler to avoid collisions”

### ares-049 — Make cache entries smaller so partial overwrites don't evict everything
Lens: jit-interpreters · Source: ares
Principle: One cache entry per large region means that any change to part of the region evicts all of it, and two programs sharing the region never coexist. Split the region into smaller independently keyed slices. More of the cache survives each change, so the hit rate goes up.
Why it is faster: —
Applies elsewhere: chunked content-defined deduplication, per-module rather than per-bundle browser caching, page-level instead of file-level invalidation in a buffer cache.
Watch out: —
Seen in ares: The RSP cache used one context and hash per 4 KB of IMEM. It became 16 contexts, each hashing 256 bytes (`context[address >> 8]`, `pools[address >> 8]`), and DMA and IMEM writes now invalidate only their own slice. The commit says this "significantly increases the cache hit ratio" because games use several small microcode programs. f1e36f277 2022-08 “n64: split rsp recompiler cache into 256-byte sized blocks”

### ares-051 — Two-level lookup: a direct-mapped fast path backed by a content-hashed store, with dirty bits
Lens: jit-interpreters · Source: ares
Principle: Put a direct-addressed array in front of the slower hashed store. Record writes as bits in a small dirty mask, and only on the next lookup evict the fast-path entries whose ranges overlap a dirty bit. On a fast-path miss, hash only the code that would actually execute and consult the long-lived store. Unchanged code then costs one array load, and repeated programs are never recompiled.
Why it is faster: —
Applies elsewhere: a TLB in front of page tables, a per-request memo in front of a shared cache, an L1 dictionary in front of Redis with version stamps for invalidation.
Watch out: —
Seen in ares: In `RSP::Recompiler::block`, writes set bits in `dirty` at 64-byte granularity. On the next call, `context[]` entries whose `mask(address, block->size)` overlaps are nulled. On a miss, `measure()` finds the block's length by scanning to its terminal branch and `hash()` covers only those bytes before checking `blocks`. The commit notes this stopped unbounded cache growth, and blocks may now wrap around the end of IMEM. 8759a7d3e 2022-08 “n64: precise rsp block invalidation”

### ares-052 — Emit short direct calls when the target is within reach
Lens: jit-interpreters · Source: ares
Principle: Calling through a register loaded with a 64-bit immediate costs a 10-byte load plus an indirect call. When the target lies within ±2 GB of the call site, a 5-byte `call rel32` is smaller and is a direct branch that the CPU predicts trivially. Check the distance at emit time and fall back to the long form only when needed.
Why it is faster: —
Applies elsewhere: linker relaxation, `-mcmodel=small`, trampolines and veneers on ARM, choosing short jumps in assemblers.
Watch out: —
Seen in ares: A new pseudo-op, `call(imm64 target, reg64 scratch)`, computes `dist = distance(target) - 5` and emits `call(imm32{dist})` when the distance fits in 32 bits, otherwise `mov scratch, target; call scratch`. All four recompilers (SH-2, N64 CPU, RSP, PS1) and the `nall` call helpers switched to it. d17a5ea04 2021-09 “recompiler: use IP-relative direct branches when possible”

### ares-053 — Reserve the JIT code buffer statically, next to the program text
Lens: memory-layout · Source: ares
Principle: Executable memory from `mmap` can land far from your binary, which forces long indirect calls into your helpers. A large zero-initialised static array lives in the binary's BSS segment, close to its code. It takes no space on disk, and the OS commits pages only when they are touched. Bump-allocate out of it and reset the whole arena when it fills up.
Why it is faster: —
Applies elsewhere: arena allocators for request-scoped data, pre-reserved pools in embedded firmware, placing hot data next to code for `rel32` addressing.
Watch out: —
Seen in ares: `ares/memory/fixed-allocator.cpp` defines `alignas(4096) u8 fixedBuffer[1_GiB]`, later 128 MiB, behind a singleton `bump_allocator`. Each core's `power()` acquires its code cache from it, and the cores flush the arena with `allocator.release()` once less than 1 MiB is available. On Apple silicon, the allocator falls back to `mmap` with `MAP_JIT`, and cache sizes were reduced "to improve likelihood of dynamic allocations being close to ares image". 2770c2000 2021-09 “recompiler: use fixed bss allocation for code caches” · b56dac15e 2021-12 “recompiler: fix page protection on Apple silicon”

### ares-054 — Share one exit sequence per generated function
Lens: jit-interpreters · Source: ares
Principle: If every early-exit point in generated code repeats the full epilogue (restore registers, fix the stack, return), code size grows with the number of exits and pressure on the instruction cache rises. Emit the epilogue once and make every exit a conditional jump to it. A label API that resolves jump offsets keeps this manageable.
Why it is faster: —
Applies elsewhere: the single-cleanup `goto out;` idiom in C, compilers merging identical return blocks (tail merging), shared error handlers in hand-written assembly.
Watch out: —
Seen in ares: Every recompiler's `emit()` repeated `pop r13/rbp/rbx; ret` after each instruction's `test rax`. Now the prologue is followed by `jmp entry; epilogue: …pops…; ret; entry:`, and each instruction ends with `jnz epilogue`. `declareLabel` and `defineLabel` replaced the hand-computed `imm32(epilogue - size() - 6)` offsets. 9b99a961f 2021-09 “recompiler: emit one epilog per block” · 394da44a0 2021-09 “recompiler: add label API to simplify emitting branches”

### ares-055 — Generate code directly for the host's calling convention
Lens: jit-interpreters · Source: ares
Principle: Writing generated code for one ABI and adding register shuffles to adapt it to another costs extra moves on every call. Instead, alias the argument registers per ABI at compile time and emit directly into them. Also avoid registers that are callee-saved only on some platforms, so prologues don't have to save them.
Why it is faster: —
Applies elsewhere: FFI stubs generated per platform (libffi), cross-platform assembly with macro-defined argument registers, Wasm runtimes emitting native-ABI trampolines.
Watch out: —
Seen in ares: Recompilers assumed System V and, on Windows, emitted `mov r9,rcx; mov r8,rdx; …` plus a `push rsi/rdi`. They now use aliases `ra0…ra5` (for example `mov(ra1d, Rn)`) and no longer touch `rsi` or `rdi`. The commit reports roughly 10% smaller generated code on Windows. 3ca1f9ebb 2021-09 “recompiler: directly emit code for target ABI”

### ares-056 — Address state relative to a base register, biased so offsets fit in one byte
Lens: jit-interpreters · Source: ares
Principle: Encoding absolute 64-bit addresses in each instruction bloats code and makes it position-dependent. Pass the state pointers in callee-saved registers and use `[base + disp]`. Point the base into the middle of the array, at `&r[16]`, so that all 32 registers fall within a signed 8-bit displacement. That uses the shortest x86 encoding.
Why it is faster: —
Applies elsewhere: the GOT and PC-relative addressing in PIC code, an interpreter keeping the frame pointer in a register, Lua and LuaJIT biasing `BASE` for short offsets.
Watch out: —
Seen in ares: The N64 recompiler loads `rbx = &ipu.r[0] + 16` and `r13 = &fpu.r[0] + 16`, with `#define Rd dis8(rbx, (Rdn - 16) * 8)`. Later the pointers arrive as function arguments (`Block::execute(self)` calls `code(&ipu.r[16], &self, &fpu.r[16])`), and `Lo`/`Hi` use `offsetof`, replacing `mem64(&cpu.ipu.hi)`. The commit reports code 5–7% smaller, and nearly 40% smaller for the RSP. 340f48d02 2021-05 “Update to ares v120 release.” · d31d5a931 2021-10 “recompiler: avoid encoding absolute data addresses”

### ares-059 — Target a portable JIT IR instead of one host ISA
Lens: platform · Source: ares
Principle: A JIT written straight to x86 encodings runs at interpreter speed on every other CPU. Retarget the front end to a small portable low-level IR library (here sljit). Every supported host then gets native code from one codebase. Make guest state a plain POD struct so the generator can address fields by `offsetof`.
Why it is faster: —
Applies elsewhere: PCRE2's JIT (also sljit), LLVM ORC or Cranelift as JIT backends, GNU lightning, eBPF JITs.
Watch out: —
Seen in ares: `nall/recompiler/amd64` was replaced by `nall/recompiler/generic`, whose `add32`, `cmp32_jump`, `call()` and so on wrap `sljit_emit_*`. All four recompilers were ported, and later commits enabled native ARM64, with MAP_JIT and `jitprotect` toggling. The SH-2 registers had moved into `struct Registers` beforehand so they could be addressed by offset. e88cb48f1 2021-12 “sh2: move registers into POD struct” · a8e35d029 2021-12 “recompiler: rewrite for machine-independence” · b56dac15e 2021-12 “recompiler: fix page protection on Apple silicon”

### ares-063 — Model only the cost of a cache, not its contents
Lens: laziness · Source: ares
Principle: When a fast path only needs a cache's timing effect, track just the tags and valid bits. Charge the hit or miss cost and skip filling the data lines. Keep the full data-carrying model for the slow, exact path.
Why it is faster: —
Applies elsewhere: cost-model simulators that estimate cache misses without moving data, cache-hit predictors in query planners, approximate LRU bookkeeping in a CDN simulator.
Watch out: —
Seen in ares: The N64 `InstructionCache` has two paths. `step(address)`, "used by the recompiler to simulate instruction cache fetch timing", sets `valid` and `tag` and charges 48 or 2 cycles. `fetch()`, used by the interpreter, calls `line.fill()`, which performs eight bus reads. 81e15695c 2022-08 “n64: a little more performance tuning”

### ares-064 — Reuse one SIMD implementation across ISAs with a translation header
Lens: simd · Source: ares
Principle: Rewriting a large set of SSE intrinsics for NEON doubles the maintenance, and a scalar fallback is far slower. A header that maps `_mm_*` intrinsics onto NEON equivalents (sse2neon) lets the same vector code compile natively on ARM64. Remove the `#if` guards that restricted it to x86.
Why it is faster: —
Applies elsewhere: porting x86 SIMD codecs, hashing or JSON parsers to Apple silicon or Graviton. SIMDe or Highway as portable SIMD layers.
Watch out: —
Seen in ares: RSP vector-unit emulation used SIMD only on amd64 (`SISD = … | !Architecture::amd64`). After vendoring `sse2neon.h`, `n64.hpp` defines `using v128 = __m128i` on ARM64 too. The `#if defined(ARCHITECTURE_AMD64)` wrappers around `VABS`, `VADD`, the `_mm_shuffle_epi8` element-select tables and others were removed. ffa7040b6 2022-08 “n64: use sse2neon to emulate RSP with SIMD instructions on ARM64”

### ares-065 — Gate SIMD per feature at compile time, with a scalar fallback
Lens: simd · Source: ares
Principle: Forcing `-msse4.2` on the whole build lets the compiler emit those instructions everywhere, so the program crashes at startup on older CPUs. Instead, compile for a chosen baseline, test the predefined feature macro (`__SSE4_1__`), select the vector or scalar implementation with `constexpr`, and tell the user when the slow path is active.
Why it is faster: —
Applies elsewhere: numeric libraries choosing AVX2 or SSE kernels, distributing x86-64-v2/v3 builds, Rust's `cfg(target_feature)`.
Watch out: —
Seen in ares: The global `-msse4.2` was removed from `n64/GNUmakefile`. `Architecture::sse41` is derived from `__SSE4_1__`, `Accuracy::RSP::SISD` includes `!Architecture::sse41`, and `RSP::power()` reports "RSP vectorization disabled". Official builds use `-march=nehalem`, and local builds use `-march=native`. Moving the baseline to x86-64-v2 was reverted because older GCC didn't accept the flag. acd2130a4 2022-04 “n64: fix SIGILL at startup when SSE4.2 is not supported” · 6399a5cd1 2022-04 “nall: sse42 -> sse41” · f7657dca4 2022-07 “desktop-ui: optimize for generic x86-64-v2 rather than nehalem” · 080b53a65 2022-07 “Revert …x86-64-v2…”

### ares-067 — Cache compiler output across rebuilds
Lens: compiler-build · Source: ares
Principle: Rebuilding unchanged translation units is pure waste. Prefixing the compiler command with a content-hashing compiler cache returns stored objects whenever the preprocessed input and the flags match.
Why it is faster: —
Applies elsewhere: ccache or sccache in CI, Bazel and Gradle remote caches, Turborepo and Nx task caching.
Watch out: —
Seen in ares: `nall/GNUmakefile` gained `ccache=true`, which prepends `ccache` to `compiler.c/.cpp/.objc/.objcpp`. 36af497fd 2022-09 “nall: add support for ccache”

### ares-069 — Keep a decoded shadow copy, updated on write
Lens: memory-layout · Source: ares
Principle: If data is stored packed (for example 4-bit pixels in 16-bit words) but read far more often than written, maintain a second, unpacked array. Each write updates both copies. Readers then index the shadow directly, with no shifting or masking in the hot loop.
Why it is faster: —
Applies elsewhere: denormalised read models in CQRS, materialised views, pre-decoded glyph bitmaps, bitsets mirrored as byte arrays for SIMD scanning.
Watch out: —
Seen in ares: In the Mega Drive performance VDP, `VRAM::write()` stores the word and also expands it into `pixels[offset<<2 | 0..3] = data >> 12/8/4/0 & 15`. `Background::renderScreen` then walks `tileData = &vdp.vram.pixels[…]` with `tileData += incrementX`. Each layer renders a whole line into its own buffer, and `render()` composites the line in one pass. c78d931e2 2022-02 “Revert "ares: remove performance renderers"”

### ares-070 — Track state from events instead of polling a syscall
Lens: io · Source: ares
Principle: Querying the OS for state that changes rarely costs a syscall each time, and hot code may ask thousands of times per frame. Subscribe to the change notifications, cache the value in a field, and answer queries from the field.
Why it is faster: —
Applies elsewhere: the `visibilitychange` event instead of polling `document.hidden`, inotify instead of `stat()` loops, caching network reachability from OS callbacks.
Watch out: —
Seen in ares: On Windows, `pWindow::focused()` returned `GetForegroundWindow() == hwnd` on every call, which caused noticeable slowdown when Game Boy games polled the joypad in a loop. `windowProc` now sets `focus` on `WM_SETFOCUS` and clears it on `WM_KILLFOCUS`, and `focused()` returns the field. a01ccb72f 2021-11 “hiro: track window focus via messages, not polling”

### ares-071 — Make huge bitmaps sparse: hash table of 64-bit words
Lens: memory-layout · Source: ares
Principle: A dense bitmap over a 64-bit key space can't be allocated. Split each key into a high part, used as a hash key, and a low 6-bit index into a 64-bit word stored in the entry. Memory then scales with the number of touched regions rather than with the size of the key space, and dense neighbourhoods still share words.
Why it is faster: —
Applies elsewhere: Roaring bitmaps, sparse visited-sets in crawlers or graph traversals, coverage maps in fuzzers.
Watch out: —
Seen in ares: The instruction tracer's "do not trace twice" mask was `vector<u08>`, sized `1 << (addressBits-3)`, which broke when the N64 moved to 64-bit addresses. It is now `hashset<VisitMask>`, where `VisitMask{upper = address>>6, u64 mask}` and `visit()` tests and sets `1ull << (address & 0x3f)`. f1b7eb058 2022-10 “debugger: use hashset to track visited addresses”

### ares-073 — Table-driven translation with a likely fast path for the common mode
Lens: lookup-tables · Source: ares
Principle: For address or permission translation, index a small precomputed table by the top bits and return it directly in the common mode. Fall into a slower decoder only for the rare mode, behind a `likely()` hint, so the hot path stays one load and one branch that is predicted correctly.
Why it is faster: —
Applies elsewhere: fast-path routing tables with a slow-path regex router, TLB-style caches over page-table walks, ASCII fast paths in UTF-8 decoders.
Watch out: —
Seen in ares: `n64::CPU::segment()` reads `context.segment[address >> 29 & 7]` and, `if(likely(context.bits == 32))`, returns it immediately. Only in 64-bit mode does it switch into `kernelSegment64` or `userSegment64`. 2973fa71e 2022-05 “n64: reactivate 64-bit memory map”

### ares-074 — Schedule context-switch code for the pipeline
Lens: concurrency · Source: ares
Principle: A context switch saves one register set and loads another. Writing all the stores and then all the loads serialises the work, and restoring the stack pointer and return address last delays the final branch. Interleave each store with the matching load and restore SP and LR first, so the memory operations overlap and the return target is known early.
Why it is faster: —
Applies elsewhere: hand-tuned coroutine or fiber switches (Boost.Context), green-thread runtimes, RTOS context switches on Cortex-A.
Watch out: —
Seen in ares: `libco/aarch64.c`'s `co_swap_function` used to store d8–d15 and x19–x30 before loading anything. It now begins with `mov x16,sp; stp x16,x30,[x1]; ldp x16,x30,[x0]; mov sp,x16` and then pairs each `stp` with its `ldp`, ending in `br x30`. 1c026f5ed 2020-02 “libco/aarch64: Optimize co_swap_function”

### ares-075 — Skip an expensive init phase, but charge its time
Lens: laziness · Source: ares
Principle: A long hardware or setup sequence can often be skipped when its only effect is reaching a known state. If later code observes elapsed time, advance the clock by the measured duration of the skipped phase at the point where it would have run. You keep the speed without breaking timing assumptions.
Why it is faster: —
Applies elsewhere: restoring a VM or container from a snapshot instead of booting it, mocking a slow dependency in tests while advancing a fake clock, warm-start caches that report the work as done.
Watch out: —
Seen in ares: The N64 forces RDRAM initialisation to be skipped unless `Accuracy::RDRAM::Broadcasting` is set. When IPL3 reads `RI_SELECT`, `RI::readWord` now calls `cpu.step(17'641'000)`, matching the count register measured on hardware. Without it, Wave Race 64 Shindou froze on the logo. Fast-boot options for PS1 and NGP were added in v114r22 and v114r23. 3212ce014 2021-09 “n64: consume cycles to compensate for skipped RDRAM initialization”

### ares-077 — Lazy invalidation with dirty bitmasks and generation counters
Lens: laziness · Source: ares
Principle: Don't evict cached entries the moment a write happens. Set a bit in a per-region dirty mask, or bump one global generation number. When a region is next looked up, compare its stored generation and mask and evict only the entries that overlap. Writes become a single OR or increment, and eviction work happens only for regions that are used again.
Why it is faster: —
Applies elsewhere: cache-busting by version stamp in web asset pipelines, epoch-based invalidation of memoized selectors, GPU descriptor caches that are marked dirty per binding range.
Watch out: —
Seen in ares: `SH2::Recompiler::invalidate(address,size)` used to null one block pointer. It now ORs a 64-bit `mask()` (one bit per 4 bytes) into `pool->dirty`. `pool()` then evicts only blocks whose `[address,size)` overlaps the mask. A cache purge does `generation++`, and pools in the cached area whose generation is stale get cleared when they are next touched. Blocks now record their `size`, so a long write no longer issues two separate invalidations. 241f0e9a3 2023-03 “sh2: precise recompiler block invalidation”

### ares-081 — Frequency-counted front cache before a linear search
Lens: memory-layout · Source: ares
Principle: If a linear search over N entries keeps hitting the same few, put a tiny cache of pointers to the hottest entries in front of it. Count hits, and on a miss replace the least-used slot. Most lookups then finish after a handful of comparisons.
Why it is faster: —
Applies elsewhere: move-to-front lists for symbol tables, a small "recent" set in front of a regex or route table, an MRU cache of open file handles.
Watch out: —
Seen in ares: `CPU::TLB::load/store` used to scan all 32 TLB entries on every mapped access. The work was split into `load(vaddr, const Entry&)` helpers, with a `TlbCache` of four `{entry*, frequency}` slots checked first. A full-scan hit calls `tlbCache.insert`, which replaces the slot with the lowest frequency. 181570e50 2023-06 “n64: add cache to speed up the lookup of the 4 most active pages”

### ares-082 — A pared-down fast path for lookups that were already validated
Lens: branching-fast-paths · Source: ares
Principle: If an earlier stage has already handled the error cases for an input, a second lookup on the same input can run without exception, debug or validity checks. Try that variant first and fall back to the full path when it returns nothing. Check that "already validated" is really true, because any check you skip that isn't covered upstream becomes a correctness bug.
Why it is faster: —
Applies elsewhere: unchecked array access inside a loop whose bounds were checked once, `get_unchecked` in Rust after a validated index, trusted internal API calls that skip input sanitizing.
Watch out: —
Seen in ares: The N64 code added `devirtualizeFast` and `TLB::loadFast`, which have no address exceptions or debugger hooks, for the recompiler's icache stepping. `CPU::instruction()` also tried `recompiler.fastFetchBlock(devirtualizeFast(pc))` before the full `devirtualize`/`block()` path. The fast block lookup was later reverted because it skipped alignment checks, and the commit notes it can come back "if something like this proves truly beneficial". 6e367916d 2023-06 “n64: add fast variant of devirtualize; fix remaining task from PR #1069” · fbaf7d811 2023-06 “n64: use devirtualizeFast for faster lookups of cached code blocks” · 1ffb4f6d5 2024-09 “n64: use devirtualize for code block lookup”

### ares-083 — Move the loop inside the callee
Lens: other · Source: ares
Principle: When a dispatcher calls a component repeatedly until the component is "caught up", put that loop inside the component. One call then does all the pending work, and the function-call and dispatch overhead is paid once per batch instead of once per step.
Why it is faster: —
Applies elsewhere: batch APIs (`writev`, bulk inserts), event handlers that drain an entire queue per wakeup, `requestAnimationFrame` callbacks that process all queued updates.
Watch out: —
Seen in ares: `CPU::synchronize()` used to run `while(vi.clock < 0) vi.main();` for each of VI, AI, RSP, RDP and PIF. Each `main()` now contains its own `while(Thread::clock < 0)` loop and is called once. The per-frame loop also moved from `System::run` into `CPU::main`, which removed another call layer. e142c3be5 2023-06 “n64: slight performance boost by moving exit logic to components rather than system”

### ares-085 — Test the steady-state case first and mark it likely
Lens: branching-fast-paths · Source: ares
Principle: A state machine that spends almost all its time in one state should check that state first, with an early return and a `likely` hint. The common path is then one predictable branch, and the compiler lays it out as straight-line code.
Why it is faster: —
Applies elsewhere: connection handlers that check "established" before handshake states, parsers that test the common token class first, the `if (cached) return` guard at the top of a function.
Watch out: —
Seen in ares: `PIF::mainHLE()` checked `state == Run` last, after the Init, challenge and other states. That test moved to the top as `if(likely(state == Run)) return;`. b697623c1 2023-02 “n64: in PIF HLT, move Rn state check to the top and mark it likely”

### ares-086 — Give low-importance components coarse time slices
Lens: concurrency · Source: ares
Principle: In a cooperative scheduler, each context switch has a fixed cost. A component whose observable behaviour doesn't need fine timing can advance in much larger quanta, so it gets scheduled far less often.
Why it is faster: —
Applies elsewhere: polling health checks every few seconds instead of every frame, batching telemetry flushes, low-priority background workers with long sleep intervals.
Watch out: —
Seen in ares: `PIF::mainHLE()` went from `step(1024)` to `step(10240*8)`. The commit says it "shouldn't matter much anyway". 1918c912c 2023-02 “n64: increase the HLE steps as it shouldn't matter much anyway”

### ares-087 — Tiered synchronization budgets between cooperating threads
Lens: concurrency · Source: ares
Principle: Synchronizing every participant on every step is correct but slow. Give each pair of participants its own minimum interval: short for tightly coupled peers, longer for full-system sync. Then tune those intervals against real workloads. Too long breaks programs and too short gives up the speed, so expect to adjust them back and forth.
Why it is faster: —
Applies elsewhere: lock-step simulations with different tick rates per subsystem, distributed caches with per-tier consistency windows, game netcode that sends physics state more often than chat or inventory.
Watch out: —
Seen in ares: The 32X SH-2s had a single `minCyclesBetweenSyncs`. It was split into `recompilerStepCycles` (20), `minCyclesBetweenFullSyncs` (200 at first, then 100, then 50 to fix a Knuckles Chaotix hang) and `minCyclesBetweenSh2Syncs`, each with its own countdown in `M32X::SH7604::step`. For plain Mega Drive and Mega CD, the 68K was set back to sync every cycle (`minCyclesBetweenSyncs = 0`). An earlier change that skipped VDP-to-CPU sync on 32X was reverted because it broke games. 0661e9858 2023-10 “md: 32x performance tuning” · 8ff104393 2024-04 “md: lower full sync interval between MD and 32X” · fa9d350a6 2024-08 “md: sync more regularly when emulating 32X” · f0449e36f 2023-09 “md: sync every cycle when 32x is not in use” · b6350b3c0 2023-03 “md: (performance) reduce vdp sync when emulating 32x” · 462afe89f 2023-03 “md: undo reduce vdp sync when emulating 32x”

### ares-088 — Synchronize at communication points, not every tick
Lens: concurrency · Source: ares
Principle: Two concurrently simulated agents only need to agree on time when one of them observes the other. Let them run apart, sync on each access to shared I/O, and add a coarse periodic sync as a backstop. You keep accuracy where it can be observed and cut the switching cost everywhere else.
Why it is faster: —
Applies elsewhere: optimistic concurrency that validates only at commit, actors that exchange messages instead of sharing clocks, lazy flushing of thread-local buffers when another thread reads them.
Watch out: —
Seen in ares: GBA: `CPU::step` used to call `Thread::synchronize()` against everything, or batch 16 cycles under `PROFILE_PERFORMANCE`. It now syncs only with `ppu, player`, adds `synchronize(apu)` every 1024 cycles, and calls `cpu.synchronize(apu)` at the top of `APU::readIO/writeIO`. Because this was as fast as the old performance profile, both profiles were removed. 32X: reads of the hcount and communication registers now call `syncOtherSh2()`. e1e1bedb2 2024-10 “gba: selectively synchronize CPU and APU, and add option to disable pixel accuracy (#1679)” · 0661e9858 2023-10 “md: 32x performance tuning”

### ares-089 — Cut a batch short when something needs attention
Lens: concurrency · Source: ares
Principle: If a worker runs a batch of N units between checkpoints, give it a way to zero its remaining budget when an event comes up that the batch boundary would otherwise delay, such as a deferred interrupt or a write to shared memory. Large batches stay the common case without adding unbounded latency.
Why it is faster: —
Applies elsewhere: cooperative schedulers whose yield check is triggered by a "need resched" flag, game loops that break out of a fixed-step catch-up when input arrives, Nagle-style batching that flushes on a priority message.
Watch out: —
Seen in ares: The SH-2 recompiler loop runs `do { block->execute(); } while(CCR < cyclesUntilRecompilerExit);`. Deferring an interrupt in a delay slot (`exceptionHandler`, `INTC::run`, the 32X `raise` macro) and writes to the uncached area now set the budget to 0, and it is restored to `recompilerStepCycles` after the loop. Without this, an interrupt could be deferred indefinitely if every exit happened inside a delay slot. 7c74821b7 2023-04 “sh2: boost sync on pending interrupt” · 0661e9858 2023-10 “md: 32x performance tuning”

### ares-090 — Offer an opt-in coarse mode that batches fine-grained steps
Lens: other · Source: ares
Principle: When precision costs a lot and only some users need it, make it a switch. In the coarse mode, merge many tiny time advances into one, and keep the same output computation.
Why it is faster: —
Applies elsewhere: "low power" modes that drop the animation frame rate, approximate query modes in analytics databases, reduced-precision physics substeps on weak devices.
Watch out: —
Seen in ares: The GBA PPU gained `setAccurate()`. In the per-pixel loop, `if(accurate) step(4);` runs once per pixel, and otherwise a single `step(960)` runs after the line. The commit reports a 10–15% speedup with the option off. e1e1bedb2 2024-10 “gba: selectively synchronize CPU and APU, and add option to disable pixel accuracy (#1679)”

### ares-091 — Move per-step invariant maintenance to the writers
Lens: jit-interpreters · Source: ares
Principle: Re-establishing an invariant after every step, such as "this slot is always zero", costs something on every step. Enforce it only where it can break: skip operations whose only effect is to break it, and repair it immediately after the few complex operations that might. Keep an assertion in debug builds.
Why it is faster: —
Applies elsewhere: maintaining a sorted-list invariant on insert instead of re-sorting each read, sanitizing strings at ingestion instead of at every render, write barriers instead of full heap scans.
Watch out: —
Seen in ares: The N64 CPU and RSP epilogues did `ipu.r[0] = 0` after every instruction. Now only the interpreter does that. The recompiler returns early for simple ops whose destination is r0 (`ADDIU`, `ORI`, `LUI` and others: `if(Rtn == 0) return 0;`), and emits `emitZeroClear(Rtn)` after calls to helpers such as `ADDI` or `LDL` that must still raise exceptions. For the RSP, the decoder's `r.def` bit 0 decides when to emit `mov32(mem(R0), imm(0))`. 401aab6d6 2024-07 “n64: clear r0 only when it was written to (#1580)” · 5453fe084 2024-08 “n64: (rsp/recompiler) clear r0 only when necessary (#1581)”

### ares-092 — Replace per-instruction helper calls with inline generated code
Lens: jit-interpreters · Source: ares
Principle: A call out of generated code into a general-purpose helper costs a call, an argument setup and a switch, all repeated per instruction. Turn the helper's state machine into plain state variables that the generated code updates inline, and handle rare cases, like an incoherent cache, by declining to compile and falling back to the interpreter. The emitted code gets larger, but the hot loop no longer leaves the JIT.
Why it is faster: —
Applies elsewhere: inlining bounds or tag checks into bytecode JIT output, replacing virtual "onTick" hooks with data-driven flags, compiling regexes to code instead of interpreting state tables.
Watch out: —
Seen in ares: `CPU::instructionEpilogue` held a 7-case `branch.state` switch. This was replaced by a `Pipeline{pc,nextpc,state,nstate}` that the recompiler updates with inline `mov`/`add`/`test` and exits on `jumpEpilog(flag_nz)`. The JIT was then restricted to code running from the icache, and the epilogue call was removed: the icache is simulated by a `jitFetch` call once per 32-byte line, plus an inline `step(2)`. Blocks whose icache is not coherent return `nullptr` and run interpreted. 12f0d0470 2024-08 “n64: remove branch state machine from epilogue (#1598)” · f43da416f 2024-09 “n64: refactor devirtualize and memory ops, force JIT only with instruction cache (#1635)” · 863593ed7 2024-10 “n64: remove call to epilogue function from the jit (#1663)”

### ares-094 — Batch ticks with compile-time counts and unswitch loops
Lens: compiler-build · Source: ares
Principle: If an inner routine runs N times in a row with the same enable flags, pass N and the per-instance constants as template arguments. Test the flags once outside the batch and loop inside the callee. The compiler can then unroll the loop and drop impossible cases, such as features a given instance never has.
Why it is faster: —
Applies elsewhere: audio DSP processing whole blocks instead of single samples, per-channel specialized mixers, image filters templated on the channel count.
Watch out: —
Seen in ares: The PC Engine PSG called `frame()` 64 times, and each call ran `channel[C].run()`, which checked `io.enable` and `noiseEnable` every tick. It now calls `frame<64>()`, tests `io.enable` once per channel, and calls `run<index, step>()`. That loops `step` times and evaluates the noise path only when `index >= 4`. The `io.output` field was removed and samples are returned directly. 6cc9f5a41 2025-01 “pce: reduce per-iteration conditional statements in psg”

### ares-096 — Compute small calendar facts arithmetically
Lens: algorithms · Source: ares
Principle: Don't build and adjust a lookup array on every call. Derive the value with a short formula. This removes the per-call table construction and the duplicated logic between callers.
Why it is faster: —
Applies elsewhere: RTC drivers in firmware, date validation in form libraries, log-rotation date math.
Watch out: —
Seen in ares: The 64DD RTC's `tickSecond()` and `valid()` each built a 12-entry `daysInMonth` array and patched February for leap years. Both now call `daysInMonth(month, year)`, which computes `30 + ((month + (month >> 3)) & 1)` and corrects February. The per-byte BCD checks were also reduced. 339278335 2023-11 “n64: optimize DD RTC checks (#1300)”

### ares-097 — Store a counter in the unit its hottest consumer uses
Lens: other · Source: ares
Principle: If a counter is converted on every read in a hot loop, for example multiplied by a clock ratio, store it already scaled. The conversion moves to the cold sites that add to it, where it is a compile-time constant.
Why it is faster: —
Applies elsewhere: keeping timestamps in the units the scheduler compares, storing money in cents, keeping a pre-scaled fixed-point value for the renderer.
Watch out: —
Seen in ares: `CPU::synchronize()` computed `clocks = Thread::clock * 2` on every sync. The CPU now counts system-rate clocks directly, and every `step(n)` site became `step(n * 2)` (icache, dcache, multiply and divide timings), so the multiply in the sync path is gone. ffcce5823 2023-08 “n64: track cpu clocks at system rate”

### ares-099 — Reserve only what was used, and zero on demand
Lens: allocation · Source: ares
Principle: If a producer asks for a worst-case buffer but writes less, give the unused tail back to the bump allocator so allocations stay dense. Don't zero the whole arena on every flush. Zero each small structure when it is created.
Why it is faster: —
Applies elsewhere: arena allocators that shrink the last allocation, `calloc`-free pools that initialize objects on acquire, growable buffers that trim after serialization.
Watch out: —
Seen in ares: ljit asks for one oversized executable allocation per block. The allocator now reserves only the bytes actually emitted, which removes dead space between blocks. Code cache flushes no longer `release(bump_allocator::zero_fill)` the whole 64 MiB. Instead, the N64, PS1 and SH-2 `pool()` functions do `*pool = {}` on allocation. 8ed4af8b5 2023-09 “recompiler: avoid overallocation in code buffer (#1247)”

### ares-100 — Reference-count nested protection toggles
Lens: platform · Source: ares
Principle: When code paths that each flip a global mode, such as a page's write/execute permission, can nest, keep a thread-local depth counter. Make the system call only on the outermost transition. This avoids redundant mode switches and stops an inner "restore" from pulling permission away from an outer caller.
Why it is faster: —
Applies elsewhere: nested `disable_irq`/`enable_irq` counters in kernels, reentrant locks, nested transaction savepoints.
Watch out: —
Seen in ares: On Apple silicon, `memory::jitprotect` called `pthread_jit_write_protect_np` on every call. It now keeps a `static thread_local s32 depth` and switches only on 0→1 for writable and 1→0 for executable, with a debug-build throw on unmatched calls. 9a8e7f8b6 2023-11 “recompiler: allow jitprotect() calls to be nested (#1306)”

### ares-101 — Generate instrumentation only while it is switched on
Lens: jit-interpreters · Source: ares
Principle: Diagnostic hooks in generated code cost something on every execution. Emit them only when the feature is enabled, and when it is toggled, flush the code cache so the code is regenerated with or without the hooks. Normal runs pay nothing.
Why it is faster: —
Applies elsewhere: dynamic tracing (eBPF or DTrace probes patched in on demand), feature-flagged logging compiled out of hot paths, re-JITting with profiling counters only during warm-up.
Watch out: —
Seen in ares: The SH-2, N64 CPU/RSP and PS1 recompilers gained `callInstructionPrologue`, which emits a `call(&...::instructionPrologue)` per instruction only when the tracer is active. `Tracer::setToggle` registers a callback that flushes the recompiler when tracing is turned on or off. cd11421d7 2023-11 “recompiler: enable instruction tracing w/recompilers (#1301)”

### ares-104 — Sleep for the computed drain time instead of spinning
Lens: io · Source: ares
Principle: When a producer must wait for a consumer to drain a buffer, work out how long the excess will take to drain and sleep that long. Don't spin on a status query. On Windows, raise the timer resolution while you do this so short sleeps are accurate.
Why it is faster: —
Applies elsewhere: rate limiters that sleep until the next token, streaming uploaders that wait on socket buffers, game loops that sleep until the next frame deadline.
Watch out: —
Seen in ares: The SDL audio driver spun on `SDL_GetQueuedAudioSize > _bufferSize`. It now converts the excess bytes into seconds using `bitsPerSample` and `frequency` and calls `usleep`. The OpenAL driver sleeps 1 ms per loop. Both call `timeBeginPeriod(1)` on Windows. On an M1 Max running the SNES core, the commit reports CPU use dropping from about 100% to 40–50%. a6d380fb5 2024-04 “ruby: Remove tight loops in SDL and OpenAL (#1458)”

### ares-105 — Bound producer queues to the latency target
Lens: io · Source: ares
Principle: A ring buffer with no fill limit lets a fast producer run far ahead. After the producer slows down, the backlog shows up as lag. Drop or overwrite once the queue reaches the latency budget so the depth stays bounded.
Why it is faster: —
Applies elsewhere: bounded channels in Go or Rust, video players that drop late frames, log shippers with fixed-size buffers.
Watch out: —
Seen in ares: The WASAPI driver could overfill its circular queue during fast-forward, which caused long pauses afterwards. It now discards the oldest sample (`queue.read++; queue.count--`) when the device event isn't ready, and asks for at least two device periods of latency. 92c1c2ec4 2023-11 “ruby: queue no more than "latency" samples in wasapi driver (#1292)”

### ares-106 — Load in the background and block only on unloaded bytes
Lens: io · Source: ares
Principle: Start a large sequential load on a worker thread and return at once. Each read waits only until the loader has passed the requested offset. The program starts responding immediately, and the worst case is no slower than a blocking load.
Why it is faster: —
Applies elsewhere: progressive image or video loading, memory-mapped files paged in on demand, streaming large JSON with readers that wait on a watermark.
Watch out: —
Seen in ares: `nall::vfs::cdrom` now decodes CUE/CHD images on `_thread`. `read()`, `write()` and `data()` call `wait(offset)`, which spins until `_loadOffset` passes the offset, while subchannel data is preloaded. `CHD::read` became `const`, with a `mutable` hunk buffer, so the loader can share it. b29fb9177 2023-08 “nall: move cdrom loading to separate thread”

### ares-107 — Use a single-instruction rounding intrinsic, guarded by feature macros
Lens: simd · Source: ares
Principle: Rounding a float in a given mode can be one SSE4.1 `roundss`/`roundsd` or one NEON `vrndn` instead of a library call or a mode switch. Guard each intrinsic with the matching ISA macro and keep a portable fallback, so builds for older baselines still work.
Why it is faster: —
Applies elsewhere: image and audio converters, numeric parsers, game math libraries with per-ISA fast paths.
Watch out: —
Seen in ares: The N64 `CPU::roundNearest/Ceil/Floor/Trunc/Current` use `_mm_round_ss/sd` with `_MM_FROUND_*` on x86 and `vrndns_f32` on ARM64. Their `#if` now also requires `ARCHITECTURE_SUPPORTS_SSE4_1`. 191393445 2023-01 “build: guard usage of sse4.1 round instructions”

### ares-108 — Raise the ISA baseline and check it before startup
Lens: compiler-build · Source: ares
Principle: Targeting a newer, widely available instruction-set level lets the compiler use better instructions everywhere. To fail safely on older machines, run a CPUID check before any static initializer can execute a new instruction, and exit with a clear message.
Why it is faster: —
Applies elsewhere: distributions moving to x86-64-v2/v3, apps shipping AVX2 builds with a launcher check, `-mcpu` baselines for Android ABIs.
Watch out: —
Seen in ares: Official amd64 builds use `-march=x86-64-v2` (SSE4.2), and local builds keep `-march=native`. On Windows, `preCppInitializer` is placed in `.CRT$XCT` so it runs before C++ global constructors. It calls `__cpuid(…,1)` and calls `FatalAppExitA` if SSE4.2 is missing. A later `nall::instruction_set` adds general CPUID feature queries. 9b521dd87 2023-02 “desktop-ui: optimize for x86-64-v2 in non-local builds” · e4fee670a 2023-02 “desktop-ui: check for SSE4.2 support in non-local builds” · 10b747c30 2025-01 “nall: add instruction_set to query cpu features (#1751)”

### ares-109 — Unity builds for whole-module optimization
Lens: compiler-build · Source: ares
Principle: Compiling each module as a single translation unit, one file that includes all its sources, lets the compiler inline and optimize across file boundaries without depending on LTO. It also parses shared headers once. Cold builds get faster and the binary gets slightly faster, at the cost of slower incremental rebuilds, so make it an option.
Why it is faster: —
Applies elsewhere: SQLite's amalgamation, CMake `UNITY_BUILD` in game engines, bundling JS modules so the minifier can inline across them.
Watch out: —
Seen in ares: The `ARES_UNITY_CORES` option adds a `<core>.cpp` per core (for example `a26.cpp` including `cpu.cpp`, `tia.cpp` and the rest), and `#pragma once` deduplicates the headers. The commit reports CI builds going from 9m16s to 6m47s, and speed gains of Knuckles Chaotix 101→108 VPS and Snowboard Kids 66→69 VPS. 2ed2ed34b 2025-01 “build: Add option to build cores as single compilation units (#1746)”

### ares-110 — ThinLTO with an incremental cache
Lens: compiler-build · Source: ares
Principle: Full LTO re-optimizes the whole program on every link. ThinLTO does summary-based cross-module optimization in parallel and can cache per-module results, so incremental links take seconds or less while keeping most of the cross-module inlining.
Why it is faster: —
Applies elsewhere: Chromium and Android native builds, Rust's `lto = "thin"`, large C++ services that need fast relinks.
Watch out: —
Seen in ares: The clang build switched from `-flto` to `-flto=thin`, with `--thinlto-cache-dir` or `-cache_path_lto` (`-lldltocache` under clang-cl), and lld became the default linker off macOS. The commit says incremental LTO links dropped "from tens of seconds to less than a second". libco's `co_swap_function` needed external linkage to work under ThinLTO. 65a3d38a2 2023-03 “build: enable clang ThinLTO” · 4b2b00a83 2023-03 “nall: use llvm's lld linker with clang”

### ares-111 — Keep frequently changing constants out of widely included headers
Lens: compiler-build · Source: ares
Principle: A value that changes every release, if it is defined in a header everything includes, forces a full rebuild each time it changes. Declare it `extern` in the header and define it in one source file, so only that file recompiles.
Why it is faster: —
Applies elsewhere: build stamps and git hashes in generated `.c` files, feature-flag tables loaded at runtime, version strings injected at link time.
Watch out: —
Seen in ares: `ares::Version`, `Name`, `SerializerSignature` and similar were `static const` in `ares/ares.hpp`. They are now `extern const`, defined in `ares/ares.cpp`. b7be6612b 2023-10 “ares: prevent rebuilding the entire project if only version constants change”

### ares-112 — Don't emit duplicated output that the presenter can scale
Lens: gpu-rendering · Source: ares
Principle: Writing every line twice into a framebuffer doubles the memory traffic and the work for every later pass. Output at native resolution and let the presentation stage scale it.
Why it is faster: —
Applies elsewhere: rendering UI at logical resolution and letting the compositor upscale, sending lower-resolution video with a scale hint, storing thumbnails instead of upscaled copies.
Watch out: —
Seen in ares: The Mega Drive VDP's `pixels()` and `frame()` used to write rows at `y * 2 * 1415` and set the screen to `visibleHeight() * 2` even for progressive content. They now use `yScale = latch.interlace ? 2 : 1`, with `setScale(0.25, 1.0 / yScale)`. 2d714d1a9 2025-01 “md: don't line-double non-interlaced content”

### ares-114 — Accumulate cycle counts and flush only where they can be observed
Lens: laziness · Source: ares
Principle: Timing only has to be correct at points where something else can see it. Sum the fixed costs of straight-line work at compile time and write the total once, just before the next call, exit or slow path. That removes a memory update, and often a call, from every instruction.
Why it is faster: —
Applies elsewhere: per-thread counters that are merged into a global counter in batches; buffered metrics emitters; deferred reference-count updates.
Watch out: —
Seen in ares: The RSP JIT keeps a compile-time `deferredClocks` and calls `emitClockFlush` only before opcodes for which `instructionMayCallf()` is true, on halt exits and at block end. A later commit drops LH/LW/SH/SW and VU ops from that list, and slow paths now flush the portion they need through `slowPathFlushedClocks`. The CPU JIT got the same rule: `if(hasBranched || info.branch() || info.jitMayCallf()) flushDeferred();`. 1f06bbc15 2026-04 “n64: batch fixed RSP JIT clocks between call sites” · 44ab78ade 2026-04 “n64: defer RSP JIT clock flushes to slow paths” · 9e20c21b3 2026-04 “n64: avoid VU preflush in RSP callf gate” · 4ed8a7792 2026-04 “n64: flush deferred CPU cycles only on branch/helper opcodes”

### ares-128 — Pack the valid flag into the tag word
Lens: memory-layout · Source: ares
Principle: Store a cache line's valid bit inside its tag word, so a hit test is a single compare against `tag | 1`. That replaces two loads and two compares (valid, then tag). It also makes the test simple enough to emit inline in generated code.
Why it is faster: —
Applies elsewhere: tagged pointers that use the low bits as flags; a hash-table slot whose key 0 means empty.
Watch out: —
Seen in ares: Both the emulated icache and dcache lines replaced `bool valid; u32 tag;` with `u32 tagKey` (bit 0 = valid). `hit()` becomes `valid() && (tagKey & ~1u) == t`, and the JIT compares `tagKey` with `(paddr & ~0xfff) | 1` in one instruction. f241c4095 2026-04 “n64: optimize dcache to use a single word with both tag and valid bit”

### ares-129 — Emulate cache lookups inline in generated code
Lens: jit-interpreters · Source: ares
Principle: An emulated cache hit is the common case, so test it inline with a computed line address and one compare, and call into the emulator only on a miss. A memory access then costs a few host instructions in place of a function call.
Why it is faster: —
Applies elsewhere: software TLB fast paths in QEMU; inline lookup of a per-thread allocation cache.
Watch out: —
Seen in ares: Instruction fetch used to call `CPU::jitFetch`. It is now `cmp32(IcacheTagKeyMem(lineIndex), imm(expectedTagKey))` with a deferred `jitIcacheFillMiss`. Aligned loads compute the dcache line pointer from `(addr>>4)&0x1ff`, compare `tagKey`, add 2 cycles and read the word directly. A miss falls back to the interpreter. b698206d0 2026-04 “n64: in CPU JIT, inline fastpath of icache hit check” · 9e6be60b9 2026-04 “n64: implement all aligned load opcodes in JIT”

### ares-130 — Speculate on conventions for stack and global pointers
Lens: jit-interpreters · Source: ares
Principle: Code compiled for a given platform almost always keeps its stack and global pointers aligned and inside RAM. Record those facts in the specialization key, and memory ops based on these registers can skip their range and alignment checks. Writes that would break an assumption end the block or update the key.
Why it is faster: —
Applies elsewhere: JITs that specialize on likely-invariant facts with deopt guards; compilers that assume stack alignment from the ABI.
Watch out: —
Seen in ares: The state key gained `gpCachedRdram`, `gpCachedRdramOff16`, `gp/spAligned4/8`. Loads with `Rsn==28/29` skip the address mismatch test (`rangeKnown`) and the alignment test (`alignmentKnown`). `addiu sp,sp,imm` updates the key via `updateStackPointerStateKey`. Decoder flags mark other writes to gp/sp as `JitStateKeyMayChange`. 5b8abb571 2026-04 “n64: in JIT, add optimizations for accesses via $sp/$gp”

### ares-131 — Store emulated flags in the host's bit layout
Lens: platform · Source: ares
Principle: Lay out the emulated status bits to match the host register's bit positions. Host results can then be masked and ORed straight into the emulated state, with no per-bit translation. Only the cases that really need special handling fall back to slow code.
Why it is faster: —
Applies elsewhere: storing data in network byte order so it can be sent without conversion; copying CPU feature masks straight from `cpuid`.
Watch out: —
Seen in ares: The FPU's flag/enable/cause records used to be structs of `n1` fields. They are now `n8 data` words whose bit indices follow MXCSR on x86-64 and FPSR on ARM64 (selected by `#if ARCHITECTURE_ARM64`). After a host FP op the JIT tests `fallbackMask` (enabled traps, denormal, and flags that can't be passed through), then executes `or8(cause, ..., hostflags & stickyMask)`. 86fa83fc2 2026-04 “n64: in FPU JIT, implement passthrough of flag bits from host processor”

### ares-133 — Make debugging hooks cost nothing where no breakpoint is set
Lens: jit-interpreters · Source: ares
Principle: Having a debugger attached should not slow down code that has no breakpoint. End compiled blocks only at the addresses that actually have breakpoints. Single-instruction compilation is then needed only where it matters, not everywhere.
Why it is faster: —
Applies elsewhere: patching in breakpoint instructions instead of single-stepping; tracepoints that are compiled out until enabled.
Watch out: —
Seen in ares: When any breakpoint existed, `block()` used to compile single-instruction blocks. Now `buildEmitPlan` stops at `GDB::server.hasBreakpointAt(pc)`. Watchpoints set a state-key bit, so only blocks that need the slow memory path get it. dd834d2c5 2026-05 “n64: revisit integration between JIT and gdb to improve performance”

### ares-134 — Resolve argument moves as a parallel copy
Lens: compiler-build · Source: ares
Principle: When a generated call passes values that are already in registers, the moves into ABI argument registers can overwrite each other. Ordering them as a parallel move, and breaking cycles with one temporary, makes it safe to pass registers directly. Spilling through memory is no longer needed.
Why it is faster: —
Applies elsewhere: SSA phi elimination ("swap problem"); register allocators' parallel-move sequentialization.
Watch out: —
Seen in ares: `callf` in `nall/recompiler/generic/encoder-calls.hpp` accepts `reg` arguments, resolves move dependencies and breaks cycles with a scratch register. RSP vector fast paths use it to pass computed DMEM pointers and sizes in registers. 4c4c5460b 2026-04 “nall: add support for register arguments to callf”

### ares-136 — Turn per-access timing calls into a counter add
Lens: laziness · Source: ares
Principle: Inside compiled code, a full timing step on every bus access can trigger synchronization in the middle of a block. Accumulate the cycles in a plain counter, and apply them all at once when the block ends.
Why it is faster: —
Applies elsewhere: batched accounting in rate limiters; coalescing writes to a shared counter.
Watch out: —
Seen in ares: 32X bus and I/O paths now call `internalStep(n)`. When the SH-2 recompiler is enabled this is `regs.CCR += clocks`; otherwise it calls `step(clocks)` as before. 048d0e331 2025-02 “md: add internalStep function for 32x recompiler”

### ares-138 — Overlap work and wait only when its result is consumed
Lens: concurrency · Source: ares
Principle: Hand the expensive operation to a worker thread and keep going. Block on its completion only at the point where the result is needed. A pair of monotonic sequence counters (submitted and completed) makes "wait until done" a simple spin on one comparison.
Why it is faster: —
Applies elsewhere: futures resolved lazily; GPU fences checked at present time; async I/O with completion counters.
Watch out: —
Seen in ares: Textured PS1 draws used to call `renderer.synchronize()` right after queueing so their texture-cache cost could be measured. Now the GPU records `textureCyclesAtQueue` and calls `finishPendingRender()` only when `io.pcounter` reaches 0. `synchronize()` spins on `completedSequence != workSequence`, which replaced a sentinel command pushed through the queue. 5d093ad9e 2026-10 “ps1: run textured draws in parallel to the cpu”

### ares-139 — Prefetch the predicted next item on a background thread
Lens: concurrency · Source: ares
Principle: If the next input can usually be predicted, decode it on a worker thread while the current one is used. When the prediction is right, swap buffers in constant time; when it is wrong, decode synchronously as before.
Why it is faster: —
Applies elsewhere: video players decoding ahead; read-ahead in file systems; speculative page loading in browsers.
Watch out: —
Seen in ares: The LaserActive video path predicts the next frame from playback mode and speed, then signals a detached worker through `std::atomic_flag` wait/notify. If `videoFramePrefetchTarget` matches the needed frame, the main thread runs `buildFrameBuffer.swap(videoFramePrefetchBuffer)`; otherwise it decodes the QOI2 frame directly. 5cc3a8118 2025-09 “ld: Added frame prefetch and video resampling coefficient precalculation”

### ares-140 — Precompute resampling weights once
Lens: lookup-tables · Source: ares
Principle: Resampling weights depend only on the source and destination sizes, not on pixel data. Compute them once per output column, and the per-frame inner loop shrinks to a multiply-accumulate with no float-to-int conversions or edge tests.
Why it is faster: —
Applies elsewhere: polyphase filter banks in audio resamplers; image scalers such as libswscale caching their filter coefficients.
Watch out: —
Seen in ares: `lineResamplingData` stores, per output pixel, `firstSamplePosX`, `lastSamplePosX`, a `sampleWeightX` vector and a `conversionFactor`, all filled on first use. The scanline loop then accumulates `pixel * weight` for all three channels in one pass, where it used to loop per plane. 5cc3a8118 2025-09 “ld: Added frame prefetch and video resampling coefficient precalculation”

### ares-141 — Use fixed-point instead of float for normalized blending
Lens: cpu-microopt · Source: ares
Principle: Blending 8-bit channels with float normalization means int-to-float conversion, divides and float-to-int conversion for every pixel. In 16.16 fixed-point, normalization is shifts and ORs and a multiply is one 64-bit integer multiply and shift. The commit reports this was 3x faster.
Why it is faster: —
Applies elsewhere: alpha compositing in software rasterizers; DSP on chips without an FPU; Q-format audio mixing.
Watch out: —
Seen in ares: MegaLD composition now uses `mul1616FixedPoint = (u64)a*b >> 16`. An 8-bit value is expanded to 16.16 by bit replication, `(v<<8)|v`, and a result is converted back with a rounding shift. 77c0718df 2025-08 “ld: Convert video composition to use 16.16 fixed point for a 3x speedup”

### ares-142 — Reject invisible items before doing per-item work
Lens: algorithms · Source: ares
Principle: A cheap bounds test at the start of the pipeline avoids fetching and rendering items that cannot contribute any output.
Why it is faster: —
Applies elsewhere: frustum culling; skipping off-screen DOM elements in virtual lists.
Watch out: —
Seen in ares: In the GBA object fetch, a sprite whose start and end x (`n9 pxMax = x + (width << affineSize)`) are both ≥ 240 now goes straight to `goToNext()`. 4c39ca23e 2026-05 “gba: skip rendering sprites that are offscreen horizontally”

### ares-143 — Initialize state directly instead of simulating up to it
Lens: algorithms · Source: ares
Principle: If a state machine can be set directly to its target state, don't step it there one increment at a time.
Why it is faster: —
Applies elsewhere: seeking a PRNG with jump-ahead; computing calendar dates arithmetically rather than looping over days.
Watch out: —
Seen in ares: On first boot the N64 cartridge RTC was initialized to the epoch and then `advance()`d by about 1.7 billion seconds. It now encodes `localtime(time(0))` straight into RAM and advances only when restoring a saved timestamp. e6130f66e 2026-08 “n64: speed up first boot of games with RTC”

### ares-145 — Sort once at construction rather than at every use
Lens: algorithms · Source: ares
Principle: If a collection does not change after it is built, sort it once when it is created. Every later reader then sees sorted data without repeating the sort.
Why it is faster: —
Applies elsewhere: materialized sorted indexes; sorting configuration lists at load time.
Watch out: —
Seen in ares: `Emulator::construct()` ends with `std::sort(emulators..., emuComparer)`. The sorts in the emulator and firmware settings panels were removed. 88bd79842 2026-01 “Sort once when vector is created”

### ares-147 — Align context-save buffers for the widest stores used
Lens: memory-layout · Source: ares
Principle: Context-switch code saves registers with aligned vector moves (`movaps`), and paired stores (`ldp/stp`), which can perform better on 16-byte-aligned memory. Declaring the save buffer `alignas(16)` guarantees that alignment.
Why it is faster: —
Applies elsewhere: `alignas(64)` for SIMD buffers and to avoid false sharing; aligned allocators for AVX data.
Watch out: —
Seen in ares: The libco `co_active_buffer[64]` is now `static thread_local alignas(16)` on amd64 and aarch64. The commit notes that amd64 requires this because of `co_swap`'s aligned moves, and that paired loads and stores on aarch64 may perform better with it. 30a3b125c 2025-12 “libco: align co_active_buffer on amd64, aarch64”

### ares-148 — Keep the device buffer small and the queue deep
Lens: io · Source: ares
Principle: If the audio device buffer is sized for the full user latency, the API may not refill it in time when the producer is bursty. Fix the device buffer at a small period and apply the requested latency to the input queue instead, with a larger refill threshold. Underruns go away and the latency setting is unchanged.
Why it is faster: —
Applies elsewhere: jitter buffers in VoIP; double-buffered DMA with a deep software ring.
Watch out: —
Seen in ares: The SDL3 driver first quadrupled `bytesToWait`. It then switched to `desired_samples = _frequency / 100` (a 10 ms device buffer) with `_bufferSize = desired_samples * _bytesPerFrame * (_latency / 10)`. 8c41639a7 2026-06 “Increase bytes to wait in SDL audio to help performance” · 6dc3f3306 2026-06 “ruby: improve SDL3 audio performance”

### ares-149 — Express all clocks in one fixed-point timebase
Lens: algorithms · Source: ares
Principle: Give each component a 64-bit clock and a per-component scalar (`Second / frequency`). Advancing time is then a single multiply-add, and comparing components running at different frequencies is a plain integer compare with no division. Subtracting the global minimum from every clock at frame exit keeps them from overflowing.
Why it is faster: —
Applies elsewhere: discrete-event simulators with a common tick unit; media pipelines that use a 90 kHz timebase.
Watch out: —
Seen in ares: `Thread::step(clocks)` is `_clock += _scalar * clocks;`, with `Second = (u64)-1 >> 1`. `Scheduler::exit` subtracts `minimum()` from every thread. A per-thread `_uniqueID` offset breaks ties deterministically. current code: ares/ares/scheduler/thread.cpp, ares/ares/scheduler/scheduler.cpp

### ares-150 — Cooperative coroutines with a hand-written register swap
Lens: concurrency · Source: ares
Principle: Running each emulated chip as a coroutine on one OS thread turns a context switch into saving and restoring the callee-saved registers and jumping. There is no kernel transition and no locking, so components can interleave very finely.
Why it is faster: —
Applies elsewhere: green threads and fibers in game engines; Go-style user-mode schedulers.
Watch out: —
Seen in ares: `co_swap_function` is a byte array of machine code: `mov [rdx],rsp; mov rsp,[rcx]; pop rax`, followed by stores and loads of rbp/rbx/r12–r15 (plus xmm6–15 on Win64) and `jmp rax`. It is placed in `.text`, or in an mprotected page aligned to 4096 bytes. current code: libco/amd64.c

### ares-151 — Synchronize lazily: switch only when the other side is behind
Lens: concurrency · Source: ares
Principle: A component runs ahead freely and catches the others up only when it touches state they share, and only if they are actually behind. Most accesses then cost one compare and no switch.
Why it is faster: —
Applies elsewhere: optimistic concurrency; conservative parallel discrete-event simulation with lookahead.
Watch out: —
Seen in ares: `Thread::synchronize(Thread& t, ...)` runs `while(t.clock() < clock() && t.handle()) co_switch(t.handle());`, a variadic template that catches up several threads in one call. The PPU's `step()` calls `Thread::synchronize(cpu)` only. current code: ares/ares/scheduler/thread.cpp

### ares-152 — Fixed-capacity binary heap with wraparound-safe compares
Lens: algorithms · Source: ares
Principle: Keep timed events in a min-heap stored in a fixed-size array. Finding the next event is O(1) and inserting is O(log n), with no allocation. Comparing 32-bit timestamps with `x - y < 0x7fffffff` stays correct across counter wraparound, so the clock never has to be rebased.
Why it is faster: —
Applies elsewhere: kernel timer wheels and heaps; TCP sequence-number comparisons (RFC 1982 serial arithmetic).
Watch out: —
Seen in ares: `priority_queue<T[Size]>` stores `{clock, event, valid}` entries. Removing a specific event just clears its `valid` flag instead of restructuring the heap. `timeToNextEvent()` gives the JIT its run budget. current code: nall/nall/priority-queue.hpp

### ares-154 — Round memories up to a power of two and pre-fill the mirrors
Lens: memory-layout · Source: ares
Principle: Allocate a power-of-two buffer and copy the mirrored contents into its padding once, at load time. Every access then becomes `data[addr & mask]`, with no modulo or range branch.
Why it is faster: —
Applies elsewhere: ring buffers with power-of-two capacity; hash tables that index by mask.
Watch out: —
Seen in ares: `Readable::allocate` sets `mask = bit::round(size) - 1`. `load()` fills `data[size..mask]` from `mirror(address, size)`, and `read()` is `self.data[address & self.mask]`. current code: ares/ares/memory/readable.hpp

### ares-156 — Build an exact three-operand saturating add from SIMD min/max
Lens: simd · Source: ares
Principle: A saturated `a + b + carry` cannot be done with two saturating adds in arbitrary order. Applying the carry to `min(a,b)` first and then adding `max(a,b)` gives the exact result. The whole operation runs as five 8-lane SSE instructions, with no widening.
Why it is faster: —
Applies elsewhere: fixed-point audio mixing; image arithmetic with clamping.
Watch out: —
Seen in ares: `VADD`: `sum=_mm_add_epi16(vs,vte); ACCL=_mm_sub_epi16(sum,VCOL); min=_mm_subs_epi16(_mm_min_epi16(vs,vte),VCOL); vd=_mm_adds_epi16(min,_mm_max_epi16(vs,vte));` current code: ares/n64/rsp/interpreter-vpu.cpp

### ares-158 — Look up compiled code by content hash, with a bitmap of dirty regions
Lens: jit-interpreters · Source: ares
Principle: For a small code memory that is overwritten constantly, key compiled blocks by a hash of their bytes plus context. A program that is uploaded again then reuses its existing code. A 64-bit dirty mask, one bit per 64-byte region, makes invalidating on a write a single OR, and the actual flushing is deferred to the next lookup.
Why it is faster: —
Applies elsewhere: content-addressed build caches; shader caches keyed by bytecode hash.
Watch out: —
Seen in ares: `block()` first checks a direct `context[2048]` array. On a miss it computes `XXH3_64bits` of the measured block, XORs in the pipeline hash and the address, and probes a `hashset<BlockHashPair>`. `invalidate()` does `dirty |= mask(address,size)`, with wraparound handled by combining two shifted masks. current code: ares/n64/rsp/recompiler.cpp, ares/n64/rsp/rsp.hpp

### ares-162 — Render whole scanlines, and skip rendering frames that won't be shown
Lens: laziness · Source: ares
Principle: A scanline-granular renderer draws each layer in one tight loop per line, which is much cheaper than drawing per dot. Frames computed only for run-ahead are never displayed, so their rendering is skipped and only the emulated state advances.
Why it is faster: —
Applies elsewhere: tile-based rendering; skipping draws on headless or offscreen game ticks.
Watch out: —
Seen in ares: `PPU::main` in the performance PPU runs `bg1..bg4.render(); obj.render(); dac.render();` once per visible line, guarded by `!runAhead()`. `Screen::frame()` and `refresh()` also return early during run-ahead. The PPU's .cpp files are `#include`d into one translation unit, so the compiler can inline across them. current code: ares/sfc/ppu-performance/ppu.cpp

### ares-163 — Batch GPU commands and wait only at explicit sync points
Lens: gpu-rendering · Source: ares
Principle: Collect emulated display-list words into a local buffer and enqueue whole commands to the GPU backend asynchronously. The CPU blocks on the GPU timeline only at the guest's own full-sync command. Scanout readback is also asynchronous, guarded by a fence.
Why it is faster: —
Applies elsewhere: command-buffer recording in Vulkan/Metal apps; batched draw submission in game engines.
Watch out: —
Seen in ares: `Vulkan::render()` copies words into `buffer[0x10000]`, keeps partial commands for the next call, and calls `processor->enqueue_command`. Only `Op::SyncFull` calls `wait_for_timeline(signal_timeline())`. `scanoutAsync` calls `scanout_async_buffer` and waits on `scanout.fence` only before the next one. current code: ares/n64/vulkan/vulkan.cpp
