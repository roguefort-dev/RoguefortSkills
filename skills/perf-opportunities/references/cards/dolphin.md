# Dolphin — technique cards (339)

Each card is exactly 7 lines. Fetch one with: grep -A 6 "^### dolphin-NNN " dolphin.md

### dolphin-001 — Patch direct jumps between compiled blocks
Lens: jit-interpreters · Source: Dolphin · Also: Flycast
Principle: When a translated code block ends, returning to a central dispatcher to look up the next block costs a hash or table lookup plus an indirect jump on every block exit. Once both the source and target blocks exist, overwrite the exit stub with a direct jump to the target. Hot loops then run block-to-block without leaving generated code.
Why it is faster: —
Applies elsewhere: threaded interpreters that patch call sites to point straight at the resolved handler, inline caches in JavaScript engines, PLT entries that rewrite themselves after the first lazy symbol resolution.
Watch out: —
Seen in Dolphin: `JitBlockCache::FinalizeBlock` records each block's exit addresses in a `links_to` multimap. `LinkBlockExits` emits `JMP(blocks[dest].checkedEntry)` over the exit stub. `LinkBlock` also re-links every earlier block that was waiting for the new block's address. The links are never undone. `DestroyBlock` instead rewrites the dead block's entry into `MOV PC, addr; JMP dispatcher`, so stale links fall back to the dispatcher safely. Linking first appeared as a partial option, and in 2009 it became the default; the commit message reports "perhaps as much as 20% more fps". ea934759e1 2008-07 “Also enabled partial block linking (see JitCache.cpp), should give a small speedup” · 2d5c0a509c 2009-02 “JIT: Made block linking the default option, it's faster and seems stable to me” • Also seen in Flycast (flycast-054): `RelinkBlock()` in `rec_arm64.cpp` emits `GenBranch(block->pBranchBlock->code)` once the target is known. Until then it emits `GenCallRuntime(ngen_LinkBlock_Generic_stub)`, and the stub (`mov x0, lr; sub x0, x0, #4; bl rdv_LinkBlock; br x0`) patches the caller. Conditional blocks get separate `cond_Branch` and `cond_Next` stubs. 75a04d6d2 2019-01 “arm64 dynarec: implement tail calling and block linking”

### dolphin-002 — Store the cache index in the source slot itself
Lens: lookup-tables · Source: Dolphin
Principle: If every lookup key is a location you control, write a tag plus the entry's index into that location when the entry is created. A lookup becomes one load and a tag test, with no hashing or tree walk. Invalidation works the same way: the moment anyone overwrites the slot, the tag is gone.
Why it is faster: —
Applies elsewhere: tagged pointers in a free list, object headers that hold a vtable or shape index, a "forwarding pointer" left in the old copy during GC compaction.
Watch out: —
Seen in Dolphin: On finalize, the JIT replaces the first guest instruction of a block with an unused opcode: `(JIT_OPCODE << 26) | block_num`. `GetBlockNumberFromStartAddress` reads one word and checks the top six bits. If they match, the low bits are the block number. The 2009 instruction-cache rework moved these stamped words into a separate `iCache` array (`*(u32*)(iCache + (addr & JIT_ICACHE_MASK))`) so guest RAM stays unmodified, and the lookup is still one read. 2d5c0a509c 2009-02 “JIT: Made block linking the default option” · b843378636 2009-10 “Lots of work on JIT (Implemented "unlimited instruction cache" CPU emulation…)”

### dolphin-003 — Track dirty registers and skip unchanged write-backs
Lens: jit-interpreters · Source: Dolphin
Principle: A register cache that mirrors memory only needs to store a value back if it changed since it was loaded. Keep a per-register dirty bit and skip the store when it is clear. Compare and load-heavy code often reads registers without writing them, so many flushes turn into no-ops.
Why it is faster: —
Applies elsewhere: write-back CPU caches, ORM change tracking that only UPDATEs modified columns, page-table dirty bits that let an OS skip writing clean pages to swap.
Watch out: —
Seen in Dolphin: In `JitRegCache.cpp` the flush path now computes `doStore = xregs[xr].dirty` and emits the `MOV` back to `ppcState` only when it is set. A fresh load sets `dirty = false`. A follow-up stopped `LoadToX64` from loading an immediate when the caller had said it would overwrite the register anyway (`doLoad` false). a6d5a7ea9e 2008-11 “Fix the dirty bit and make the register cache use it to be a bit more clever” · 6dd9fa75f8 2008-12 “Minor JIT optimization; doesn't trigger often, but the code looks really stupid when it does.”

### dolphin-004 — Fold constants at code-generation time
Lens: jit-interpreters · Source: Dolphin
Principle: When a code generator knows an operand's value at compile time, it can compute the result right away and record it as a constant. It emits no instructions for that operation, and later instructions that use the result can fold too.
Why it is faster: —
Applies elsewhere: template engines that pre-render static fragments, SQL planners that evaluate constant expressions once, shader compilers that specialize on uniform values.
Watch out: —
Seen in Dolphin: `Jit64::rlwinmx` used to special-case only a zero source. It now evaluates rotate-and-mask for any immediate source (`result = rotl(imm, SH) & Helper_Mask(MB, ME); gpr.SetImmediate32(a, result)`). In the IR JIT, `FoldICmpCRSigned`/`Unsigned` turn a compare of two constants into a constant CR value. `FoldUOp` rewrites a double-precision op whose inputs are both duplicated singles into the single-precision op. 590adfce53 2008-12 “Generalize constant folding for rlwinm.” · 94ee9afdb2 2009-02 “JitIL: A bit more constant folding, and a bit of cleanup.” · 9ed368b35a 2009-02 “JitIL: a couple small optimizations.”

### dolphin-005 — Fuse a compare with the branch that consumes it
Lens: jit-interpreters · Source: Dolphin
Principle: Materializing a comparison result into a flags variable, then testing it again in the next instruction, costs several stores, jumps and a reload. When the very next operation is a branch on that result, emit the native compare and conditional jump directly. Reordering independent instructions can move compares next to their branches, which creates more chances to fuse.
Why it is faster: —
Applies elsewhere: macro-op fusion in CPUs (cmp+jcc), bytecode superinstructions like `IF_ICMPLT`, query engines that fuse a filter into the scan loop instead of producing a boolean column.
Watch out: —
Seen in Dolphin: `Jit64::cmpXX` peeks at `js.next_inst`. If it is a `bc` that tests only the same CR field (no CTR decrement, no link), it emits `CMP` followed by the guest branch, and sets `js.cancel` so the branch is not compiled twice. A pass in `PPCAnalyst` (`CanSwapAdjacentOps`) then moves compares toward their branches. It skips instructions flagged `FL_EVIL`. The author reported a "nearly unnoticable" speedup on its own. 3de4df29a3 2008-12 “merge cmp/branch when possible.” · a4fc0da8cc 2008-12 “activate code reordering for better branch merging”

### dolphin-006 — Unpack bitfields into one byte per field
Lens: memory-layout · Source: Dolphin
Principle: When code mostly reads and writes individual 4-bit fields of a packed register, each update costs a load, AND-mask, shift, OR and store. Store each field in its own byte instead. Writes become a single byte store and tests a single byte test, and the rare whole-register access pays the pack/unpack cost.
Why it is faster: —
Applies elsewhere: struct-of-bools instead of a flags word in hot game-state code, a byte-per-pixel mask instead of a bitmask when per-pixel writes dominate.
Watch out: —
Seen in Dolphin: The 32-bit PowerPC CR became `ppcState.cr_fast[8]`. Compare results are now `MOV(8, M(&cr_fast[crf]), Imm8(0x8))` instead of an AND/OR on the full word, and `bcx` tests `TEST(8, M(&cr_fast[BI>>2]), Imm8(8>>(BI&3)))`. Only `mfcr`/`mtcrf` rebuild or split the word. The interpreter's CR logical ops changed the same way, to `SetCRBit(d, GetCRBit(a) & GetCRBit(b))` instead of repacking the whole CR. a44c421d01 2008-12 “Somewhat faster CR flag storage.” · c4f27aaae1 2008-12 “Avoid GetCR/SetCR in the cr bitwise ops”

### dolphin-007 — Lower to a compact IR, then drop dead values
Lens: jit-interpreters · Source: Dolphin
Principle: Translating straight from source instructions to machine code makes every side effect, such as a register write-back or a carry flag, real even when nothing reads it. Go through a simple SSA-like IR where guest-register loads and stores are separate operations. Then one backward liveness pass can delete redundant stores and any computation whose result is never used.
Why it is faster: —
Applies elsewhere: any compiler middle end, query optimizers that prune unused columns, build systems that skip targets no output depends on.
Watch out: —
Seen in Dolphin: `Jit64IL/IR.cpp` introduced a nanojit-style IR. Each instruction is one 32-bit word with 8-bit back-offsets to its operands. Redundant register loads and stores are removed and constants folded while the IR is built. A liveness pass marks dead instructions and last uses, and codegen skips any instruction with `!thisUsed`, so for example an unused carry from `srawx` disappears. A greedy register allocator follows. The author measured roughly 25% faster than the old JIT on an integer-bound test. 68c451f008 2008-12 “Some WIP work on the JIT… There's a big comment in Jit64IL/IR.cpp with a high-level overview”

### dolphin-008 — Instrument once, then recompile with what you observed
Lens: jit-interpreters · Source: Dolphin
Principle: A static compiler must emit checks for every case an instruction might hit, but at run time each site usually sees only one case. First compile a version that records what actually happens, such as which memory region each load touches. Then recompile with that fact built in, so the per-execution check and branch disappear.
Why it is faster: —
Applies elsewhere: profile-guided optimization, JVM tiered compilation with type profiles, monomorphic inline caches.
Watch out: —
Seen in Dolphin: The IR JIT's 32-bit "fast memory" compiles each block twice. The first build instruments its loads and stores and then jumps back into the JIT. The second build reads the recorded addresses and drops the mask-and-branch on each access. The author called this "the edge over the current JIT". In 2009 the option became `bJITProfiledReJIT`: `DoWriteCode(..., UseProfile, MakeProfile)` gained a separate flag, and the recompiled block is finalized and linked through `FinalizeBlock`. The commit marks it as unsafe, because it depends on a guarantee the PowerPC does not make. 68c451f008 2008-12 “Some WIP work on the JIT” · ef1ecbcd0f 2009-06 “Fixed profiled ReJIT. It's very experimental though. Makes a great performance boost, but is way too unsafe.”

### dolphin-009 — Let the MMU fault on the rare case, then patch that site
Lens: jit-interpreters · Source: Dolphin
Principle: Instead of testing every memory access for "is this special I/O?", reserve a large virtual region, map ordinary memory into it, and emit a plain load. The rare access that hits an unmapped page faults. The fault handler decodes the faulting instruction and rewrites it into a call to a slow-path trampoline. The common path stays one instruction, and each special site pays the cost once.
Why it is faster: —
Applies elsewhere: WebAssembly runtimes that use guard pages instead of bounds checks, copy-on-write via page protection, garbage collectors that use page faults as read or write barriers.
Watch out: —
Seen in Dolphin: `JitBackpatch.cpp` `BackPatch()` runs from the exception handler. It disassembles the faulting `MOV` (`DisassembleMov`), checks that the base register is RBX (the fastmem base), and emits a trampoline: `ABI_PushAllCallerSavedRegsAndAdjustStack`, call `Memory::Read_U32`, pop. It then overwrites the original `MOV`+`BSWAP` with `CALL trampoline` plus NOP padding. A code comment explains that the trampolines are heavy, but "only instructions that access I/O will get these". Later commits ported the fault handler and caller-saved push/pop to Linux. e3d21c0b11 2008-08 “32-bit speedup (videos mostly affected). Lots of various cleanup and future proofing.” · 7eefb7743b 2008-12 “Get the "MemTools" and backpatching code to build on linux.” · 0412d0811a 2008-12 “Fastmem for linux.”

### dolphin-010 — Natively compile whatever tops the profile
Lens: jit-interpreters · Source: Dolphin
Principle: A fast path with fallbacks is only as fast as its most common fallback. Profile, find which operations still drop to the slow generic handler, and give each one a native implementation, starting with the most frequent. Each one is small, but together they remove the expensive context switch to the slow path.
Why it is faster: —
Applies elsewhere: adding a dedicated SQL fast path for the query shape that dominates logs, writing a native method for the hottest interpreted builtin, vectorizing only the top kernel shown in a flame graph.
Watch out: —
Seen in Dolphin: The commit messages name the profile directly. `fabsx`/`fnegx`/`fnabsx` were each falling back to the interpreter. One `Jit64::fsign` now handles all three with a single `ANDPD`/`XORPD`/`ORPD` against a sign mask. `bclrx` got native code for conditional `blr`/`blrl`: a `TEST` on `cr_fast`, then `WriteExitDestInEAX`. `frsqrte` became `RSQRTSS` in the IR JIT, and `subfcx` and `lha` were added in 2008. 9ccf22093d 2009-07 “fabsx was showing up in TD's profiles, so jit it.” · a1904ba721 2009-07 “JIT: this should remove bclrx from TD's profiles...” · 1b936a4f71 2009-07 “JIT for frsqrte; … drops it off a profile I'm looking at.” · c305371725 2008-08 “More 32-bit speed”

### dolphin-011 — Peephole away moves and saves the code doesn't need
Lens: jit-interpreters · Source: Dolphin
Principle: Generic emitter helpers often produce a copy into a scratch register followed by a copy out, a zero-extend between registers of the same width, or a push/pop of a register that holds nothing live. Check for these cases while emitting. Each one saved is small, but it runs on every guest instruction of that kind.
Why it is faster: —
Applies elsewhere: SQL code generators that skip identity projections, serializers that skip a re-encode when source and destination formats match.
Watch out: —
Seen in Dolphin: `MOVZX`/`MOVSX` now degrade to a plain `MOV` when source and destination widths are equal. `UnsafeLoadRegToReg` now byteswaps and sign-extends in the destination register, not via `EAX`, and uses `SAR` instead of `SHR`+`MOVSX` for signed 16-bit loads. The IR `regWriteExit` skips `MOV EAX, x` when x is already in EAX. On x64 `regEmitMemLoad` no longer saves EAX, which the x64 allocator never assigns. c305371725 2008-08 “More 32-bit speed” · 9fe8f8b6dd 2008-11 “Obvious enhancements to load generation; should be a slight speedup.” · ef1ecbcd0f 2009-06 “Fixed profiled ReJIT.” · daf11ee708 2009-02 “remove unnecessary push/pop from 64-bit build (IR.cpp)”

### dolphin-012 — Make the code cache big enough to hold the working set
Lens: jit-interpreters · Source: Dolphin · Also: Flycast
Principle: When a translation cache fills, it is usually flushed wholesale and everything is recompiled. If the cache is smaller than the program's hot code, the program thrashes: compile, flush, compile again. Size the cache, both bytes and entries, generously so steady-state execution never triggers a flush.
Why it is faster: —
Applies elsewhere: raising a regex or prepared-statement cache limit, increasing the JVM `ReservedCodeCacheSize`, sizing a CDN cache above the daily working set.
Watch out: —
Seen in Dolphin: `JitCache.cpp`: `CODE_SIZE` went from 8 MB to 16 MB and `MAX_NUM_BLOCKS` from 65536 to 131072. 7dfbc574ae 2008-11 “Increase default JIT cache size to 16MB.” • Also seen in Flycast (flycast-058): `CODE_SIZE` in `core/hw/sh4/dyna/ngen.h` went from 6 MB to 10 MB. The commit message says frequent invalidation for lack of space “kills performance (Extreme Sports)”. 0df91770d 2018-07 “Increase dynarec code cache size to 10 MB”

### dolphin-013 — Recognize busy-wait loops and fast-forward time
Lens: algorithms · Source: Dolphin · Also: Flycast
Principle: A simulated program that spins waiting for an event burns host CPU while doing nothing. Detect the spin, either by matching the loop's instruction pattern or with a per-address hint, and jump the simulated clock straight to the next scheduled event. The emulated result is the same, and the host does no work.
Why it is faster: —
Applies elsewhere: discrete-event simulators that advance to the next event timestamp, test frameworks with fake timers that skip sleeps, VM hypervisors that trap the guest's HLT/PAUSE instead of letting it spin.
Watch out: —
Seen in Dolphin: The JIT's `lwz` matched the exact OS idle loop: `lwz rX, d(r13)`, `cmplwi rX,0`, `beq -8`. It now calls `PowerPC::OnIdle`, which calls `CoreTiming::Idle()` if the next-thread pointer is 0, and this was extended to dual-core mode. A per-game `[Speedhacks]` INI section (`0x80375c68=200`) makes the block at that address cost extra cycles, for games with unusual idle loops. For the DSP, `DSPAnalyzer` scans microcode for mailbox-poll signatures and marks them `CODE_IDLE_SKIP`. `gdsp_run_cycles` then returns its time slice early when it reaches one. e6a12b553b 2008-09 “Idle skipping in DC added (speedup).” · f87c709aa9 2008-11 “Added speed hack engine (way to make block take more cycles).” · 541832e4f0 2009-04 “Add preliminary DSPAnalyzer, which looks for idle skip opportunities.” · 8c3bb3796b 2009-04 “DSP: New non-threaded mode for the LLE plugin, with some idle skipping enabled.” • Also seen in Flycast (flycast-005): When `settings.dynarec.idleskip` is on, `dec_DecodeBlock` checks two things. A block whose hash is in a known `idle_hash` list gets `guest_cycles = max_cycles*100`. A short block (<6 ops) that has memory reads but no writes or FPU and branches backwards gets `guest_cycles*=3`, or `*=10` if it branches to itself. The Android frontend made idle skip the default. 90db7054a 2014-02 “idleskip should default to true”

### dolphin-014 — Run the work on the calling thread instead of handing it off
Lens: concurrency · Source: Dolphin
Principle: If thread A only starts thread B and then sleeps in a loop until B finishes, the second thread adds a context switch and sleep latency and runs nothing in parallel. Run B's function directly on A. The same applies to a coprocessor thread that has to stay in lockstep anyway: interleave it in slices on the main thread.
Why it is faster: —
Applies elsewhere: running a task inline when the thread pool would block on it immediately, single-threaded event-loop servers, coroutine schedulers that resume a task inline.
Watch out: —
Seen in Dolphin: `Core.cpp EmuThread`: in single-core mode it used to spawn `CpuThread` and loop in a "Idle thread" with sleeps. It now calls `CpuThread(pArg)` directly. The DSP LLE plugin gained a non-threaded mode in which the core calls `gdsp_run_cycles(cycles)` from its own timing loop instead of running a free-running DSP thread. ba90c68742 2008-09 “In single core mode Emulation and CPU threads are now merged. Performance gain due to sleep elimination.” · 8c3bb3796b 2009-04 “DSP: New non-threaded mode for the LLE plugin”

### dolphin-015 — Pick the wait primitive by expected wait length
Lens: concurrency · Source: Dolphin
Principle: Sleep(0) or an empty spin burns a core and pollutes profiles. Sleep(1) wakes up late, often 1–15 ms. Kernel events cost a syscall per signal. For long or unpredictable waits, block on an event. For handoffs that finish in microseconds, spin on an atomic flag with a CPU pause instruction.
Why it is faster: —
Applies elsewhere: adaptive mutexes that spin briefly before parking, `futex`-based locks, game loops that spin-wait for the last fraction of a frame, `Atomics.wait` versus a busy loop in Web Workers.
Watch out: —
Seen in Dolphin: The 2009 commits show this decision being made in several places. The audio mixer went from `sleep(0)` to `usleep(1000)` because the busy loop "pollutes profiles", and the FIFO waits gained `SleepCurrentThread(1)` "for the planet". Then `YieldCPU()` (`YieldProcessor`) replaced `SleepCurrentThread` inside the Command Processor waits. `WaitForFrameFinish` and the FIFO drain wait now block on `s_fifoIdleEvent`, set by the video thread. Finally, EFB peeks and swap requests dropped `Common::Event` in favor of spinning on `AtomicLoadAcquire(flag)` with `YieldCPU()`, because "events cause kernel calls and it is sometimes more efficient to do spin-waits". 57c5f00337 2009-01 “Sleep for 1ms instead of zero time; the busy-sleep loop pollutes profiles.” · 32ef02c959 2009-01 “For the planet.” · ed3281290d 2009-01 “CPU sleeps when fifo HiWaterMark reached now.” · a5d65b3b22 2009-07 “No more SleepCurrentThreads in Command Processor.” · ccfc004161 2009-07 “CP: Replace a couple spin-waits with event signals.” · 8027ab53d0 2009-07 “Don't use events for efb access.”

### dolphin-016 — Replace a lock around a counter with an atomic add
Lens: concurrency · Source: Dolphin
Principle: If a critical section only adjusts one shared integer, a lock adds acquire/release overhead and a chance of contention for a single read-modify-write. A hardware atomic add (`lock xadd`) does the same job in one instruction. Where you do need ordering, use acquire/release on the flag rather than full fences on every access.
Why it is faster: —
Applies elsewhere: request counters and refcounts (`std::atomic<int>::fetch_add`, `AtomicInteger`), lock-free ring buffer indices, statistics in high-throughput servers.
Watch out: —
Seen in Dolphin: `GatherPipeBursted` used to wrap `fifo.CPReadWriteDistance += 32` in `fifo.sync->Enter()/Leave()`. It now calls `Common::InterlockedExchangeAdd`, which wraps `__sync_add_and_fetch`, and the video FIFO's decrement does the same. Later, `AccessEFB` dropped two redundant `MemFence()` calls, and the EFB/swap flags moved to `AtomicStoreRelease`/`AtomicLoadAcquire`. 3d6d64c632 2008-10 “Get rid of locks added InterlockedExchangeAdd function in thread.cpp” · d99ac909b9 2009-07 “Remove a couple unnecessary MemFences” · 8027ab53d0 2009-07 “Don't use events for efb access.”

### dolphin-017 — Snapshot the data, then do slow I/O on a background thread
Lens: concurrency · Source: Dolphin
Principle: Disk writes, compression and image encoding can stall a real-time loop for hundreds of milliseconds. Copy the data into a private buffer on the hot thread, which is fast, and hand that buffer to a worker that compresses and writes. The hot thread pays only for the copy.
Why it is faster: —
Applies elsewhere: Redis `BGSAVE` (fork as a snapshot), async loggers with a queue to a writer thread, games that save checkpoints off the render thread, a browser encoding a screenshot in a worker.
Watch out: —
Seen in Dolphin: `CEXIMemoryCard::Flush` packs filename, pointer, size and flags into a `flushStruct` and starts `innerFlush` on a `Common::Thread`. Save states serialize into a `new u8[sz]` buffer and pass it to `CompressAndDumpState` on `saveThread`. PNG screenshots go to a `TakeScreenshot` thread with a `wxImage`. DVD command execution moved to `DVDThreadFunc`, which posts the completion interrupt back through `CoreTiming::ScheduleEvent_Threadsafe`. 2f0c872cd2 2009-06 “Flushing the memory card data to the hard-drive is now being done in a separate thread.” · 46b0fd3c96 2009-06 “Saving the state (compression and memory dump to file) is now done on a separate thread” · 28198d0062 2009-07 “Should reduce/fix the little stuttering when saving a screenshot as PNG” · 402abca90a 2009-08 “Separate DVD thread.”

### dolphin-018 — Skip the flush when nothing changed
Lens: laziness · Source: Dolphin
Principle: Periodic or shutdown-time persistence often rewrites data that hasn't changed. Set a dirty flag on every mutating path and return early from the flush when it's clear.
Why it is faster: —
Applies elsewhere: editors that skip saving an unmodified buffer, config managers that only write on change, a React `shouldComponentUpdate` that short-circuits.
Watch out: —
Seen in Dolphin: `CEXIMemoryCard` gained `m_bDirty`. It is set in the `SetCS` command handlers that erase or write, and `Flush()` returns immediately when it is false. As a result, stopping emulation became "uber-fast". 06b65ea425 2009-07 “Uber-fast stop (thanks to smart memory card flushing)”

### dolphin-019 — Shadow API state and drop redundant updates
Lens: gpu-rendering · Source: Dolphin · Also: Flycast
Principle: Every graphics API call costs driver validation and CPU time, even when it sets a value that is already set. Keep a CPU-side copy of the last value sent, such as constants, the bound shader or render state, and make the call only when the new value differs. When a register write arrives, compute which bits changed and act only on those.
Why it is faster: —
Applies elsewhere: virtual DOM diffing, state caches in WebGL wrappers (three.js `WebGLState`), skipping a `setState` when the value is equal.
Watch out: —
Seen in Dolphin: `SetPSConstant4f`/`SetVSConstant4f` in both the D3D and GL plugins compare against a `lastPSConstants[][4]` shadow before calling `SetPixelShaderConstantF` or `glProgramEnvParameter4fARB`. `SetShader`/`GetShader` return early if the shader UID equals `last_entry_uid`. For BP register writes, `LoadBPReg` computes `changes = (oldval ^ newval) & 0xFFFFFF`, and `BPWritten` flushes the vertex batch only `if (changes)`. The vertex loader flushes only when `g_nativeVertexFmt != &m_NativeFmt`. The authors themselves saw little FPS change on fast GPUs. c1665ec6cd 2009-09 “D3D: Reduce the number of DirectX API calls.” · 891aa44dd0 2009-09 “OGL: API call reduction.” · dcbc8e78d4 2008-10 “A minor speedup for BP writes - merged the two switch()-es.” · f14fca914d 2008-10 “cache pointer to NativeVertexFormat instead.” • Also seen in Flycast (flycast-067): `core/rend/gles/glcache.h` adds a `GLCache` whose `BindTexture`, `BlendFunc`, `DepthFunc`, `DepthMask`, `Enable`, `UseProgram` and similar methods call GL only on change. A follow-up adds `std::map<GLuint, TextureParameters>` so that `TexParameteri` filter and wrap settings are cached per texture. That replaced a fragile one-entry cache in `gldraw.cpp`. The same commit generates texture IDs in batches (`TEXTURE_ID_CACHE_SIZE 32`) and skips modifier-volume work entirely when no volumes are present. de046c462 2018-05 “Use OpenGL state cache. Allocate texture ids in bulk. Avoid modifier volumes entirely when not needed.” · 80c193b3d 2018-05 “Cache texture parameters.” • Also seen in Flycast (flycast-090): `core/rend/gles/glcache.h` wraps `BindTexture`, `BlendFunc`, `UseProgram`, the stencil and depth calls and so on, each guarded by `if (value != _cached || _disable_cache)`. A one-line bug had left `_disable_cache = true`, so every call went to the driver. Fixing it turned the cache on. The ImGui OpenGL backend was then moved onto glcache and its save/restore of all GL state was deleted. f83a06d9c 2019-04 “glcache was always disabled” · 05df2f06e 2019-04 “imgui: use glcache, don't save/restore state”

### dolphin-020 — Batch many small draws into one call
Lens: gpu-rendering · Source: Dolphin · Also: Flycast
Principle: Fixed per-call overhead dominates when a frame issues thousands of tiny draws. Group primitives that share state and submit them together, either with a multi-draw entry point or by converting strips, fans and quads into one indexed triangle list. Then one call covers the whole batch.
Why it is faster: —
Applies elsewhere: sprite batching in 2D engines, instanced rendering, bulk INSERT instead of row-by-row, coalescing small network writes (Nagle, `writev`).
Watch out: —
Seen in Dolphin: `VertexManager` first collected `s_vertexFirstOffset`/`s_vertexGroupSize` per primitive type and called `glMultiDrawArrays` once per group instead of `glDrawArrays` per strip. The commit reports about 10% in one case. Later, an `IndexGenerator` (`AddStrip`, `AddFan`, `AddQuads`) rewrote every primitive into triangle, line or point lists. Batches only break when that render primitive changes, and each flush is a single `glDrawElements` (or `glDrawArrays` when the batch holds only lists). `MAX_BUFFER_SIZE` also grew from 0x4000 to 0x50000. 90c9810b85 2009-02 “Use glMultiDrawArrays instead of glDrawArrays, which significantly reduces the number of OpenGL calls” · 29808cdde2 2009-09 “OpenGL: commit rodolfoosvaldobogado's … speedup patches.” • Also seen in Flycast (flycast-070): `DrawModVols` in `gldraw.cpp` used to switch VBOs and draw a full-screen quad (`glDrawArrays(GL_TRIANGLE_STRIP, 0, 4)`) to sum up each closed modifier volume. It now tracks `mod_base` and issues `glDrawArrays(GL_TRIANGLES, mod_base*3, (end - mod_base)*3)` over the volume's own triangles. 6871061ca 2018-06 “Modifier Volumes: summing up with triangles is faster than using a quad.”

### dolphin-021 — Upload contiguous ranges in one call
Lens: gpu-rendering · Source: Dolphin
Principle: If an API accepts a count, send adjacent registers or array slots in one call rather than looping one slot at a time. Validation and call overhead are paid once.
Why it is faster: —
Applies elsewhere: `glUniform4fv(loc, n, ptr)` instead of n calls, Redis `MSET`/pipelining, batched GraphQL or REST endpoints.
Watch out: —
Seen in Dolphin: `VertexShaderManager` now uploads transform, post-transform, texture and projection matrices with `SetMultiVSConstant4fv(start, count, ptr)`, which maps to one `SetVertexShaderConstantF(..., count)`, instead of three or four `SetVSConstant4fv` calls. The commit reports a "very tiny speedup". 52ea8a0fd1 2009-09 “D3D supports setting multiple shader constants at once, so let's add support for that.”

### dolphin-022 — Reuse resources instead of destroying and recreating them
Lens: gpu-rendering · Source: Dolphin
Principle: Creating a texture or device buffer allocates driver memory, validates it and possibly syncs. If the replacement has the same size and format, write new contents into the existing object. Allocate scratch buffers once at their maximum size.
Why it is faster: —
Applies elsewhere: reusing `ArrayBuffer`s or `ImageData` per frame, `texSubImage2D` instead of `texImage2D`, connection and buffer pools.
Watch out: —
Seen in Dolphin: `TextureMngr::Load` (GL) and `TextureCache::Load` (D3D) check `width == entry.w && height == entry.h && format == entry.fmt`. On a match they rebind the existing texture and skip `glGenTextures`/`CreateTexture`, and D3D uses `ReplaceTexture2D` to upload. In the OpenCL decoder, a fresh `clCreateBuffer(CL_MEM_COPY_HOST_PTR)` per texture became two 4 MB buffers created at init and filled with `clEnqueueWriteBuffer`. The commit reports "a visible performance boost". f3eda7d85a 2008-12 “GL: Don't create a new texture every time the contents of one is updated.” · 7a8f6bdd6d 2009-09 “texture replace instead of destroy/create when possible” · 540f59ef07 2009-10 “OpenCL: More formats … Optimized the buffer creation.”

### dolphin-023 — Cache failures too
Lens: laziness · Source: Dolphin
Principle: If a cache stores only successful results, every request for an input that fails repeats the expensive attempt. Store the failure as a cache entry, for example a null value. Later lookups then fail quickly instead of recompiling, refetching or re-resolving.
Why it is faster: —
Applies elsewhere: DNS negative caching (NXDOMAIN TTL), HTTP caching of 404s, memoizing "not found" in a module resolver.
Watch out: —
Seen in Dolphin: D3D `PixelShaderCache::SetShader`/`VertexShaderCache::SetShader` now insert a `PSCacheEntry` even when compilation returned null. A cache hit with `entry.shader == NULL` returns false right away, so a bad shader is no longer "re-generated / re-compiled again, and again". 47a16da49a 2009-09 “Some shader manager changes, to avoid a shader be re-generated / re-compiled again, and again, if it is failed at the first time.”

### dolphin-024 — Specialize generated code and leave out identity operations
Lens: compiler-build · Source: Dolphin
Principle: When you generate source code from configuration, emitting the general formula and hoping the downstream compiler simplifies it is a missed opportunity. Check for the trivial configurations, such as multiply by 1, lerp between zeros, or no clamp, and emit the simpler expression. The output gets shorter and cheaper to run on every invocation.
Why it is faster: —
Applies elsewhere: query builders that skip `WHERE 1=1` terms, CSS-in-JS compilers that drop default-valued declarations, code generators that skip an empty loop.
Watch out: —
Seen in Dolphin: `PixelShaderGen.cpp WriteStage` now omits `tevScaleTable[shift]*` when `shift == 0` and emits `0` instead of a `lerp` when both inputs are zero. It wraps the stage in `saturate()` only when the stage clamps. The commit measured 20–30% fewer instructions in compiled Zelda TP shaders, for example 38 down to 28. 718c88ff6a 2009-08 “Some optimization on the pixel and vertex shader generators … reduce the number of lines in the compiled pixel shader program by 20% to 30%”

### dolphin-025 — Upload in the narrowest format and let the GPU build mipmaps
Lens: gpu-rendering · Source: Dolphin
Principle: Expanding single-channel data to RGBA on the CPU quadruples the bytes decoded, copied and uploaded. Upload luminance or intensity data in its native 8-bit format. Ask the GPU to generate mipmaps instead of computing them in a CPU library call.
Why it is faster: —
Applies elsewhere: uploading grayscale masks as R8 or alpha-only textures in WebGL, `generateMipmap()` instead of CPU resampling, sending compressed textures as-is.
Watch out: —
Seen in Dolphin: I4/I8/IA formats now decode to `PC_TEX_FMT_I8`/`IA8` and upload as `GL_LUMINANCE`/`GL_INTENSITY` instead of expanded 32-bit RGBA, which removed about 190 lines of RGBA-expansion code from `TextureDecoder.cpp`. `gluBuild2DMipmaps`, which resamples on the CPU, was replaced by `glTexParameteri(GL_GENERATE_MIPMAP, GL_TRUE)` around a single `glTexImage2D`. d2d097fba5 2009-02 “Committing magumagu9's work on texture decoding transferral to OGL”

### dolphin-026 — Move per-pixel decoding to a GPU compute kernel
Lens: gpu-rendering · Source: Dolphin
Principle: Texture and color-format decoding is fully data-parallel: every output block depends only on its own input bytes. One kernel invocation per output block turns a serial CPU loop into thousands of parallel GPU threads. The cost moves to the host-device transfer, so keep buffers persistent.
Why it is faster: —
Applies elsewhere: image filters in WebGL or WebGPU compute, video color-space conversion in shaders, CUDA kernels for decoding compressed columns in databases.
Watch out: —
Seen in Dolphin: `VideoCommon/Src/OpenCL/TextureDecoder.cpp` embeds kernels such as `DecodeRGBA8`, `DecodeRGB565` and `DecodeRGB5A3`. Each kernel computes `x = get_global_id(0)*4, y = get_global_id(1)*4` and decodes one 4×4 tile. The host picks the kernel by format, writes the source with `clEnqueueWriteBuffer` and reads back the result. XFB YUV→RGB conversion got an OpenCL path in the same series. 0146f3f58f 2009-10 “More OpenCL work, got XFB converting which needs more optimization.” · 540f59ef07 2009-10 “OpenCL: More formats … RGBA8, RGB565, RGB5A3.”

### dolphin-027 — Decode pixels 16 bytes at a time, with runtime CPU dispatch
Lens: simd · Source: Dolphin
Principle: Nibble expansion, byte shuffles and channel swizzles map directly onto SIMD AND/shift/OR and byte-shuffle instructions, so one instruction handles 16 pixels or channels. Make the per-block inner functions fixed-size so the compiler can unroll them. Pick the best instruction set (SSE2 or SSSE3 `pshufb`) at run time from CPUID.
Why it is faster: —
Applies elsewhere: SIMD base64 or UTF-8 validation, simdjson's runtime ISA dispatch, image libraries (libjpeg-turbo, Skia) with SSE/AVX/NEON paths chosen at startup.
Watch out: —
Seen in Dolphin: `TextureDecoder.cpp`: `decodebytesI4` and the other decoders lost their `numbytes` parameter in favor of constant 4- or 8-iteration loops. An SSE2 I4 path does `sl = s & 0x0F; sl | (sl << 4)` on 16 bytes at a time. A later SSSE3 path does the expansion with `_mm_shuffle_epi8` against eight precomputed masks, guarded by `cpu_info.bSSSE3`. Earlier, ARGB8 decoding merged two passes over the destination (`pass1`/`pass2`) into one fixed 4-pixel `decodebytesARGB8_4`. 2cc477b7d5 2008-10 “"as is" sse in some cases in TextureDecoder.cpp” · b252be40cf 2009-01 “Added S-SSE3 implementation of one perf. critical texture decoder” · 683ba7ecae 2008-08 “Minor speedup in texdecoder.”

### dolphin-028 — Precompute each input byte's vector contribution
Lens: simd · Source: Dolphin
Principle: A linear transform such as a color-space conversion is a sum of per-input terms. Precompute, for every possible byte value, a 4-lane vector of its contribution to every output channel. Then each pixel costs a few table loads and SIMD adds. Clamping comes free from saturating pack instructions instead of per-channel compare-and-branch.
Why it is faster: —
Applies elsewhere: YUV↔RGB in video players, CRC slicing-by-8 tables, gamma or tone curves as LUTs feeding SIMD.
Watch out: —
Seen in Dolphin: `XFBConvert.cpp`: `ConvertFromXFB` builds `_y[256]`, `_u[256]` and `_v[256]` as `__m128i` tables of fixed-point coefficients. Each pixel pair is `_mm_srai_epi32(y + u + v, 16)`, then `_mm_packs_epi32`/`_mm_packus_epi16`, which also clamps to 0–255. The first step had already swapped the clamp's nested ternary for a `bound_lut` table and factored out `76283*(y-16)`. `ConvertToXFB` uses `_r1/_g1/_b1` tables the same way. 34a4fee1c5 2008-10 “Substantial XFB speedup.” · db3a5e3822 2008-10 “More XFB speedup. It's all SSE now.”

### dolphin-029 — Fold constant offsets into the lookup table
Lens: lookup-tables · Source: Dolphin
Principle: If every use of a table entry adds the same constant, add it once when building the table. That removes one add from every element processed.
Why it is faster: —
Applies elsewhere: pre-biasing a histogram or gamma table, baking a base address into a jump table, storing pre-offset glyph positions in a font atlas.
Watch out: —
Seen in Dolphin: `InitXFBConvTables` now stores `_r1[i] = _mm_add_epi32(..., _bias1)` (and `_r2` with `_bias2`), so `ConvertToXFB` no longer adds `_bias1`/`_bias2` per pixel pair. The same commit switched to aligned stores. caf0d58047 2008-10 “Further tiny XFB speedup: baked the bias constants into the R tables”

### dolphin-030 — Align buffers and strides for the consumer
Lens: memory-layout · Source: Dolphin
Principle: Aligned SIMD stores are cheaper than unaligned ones, and GPUs fetch vertex attributes fastest when each attribute and the stride are 4-byte multiples. Allocate buffers on page or 16-byte boundaries. Pad 3-component attributes to 4.
Why it is faster: —
Applies elsewhere: `aligned_alloc` for SIMD arrays, std140 padding in uniform buffers, padding WebGL vertex attributes, cache-line aligning hot structs.
Watch out: —
Seen in Dolphin: `XFB_Init` allocates `efb_buffer` with `AllocateMemoryPages`, which is page-aligned, instead of `new u8[]`, so `ConvertFromXFB` can use `_mm_store_si128` instead of `_mm_storeu_si128`. The vertex loader now writes byte and short normals as 4 components (`offset += 4`/`8`), so every component starts on a 4-byte boundary. It treats any odd stride as an error instead of adding a `m_VBStridePad`. An earlier comment had noted that some cards ran 3× faster with strides divisible by 4. caf0d58047 2008-10 “switched to aligned writes, and made sure efb_buffer is aligned.” · 5129341a9c 2008-10 “Aligned all vertex components to 4 byte boundaries”

### dolphin-031 — Compute small conversions with shifts instead of tables
Lens: lookup-tables · Source: Dolphin
Principle: A tiny lookup table still costs a memory load and cache space. Scaling an n-bit value to 8 bits can be done exactly by replicating its bits (`(v<<3)|(v>>2)` for 5→8) in a couple of register ops. In the same spirit, replace divides by 3 and float `ceil` with integer shifts and `+1` rounding.
Why it is faster: —
Applies elsewhere: RGB565→RGB888 in display drivers, PNG bit-depth scaling, integer blend weights (3/8, 5/8) in DXT/BCn decoders.
Watch out: —
Seen in Dolphin: `LookUpTables.cpp` was deleted (`lut3to8`, `lut4to8`, `lut5to8`, `lut6to8`). Inline `Convert3To8` through `Convert6To8` replace them, for example `(v << 2) | (v >> 4)` for 6→8. `decodeDXTBlock` now computes interpolants as `((c2-c1)>>1) - ((c2-c1)>>3)` (3/8) instead of `/3`, and averages with `(a+b+1)/2` instead of `ceil((float)(a+b)/2)`. The commit notes this is "slightly more accurate and faster". 8ab4814d73 2009-07 “Don't use lookup tables. It's better to use CPU registers and reduce memory accesses.” · 97af2f1126 2009-09 “GL/D3D: slightly more accurate and faster DXT decoding”

### dolphin-032 — Detect changes with a sparse sampling hash
Lens: algorithms · Source: Dolphin
Principle: To decide whether a large buffer changed, hashing every byte costs as much as reprocessing it. Hash a fixed number of evenly spaced words instead, about 13 regardless of size, and fully hash only small inputs. You trade a small chance of missing an edit for a constant-time check.
Why it is faster: —
Applies elsewhere: rsync weak checksums, perceptual or sampled hashes for media dedup, sampling a large file's head, middle and tail before a full compare.
Watch out: —
Seen in Dolphin: `TexDecoder_GetSafeTextureHash`: below 2048 bytes it XOR-rotates every 13th word. Above that it uses `step = sz / 13 / 4`, so it reads a constant number of words. The "safe texture cache" compares this hash to decide whether a cached decoded texture is stale. A matching `TexDecoder_GetTlutHash` covers palettes. d0a48f654a 2008-11 “Improve safe texture cache - now using sparse CRC to identify textures.” · bc3a7bdce3 2008-12 “Better tlut hash for fixing MPs font. Didn't notice any slowdown for now.”

### dolphin-033 — Generate a straight-line routine per data format
Lens: jit-interpreters · Source: Dolphin
Principle: A generic decoder walks an array of function pointers for every record and re-checks the format each time. For each distinct format, emit machine code that calls exactly the needed steps in sequence, and put the record loop inside it too. Per-record dispatch and loop overhead go away.
Why it is faster: —
Applies elsewhere: generated serializers (protobuf codegen, simdjson on-demand), JIT-compiled regexes, eBPF programs compiled per filter.
Watch out: —
Seen in Dolphin: `VertexLoader::CompileVertexTranslator` used to fill `m_PipelineStages[]`, and `RunVertices` looped over it per vertex. It now emits `CALL`s into executable memory (`m_compiledCode`). The next commit moved the vertex loop into the generated code (`SUB loop_counter; J_CC(CC_NZ, loop_start)`). It also emits the `tcIndex`/`colIndex`/`s_texmtxwrite` resets only when the vertex format actually has texcoords, colors or matrix indices. 5c58227702 2008-11 “Optimize vertex loader with a mini JIT” · 72af432491 2008-11 “Put the vertex loop inside the JITted code in the vertex loader.”

### dolphin-034 — Move per-item branches to setup time
Lens: branching-fast-paths · Source: Dolphin
Principle: A branch on a value that is constant for a whole batch, such as a component count or a format flag, still runs once per item. Decide it once at setup: pick a specialized function variant, or precompute the flags into plain booleans. The inner loop then has no data-dependent branch.
Why it is faster: —
Applies elsewhere: choosing a specialized parser for a CSV dialect once per file, monomorphizing generics, picking a blit function per pixel-format pair.
Watch out: —
Seen in Dolphin: `VertexLoader_Position.cpp` had one reader per format that checked `PosElements` (2 vs 3) for every vertex. It was split into `Pos_ReadDirect_UByte2`/`UByte3` and similar variants for every format and index mode, and `CompileVertexTranslator` picks the right one. In D3D, `DecodedVArray::ComputeComponents()` now runs once in `SetComponents` and caches `hasPosMatIdx`, `hasTexMatIdx[8]` and `hasNrm`, so the transform loop no longer tests component bits per vertex. c49f969563 2009-02 “vertexloader_pos cleanup - remove the branch per vertex.” · 8eef6f9cd5 2008-10 “Moved components' perks computation to the DecodedVArray in DX9. Performance boost.”

### dolphin-035 — Memoize expensive objects by a key of their inputs
Lens: laziness · Source: Dolphin
Principle: When building an object is costly (generated code, parsed metadata, an opened container) and the same inputs keep coming back, key a map on a compact UID of the inputs and reuse the built object. Mark keys dirty when inputs change and look up again only then. Persist the cache to disk when the cost comes from I/O.
Why it is faster: —
Applies elsewhere: compiled-template caches, Babel or Webpack persistent caches keyed by content hash, prepared-statement caches, thumbnail caches.
Watch out: —
Seen in Dolphin: `VertexLoaderManager` replaced its 8 fixed loaders with `std::map<VertexLoaderUID, VertexLoader*>` and an `s_attr_dirty` bitfield. `RefreshLoader` looks up or builds a loader only when that attribute group is dirty. The ES device caches `CNANDContentLoader`s by path because "we open this content files ways to often". `GameListItem` saves its parsed banner, name and description to a per-ISO cache file (`CChunkFileReader::Save`, `CACHE_REVISION`) and loads it with `LoadFromCache()` before opening the disc. f77624147d 2008-10 “Cachable vertex loaders.” · 0b76db1c04 2009-03 “added cached for NANDContentLoader” · 4ad86ef180 2008-11 “added caching on GameListItem level”

### dolphin-036 — Translate an address once, then walk a raw pointer
Lens: memory-layout · Source: Dolphin
Principle: Going through a general accessor for every byte means repeating the whole address decode (region checks, masking, table lookup) for adjacent bytes. Translate the base address once into a host pointer and read fields relative to it. Better still, cache the translated base whenever the guest base register is written.
Why it is faster: —
Applies elsewhere: getting a direct `DataView` or typed-array view once instead of calling a getter per field, mapping a file once and parsing in place, resolving a memory-mapped I/O base at init.
Watch out: —
Seen in Dolphin: The vertex loaders' indexed readers did three `Memory_Read_U8/U16/U32(iAddress+k)` calls per component, and each one went through `Memory::GetPointer`. They now call `Memory_Read_U8_Ptr(iAddress)` once and index `pData[k]`. A follow-up added `cached_arraybases[16]`. The base is translated with `Memory_GetPtr` when the CP array-base register is written, so position reads become `cached_arraybases[ARRAY_POSITION] + Index * stride`. c450e44b08 2009-02 “optimize memory access. reduce Memory_Read_U##_type call (it uses Memory::GetPointer, which is complex)” · d209d4a577 2009-02 “Cache vertex array pointers, a la magumagu's idea.”

### dolphin-037 — Hoist expensive lookups out of hot paths
Lens: other · Source: Dolphin
Principle: Singleton accessors, plugin registries, symbol-table lookups and OS time calls look cheap at the call site but can involve locks, maps or syscalls. Look the value up once, before the loop or at init, and keep the result. When a lookup must repeat, memoize the last key and answer.
Why it is faster: —
Applies elsewhere: caching `document.getElementById` outside a loop, reading `Date.now()` once per frame, keeping a logger reference instead of calling `getLogger()` per message.
Watch out: —
Seen in Dolphin: `CatchUpGPU` moved `CPluginManager::GetInstance().GetVideo()` out of its per-32-byte loop. `DSP.cpp` caches `dsp_plugin` at `Init` instead of calling `GetDSP()` on every mailbox access, after it showed up in oprofile. `VideoInterface::Update` calls `timeGetTime()` once instead of twice. `LogManager::Log` memoizes `lastPC`/`lastSymbol`, because `g_symbolDB.GetDescription(PC)` on every log line made games "crawl to a halt". `PPCAnalyst::Flatten` stopped calling `GetSymbolFromAddr` for every block. 013e9bed93 2009-02 “GetVideo() is a kind of bottleneck, so move it out of loop” · 37332216ef 2009-07 “Cache pointer to dsp plugin. turns out the lookup is pretty expensive” · 16d9a65d25 2009-08 “less timeGetTime calls, which showed up in TD's profiles.” · 290966a4f8 2008-10 “the symbol lookup would make the game crawl to a halt” · a9c6a59307 2008-08 “don't lookup things in symbol map unnecessarily in the ppcanalyst”

### dolphin-038 — Replace a virtual stream reader with an inline pointer bump
Lens: compiler-build · Source: Dolphin
Principle: A virtual `Read8()`/`Read32()` interface costs an indirect call per field and blocks inlining. When there is really only one hot implementation, expose the read cursor as a plain pointer and make the readers inline functions that dereference and advance it. The compiler can then fuse consecutive reads.
Why it is faster: —
Applies elsewhere: byte-buffer cursors in binary parsers (Rust `bytes::Buf` with a concrete type, Netty `ByteBuf`), replacing an `InputStream` chain with direct array access in a hot decoder.
Watch out: —
Seen in Dolphin: The GL plugin's `DataReader` was an abstract class with `CDataReader_Fifo` and `CDataReader_Memory` subclasses. Under `DATAREADER_INLINE` it became a global `g_pVideoData` pointer and inline `DataPeek8`/`DataReadU32` functions. A `DATAREADER_DEBUG` mode cross-checks the new cursor against the old virtual reader. Later the readers became templates (`DataRead<u16>()`) with specializations that byteswap. df9eba79b2 2008-09 “DataReader migration to faster one: first step.”

### dolphin-039 — Recycle fixed-size nodes through a free list
Lens: allocation · Source: Dolphin
Principle: A scheduler that allocates and frees one small node per event pays allocator cost on the hottest path. Keep freed nodes on an intrusive singly linked free list and pop from it before calling `new`. Steady state then does no heap traffic at all.
Why it is faster: —
Applies elsewhere: timer-wheel node pools in network stacks, object pools in game engines, slab allocators in kernels.
Watch out: —
Seen in Dolphin: `CoreTiming.cpp` added `eventPool`/`eventTsPool`. `GetNewEvent()` pops from the pool or falls back to `new Event`, and `FreeEvent()` pushes the node back using its existing `next` field. Thread-safe events use their own pool under `externalEventSection`. The same commit made `GetFilename()` return `const std::string&` instead of a copy. b7c398574f 2009-02 “Memory pools are set up in core timing to avoid frequent calls to new and delete.”

### dolphin-040 — Store values contiguously and skip temporary strings
Lens: allocation · Source: Dolphin
Principle: A vector of pointers to small structs costs one heap allocation per element plus a pointer chase on every access. Store the structs by value and hand out pointers into the vector. Likewise, pass `const char*` or a reference where a function only reads a string, and don't wrap a stack buffer in a `std::string` just to call `c_str()` on it.
Why it is faster: —
Applies elsewhere: `Vec<T>` instead of `Vec<Box<T>>`, `std::string_view` parameters, avoiding `String` concatenation in Java logging hot paths.
Watch out: —
Seen in Dolphin: `CFileSystemGCWii`: `std::vector<SFileInfo*>` became `std::vector<SFileInfo>`, and `GetFileList` returns `&m_FileInfoVector[i]` instead of `new SFileInfo(copy)`. `File::Exists`/`IsDirectory` and the other helpers take `const char*`. `Renderer::SwapBuffers` renders its stats directly from `debugtext_buffer` instead of copying it into a `std::string text` first. cca92d4e38 2008-09 “changed vector<FileInfo*> to vector<FileInfo> for less allocs and more speed in debug mode” · 38f04809f1 2008-09 “Reduce the use of string objects slightly.” · 33ea7313ba 2009-02 “remove unnecessary string alloc in overlay stats string code”

### dolphin-041 — Use a fixed-capacity ring buffer for streaming queues
Lens: allocation · Source: Dolphin
Principle: General-purpose queues (`std::queue`/`deque`) allocate chunks as they grow and carry iterator and bounds machinery. When the maximum depth is known, a preallocated array with head and tail indices that wrap gives O(1) push and pop with no allocation and contiguous memory.
Why it is faster: —
Applies elsewhere: audio callback buffers, UART RX buffers in embedded firmware, the LMAX Disruptor, bounded log buffers.
Watch out: —
Seen in Dolphin: `Common/Src/FixedSizeQueue.h` adds `template <class T, int N> class FixedSizeQueue`: a `new T[N]` array with `head`, `tail` and `count`. The DSP HLE `Mixer.cpp` sample FIFO switched from an STL queue to it. 48c9333c0f 2008-09 “Switch audio fifo queue to a faster non-STL one.”

### dolphin-042 — Convert byte order at load time, not on every access
Lens: memory-layout · Source: Dolphin
Principle: If memory is stored in a foreign byte order and swapped on every read and write, the hottest paths pay for a conversion that only matters at the boundary. Swap once when data enters (ROM load, DMA in) and leaves (DMA out). The interpreter then reads native words.
Why it is faster: —
Applies elsewhere: parsing network-order headers into native structs once, converting a big-endian file format at load, keeping UTF-16 text decoded rather than re-decoding per character.
Watch out: —
Seen in Dolphin: DSP LLE `gdsp_memory.cpp`: `dsp_imem_read`/`dsp_dmem_read`/`write` lost their `dsp_swap16`. The IROM and COEF ROMs are byteswapped once after loading, and the DMA routines (`gdsp_ddma_in`/`out`, `gdsp_idma_in`) apply `Common::swap16` while copying to or from emulated RAM. 8c861ad58d 2009-04 “DSP LLE: Remove byteswaps and other bloat from the hot path (do them at load/dma time).”

### dolphin-043 — Check rare conditions once per slice, not every step
Lens: branching-fast-paths · Source: Dolphin
Principle: Testing for an external interrupt or pause request before every simulated instruction puts a load and branch in the innermost loop, though the condition changes rarely. Check it at slice boundaries, every few hundred steps or once per call, and run the steps between checks unconditionally.
Why it is faster: —
Applies elsewhere: checking a cancellation token every N iterations, interpreters that poll for signals at loop back-edges only, game loops that read input once per frame.
Watch out: —
Seen in Dolphin: `gdsp_step` used to call the external-interrupt check every cycle. That check moved to `gdsp_check_external_int()`, called once per 500-step batch in `gdsp_run` and once at the start of `gdsp_run_cycles`. The same commit's analyzer marks loop endpoints (`CODE_LOOP_END`) "so that we can avoid checking for loops every cycle". 0f26228ef2 2009-04 “Don't check for external interrupts every cycle. Loop endpoint detector added to Analyzer”

### dolphin-044 — Compile out debug checks in release builds
Lens: compiler-build · Source: Dolphin
Principle: Breakpoint lookups, memory watchpoints and checked iterators each add work to every operation. Put them behind build-time macros that are on only in debug configurations. The release binary then runs the plain path with no runtime flag test.
Why it is faster: —
Applies elsewhere: `NDEBUG` asserts, `_ITERATOR_DEBUG_LEVEL=0` in MSVC, Rust `debug_assert!`, stripping dev-only checks with a bundler's dead-code elimination.
Watch out: —
Seen in Dolphin: `DSPCore_RunCycles` now checks breakpoints only `#if defined(_DEBUG) || defined(DEBUGFAST)`. `Memmap.h` defines `ENABLE_MEM_CHECK` only for debug builds with high log levels, and every `Read_U*`/`Write_U*` memcheck block now tests that define. The old `NOCHECK` define sat in the wrong file and never took effect in the accessor file. Project settings require `_SECURE_SCL=0`, which turns off MSVC's checked STL iterators, and a compile-time assert enforces it. 1c95100eb1 2009-11 “DSPLLE: moved breakpoint to debug to gain some speed” · 45d9d3191b 2009-07 “Move define to the right place, get some free speed” · 8e3bd6781e 2008-10 “added compile time assertion for _SECURE_SCL=0.”

### dolphin-045 — Stop a scan at a sentinel instead of checking every slot
Lens: branching-fast-paths · Source: Dolphin
Principle: Scanning a fixed-size array and testing each slot for "used" visits every slot even when only one or two are filled. Keep used entries packed at the front with a sentinel after the last, and loop until the sentinel. Reserve one extra slot so the sentinel always fits.
Why it is faster: —
Applies elsewhere: NUL-terminated arrays, sentinel nodes in linked lists, `argv`-style NULL-terminated vectors.
Watch out: —
Seen in Dolphin: `DSPIntExtOps.cpp applyWriteBackLog` went from `for (i < WRITEBACKLOGSIZE) if (idx[i] != -1) …` to `for (i = 0; writeBackLogIdx[i] != -1; i++)`. It relies on an extra trailing `-1` entry, as a comment notes. 56c4eea4c1 2009-08 “DSPLLE: trying to optimize a bit (based on drkrazi's suggestion)”

### dolphin-046 — Skip output you won't show, but still parse the stream
Lens: laziness · Source: Dolphin
Principle: When the consumer can't keep up, drop whole frames of output while still processing the side effects that keep the state machine correct. In the input stream, parse just enough to find each command's length. Skip bulk payloads such as vertex data, and apply only the state-changing commands.
Why it is faster: —
Applies elsewhere: video players dropping frames while still decoding reference frames, log shippers that sample, game servers that skip rendering on headless instances but still run the simulation.
Watch out: —
Seen in Dolphin: Frameskipping sets `g_bSkipCurrentFrame` through `Video_SetRendering`. `Renderer` skips `RenderToXFB` and swaps, and `VertexManager::Flush` just resets the buffer. The more aggressive version passes `skipped_frame` to `OpcodeDecoder_Run`. That path skips `numVertices * GetVertexSize(vat)` bytes of each draw with `DataSkip` instead of decoding them, while still applying BP register writes because they carry sync tokens. 51ddedf512 2009-08 “Frameskipping! … Even my most naive approach resulted in great speeds” · 6003c9ecd2 2009-08 “Frameskipping more aggressive (minor speedup…)”

### dolphin-047 — Test the common case first and let rare cases fall through
Lens: branching-fast-paths · Source: Dolphin
Principle: Checking for NaN on every float compare before the ordinary <, >, == tests puts the rare case on the hot path. Comparisons involving NaN are all false, so test <, >, == first and handle NaN in the final else. When full exactness still costs extra, make it an opt-in setting.
Why it is faster: —
Applies elsewhere: sort comparators that handle NaN or null last, hot validation code that checks the expected format before error cases, "strict mode" flags in numeric libraries.
Watch out: —
Seen in Dolphin: The interpreter's `fcmpo`/`fcmpu` used to start with `if (IsNAN(fa) || IsNAN(fb))`. They are now ordered `fa < fb` → 8, `fa > fb` → 4, `fa == fb` → 2, and the else branch handles the unordered result. In the JIT, the slower exact compare path (`jo.fpAccurateFcmp`) is now tied to `bEnableFPRF` and off by default, which the author said gives "a few fps in ZWW". ce029df144 2009-09 “FPU: not need to check NaN, slightly faster.” · 486445c6c7 2009-07 “Turn fpAccurateFcmp off by default.”

### dolphin-048 — Replace a general math call with its special case
Lens: other · Source: Dolphin
Principle: Library functions like `pow` handle arbitrary exponents with logs and exponentials. When the exponent is a known small integer, a direct multiply is exact and far cheaper.
Why it is faster: —
Applies elsewhere: `x*x` instead of `Math.pow(x, 2)` in JS physics loops, squared distances to skip `sqrt` in comparisons, shifts for power-of-two multiplies.
Watch out: —
Seen in Dolphin: `MathUtil.h` adds `inline float pow2f(float x) { return x * x; }`. `EmuDynamics.cpp PitchDegreeToAccelerometer` now uses `pow2f(tanf(0.5f*_Roll)) - 1` instead of `powf(..., 2)`. d3f7349692 2009-02 “pow is total overkill for x*x, introduce pow2 inline function to fix it.”

### dolphin-049 — Read aligned runs in one request
Lens: io · Source: Dolphin
Principle: A block reader that fetches one sector at a time through a one-entry cache pays per-request overhead for every sector of a large read. When a read starts on a block boundary and spans many blocks, send all the whole blocks in one request straight to the destination, bypassing the cache. Use the cached path only for the partial head and tail.
Why it is faster: —
Applies elsewhere: `readv` or large `pread` instead of per-page reads, HTTP range requests that cover many chunks, database read-ahead for sequential scans.
Watch out: —
Seen in Dolphin: `SectorReader::Read` now calls `ReadMultipleAlignedBlocks(block, num_blocks, out_ptr)` when `positionInBlock == 0 && remain > m_blocksize`. Subclasses such as the physical-drive blob can override it with one bulk read. The commit says it "didn't help very much" for drive reads. It was disabled because of a bug and re-enabled the same day after a `memcpy` offset fix. e3d1704979 2009-02 “Attempt to speed up drive reading by doing larger block reads” · 8224c3b449 2009-02 “Fix bug, reenable the drive optimization.”

### dolphin-050 — Use a speed-first compressor for snapshots
Lens: io · Source: Dolphin
Principle: For data written often and read rarely, such as save states and checkpoints, compression time matters more than ratio. A byte-oriented LZ variant like LZO compresses at near memory speed. It shrinks the write without becoming the bottleneck.
Why it is faster: —
Applies elsewhere: LZ4 or Snappy for database pages and Kafka batches, zstd at level 1 for hot backups, compressed swap (zram, lz4).
Watch out: —
Seen in Dolphin: `State.cpp` switched `bCompressed` to true and compresses the serialized state in chunks with minilzo's `LZO1X_1`, using a static `wrkmem` work buffer. 795c3bc4e0 2008-09 “Added the LZO external library. Added very fast LZO save state compression.”

### dolphin-051 — Suspend redraws and narrow scans during bulk UI work
Lens: other · Source: Dolphin
Principle: Inserting thousands of items into a visible widget can trigger a layout and repaint after each insert. Freeze the widget, insert everything, then thaw for one repaint. For file scans, filter by the extensions the user has enabled before touching the filesystem, instead of scanning everything and filtering afterward.
Why it is faster: —
Applies elsewhere: `DocumentFragment` or `requestAnimationFrame` batching in the DOM, `beginUpdate`/`endUpdate` in WinForms, glob patterns pushed down into a directory walker.
Watch out: —
Seen in Dolphin: `CCodeWindow::NotifyMapLoaded` wraps filling the symbol list in `symbols->Freeze()`/`Thaw()` instead of hiding and showing the control. `CGameListCtrl::ScanForISOs` adds `*.gcm`, `*.iso`/`*.gcz` and `*.wad` only if GC, Wii or WAD listing is enabled. The commit reports 4–10 s saved per refresh. 1b12bc38ac 2009-01 “Use wx style ListBox fast fill: Freeze and Thaw” · c6abb79e6e 2009-06 “Only scan for extensions that apply to the type of games set to show”

### dolphin-052 — Derive a ticking counter from a start timestamp when it is read
Lens: laziness · Source: Dolphin
Principle: If a value changes at a fixed rate, don't update it on every tick. Store a base value and the time it was set, and compute `base + (now - t0) / rate` only when someone reads it. The per-tick work goes away completely, and reads get one subtraction and one divide.
Why it is faster: —
Applies elsewhere: rate-limiter token buckets that refill when checked rather than from a timer, TTL caches that compare against the clock on lookup, embedded uptime counters derived from a free-running hardware timer.
Watch out: —
Seen in Dolphin: `SystemTimers` registered an `AdvanceCallback` that rewrote the PowerPC timebase (`TL/TU`) and a `fakeDec` counter every time the CPU advanced. It now records `fakeTBStartValue/fakeTBStartTicks` and `fakeDecStartValue/fakeDecStartTicks` when the register is written. `mftb`/`mfspr` call `GetFakeTimeBase()` or `GetFakeDecrementer()`, e.g. `fakeDecStartValue - (GetTicks() - fakeDecStartTicks) / TIMER_RATIO`, and the advance callback is no longer registered. 9e17007c90 2010-09 “Fixed TimeBase and Decrementer registers so that they are updated only when needed”

### dolphin-053 — Parse a container's index only when someone asks for a file
Lens: laziness · Source: Dolphin
Principle: Opening something to list it often needs only a header. Validate the header up front and defer building the full index until the first call that needs it, guarding each accessor with an `if (!initialized) init()`. Bulk scans then touch a few bytes per item instead of every directory table.
Why it is faster: —
Applies elsewhere: zip/tar browsers that read only the central directory, ORMs that defer loading relations, IDEs that index a project only when you search it.
Watch out: —
Seen in Dolphin: The game list built a `CFileSystemGCWii` per disc and ran `InitFileSystem()` in the constructor, even though Wii banners live outside the disc. The constructor now only runs `DetectFileSystem()`. `GetFileSize`, `ReadFile`, `ExportFile`, `GetFileList` and `FindFileInfo` each call `InitFileSystem()` on first use. 5539dd4e8b 2010-06 “Gamelist loading speedup: Be lazy about parsing the file system”

### dolphin-054 — Persist slow compiler output to disk, keyed and versioned
Lens: io · Source: Dolphin
Principle: When an external compiler (shader compiler, OpenCL, regex engine) is slow and deterministic, save its binary output keyed by the input, with a version header. Later runs load the bytes instead of compiling, so the stall happens once per install rather than once per session. Throw the file away on a version mismatch.
Why it is faster: —
Applies elsewhere: Python `.pyc` files, the GPU pipeline caches in browsers and game engines, `ccache`, and persisted V8 code caches.
Watch out: —
Seen in Dolphin: A new `LinearDiskCache` appends key/value records to a file whose header holds an ID and a version. The D3D9 `PixelShaderCache`/`VertexShaderCache` read it back at startup, so the “freeze jerks” from D3D shader compilation happen only the first time. The same idea was applied to OpenCL: compiled kernels go to `User/OpenCL/kernel.bin`, behind a 10-byte header holding the revision string. 3e01152793 2010-01 “Shader Disk Cache implementation for D3D” · 9114c96c78 2010-12 “Cached the compiled OpenCL kernels to disk”

### dolphin-055 — Hash the key once and switch the ordered map to a hash map
Lens: algorithms · Source: Dolphin
Principle: If a lookup table never needs ordered iteration, a hash map answers in O(1) where a tree takes O(log n) multi-word comparisons. Compute the key's hash when the key is built and store it inside the key, so lookups don't rehash. Where a comparator is still needed, compare the first differing word with an early exit rather than calling a generic lexicographic routine.
Why it is faster: —
Applies elsewhere: interned strings that carry a precomputed hash (Java `String.hash`, CPython), and caches of compiled query plans keyed by a fingerprint.
Watch out: —
Seen in Dolphin: `VertexLoaderManager` kept compiled vertex loaders in `std::map<VertexLoaderUID, VertexLoader*>`, compared with `std::lexicographical_compare` over five words. `VertexLoaderUID::InitFromCurrentState` now stores `hashValue = h*137 + vid[i]`, the map is a `hash_map`, and `operator<` is a hand-unrolled compare that exits early. d69478e539 2010-03 “rewrote map to hash_map”

### dolphin-056 — Hoist repeated lookups out of loop conditions and expressions
Lens: algorithms · Source: Dolphin
Principle: A function call in a loop condition, or the same call written twice in one expression, runs every time unless the compiler can prove it has no side effects, and it rarely can. Evaluate it once into a local. That also applies to resolving an object through a manager or singleton on a hot path: fetch the pointer once and keep it.
Why it is faster: —
Applies elsewhere: `for (i = 0; i < strlen(s); i++)`, DOM `querySelector` inside a loop, repeated `map.find()` for the same key.
Watch out: —
Seen in Dolphin: `WII_IPC_HLE::UpdateDevices` called `g_DeviceMap.lower_bound(IPC_FIRST_FILEIO_ID)` on every iteration; that one routine was taking more than 4% of a thread. `GPFifo::CheckGatherPipe` went through `CPluginManager::GetInstance().GetVideo()->Video_GatherPipeBursted` on each burst and now calls a pointer cached in `m_GatherPipeBursted`. In the JitIL allocator, `regLocForInst()`/`fregLocForInst()` results are now held in locals. cff39f5382 2010-02 “Omitted extra lower_bound() calls” · c1ca3ae96c 2010-06 “Optimized FIFO gather a tiny bit.” · 9c98d0ab85 2010-08 “JitIL: Extracted local variables to prevent calling regLocForInst()/fregLocForInst() many times”

### dolphin-057 — Test for the dominant case first in a generic slow path
Lens: branching-fast-paths · Source: Dolphin
Principle: When one specific input reaches a general handler far more often than the others, check for it first with an exact compare and return. The common case then skips the whole chain of range tests and dispatch.
Why it is faster: —
Applies elsewhere: an HTTP router that checks `/health` before the regex table, or a parser that tests ASCII before full Unicode classification.
Watch out: —
Seen in Dolphin: `WriteToHardware<T>` in `MemmapFunctions.cpp` now starts with `if (em_address == 0xCC008000)` and dispatches to `GPFifo::Write8/16/32` before any of the MMIO region decoding. The FIFO write port is “probably the most common reason we end up in this function”. 470db5965e 2010-03 “Minor optimization that's been sitting on my harddrive for a while.”

### dolphin-058 — Filter redundant state uploads with a shadow copy and one memcmp
Lens: gpu-rendering · Source: Dolphin
Principle: Driver calls are expensive. Keep a shadow copy of what was last sent and skip the call when nothing changed. Do the comparison as a single `memcmp` over the whole range and the update as one `memcpy`, not as per-element loops with early-break flags, so it compiles to tight vectorized code.
Why it is faster: —
Applies elsewhere: virtual-DOM diffing, skipping `glUniform` calls when values are unchanged, dirty-checking a config block before writing it to flash.
Watch out: —
Seen in Dolphin: `SetMultiVSConstant4fv` in the DX9 plugin compared and copied `lastVSconstants` component by component in two nested loops. It is now: `if (memcmp(&lastVSconstants[n], f, count*16)) { memcpy(...); D3D::SetVertexShaderConstantF(...); }` 7923903f92 2010-05 “A minor speed up for SetMultiVSConstant4fv()”

### dolphin-059 — Collapse a row-by-row copy into one memcpy when the strides match
Lens: memory-layout · Source: Dolphin
Principle: If the source and destination pitches are equal and no per-pixel conversion is needed, the 2D copy is really one contiguous block. A single large `memcpy` moves it at full memory bandwidth, with no per-row loop overhead.
Why it is faster: —
Applies elsewhere: image libraries that check `stride == width*bpp`, framebuffer blits, and NumPy's contiguous fast path.
Watch out: —
Seen in Dolphin: `CreateTexture2D`/`ReplaceTexture2D` (D3D9, A8R8G8B8) now check `if (pitch*4 == Lock.Pitch && !swap_r_b) memcpy(Lock.pBits, buffer, Lock.Pitch*height);` and fall back to the row loop otherwise. The commit says EFB-to-RAM copies, which upload big textures, get faster. 4f10ded2b6 2010-05 “Slightly optimize uploading D3DFMT_A8R8G8B8 data to textures”

### dolphin-060 — Trigger a transfer once, when its final parameter is written
Lens: io · Source: Dolphin
Principle: When a command is configured through several register or field writes, don't re-check readiness after each write. Act on the write that has to come last, which also lets you drop the bookkeeping flags. The whole transfer then runs as one bulk operation.
Why it is faster: —
Applies elsewhere: committing a builder object on `build()` instead of validating each setter, and kicking a DMA only on the doorbell register write.
Watch out: —
Seen in Dolphin: In `HW/DSP.cpp`, writes to `AR_DMA_CNT_H` and `AR_DMA_CNT_L` each set a `CntValid[]` flag and called `Update_ARAM_DMA()`, which returned early until both were set. Now only the `CNT_L` write (or the 32-bit write) calls `Do_ARAM_DMA()`, the flags are gone, and the DMA runs “less often, and in larger chunks”. 4927e5607e 2010-01 “Do ARAM DMA less often, and in larger chunks.”

### dolphin-061 — Drain a staging buffer by offset instead of shifting it
Lens: memory-layout · Source: Dolphin
Principle: Consuming a buffer in fixed chunks by `memmove`-ing the remainder to the front costs O(n) per chunk. Walk a read offset through it instead and compact once at the end. Resolve the destination pointer once and advance it, rather than translating the address for every chunk.
Why it is faster: —
Applies elsewhere: network receive buffers that parse frames by offset, and stream decoders that keep an index into a `Uint8Array` rather than slicing it.
Watch out: —
Seen in Dolphin: `GPFifo::CheckGatherPipe` copied 32 bytes to `Memory::GetPointer(Fifo_CPUWritePointer)`, then `memmove`d the rest of `m_gatherPipe` down, once per burst. It now gets `curMem` once and loops `memcpy(curMem, m_gatherPipe + cnt, 32)`, advancing `curMem` and wrapping it at `Fifo_CPUEnd`. The same commit fixed an out-of-bounds wrap check. 857f1f4d38 2010-06 “Fixed GPFifo out of bounds by 32 bytes … plus a bit optimization of CheckPipe()”

### dolphin-062 — Use a spinlock for very short critical sections when the platform mutex is heavy
Lens: concurrency · Source: Dolphin
Principle: Some OS mutexes always go through a kernel or semaphore path, even when uncontended. For critical sections that last a few instructions, a user-space spinlock costs one atomic operation to acquire and release. Keep the mutex for anything that can block for long.
Why it is faster: —
Applies elsewhere: per-bucket locks in a concurrent hash table, lock-protected counters in an allocator, ring-buffer indices shared between an audio callback and a producer.
Watch out: —
Seen in Dolphin: On OS X, `Common::CriticalSection` (and the event class) used `pthread_mutex`. The author notes they “are really semaphores” there and switched to `OSSpinLockLock/Try/Unlock`, with a “decent speed up”. Other POSIX platforms kept the mutex. 3103b920b6 2010-04 “[OSX] Use spinlocks instead of mutexes since mutexes are really semaphores”

### dolphin-063 — Compile diagnostic checks out of release builds
Lens: compiler-build · Source: Dolphin
Principle: Error polling and return-code logging that is cheap to write can be expensive to run, especially when it forces a synchronous round trip such as `glGetError`, which can stall a driver's pipeline. Wrap it in a macro that expands to nothing (or a constant) in release builds.
Why it is faster: —
Applies elsewhere: WebGL apps that call `gl.getError()` only in debug builds, `assert` versus `NDEBUG`, and log statements guarded by a level check at compile time.
Watch out: —
Seen in Dolphin: The OpenGL plugin called `glGetError()` directly in hot paths. These calls now go through `GL_REPORT_ERROR()`, which reports only under `_DEBUG`/`DEBUGFAST`. The commit message says this gives a speed-up in the OpenGL plugin. On POSIX, `CriticalSection::Enter/Leave` now check the `pthread_mutex_lock` return value and call `ERROR_LOG` only under `DEBUG`. c76ba17ff4 2010-08 “Stub out glGetError() calls from the release build” · 2a64643fdc 2010-09 “speed up CriticalSection on systems that use POSIX-threads”

### dolphin-064 — Shrink lock scope to the work that actually shares data
Lens: concurrency · Source: Dolphin
Principle: Taking a lock around the whole loop iteration, or on every producer write "just for safety", serializes threads that could run in parallel and adds an atomic operation per call. Lock only around the block that touches shared state, and drop locks that protect nothing.
Why it is faster: —
Applies elsewhere: a server worker that holds a global mutex only while it pops from the queue, not while it handles the request.
Watch out: —
Seen in Dolphin: `CommandProcessor::Write16` and `GatherPipeBursted` wrapped register updates in `FifoCriticalEnter/Leave`, and the GPU thread's `Fifo_EnterLoop` held the lock for the entire loop body. Those producer-side locks are now commented out, and the consumer takes the lock only inside the branch that actually processes FIFO data. b0ef811786 2009-12 “Disabled thread synchronization check so the speed should be back”

### dolphin-065 — Replace per-call reciprocal math with a constant table
Lens: lookup-tables · Source: Dolphin
Principle: A value that depends on a small integer, such as `1.0f / (1 << n)`, costs an int-to-float conversion and a divide each time it is computed. A `static const` table indexed by `n` turns that into one load, and making it constant at compile time also removes the setup loop.
Why it is faster: —
Applies elsewhere: fixed-point to float conversion in audio codecs, and mip-level scale factors in renderers.
Watch out: —
Seen in Dolphin: `VertexLoader::RunVertices` computed `tcScale[i] = 1.0f / float(1 << Frac)` and `posScale` the same way for every batch. The first change filled a `texCoordFrac[32]` table in the constructor. The second replaced it with a compile-time `static const float fractionTable[32]` used for both positions and texture coordinates. 5cf491af64 2010-06 “VideoCommon: A minor speed up.” · 08b999dbfa 2010-09 “VideoCommon: A minor optimization.”

### dolphin-066 — Evict the register whose value is needed furthest in the future
Lens: jit-interpreters · Source: Dolphin
Principle: Round-robin register allocation spills values that are about to be used again. Record each value's last use and, when you need a free register, evict the one whose live range ends latest (or reuse an operand's register the moment its live range ends). This is linear-scan allocation, and it cuts spills, reloads and copies.
Why it is faster: —
Applies elsewhere: Belady-style cache eviction when the access pattern is known in advance, and buffer pools in query engines that use the planned access order.
Watch out: —
Seen in Dolphin: In JitIL's `regFindFreeReg`/`fregFindFreeReg`, `RegAllocOrder[nextReg++ % size]` was replaced by a scan over `RI.lastUsed[]` that picks the register with the greatest end. A later change added `fregURegWithoutMov`/`fregBinLHSRegWithMov`, which recycle op1's XMM register when its life ends at this instruction and remove extra `MOVAPD`s. c9855f2c22 2010-07 “JitIL: Implemented linear scan algorithm for register allocation.” · 6f6acf3970 2010-08 “JitIL: Improved the register usage and remove extra MOVAPD instructions.”

### dolphin-067 — Algebraic simplification and constant folding in the IR builder
Lens: jit-interpreters · Source: Dolphin
Principle: Apply rewrite rules when IR nodes are created: fold constants, merge chained immediates, and replace expensive operations with cheaper equivalent ones. Examples are `x + x → x << 1`, multiplying by a power of two becoming a shift, and `-a + b → b - a`. Every rule that fires removes host instructions from every execution of the block.
Why it is faster: —
Applies elsewhere: SQL optimizers that simplify predicates, shader compilers, the folding that LLVM's InstCombine does, and spreadsheet engines that simplify formulas before evaluating them.
Watch out: —
Seen in Dolphin: `IRBuilder::FoldAdd` gained rules such as `Add x 0 => x`, `Add (Add x i0) i1 => Add x (i0+i1)`, `Add x x => Shl x 1`, factoring of common multipliers, and negation rules. The new `FoldMulHighUnsigned` turns `(x*(1<<i))>>32` into a shift. In Jit64, `regimmop` now folds whenever the source register holds a known immediate, not only when `a == d`. 9f3e2ac2cc 2010-07 “JitIL: Added operand folding rules” · aaa083f8f0 2010-07 “JitIL: Added instruction combining rules” · a3df65bd02 2010-08 “JitIL: Added a new IR instruction, MulHighUnsigned” · 1faff6aeb7 2010-08 “Improved constant folding/propagation in integer instructions”

### dolphin-068 — Fuse common instruction pairs into one host operation
Lens: jit-interpreters · Source: Dolphin
Principle: Some source instruction sets need two instructions for one idea, such as loading a 32-bit constant, or comparing and then branching on the result. Peek at the next instruction while translating. If it completes the idiom, emit a single fused host sequence and skip the next instruction, which avoids materializing the intermediate value or flags.
Why it is faster: —
Applies elsewhere: macro-op fusion in CPUs (cmp+jcc), bytecode superinstructions in interpreters such as CPython and Lua, and SQL operator fusion.
Watch out: —
Seen in Dolphin: In `Jit64::reg_imm`, `lis rD` followed by `addi rD,rD` or `ori rD,rD` now becomes one `gpr.SetImmediate32`, and `js.skipnext` skips the second instruction. `cmpXX` already merged with a following `bcx`; it now also merges with `bcctrx` and `bclrx`. 99ffecd675 2010-07 “Merge JIT instructions when loading 32-bits immediate values” · f09c5ad6c1 2010-07 “Try to merge cmpXX with all types of conditional branches”

### dolphin-069 — Inline short shared helpers into generated code
Lens: jit-interpreters · Source: Dolphin
Principle: Calling a tiny shared routine from generated code costs argument moves into fixed registers, a call/return pair, and pressure on the return predictor. Emitting the routine inline makes the code a little larger, but the operand can stay wherever it already is, so the moves disappear along with the call.
Why it is faster: —
Applies elsewhere: `inline`/LTO for small getters, Duff's-device-style template expansion, and inlined write barriers in garbage-collected runtimes.
Watch out: —
Seen in Dolphin: Every Rc-form integer instruction did `MOV(32, R(EAX), gpr.R(d)); CALL(asm_routines.computeRc)`. The new `Jit64::ComputeRC(const OpArg&)` emits the compare and the three-way `cr_fast[0]` store inline, directly on the operand. The same series removed `GenerateCarry`'s temporary register and redundant `MOV`s to `PC` in branches. JitIL's `WriteExitDestInOpArg` stores the exit PC straight from wherever it lives instead of staging it through `EAX`. c7b4df19d7 2010-07 “Inline the call to computeCR” · cbc66f9467 2010-07 “More performance work on the PowerPC JIT compiler” · d340b9497e 2010-07 “Removed unneeded MOV in branch instructions” · 43dc70eace 2010-08 “JitIL: Omitted some extra MOVs when a next program counter is in a register.”

### dolphin-070 — Write back only dirty cached values
Lens: jit-interpreters · Source: Dolphin
Principle: A write-back cache, whether of registers, pages or rows, should track a dirty bit and skip the store when a value was only read. Flushing every cached entry unconditionally doubles the memory traffic for read-mostly data.
Why it is faster: —
Applies elsewhere: OS page caches, ORMs that track changed fields and emit only the changed columns, and game save systems that rewrite only modified chunks.
Watch out: —
Seen in Dolphin: Jit64's `RegCache::StoreFromRegister` had `// if (doStore) //<-- Breaks JIT compilation`, so it always stored. With `KillImmediate(preg, doLoad, makeDirty)` fixed to set `xregs[...].dirty` correctly, the `if (doStore)` check is back on and flushes write only dirty registers. 9b6180ab27 2010-08 “Fixed JIT register allocator. Now it only writes back dirty registers when flushed”

### dolphin-071 — Snapshot and restore cached state around a side exit
Lens: jit-interpreters · Source: Dolphin
Principle: A rarely taken exit in the middle of a block must leave memory consistent, but flushing permanently forces the hot fall-through path to reload everything. Save the cache's bookkeeping, emit the flush only on the exit path, then restore the bookkeeping so the fall-through code keeps using values that are still in registers.
Why it is faster: —
Applies elsewhere: exception paths in compilers that spill only on the throwing edge, and transactions that write a checkpoint on rollback while the commit path stays in memory.
Watch out: —
Seen in Dolphin: `Jit64::WriteExit`, `WriteExitDestInEAX` and the exception exits now do `gpr.SaveState(s); fpr.SaveState(s2); Flush(FLUSH_ALL); … gpr.LoadState(s); fpr.LoadState(s2);`. The flush code is emitted for the jump to the dispatcher, but the allocator returns to its cached view for the instructions that follow. 5a248b46a1 2010-08 “Fixed the register allocator so that it can restore the registers state after jumping to the dispatcher”

### dolphin-072 — Fold several range checks into one mask test
Lens: branching-fast-paths · Source: Dolphin
Principle: When the address regions you need to catch are marked by specific high bits, one `TEST addr, MASK` plus one branch replaces a chain of compares and jumps. The common case then costs one predictable branch.
Why it is faster: —
Applies elsewhere: pointer tagging in JavaScript engines (one AND to tell a small integer from a pointer), and permission bitmasks checked with a single AND.
Watch out: —
Seen in Dolphin: `SafeLoadRegToEAX`/`SafeWriteRegToReg` emitted `CMP 0xf0000000`, `TEST 0x20000000` and `TEST 0x0C000000`, each with its own branch. They now emit a single `TEST(reg, ADDR_MASK_HW_ACCESS | ADDR_MASK_MEM1)`. `Read_Opcode` and `DoJit` replaced five `(addr >> 28) != X` compares with `addr & ADDR_MASK_MEM1`. fe0f25c737 2010-08 “Optimised the memory bounds check in MMU and MMU speed hack games.” · ea7004ffa7 2010-08 “Optimised the JIT cache lookup in JIT and JITIL.”

### dolphin-073 — Specialize generated code to the configuration at generation time
Lens: jit-interpreters · Source: Dolphin · Also: Flycast
Principle: If a hot routine is generated at runtime, settings that are fixed for the session can be checked once while generating it. Paths for disabled modes are simply never emitted, so the run-time code doesn't test flags that cannot change.
Why it is faster: —
Applies elsewhere: SQL engines that compile a query to machine code after the schema is known, regex engines that build an engine for the given flags, and shader permutations selected at load time.
Watch out: —
Seen in Dolphin: The dispatcher in `JitAsm.cpp` always tested `JIT_ICACHE_VMEM_BIT` and `JIT_ICACHE_EXRAM_BIT` before indexing the instruction cache. It now builds `mask` from `bWii`, `bMMU` and `iTLBHack` while generating the code. For a GameCube game without MMU it emits only `AND EAX, JIT_ICACHE_MASK; MOV EAX, [icache+EAX]`. The commit reports a speed-up of under 5% in GameCube games. ea7004ffa7 2010-08 “Optimised the JIT cache lookup in JIT and JITIL.” · eaa3371f92 2010-08 “Further optimised the JIT cache lookup.” • Also seen in Flycast (flycast-102): `rec_arm64.cpp` replaced its inline-asm `ngen_mainloop` with a `generate_mainloop()` that emits different code for MMU and non-MMU, plus separate `ngen_FailedToFindBlock_mmu`/`_nommu` stubs. x64 followed with `genMainloop()`, which removed `msvc.asm`. `CheckBlock` now loads the block address only `if (mmu_enabled() || force_checks)`. cc4b59f5f 2019-05 “rec-arm64: generate main loop dynamically” · 594241ffa 2021-01 · e773880e6 2021-01 “x64 rec: save one op in CheckBlock”

### dolphin-074 — Link compiled blocks directly and use stubs for missing ones
Lens: jit-interpreters · Source: Dolphin
Principle: Returning to a central dispatcher after every block costs a lookup and an indirect jump. When the jump target is a constant and that block is already compiled, jump straight to it. Fill unfilled table slots with a stub that compiles on demand, so callers never need a null check. Retry linking later for targets that were not compiled yet.
Why it is faster: —
Applies elsewhere: PLT/GOT lazy binding in dynamic linkers (unresolved entries point at a resolver stub), and inline caches in JavaScript VMs.
Watch out: —
Seen in Dolphin: DSP LLE JIT: `r_jcc`/`r_call` emit `JMPptr(M(&blockLinks[dest]))` when the destination is already compiled, which the commit measures at about 7–10%. A follow-up flags blocks with unresolved calls using `CODE_CALL` and recompiles them as more blocks appear. Separately, `blocks[i]` now starts as `stubEntryPoint`, which calls `CompileCurrent`, so the ASM dispatcher no longer checks whether a block exists. c584bd5333 2010-12 “LLE JIT: Added simple block linking” · a90d0e8985 2010-12 “LLE JIT: Used a reiterative approach to linking blocks.” · c79f41d114 2010-11 “LLE JIT: Speed up the dispatcher by removing the compiled block check.”

### dolphin-075 — Move per-execution decisions from the dispatcher to compile time
Lens: jit-interpreters · Source: Dolphin
Principle: If a property of a block can be known when it is compiled, such as "this block is an idle loop", bake its effect into the block's code. The dispatcher, which runs between every pair of blocks, then stays minimal. Drop dispatcher checks that only guard rare edge cases when a small overshoot is acceptable.
Why it is faster: —
Applies elsewhere: an event loop that removes per-iteration feature-flag checks by installing different handlers at startup, and a request pipeline that resolves middleware at configuration time.
Watch out: —
Seen in Dolphin: The new ASM DSP dispatcher looked up `code_flags[pc] & CODE_IDLE_SKIP` on every block. `DSPEmitter::Compile` now emits `MOV EAX, DSP_IDLE_SKIP_CYCLES` as the return value for idle blocks, and the lookup is gone. The `CMP ESI, EAX` “enough cycles?” check was also removed, at the cost of slightly over-running the cycle budget. 6a6fb9cce9 2010-11 “LLE JIT: Added an ASM dispatcher.” · 9c1fb241c7 2010-11 “Speed up the idle skip detection by moving the logic into the block” · 93a901dd1d 2010-11 “Speed up the dispatcher by removing one of the cycle checks.”

### dolphin-076 — Grow translation units past branches and calls
Lens: jit-interpreters · Source: Dolphin
Principle: Short compiled regions hand control back to the dispatcher often and give the optimizer little to work with. Keep compiling past unconditional jumps, into called functions and back through their returns, and past conditional branches, using an inline exit check on the taken side. Larger regions give register caching and folding more room, and the dispatcher runs less often.
Why it is faster: —
Applies elsewhere: trace-based JITs such as LuaJIT and TraceMonkey, and profile-guided function inlining in AOT compilers.
Watch out: —
Seen in Dolphin: `PPCAnalyst::Flatten` now follows `bx` into its destination, governed by `FUNCTION_FOLLOWING_THRESHOLD`, and the code buffer grew to 32 MB. It also tracks `returnAddress`, so an unconditional `bclrx` resumes in the caller and a small function is inlined; `mtspr LR` cancels this. In the DSP JIT, a conditional branch no longer ends the block: the code compares `g_dsp.pc` and returns only if the branch was taken. ccb96be9b3 2010-09 “Jit64/JitIL: Enabled block merging” · 5e806eb7b2 2010-09 “Implemented simple function inlining for speedup.” · 6d2c35a332 2010-10 “Jit: only end emitting the current block when hitting an unconditional branch”

### dolphin-077 — Materialize derived state only for consumers that read it
Lens: laziness · Source: Dolphin · Also: ares
Principle: Keeping a mirrored variable up to date after every step, such as a program counter or a cursor, costs a store each time even though few readers look at it. Track it at compile time instead, and write the real value only before operations marked as reading it, such as slow-path fallbacks and exits.
Why it is faster: —
Applies elsewhere: lazy line and column tracking in parsers (computed from the offset when an error is reported), and deferred layout in UI frameworks.
Watch out: —
Seen in Dolphin: The DSP JIT emitted `ADD [g_dsp.pc], 1` after every instruction. It now tracks `compilePC`. `MainOpFallback` stores `compilePC+1` only when `opTable[inst]->reads_pc`, and `checkExceptions` writes the PC only at exit points. f67afb4fca 2010-12 “LLE JIT: Optimised the updating of g_dsp.pc … Marked instructions that update g_dsp.pc in the DSP table” • Also seen in ares (ares-124): `setupPipeline()` is idempotent (`emitPipelineSetupDone`) and is called only for `info.branch()`, for a needed commit, or before a slow path. It writes JIT-time immediates when the PC is known. Callf setup in the CPU JIT also moved to the paths that actually emit a helper call. bb75c2f05 2026-05 “n64: in CPU JIT, do not emit pipeline prologue for linear instructions” · d91f838e3 2026-04 “n64: lazily emit callf setup in CPU JIT”

### dolphin-078 — Use a fixed-capacity stack array instead of a growable vector on hot paths
Lens: allocation · Source: Dolphin
Principle: If an upper bound on the size is known, a local array with a count avoids heap allocation, the capacity check on each push and the destructor. It also stays in cache-hot stack memory. Assert on overflow instead of growing.
Why it is faster: —
Applies elsewhere: `SmallVector`/`absl::InlinedVector`, a fixed-size scratch buffer in an interrupt handler, and Go's escape-analysis-friendly arrays.
Watch out: —
Seen in Dolphin: `Jit64::DoJit` and `JitIL::DoJit` created a `std::vector<u32> merged_addresses` for every block compile. They now use `u32 merged_addresses[32]` with `capacity`/`size` parameters passed to `PPCAnalyst::Flatten`, which panics if the capacity is below `FUNCTION_FOLLOWING_THRESHOLD`. 97a9c375f9 2010-09 “JitIL/Jit64: Removed std::vector for speed up.”

### dolphin-079 — Make sure one-time initialization really runs only once
Lens: other · Source: Dolphin
Principle: A "build the table if it is empty" guard that tests a table entry can fail silently when that entry is legitimately zero after the build, so the table is rebuilt on every call. Use an explicit `initialized` flag, or better a static initializer or a constant table.
Why it is faster: —
Applies elsewhere: memoization keyed on a falsy result (`if (!cache[k])` when the value can be 0), and lazy singletons whose sentinel collides with a valid state.
Watch out: —
Seen in Dolphin: `IRBuilder::getNumberOfOperands` guarded its setup with `if (numberOfOperands[0] == 0)`. Opcode 0 has zero operands, so the whole table was rebuilt on every call. It now uses `static bool initialized`, which noticeably speeds up JitIL compilation. 6dc16b7416 2010-09 “JitIL: Improved the compilation speed by fixing the bug that getNumberOfOperands() initializes the table each time it is called.”

### dolphin-080 — Precompute per-address analysis flags and test them in the hot loop
Lens: lookup-tables · Source: Dolphin
Principle: Facts about static code or data, such as "a loop ends here", "this is an idle loop" or "this uses the FPU", can be computed once in an analysis pass and stored as flags. The hot loop then does a single bit test, or keeps the decoded info alongside the item, instead of calling a handler or re-decoding every step.
Why it is faster: —
Applies elsewhere: precomputed "has side effects" bits in bytecode, and a CSS engine that caches selector-match hints per node.
Watch out: —
Seen in Dolphin: The DSP interpreter's `Step()` called `HandleLoop()` after every instruction. It now calls it only when `DSPAnalyzer::code_flags[pc-1] & CODE_LOOP_END`. In the PowerPC JIT, `PPCAnalyst::Flatten` stores `code[i].opinfo` once, and `CompileInstruction(CodeOp&)`/`CanSwapAdjacentOps` read it instead of calling `GetOpInfo` again. FPU use comes from the new `FL_USE_FPU` table flag rather than a separate `UsesFPU()` decode. 6c8fc66698 2010-10 “small HandleLoop() optimization for dsplle-int” · f8ce075ab1 2010-07 “Commit patch submitted in issue 2951 (PowerPC JIT optimizations)”

### dolphin-081 — Detect changes with a sampled, cheap hash
Lens: algorithms · Source: Dolphin
Principle: To decide whether a large buffer changed, you don't always need to hash every byte. Hash a fixed number of evenly spaced words with a light multiply/xor mix, and expose the sample count as a speed/accuracy setting. That makes the check O(samples) instead of O(size). Keep full hashing for small, high-risk data such as palettes.
Why it is faster: —
Applies elsewhere: rsync's rolling checks, file-dedupe tools that hash the first, middle and last blocks before a full hash, and video encoders' sampled scene-change detectors.
Watch out: —
Seen in Dolphin: The safe texture cache adopted a MurmurHash64A-style `TexDecoder_GetFullHash` and fully hashed TLUT palettes. A test commit then dropped some of the multiplies in the mix to make it cheaper. Configuration was reduced to `SafeTextureCacheColorSamples` (e.g. 37 samples, or 0 meaning all) plus a TLUT size cap. `GetHash64(src, len, samples)` in `Hash.cpp` sets `Step = (len/8)/samples` and skips words in between. e93e777ffb 2010-02 “implemented full hashing of tlut textures” · 6200c99dd4 2010-02 “please test if this improve the performance with the safe texture cache” · 02189ec5e0 2010-02 “Simplify the params for the safe texture cache” · 9b0357b5e2 2010-08 “use plain vertex arrays instead of VBOs”

### dolphin-082 — Skip regenerating output whose content hash is already cached
Lens: algorithms · Source: Dolphin
Principle: Before decoding or uploading something expensive, hash the source and check whether the cache already holds an entry with that hash at that key. Most frames reproduce the same data, and those frames become a hash and a lookup. Use bounded range queries (`lower_bound`/`upper_bound`) on an ordered index to invalidate only overlapping entries rather than scanning all of them.
Why it is faster: —
Applies elsewhere: HTTP ETags, build systems keyed by content hash (Bazel, Nix), and React's memoization by props identity.
Watch out: —
Seen in Dolphin: `EncodeToRamFromTexture` (D3D9/OGL) now computes `GetHash64(dest_ptr, …)` and returns early if `TextureCache::Find(address, hash)` matches. The commit reports a 35% speed-up in New Super Mario Bros. Wii. `TextureCache::InvalidateRange` now iterates from `lower_bound(start)` to `upper_bound(start+size)` instead of from `begin()` to `end()`. A follow-up moved the check so the destination texture isn't invalidated, and it was later put behind an option, `bEFBCopyCacheEnable`. 8775abacc6 2010-11 “Optimised EFB copy to RAM.” · 6666b400a3 2010-11 “Updated with a less aggressive optimisation to EFB copy to RAM.” · 60ed57b32a 2010-12 “Add an option to toggle caching EFB->RAM copies.”

### dolphin-083 — Record a command stream once, replay the compiled result
Lens: caching · Source: Dolphin
Principle: If the same command list is submitted over and over, interpret it once and record the expensive outputs, such as decoded vertex data and register writes. Emit a compact replay routine keyed by address, size and a content hash. Later submissions call the replay routine and skip parsing and decoding entirely.
Why it is faster: —
Applies elsewhere: OpenGL display lists and Vulkan secondary command buffers, prepared SQL statements, and template engines that compile a template to a function.
Watch out: —
Seen in Dolphin: `DLCache.cpp` runs a GameCube display list in an analysis pass (`AnalyzeAndRunDisplayList`), then compiles it with `CompileAndRunDisplayList`. XF register writes are copied into owned buffers (`dl->Vdata`), vertex data is included, and native code replays it through `ABI_CallFunctionCCCP`. A second step hashes referenced array data, so a list is reused only if its inputs are unchanged. 3e7dafbbd7 2010-08 “fully implemented display list cache with vertex data included” · 73adab0ce1 2010-09 “Dlist Cache step 2: include referenced array data in the hash”

### dolphin-084 — Match the upload mechanism to how often data is reused
Lens: gpu-rendering · Source: Dolphin
Principle: GPU buffer objects pay for allocation, mapping and synchronization, which is worth it only when the data is drawn more than once. For data that is rewritten every draw, a client-side array the driver copies once can be cheaper. For many small writes from utility code, one large dynamic buffer used as a ring is cheaper: append with "no overwrite" mapping and discard only on wrap, which avoids a stall per write.
Why it is faster: —
Applies elsewhere: streaming vertex rings in game engines, `GL_MAP_UNSYNCHRONIZED_BIT` with fences, and the per-frame uniform ring buffers used with WebGPU.
Watch out: —
Seen in Dolphin: The OpenGL `VertexManager` rotated through `s_vboBuffers` with `glBufferData(GL_STREAM_DRAW)` per flush. It now draws straight from `LocalVBuffer` as a plain vertex array, “as the nature of the data make VBOs slower”. In DX11, the `drawShadedTexQuad`/clear helpers now share a `UtilVertexBuffer` that maps with `D3D11_MAP_WRITE_NO_OVERWRITE` and falls back to `WRITE_DISCARD` only when `offset + size >= max_size`. 9b0357b5e2 2010-08 “use plain vertex arrays instead of VBOs to render in Opengl plugin” · f9187ab508 2010-10 “DX11: Use a ring buffer in the utility functions for speedup.”

### dolphin-085 — Produce two outputs in one pass instead of drawing twice
Lens: gpu-rendering · Source: Dolphin
Principle: If a second pass over the same geometry exists only to write a different value into one channel, ask the hardware to emit both values from one shader invocation. That halves vertex processing and state changes for the affected draws.
Why it is faster: —
Applies elsewhere: multiple render targets in deferred shading, and fused map-reduce kernels that write a result and a statistic in one sweep.
Watch out: —
Seen in Dolphin: For destination alpha, the DX11 plugin drew geometry twice. `EmuGfxState::AlphaPass` rebuilt a blend state with an alpha-only write mask and drew again. With dual-source blending, the pixel shader now writes `ocol1 = ocol0` for blending and `ocol0.a = alpha constant` for the stored alpha. `AlphaPass` is gone, and Super Mario Galaxy, which uses destination alpha heavily, runs faster. cb453a0fb3 2010-10 “DX11 plugin: Do destination-alpha in a single pass.”

### dolphin-086 — Specialize generated programs for known constant inputs
Lens: jit-interpreters · Source: Dolphin
Principle: When you generate source code such as shaders or SQL from a configuration, check for degenerate cases while generating. A lerp whose two endpoints are equal, a factor of 0 or 1, or a test that always fails should produce the simplified expression, or an empty program, rather than relying on the downstream compiler. Emit expensive correction code only when the state that needs it is present.
Why it is faster: —
Applies elsewhere: ORMs that drop `WHERE 1=1` clauses, template engines that remove dead conditionals, and code generators that specialize on feature flags.
Watch out: —
Seen in Dolphin: `PixelShaderGen`'s `WriteStage` emits `tevCInputTable[a]` directly when `cc.a == cc.b` or `c` is ZERO, and `[b]` when `c` is ONE, instead of a `lerp`. `AlphaPreTest()` detects an alpha test that always fails and emits a shader that only discards, or one that only writes alpha when the test always passes. `RegisterStates[]` tracks which TEV registers can overflow, so the wrap-around math is generated only for those registers. b73b77014f 2010-05 “Rework the logic for avoiding lerp() in shader code” · d511b50612 2010-06 “Optimize pixelshadergen to avoid redundant register overflow math” · 7797b9d753 2010-10 “a little bugfix and a small optimization”

### dolphin-087 — Byte-swap and widen in SIMD registers, dispatching on CPU features
Lens: simd · Source: Dolphin
Principle: Endian conversion of a buffer is a fixed byte permutation, so one `pshufb` swaps 16 bytes at once. Use 64-bit or 128-bit loads and stores instead of pairs of 32-bit ones. Use non-temporal stores for output that won't be read back soon. Pick the SIMD or scalar routine at startup from CPUID, through a function pointer or a flag, so one binary runs everywhere.
Why it is faster: —
Applies elsewhere: network stacks converting big-endian packet fields, image codecs (libjpeg-turbo, simdjson's runtime dispatch), and audio sample-format conversion.
Watch out: —
Seen in Dolphin: `TextureDecoder.cpp` added SSSE3 paths that use `_mm_shuffle_epi8(a, kMaskSwap16/32)` and `_mm_stream_si128`, plus SSE4.1 `_mm_stream_load_si128`. `VertexLoader_*` picks between SSSE3 and scalar readers through function pointers, after `cpu_info` detection. JitIL's `LoadDouble` became `MOVQ` + `PSHUFB` when `cpu_info.bSSSE3`, replacing two 32-bit loads and a `PUNPCKLDQ`. `StoreDouble` gained a 64-bit `PSHUFB`+`MOVQ` fast path, guarded by one address-mask test. 6136c94de5 2010-04 “VideoCommon: merged SSSE3/SSE4.1 codes.” · 956b8eb54d 2010-04 “Added automatic selection routines for SSSE3/SSE4.1 codes.” · 014e87157c 2010-08 “JitIL: Modified "LoadDouble" with SSSE3” · 402b1d83e2 2010-09 “JitIL: Implemented 64-bit access in StoreDouble for speed up.”

### dolphin-088 — Expand bytes to wider pixels with SIMD broadcast-and-mask
Lens: simd · Source: Dolphin
Principle: Turning one intensity byte into a 4-byte RGBA pixel is a broadcast. Splat each source byte across a lane with `_mm_set1_epi8`, AND with a lane-selection mask, OR four of them together, and do an aligned store. That writes four pixels per instruction group instead of one store or `memset` per pixel.
Why it is faster: —
Applies elsewhere: converting grayscale to RGBA for canvas `ImageData`, palette expansion in image decoders, and LED-strip pixel packing on SIMD-capable MCUs.
Watch out: —
Seen in Dolphin: The `GX_TF_I8` case of `TexDecoder_Decode_RGBA` replaced its per-texel `memset` loop with SSE2: `_mm_or_si128(_mm_and_si128(_mm_set1_epi8(src[0]), lane0mask), …)` stored with `_mm_store_si128(quaddst, m0)`. The author reports 6–7 more FPS in Mario Kart Wii's intro movie. b038df64bf 2010-12 “TextureDecoder.cpp: new SSE2 optimized GX_TF_I8 decoder.”

### dolphin-089 — Template-unroll small variable-count loops behind a function table
Lens: compiler-build · Source: Dolphin
Principle: A loop whose count is small and varies at run time (1 to 16) pays for loop control and per-element calls. Instantiate a template for each count, so every version is a straight-line `memcpy` plus N fixed operations, and index a function-pointer table by count. Each call does one indirect jump and then branch-free code.
Why it is faster: —
Applies elsewhere: specialized `memcpy` for small sizes in libc, protobuf varint decoders that dispatch on length, and fixed-size matrix kernels generated per dimension.
Watch out: —
Seen in Dolphin: XF register loads did `for (i<transfer_size) data_buffer[i] = DataReadU32();` in `OpcodeDecoding.cpp` and `DLCache.cpp`. The new `DataReadU32xN<N>` does `memcpy` followed by `if (N>=k) buf[k-1] = swap32(buf[k-1])` for 16 values, and callers use `DataReadU32xFuncs[transfer_size-1](data_buffer)`. b038df64bf 2010-12 “OpcodeDecoding.cpp and DLCache.cpp: optimization”

### dolphin-090 — Force-inline tiny stream readers used from templates
Lens: compiler-build · Source: Dolphin
Principle: The compiler's inlining heuristics can give up on small helpers once they are reached through layers of templates, which leaves a real call for every byte read. For one-line accessors on the hottest path, force inlining with `__forceinline` or `always_inline`.
Why it is faster: —
Applies elsewhere: binary deserializers' `readU16()`, iterator `operator*` in container libraries, and the bit-reader in a compression codec.
Watch out: —
Seen in Dolphin: `DataReader.h`'s `DataSkip`, `DataPeek8/16/32`, `DataReadU8/U16/U32` and `DataRead<T>` changed from `inline` to `__forceinline`, because GCC did not always inline them when they were called from other templates. `Pos_ReadDirect` was also explicitly instantiated, since only JIT code calls it. faf586e8f1 2010-07 “force DataRead* inline as gcc didn't always do that itself”

### dolphin-091 — Decode directly into the format the consumer wants
Lens: memory-layout · Source: Dolphin
Principle: Decoding to an intermediate format and then converting to what the API wants costs an extra pass and an extra buffer. Add a decoder path that writes the final layout, here RGBA8 with the channel order expected, so the data is touched once and can be uploaded as is.
Why it is faster: —
Applies elsewhere: image decoders with an output-color-space parameter (libpng transforms, `createImageBitmap` options), and JSON parsers that build target structs directly instead of a DOM.
Watch out: —
Seen in Dolphin: `TexDecoder_Decode_RGBA` was added, with `decode5A3RGBA`, `decode565RGBA` and `decodeIA8Swapped` helpers that emit `a<<24 | b<<16 | g<<8 | r`. The DX11 `D3DTexture` code then replaced its `texbuf` staging and per-format expansion with `ReplaceRGBATexture2D`. 8c6ae1f6f4 2010-06 “add a path to texture decoder to produce only rgba textures” · 208c33f09c 2010-06 “Use the new texture decoder path to speed up texture loading in DX11.”

### dolphin-092 — Recognize busy-wait loops and fast-forward through them
Lens: other · Source: Dolphin
Principle: Code that spins on a status register burns host time without changing anything. Recognize these loops, by matching instruction signatures or by spotting a branch to self, and charge a large block of virtual time in one step so the event that ends the wait arrives sooner.
Why it is faster: —
Applies elsewhere: `pause` and back-off in spin loops, simulators that skip idle cycles, and test harnesses with fake clocks that jump to the next timer.
Watch out: —
Seen in Dolphin: `DSPAnalyzer` matches `idle_skip_sigs`. Two new signatures (`lrs $AC0.M,@DMBH; andf #0x8000; jlnz` and the `$AC1` variant) recognize mailbox polling, and matching code is flagged `CODE_IDLE_SKIP`, which consumes `DSP_IDLE_SKIP_CYCLES` at once. In the PowerPC JIT, `bx` to itself adds to `js.downcountAmount` to “make idle loops go faster”. 11a215567b 2010-03 “DSPLLE - idleskip-ing improved a little” · ccb96be9b3 2010-09 “Jit64/JitIL: Enabled block merging”

### dolphin-093 — Small set-associative cache in front of a slow translation
Lens: memory-layout · Source: Dolphin
Principle: Address translation that walks page tables or hash tables is expensive, but accesses cluster on a few pages. A tiny 2-way set-associative cache indexed by page number, with a most-recently-used bit for replacement, answers most lookups with one tag compare.
Why it is faster: —
Applies elsewhere: DNS resolver caches, a memoized URL router, and a software TLB in a database's buffer manager.
Watch out: —
Seen in Dolphin: The MMU work added `tlb_entry tlb[NUM_TLBS][TLB_SIZE/TLB_WAYS][TLB_WAYS]`, with separate data and opcode sets, 128 entries and 2 ways. `LookupTLBPageAddress` checks both ways for `tag == (vpa & ~0xfff)`, updates `TLB_FLAG_MOST_RECENT`, and returns `paddr | (vpa & 0xfff)`. A miss falls back to the page-table walk. 9ff5e836eb 2010-07 “Implemented MMU Demand Paging … Added fast TLB cache implementation”

### dolphin-094 — Put costly accuracy behind per-workload switches
Lens: other · Source: Dolphin
Principle: When a faithful but slow code path matters to only a few inputs, make it opt-in per item, with the fast approximation as the default. Most users get the speed, and the few that need accuracy turn it on. The hot path's check becomes one boolean.
Why it is faster: —
Applies elsewhere: per-route strict validation in an API gateway, per-table synchronous replication, and per-asset quality settings in a game.
Watch out: —
Seen in Dolphin: Block Address Translation in the MMU was put behind a per-game "Enable BAT" option, with `tlb_addr = bMMUBAT ? TranslateBlockAddress(...) : 0`. Loading native mipmaps became the `bUseNativeMips` option, after it was found to cost speed in games that don't need it. A game-specific hack skips `FlushPipeline()` on minimap BP writes in Zelda: Twilight Princess, which removes the Hyrule Field slowdown. c710ae7529 2010-09 “Added a speed hack for the MMU games.” · 0ea01d8462 2010-05 “make native mips loading an option to prevent performance lost” · 040a6e1eb3 2010-08 “This commit addresses the Hyrule field slowdown issue”

### dolphin-095 — Let the real-time callback pull data directly
Lens: platform · Source: Dolphin
Principle: An extra thread that fills a buffer the audio callback then copies adds a hop, a copy and latency. In a pull-model API, render straight into the buffer the callback is given, which removes the intermediate thread, buffer and polling.
Why it is faster: —
Applies elsewhere: Web Audio `AudioWorklet`s that generate samples in `process()`, and the I2S/DMA half-complete interrupts in embedded audio that fill the next half buffer.
Watch out: —
Seen in Dolphin: The Core Audio backend ran a `SoundLoop` thread that filled `realtimeBuffer[1024]`, and the AudioUnit callback `memcpy`'d from it. The callback now calls `CoreAudioSound::RenderSamples(ioData->mBuffers[i].mData, size)`, which runs `m_mixer->Mix()` into the output buffer. fdd532356a 2010-05 “Have the AudioUnit callback drive the internal mixer directly instead of going through an intermediate thread and buffer.”

### dolphin-096 — Pin heavy threads to distinct physical cores
Lens: platform · Source: Dolphin
Principle: On SMT CPUs, logical cores 2n and 2n+1 often share one physical core's execution units. Pinning two busy threads to siblings makes them compete. When pinning, choose logical CPU indices that map to different physical cores.
Why it is faster: —
Applies elsewhere: DPDK and low-latency trading setups that isolate one thread per physical core, and game engines placing render and simulation threads on separate cores.
Watch out: —
Seen in Dolphin: With “Lock threads to cores” enabled, `EmuThread` pinned the video thread to affinity 2. On HyperThreaded CPUs that is the CPU thread's sibling. It now uses affinity 3 when `cpu_info.num_cores > 3` (“third, non-HT core”). 7c0e08b0e7 2010-12 “Forced the video thread to the third core when "Lock threads to cores" is enabled.”

### dolphin-097 — Widen packed data with unpack instructions, not per-element broadcasts
Lens: simd · Source: Dolphin
Principle: To expand narrow elements into wider ones (for example 8-bit intensity into 32-bit RGBA), load a whole run of source bytes into one vector register and interleave it with itself using unpack. Each unpack doubles the width of every element in one instruction. Broadcasting each element separately and masking it into place costs several instructions per element.
Why it is faster: —
Applies elsewhere: grayscale-to-RGBA conversion in image libraries, widening 16-bit PCM to 32-bit in audio mixers, decoding palette or bit-packed formats in codecs.
Watch out: —
Seen in Dolphin: `TexDecoder_Decode_RGBA` in `VideoCommon/Src/TextureDecoder.cpp` decoded `GX_TF_I8` by doing `_mm_set1_epi8(src[i])` for each byte, then AND and OR into position. It now does `_mm_loadl_epi64` of 8 texels, then `_mm_unpacklo_epi8(r0,r0)` followed by `unpacklo/unpackhi` again, which gives four 32-bit pixels per register. The commit reports about 80% faster. RGB565 got the same SSE2 treatment: unpack, then shift and mask all four channels in parallel (about 78%). The S3TC/CMPR decoder got faster when it stopped forcing per-pixel colors back into a vector, extracted the 2-bit selectors to scalars and wrote through plain pointers. 343e3f7c75 2010-12 “~80% speed improvement in decoding GX_TF_I8 textures” · 4b15325acd 2011-01 “GX_TF_RGB565 texture decoder optimized with SSE2” · 2841d67ce3 2011-01 “Faster SSE2 optimized GX_TF_CMPR texture decoder”

### dolphin-098 — Use one byte-shuffle for any fixed byte permutation
Lens: simd · Source: Dolphin
Principle: Channel swaps (RGBA to BGRA) and endianness conversion are fixed permutations of bytes. A byte-shuffle instruction such as SSSE3 `pshufb` applies an arbitrary 16-byte permutation from a constant mask in one operation. Building the same permutation from shifts, ANDs and ORs takes many operations, and a scalar loop takes one operation per element.
Why it is faster: —
Applies elsewhere: byte-swapping network or file-format integers in bulk, converting pixel formats before a GPU upload, reordering struct-of-bytes data in parsers.
Watch out: —
Seen in Dolphin: D3D9 `CreateTexture2D`/`ReplaceTexture2D` (`D3DTexture.cpp`) swapped R and B one byte at a time. A first SSE2 version used mask and shift (45% faster). `ConvertRGBA_BGRA_SSSE3` then reduced the inner loop to `_mm_storeu_si128(dst, _mm_shuffle_epi8(_mm_loadu_si128(src), mask))`, which the commit puts at about 93% over C. The same trick was used for big-endian command-stream reads (`DataReadU32xN_SSSE3` in `DataReader.h`, with a fall-through `switch(N)` so each count is straight-line code) and for DSP DMA `swap16` plus copy in `DSPHWInterface.cpp`. b6a1e9376f 2011-01 “45% speed improvement in RGBA -> BGRA conversion” · c4477f68e1 2011-01 “~93% speed improvement … for processors that support SSSE3” · f9e4e73e42 2011-01 “Use SSSE3 shuffle for DataReader's DataReadU32xN” · 559fb7434e 2011-01 “Used SSSE3 instructions to swap16 and memcpy the DSP DMA transfers”

### dolphin-099 — Dispatch on CPU features once, outside the loop
Lens: branching-fast-paths · Source: Dolphin
Principle: Checking a capability flag inside a per-pixel or per-vertex loop adds a load and a branch to every iteration, and the branch can stop the compiler from keeping constants in registers. Check once and pick a whole loop. Better still, fill a function-pointer table at startup with specialized variants, generated with templates where possible. The hot path then makes one indirect call with no feature tests at all. The table has to be filled completely: a missed slot silently keeps the slow path.
Why it is faster: —
Applies elsewhere: picking AVX2, SSE or NEON kernels for memcpy, JSON parsing or compression at program start, and resolving variants through GNU `ifunc` or a strategy object.
Watch out: —
Seen in Dolphin: The SSSE3 texture decoders tested `if (cpu_info.bSSSE3)` inside the innermost `for`, and their shuffle masks were rebuilt there too. The fix moved the test above the loop nest and hoisted the masks; the commit reports the change “speeds up most textures dramatically (where it has previously slowed them)”. Vertex loading uses dispatch tables (`DataReadU32xFuncs`, `tableReadTexCoord`) that are swapped for SSSE3 or SSE4 versions at init. A template on index width (`TexCoord_ReadIndex_Float2_SSSE3<u8/u16>`) now covers 8-bit indices as well as 16-bit. A later fix found that only the first table entry had been replaced (`*DataReadU32xFuncs = *DataReadU32xFuncs_SSSE3`) and changed it to copy all 16. 5c725262ba 2011-01 “Refactor all the SSSE3 functions in TextureDecoder so that the cpu_info check isn't looped over” · 7987d6babb 2013-02 “Use templates for 8+16 bit indexed versions of SSSE3/SSE4 TextCoord reading” · 8597660855 2012-08 “Replace all of the opcode data read functions by their SSSE3 equivalent”

### dolphin-100 — Hash with the hardware CRC instruction
Lens: simd · Source: Dolphin
Principle: If a hash is only needed to detect changes, not to resist attacks, the CPU's CRC32 instruction can mix 8 bytes per instruction. A multiply-xor-shift software hash spends several dependent multiplies on each word. Keep the software hash as a fallback for CPUs without the instruction.
Why it is faster: —
Applies elsewhere: content hashing for asset caches, dedup of chunks in backup tools, checksum of packets in storage engines (as RocksDB and many databases do with CRC32C).
Watch out: —
Seen in Dolphin: `GetHash64` in `Common/Src/Hash.cpp` used a MurmurHash-style loop (`k *= m; k ^= k >> r; …`). A new `GetCRC32` does `h = _mm_crc32_u64(h, data[0])` per sampled word, and `GetHash64` calls it when `cpu_info.bSSE4_2` is set. The texture cache, EFB-to-RAM and texture IDs all use it. The commit reports about 2x faster hashing. d698e022f5 2011-02 “Changed the hash algorithm to CRC32 utilising the SSE4.2 instruction”

### dolphin-101 — Revalidate a cache only after events that could change its source
Lens: laziness · Source: Dolphin
Principle: Re-hashing cached data on every use is expensive. Find the points where the underlying data could actually change, and bump a generation counter there. On use, compare the entry's stored generation with the current one, and do the expensive check only when they differ.
Why it is faster: —
Applies elsewhere: invalidating memoized layout only after DOM mutations, an ORM identity map that refreshes only after a transaction boundary, re-statting a config file only after an inotify event.
Watch out: —
Seen in Dolphin: The display-list cache (`DLCache.cpp`) re-hashed list memory with `GetHash64` “every N draws”. Now `IncrementCheckContextId()` is called from `CommandProcessor.cpp` and `PixelEngine.cpp`, where the emulated CPU might have rewritten the list (after it is processed, at frame finish, and on FIFO breakpoints). The hash and region checks run only when the context ID has changed. 00c35451f6 2011-02 “second step in dlist optimizations: test the hashes only when the cpu could change the contents of the dlist”

### dolphin-102 — Block on an event instead of spinning, and choose yields by measurement
Lens: concurrency · Source: Dolphin
Principle: A thread that loops on a shared variable waiting for another thread burns a whole core and competes with the thread it is waiting for. A pair of events, one in each direction, lets the waiter sleep in the kernel and wake exactly when signalled. Where polling can't be avoided, time the real workload before choosing a yield primitive: `pause` keeps the core busy, `usleep(1)` can oversleep, and `sleep(0)` hands off only when another thread is runnable. Yield only while the peer is active, and sleep once it goes idle.
Why it is faster: —
Applies elsewhere: producer/consumer queues using condition variables instead of polling, worker pools parked on a semaphore, adaptive spin-then-park mutexes.
Watch out: —
Seen in Dolphin: When DSP LLE ran on its own thread, `DSPLLE.cpp` had the CPU side spin with `while (m_cycle_count != 0) ;` and the DSP side call `YieldCPU()`. These became `ppcEvent`/`dspEvent` (`Common::Event`): each side does `Set()` on the other's event and `Wait()`s on its own. The commit reports about a third less CPU usage. Where polling remains, the choice of yield was made by measurement. `Common::YieldCPU` (`Thread.h`) went from `_mm_pause()` to `usleep(1)`, which cut idle CPU in the video thread but made DSP-on-thread “painfully slow”, and then to `sleep(0)`, which time trials showed to be fastest. On Windows `Sleep(0)` became `SwitchToThread()`. The FIFO wait loop in `Fifo.cpp` now yields only if `fifo.isGpuReadingData` was set since the last check, and otherwise does `SLEEP(1)`. 8f83a89416 2012-02 “Removed the busy-wait from DSP LLE on thread” · af139f4661 2011-02 “Used usleep(1) instead of _mm_pause on Linux and Mac OSX” · 12cd2d41ec 2011-03 “Change YieldCPU to sleep(0) like on windows” · daefb3b550 2012-01 “a small thread synchronization speedup for dual core mode”

### dolphin-103 — Allocate buffers at the alignment the consumer wants
Lens: allocation · Source: Dolphin · Also: PostgreSQL
Principle: SIMD loads and stores are fastest on 16-byte-aligned addresses, and some drivers can only DMA straight from page-aligned memory. Allocating with `_aligned_malloc`/`posix_memalign` at the needed boundary opens up those fast paths: aligned vector instructions, or zero-copy "pinned" uploads.
Why it is faster: —
Applies elsewhere: `O_DIRECT` I/O buffers, NIC ring buffers, Vulkan/CUDA host-visible staging memory, cache-line-aligned per-thread counters.
Watch out: —
Seen in Dolphin: `AllocateAlignedMemory`/`FreeAlignedMemory` were added to `MemoryUtil.cpp`, and the texture decode buffer in `TextureCacheBase.cpp` now comes from them so the SSE decoders can use aligned stores. In the OpenGL `StreamBuffer.cpp`, the buffer for `GL_AMD_pinned_memory` is allocated 4096-byte-aligned (and its size rounded up), so the driver can read client memory directly instead of using `MAP_AND_SYNC`. eef715b1cf 2011-02 “added the possibility to allocate aligned memory, an use it to allocate the buffer utilized in texture decoding” · 6e20af9ee9 2013-02 “align memory, use pinned memory if available” • Also seen in PostgreSQL (postgresql-101): 5aa29e88e99 2003-09-21 Aligned shared disk buffers to at least 32 bytes, which "makes a noticeable difference in the speed of transfers to and from kernel space". • 439f61757f0 2022-12-22 Added `palloc_aligned()` and `MemoryContextAllocAligned()`. • faeedbcefd4 2023-04-08 Added `PG_IO_ALIGN_SIZE` (4096) and aligned heap I/O buffers, stack blocks and the shared buffer pool.

### dolphin-104 — Use the processor's native number type in hot shader math
Lens: gpu-rendering · Source: Dolphin
Principle: GPUs have far more float throughput than integer throughput, and every int-to-float conversion is an extra instruction. Keep coordinates and color math in float, and replace shifts with multiplies by the target range, such as `r*31` instead of `r>>3`, so the whole shader stays on the fast path.
Why it is faster: —
Applies elsewhere: WebGL/compute shaders doing image processing, DSP code on float-only cores, ML kernels kept in the accelerator's native type.
Watch out: —
Seen in Dolphin: In the D3D11 EFB-to-texture encoder (`PSTextureEncoder.cpp`), the constant buffer fields `NumHalfCacheLinesX`, `PosX` and the others went from `UINT` to `FLOAT`. The fetch and coordinate helpers now take `float2`, and encoders such as `EncodeRGB565(float4)` compute `pixel.r*31` where they used to shift integers. 9abf449e7c 2011-03 “DX11: Eliminate most int-to-float conversions and use float math in more places”

### dolphin-105 — Let the consumer read straight into its own buffer
Lens: io · Source: Dolphin
Principle: A reader that loads each record into a temporary buffer and then hands it to a callback costs one allocation and one copy per record. Pass the callback the key and size plus a `get_data(dst)` function instead, so the consumer allocates its final buffer and the bytes are read once, directly into it.
Why it is faster: —
Applies elsewhere: streaming HTTP body parsers that hand a sink to the application, `read_into` APIs, protobuf parsers that decode into a caller-owned arena.
Watch out: —
Seen in Dolphin: `LinearDiskCache::OpenAndRead` (`Common/Src/LinearDiskCache.h`) did `value = new V[value_size]` and called a virtual `Read(key, value, size)`. It is now a template on the reader type and calls `reader(key, value_size, [this,&value_size](V* data){ Read(data, value_size); })`. The D3D shader-cache inserters read the bytecode directly into the buffer they keep. 306c3570d1 2011-03 “Eliminated a memcpy in DX11's vertex shader disk cache loading”

### dolphin-106 — Key a compile cache on exactly the inputs that change the output
Lens: laziness · Source: Dolphin
Principle: If a cache key includes state that doesn't affect the generated artifact, identical artifacts get compiled again under different keys. If it leaves state out, results are wrong. Build the key from the same code paths that generate the output, so only bits the output actually depends on go in. A field that loops over a fixed maximum when the actual count is smaller is a common leak. To make lookups cheap, put the count-dependent parts at the end of the key and record how many words are in use, so hashing and comparison stop there.
Why it is faster: —
Applies elsewhere: build-system action keys (Bazel, ccache), template render caches, query-plan caches keyed on normalized SQL, length-prefixed small-string keys.
Watch out: —
Seen in Dolphin: `GetPixelShaderId` (`PixelShaderGen.cpp`) packed large swaths of BP/XF state into the UID. It was rewritten so that only state the generator actually reads goes into the key, and the commit reports about 30% fewer redundant compiles, which reduced stutter in F-Zero GX and Red Steel 2. In `VertexShaderGen.cpp` the color-channel loop changed from `for (i < 2)` to `for (i < xfregs.numChan.numColorChans)`, removing most of the redundant vertex shader compiles. Key comparison was made cheaper too. Per-TEV-stage state in the pixel shader UID was spread across bit-packed fields like `cc_n_d |= cc.d << (4*n)`. It was regrouped into a `stagehash[n]` array of raw register slices (`cc.hex & 0xFFFFFF`), which produced more compact generator assembly. Later, `uid_data.num_values` was set to `offsetof(uid_data, stagehash[numStages])/4` so `NumValues()` covers only the active stages. Vertex UID fields were reordered so bitfields pack into fewer words. 7f01139d13 2011-08 “Replace the pixel shader UID generation algorithm with a better one which reduces the number of redundant shader compilations by around 30%” · bcb8d11c1b 2011-08 “Reduced the number of redundant vertex shader compilations (possibly to zero)” · 16ada5fa3d 2013-06 “Compactify generated UID generator assembly … by storing tev stage hash values in a struct” · bdc28106ee 2013-06 “Optimize shader uid checks by checking the number of uid values which are actually used” · 7a1940020d 2013-08 “VertexShaderGen: Optimize shader uid data order”

### dolphin-107 — Skip output-only work when a generator runs just to compute a key
Lens: other · Source: Dolphin
Principle: When one routine both computes a cache key and emits the artifact, the key-only pass runs on every lookup, so any string formatting, canary checks or driver queries it performs are pure waste. Hoist a single `is_writing` flag and guard all output-only work with it. Also replace helpers that format into temporary strings with compile-time format fragments.
Why it is faster: —
Applies elsewhere: a "dry-run" mode in code generators, React-style render functions that compute a memo key before rendering, logging calls that format only when the level is enabled.
Watch out: —
Seen in Dolphin: `GeneratePixelShaderCode`/`GenerateVertexShaderCode` run with a NULL buffer to build the UID. `c05aa014` added `const bool is_writing_shadercode = (out.GetBuffer() != NULL)` and gated the canary, early-depth code and the `DriverDetails::HasBug` query behind it. In `LightingShaderGen.h`, helpers such as `LightCol()` that `snprintf`'d into static buffers became macros (`LIGHT_COL "%s[5*%d].%s"`) concatenated into the caller's format string. `22d97367` moved `AlphaTest::TestResult()` inline into the header. `7acee71c` builds the swap table as a local, so UID generation no longer writes a global. c05aa0141d 2013-08 “ShaderGen: Optimize out most function calls for uid generation” · fe2ca814c5 2013-08 “LightingShaderGen: Use macro magic instead of snprintf” · 22d9736787 2013-08 “ShaderGen: Static inline everything” · 7acee71c0c 2013-09 “Move swapModeTable to a local variable”

### dolphin-108 — Defer persistent-cache flushes to shutdown
Lens: io · Source: Dolphin
Principle: Syncing an append-only cache file to disk after every insert puts an fsync-class stall on the hot path. The data is only a cache: if the process crashes, it gets regenerated. So write in memory or append without syncing, and sync once at shutdown.
Why it is faster: —
Applies elsewhere: compiler and package-manager caches, browser HTTP cache indexes, build-tool artifact databases.
Watch out: —
Seen in Dolphin: The D3D9 and D3D11 `PixelShaderCache.cpp`/`VertexShaderCache.cpp` called `g_ps_disk_cache.Sync()`/`g_vs_disk_cache.Sync()` after every newly compiled shader. Those calls were removed, and syncing happens on stop. 7eb06430ed 2011-09 “Only sync shader caches to disk on stop”

### dolphin-109 — Turn many tiny readbacks into one block read plus a cache
Lens: io · Source: Dolphin
Principle: Every synchronous read from a device (GPU, disk, remote service) pays a large fixed round-trip cost no matter how small it is. Requests that cluster in space can be served by fetching a whole surrounding tile once, caching it, and answering later point reads from memory until the source changes.
Why it is faster: —
Applies elsewhere: page-granular disk reads, fetching whole rows or pages instead of single fields from a database, tile caches for map servers.
Watch out: —
Seen in Dolphin: EFB peeks in the OpenGL `Renderer::AccessEFB` (`Render.cpp`) did a 1x1 `glReadPixels` per request. They now read the enclosing 64x64 tile into `s_efbCache[type][rect]`, mark it valid in `s_efbCacheValid`, and serve further peeks from it. `ClearEFBCache()` runs on each flush in `VertexManager.cpp`. The commit reports the Baten Kaitos deck menu going from 3 to 54 FPS. 1e558aedeb 2011-08 “Added an EFB peek cache to the GL video plugin”

### dolphin-110 — Pick the buffer-upload path per driver, and say what you will overwrite
Lens: gpu-rendering · Source: Dolphin
Principle: Mapping a GPU buffer, sub-data copies and orphaning cost different amounts on different drivers. Mapping can stall or take a slow path, while `glBufferSubData` may just be a memcpy into driver-owned memory. Make the strategy a switchable choice. When you do map, set an invalidate flag so the driver knows the old contents can be thrown away and doesn't have to preserve or synchronize them. Separating "reserve space" from "upload data" lets each strategy do its part in the right place.
Why it is faster: —
Applies elsewhere: WebGL/WebGPU uniform ring buffers, Vulkan staging-buffer strategies per vendor, choosing mmap or write() for file output.
Watch out: —
Seen in Dolphin: The GLSL uniform path in `ProgramShaderCache.cpp` dropped `glMapBuffer`/`glUnmapBuffer` (“glMapBuffer was slow”) for one combined UBO updated with `glBufferSubData`. `StreamBuffer` later gained a `BUFFERSUBDATA` mode and a separate `Alloc(size)` step ahead of `Upload()`. The UBO `glMapBufferRange` call got `GL_MAP_INVALIDATE_RANGE_BIT` next to `GL_MAP_UNSYNCHRONIZED_BIT`. c908e1173d 2011-12 “glMapBuffer was slow, go back to glBufferSubData, single combined ps/vs ubo now” · 878bd7f26c 2013-02 “implement streaming by bufferSubData, split upload and allocation in ringbuffer” · 4fadb65259 2013-01 “invalidate range on bindbufferrange since we don't care about what is in the range”

### dolphin-111 — Persist expensive compiled artifacts across runs
Lens: laziness · Source: Dolphin
Principle: Compiling and linking GPU programs at first use causes visible hitches. If the platform can export the final binary, store it on disk under the program's key and load it on the next run, which skips both compile and link. Record the binary format and a version, so a driver change makes the cache miss instead of failing.
Why it is faster: —
Applies elsewhere: Vulkan pipeline caches, Python `.pyc`, V8 code cache, JIT warm-start profiles.
Watch out: —
Seen in Dolphin: `ProgramShaderCache.cpp` gained `g_program_disk_cache` (a `LinearDiskCache<PROGRAMUID,u8>`). It sets `GL_PROGRAM_BINARY_RETRIEVABLE_HINT`, saves the program binary with its `ProgramFormat` and reloads it at startup when `bSupportsGLSLCache` is set, which the commit says reduced F-Zero startup stutter. Uniform locations are also queried once per program and cached in the entry instead of being looked up each time. a10656b1b2 2011-12 “Implement Program shaders cache”

### dolphin-112 — Don't load a register you are about to overwrite
Lens: jit-interpreters · Source: Dolphin
Principle: Before an instruction writes a destination, a register allocator only needs to load that destination's old value if the instruction also reads it. Pass "needs load" as a precise condition (destination equals a source) rather than always true. That removes a memory load and a move from every such instruction.
Why it is faster: —
Applies elsewhere: write-allocate versus no-allocate decisions in caches, "discard" load ops for render-pass attachments, skipping read-before-write in ORMs for full-row updates.
Watch out: —
Seen in Dolphin: In Jit64 `Jit_Integer.cpp`, several ops called `gpr.BindToRegister(a, true, true)`. They now call `gpr.BindToRegister(a, (a == b || a == s), true)`, so the guest register is loaded into the host register only when it is also an input. 60cddf0823 2012-01 “Fixed some BindToRegister calls (removing unnecessary MOV instructions for speedup)”

### dolphin-113 — Strength-reduce arithmetic on known constants and use addressing modes
Lens: jit-interpreters · Source: Dolphin
Principle: When an operand is a compile-time constant, multiply by 0, -1 or a power of two becomes XOR, NEG or SHL, and constant unsigned division becomes multiply-and-shift. Address arithmetic like `x*4` or `base + x*8` fits in one `LEA` or a scaled memory operand, with no separate MOV and SHL. In hand-written loops, step the induction variable by the stride rather than computing `i*3+k` after a division for the trip count, and force-inline tiny per-element emitters.
Why it is faster: —
Applies elsewhere: any compiler or query engine with constant folding, bytecode VMs specializing on immediate operands, index-buffer builders and pixel loops.
Watch out: —
Seen in Dolphin: `mulli`/`mullwx` in `Jit_Integer.cpp` check the immediate: 0 becomes `XOR`, -1 becomes `NEG`, and `(imm & (imm-1)) == 0` becomes `SHL` with the shift found by bit tests, falling back to `IMUL`. `divwx` gained a compiled path, and constant unsigned division was optimized. `CMP r,0` became `TEST r,r`. In the DSP JIT and JitIL, `MOV`+`SHL` pairs became `LEA … MScaled(EAX, SCALE_4)`, and quantized-table calls use `CALLptr(MScaled(EDX, SCALE_8, table))`. The same idea in C++: `IndexGenerator.cpp` loops (`AddList`, `AddQuads`, `AddLineList`) replaced `numTris = numVerts/3` and `index + i*3 + k` with `for (i = 2; i < numVerts; i += 3)` and `index + i - 2`, and `WriteTriangle` became `__forceinline`. a7f3f61598 2012-01 “[Jit64] Constant unsigned division and constant multiplication optimizations” · ea95e603b2 2011-06 “JIT: Get rid of SHLs with shifts of 1, 2, 3” · a2e132dd4b 2013-06 “small index generator optimiztions” · 86935ab4fd 2013-02 “forceinline IndexGenerator::WriteTriangle”

### dolphin-114 — Use wider operations: to absorb edge cases, and to merge narrow stores
Lens: jit-interpreters · Source: Dolphin
Principle: Guest semantics often differ from the host at the edges. For example, a 32-bit shift by 32 or more must give 0, but x86 masks the count. Doing the operation at double width (a 64-bit shift of a zero-extended 32-bit value) gets the guest result for every count, so no compare-and-branch is needed for large counts. Width helps with memory too: four adjacent 16-bit constant stores can become two 32-bit stores of the same bytes.
Why it is faster: —
Applies elsewhere: overflow-safe arithmetic by computing in 64-bit and checking the high half, saturating DSP math in wider accumulators, compiler store-merging passes.
Watch out: —
Seen in Dolphin: On x64, `srwx`/`slwx`/`srawx` in `Jit_Integer.cpp` now move the shift count into ECX and do `SHR(64, a, ECX)`/`SHL(64, …)` on the 64-bit register, then truncate with a 32-bit `MOV`. The old version emitted branchy special cases for counts of 32 and above. The DSP JIT `clrp` (`DSPJitMultiplier.cpp`) went from four `MOV(16, …)` into `prod.l/m/h/m2` to two `MOV(32, …, Imm32(…))`. fd616ccb0c 2012-01 “[Jit64] Register-based shift instructions optimized on 64-bit processors” · 1f170130e8 2011-06 “DSP/Jit: Combine clrp memory writes back to two 32bit writes”

### dolphin-115 — Invalidate the smallest region, and only for writes that can matter
Lens: jit-interpreters · Source: Dolphin · Also: ares
Principle: Over-broad cache invalidation turns into recompilation storms. Shrink the invalidated range to the exact bytes. Record only the writes that actually need the special handling. Recognize idioms where a flush cannot change contents, such as a prefetch followed by a flush of the same line, and skip the invalidation for them.
Why it is faster: —
Applies elsewhere: fine-grained CDN purge keys, incremental build invalidation by file hash instead of by directory, React keyed reconciliation.
Watch out: —
Seen in Dolphin: In `GPFifo.cpp`, FIFO-write tracking invalidated `(PC-4, 8)` bytes and is now `(PC, 4)`. It then started recording only real store opcodes (`OPTYPE_STORE`, `STOREFP`, `psq_st`) in `fifoWriteAddresses`, so other instructions no longer force a recompile with exception checks. Jit64/JitIL gained a `dcbst` handler that does not invalidate the JIT cache when the previous instruction is `dcbt`. That fixed constant “Clearing code cache” in The Last Story. 5de8366db2 2012-03 “Made the invalidation of the FIFO write JIT blocks more selective (efficient)” · 0c882cca66 2013-02 “Profile stores, fp stores and ps stores only to the fifo write addresses list” · 99b7c91df5 2012-05 “Checked if dcbst instructions are preceded by dcbt” • Also seen in ares (ares-047): The SH-2's `invalidate(address)` used to null the whole pool, `pools[address >> 8] = nullptr`. It now clears `pool(address)->blocks[address >> 1 & 0x7f]`, and a later fix stopped this from allocating empty pools. The N64 CPU got the same change, but it was reverted to whole-pool eviction because overlapping blocks crashed Jet Force Gemini. Single-block eviction is only safe with exact overlap tracking. 657b0c0f5 2021-07 “sh2: invalidate single blocks, not entire pools” · b56dac15e 2021-12 “recompiler: fix page protection on Apple silicon” · 1808e3f50 2022-08 “n64: make the recompiler invalidate single blocks rather than entire pools” · c6ecbe81b 2022-08 “n64: restore old recompiler invalidate behavior”

### dolphin-116 — Put a bitset in front of an ordered-map range query
Lens: algorithms · Source: Dolphin
Principle: If most queries ask "is anything here?" and the answer is usually no, a dense bitset with one bit per fixed-size chunk answers in a single memory access. Only when the bit is set do you fall through to the logarithmic ordered-map search. This is the same idea as a Bloom filter, but exact, because the chunks are aligned.
Why it is faster: —
Applies elsewhere: page-present bitmaps in allocators, dirty-card tables in garbage collectors, sparse-file extent maps, spatial-hash occupancy grids.
Watch out: —
Seen in Dolphin: `JitBlockCache::InvalidateICache` (`JitCache.cpp`) did `block_map.lower_bound()` on a `std::map<pair<u32,u32>,u32>` for every call, and most calls come from 32-byte `dcb*`/`icbi`. A `valid_block` bitset with one bit per 32 bytes of RAM is now set as blocks are compiled. For `length == 32`, a clear bit skips the map search entirely. The commit reports InvalidateICache more than 100% faster. 3990002250 2012-10 “Optimize JitCache::InvalidateICache by maintaining a "valid blocks" bitset”

### dolphin-117 — Emit checks only where analysis shows they're needed
Lens: jit-interpreters · Source: Dolphin · Also: ares
Principle: A per-instruction or per-block runtime check (interrupts, exceptions, breakpoints) costs on every pass. Run a pre-pass that marks only the instructions that can actually trigger the condition, and emit the check after those. Handle externally triggered events at the moment they are raised instead of polling for them.
Why it is faster: —
Applies elsewhere: safepoint placement in managed runtimes (only at loop back-edges and calls), bounds-check elimination, cancellation checks only around blocking calls.
Watch out: —
Seen in Dolphin: The DSP LLE JIT called `checkExceptions` and an external-interrupt check after every instruction. `DSPAnalyzer.cpp` now sets `CODE_CHECK_INT` on the instruction following opcodes that can raise exceptions, and `DSPEmitter` emits the check only there. External interrupts are processed in the DSP control-register write handler (`main.cpp`) when the CPU sets the flag. ae71e63872 2011-01 “LLE JIT: Minimised exception checking” • Also seen in ares (ares-076): In the SH-2 recompiler, `emitInstruction` used to return a bool. It now returns a `Branch` kind, and `emit` tracks `inDelaySlot = branch == Branch::Slot`. `checkDelaySlot` now starts with `if(!inDelaySlot) return body();`, so it emits the compare-and-jump on `PPM` only after a delayed branch, or on the first instruction of a block, where it is forced on. 8156a7b76 2023-02 “sh2: elide runtime delay slot check when possible” • Also seen in ares (ares-113): The RSP JIT first added an inline epilogue that tested `branch.state` at runtime and called the C++ `instructionEpilogue` only when the state was nonzero. The next commit removed that runtime test: the emitter tracks `delaySlot = hasBranched` itself and emits the legacy call only for real delay slots. Ordinary instructions get a straight-line clock add and PC store. 3e913fda5 2026-04 “n64: add toggled fast RSP instruction epilogue path” · a3d75ad5d 2026-04 “n64: specialize RSP JIT epilogue by delay-slot position” • Also seen in ares (ares-123): The RSP JIT emits the `status.halted` test only when `instructionMaySetHalted()` is true, which covers BREAK and MTC0 to SP_STATUS. The CPU JIT emits its EndBlock test and pipeline commit only when `firstInstruction || delaySlot || info.likelyBranch() || emitCallfEmitted`. f43063803 2026-04 “n64: gate RSP halted checks to relevant opcodes” · 105e8da92 2026-05 “n64: in CPU JIT, make epilogue code conditional in fast paths”

### dolphin-118 — Guard an expensive slow path with cheap state tests
Lens: branching-fast-paths · Source: Dolphin
Principle: Leaving the fast path unconditionally "just in case" (for instance, to check for pending interrupts) costs the full exit every time. Test the few flags that decide whether the slow path could do anything, and branch to it only when they are set.
Why it is faster: —
Applies elsewhere: checking `queue.empty()` before taking a lock, skipping event dispatch when there are no listeners, checking a dirty flag before a re-render.
Watch out: —
Seen in Dolphin: Jit64 `mtmsr` (`Jit_SystemRegisters.cpp`) always ended with `WriteExceptionExit()`. It now tests `MSR & 0x8000` (EE enabled) and `ppcState.Exceptions`, takes `WriteExternalExceptionExit()` only if both are set, and otherwise does a normal `WriteExit`. The gather-pipe exception check in `Jit.cpp` was narrowed the same way: it skips when non-external exceptions are pending, and tests `msr` EE and the CP interrupt cause before exiting. 76a13604ef 2012-06 “Optimize mtmsr by going to an exception exit only if EE are enabled and some exceptions are pending” · 352ab2ba43 2012-03 “Improved the CheckException for the GatherPipe writes in JIT”

### dolphin-119 — Jump between generated blocks instead of calling them
Lens: jit-interpreters · Source: Dolphin
Principle: If a dispatcher CALLs each compiled block, every block has to save and restore callee-saved registers and return. If the dispatcher saves state once on entry and blocks JMP back to it (or directly to each other), that per-block prologue and epilogue disappear, and state that used to live in a dedicated register can stay in memory.
Why it is faster: —
Applies elsewhere: threaded-code and tail-call interpreters (`musttail` in Clang, computed goto in CPython), coroutine trampolines.
Watch out: —
Seen in Dolphin: In the DSP JIT `DSPEmitter.cpp`, the dispatcher's `CALLptr(MComplex(EBX,ECX,SCALE_4,0))` became `JMPptr`. Each block's `ABI_PopAllCalleeSavedRegsAndAdjustStack(); RET();` became `JMP(returnDispatcher)`, and the cycle counter moved from a pinned register to `SUB(16, M(&cyclesLeft), …)`. Jit64 block exits in `Jit.cpp` now `JMP(asm_routines.dispatcher)` instead of testing state, popping registers and returning. 7e9f02869a 2011-01 “LLE JIT: Changed the ASM dispatcher so that it jumps to the blocks instead of calling them” · e5d527340d 2012-05 “Optimised the JIT path.”

### dolphin-120 — Track what each unlinked jump waits for, and relink precisely
Lens: jit-interpreters · Source: Dolphin
Principle: Linking compiled blocks directly saves a trip through the dispatcher. But if a target doesn't exist yet, crude strategies such as periodically flushing every block with unresolved links recompile far too much. Keep a per-block list of the targets it is waiting on. When a target gets compiled, recompile only the blocks waiting on it.
Why it is faster: —
Applies elsewhere: incremental linkers, dependency-tracked rebuilds, lazy symbol binding with PLT patching.
Watch out: —
Seen in Dolphin: The DSP JIT used to wipe every block with `CODE_CALL` set after every 10 compiles and give up when more than 8 calls were unresolved. `DSPEmitter::Compile` now keeps `unresolvedJumps[start_addr]` as a list of target addresses (removing self-links). Both jumps and calls can link, and a linked jump checks the cycle count before entering the next block. a6637c257f 2011-01 “LLE JIT: Reworked the block linking code”

### dolphin-121 — Resolve rarely-changing conditions when you generate code
Lens: jit-interpreters · Source: Dolphin
Principle: If a condition almost never changes, decide it once when the code is generated or the program is built, rather than testing it every time the code runs. Only the variant that's needed is emitted. The cost is that a change has to trigger regeneration, or that the result is knowingly a little inexact.
Why it is faster: —
Applies elsewhere: feature flags baked into compiled bundles, specializing SQL plans on parameter nullness, shader permutations instead of uniform branches.
Watch out: —
Seen in Dolphin: The Jit64/JitIL block prologue emitted `CMP(MMCR0,0)` plus a conditional `UpdatePerformanceMonitor` call in every block. It now checks `if (MMCR0.Hex || MMCR1.Hex)` at compile time; the code comments call this a “SPEED HACK”. The Z-comploc pixel shader pass is skipped when `PixelShaderManager::AlphaPreTest()` can prove from the alpha-test configuration that the test always passes or always fails. 853d12b42c 2012-04 “Changed the performance monitor check to a compile time check instead of run-time” · 0efd4e5c29 2012-08 “Skipped the ZCompLoc pass if the result can be determined at compile time”

### dolphin-122 — Pay for debugging hooks only when they're used
Lens: other · Source: Dolphin
Principle: Debug support can quietly tax every user. Examples are single-instruction blocks so breakpoints work, or checking a compile or link status that forces the driver to synchronize. Turn the cost on only when it is needed: split blocks only while stepping, put a check only at addresses that have breakpoints and invalidate those addresses when breakpoints change, and compile status checks in only for debug builds.
Why it is faster: —
Applies elsewhere: conditional instrumentation with zero-cost tracepoints, `glGetError` only in debug builds, source maps loaded only when DevTools are open.
Watch out: —
Seen in Dolphin: Jit64 forced `blockSize = 1` whenever the debugger was enabled. It now does so only in `CPU_STEPPING`, and otherwise emits a `CheckBreakPoints` call just at addresses that have breakpoints (JitIL uses a new `BreakPointCheck` IR op). `BreakPoints::Add/Remove/Clear` invalidate the affected JIT block. In the OpenGL `PixelShaderCache.cpp`/`VertexShaderCache.cpp`, the compile-status query is now wrapped in `#if defined(_DEBUG) || defined(DEBUGFAST) || defined(DEBUG_GLSL)`. 8ed6ea3b07 2012-03 “Sped up the Dolphin debugger in JIT mode by splitting a block only while stepping or when it contains a breakpoint” · 4fadb65259 2013-01 “Only check shader compilation status in debug or with DEBUG_GLSL defined”

### dolphin-123 — Shadow the state you set and drop no-op updates
Lens: gpu-rendering · Source: Dolphin
Principle: Driver and hardware state changes are expensive, and many writes merely repeat the current value. Keep a CPU-side copy of what was last set (bound texture per unit, active program, shader constants, register values) and return early when the new value is the same. Watch for writes that trigger an action even when the value is unchanged, and still pass those through.
Why it is faster: —
Applies elsewhere: React/virtual-DOM diffing, avoiding redundant `setState`, caching the last `fcntl`/socket option set, skipping MMIO writes with an unchanged value in drivers.
Watch out: —
Seen in Dolphin: `BPWritten` (`BPStructs.cpp`) now returns early when `bpmem[addr] == newvalue`, except for trigger registers (EFB copy, bbox clear, draw done, PE tokens, TLUT loads, texture invalidate and preload). The OpenGL `TextureCache.cpp` tracks `s_Textures[8]` and `s_ActiveTexture` to skip `glActiveTexture`/`glBindTexture`. `ProgramShaderCache::SetShader` returns when the (PS,VS) pair is already bound. `PixelShaderManager` keeps `s_constant_cache` and skips matching constant uploads, and a missing reset of `nPostTransformMatricesChanged` in `VertexShaderManager` was causing redundant constant uploads every draw. D3D9 `D3DBase.cpp` caches stream sources, the index buffer, the vertex declaration and shaders with change flags. c95baf614d 2012-05 “Avoid changing video state on useless BP writes” · d73c950623 2013-01 “cache texture bindings” · bb200acdd8 2013-01 “also don't search in the shader cache if the shader isn't changed” · 9f1582843d 2012-09 “PixelShaderManager: Reduce number of redundant shader constant updates” · bed260aa14 2012-12 “VertexShaderManager: Fix redundant shader constant updates” · ee72852491 2012-10 “implement some code to reduce the amounts of calls to setup vertex format”

### dolphin-124 — Cache small immutable GPU objects keyed by their parameters
Lens: gpu-rendering · Source: Dolphin
Principle: Creating or re-uploading tiny objects like samplers, fullscreen-quad VBOs or state blocks every draw costs driver calls and allocations. Pack the defining parameters into a key and keep a map from key to object. A repeat request binds the existing object.
Why it is faster: —
Applies elsewhere: D3D11/Vulkan state-object caches, interned strings, a font-glyph atlas keyed on (glyph, size).
Watch out: —
Seen in Dolphin: A new OpenGL `SamplerCache` (`SamplerCache.cpp`) maps `TexMode0`/`TexMode1` and anisotropy to GL sampler objects, replacing `glTexParameter` calls on every texture bind. `TCacheEntry::FromRenderTarget` used to re-upload a quad with `glBufferData` on every EFB copy. It now keeps `std::map<u32,VBOCache> s_VBO` keyed on the packed source rectangle. f2647a1216 2013-02 “Implement OGL sampler cache” · 34b1451fbe 2012-12 “cache vbos in TextureCache::TCacheEntry::FromRenderTarget”

### dolphin-125 — Pack composite keys into one integer and use a hash map
Lens: algorithms · Source: Dolphin
Principle: A `std::map<std::pair<u32,u32>,V>` pays for tree pointer-chasing and two-field comparisons. Two 32-bit IDs fit in a single `u64` (`a<<32 | b`), which hashes and compares in one instruction, and an `unordered_map` on it gives O(1) average lookup with better locality.
Why it is faster: —
Applies elsewhere: graph edge maps keyed on (src,dst), tile caches keyed on (x,y), join keys in query engines.
Watch out: —
Seen in Dolphin: In the OpenGL `ProgramShaderCache.h`, `typedef std::map<ShaderUID, PCacheEntry>` with `ShaderUID = std::pair<u32,u32>` became `std::unordered_map<u64, PCacheEntry>` with `Create_Pair(psid, vsid)`. `CurrentShaderProgram` also became a `u64`. d94f3c4155 2013-01 “Stop using std::pair and std::map. Switch over to u64 and std::unordered_map”

### dolphin-126 — Diff settings and invalidate only what each change affects
Lens: laziness · Source: Dolphin · Also: Flycast
Principle: Throwing away a whole cache whenever a settings dialog opens, or on any state reload, discards data that is still valid. Keep a copy of the previous settings, compare field by field, and invalidate only the subsets each changed field affects. Where you can, mark entries stale (for example, reset their hash) rather than destroying them.
Why it is faster: —
Applies elsewhere: hot-reloading config in servers without dropping connection pools, CSS changes that invalidate only matching selectors, cache tags in HTTP reverse proxies.
Watch out: —
Seen in Dolphin: Opening `VideoConfigDiag` used to call `TextureCache::InvalidateDefer()`. `TextureCache::OnConfigChanged` (`TextureCacheBase.cpp`) now compares against `backup_config`. Only changes to texture-affecting settings (color samples, format overlay, hi-res textures) call `Invalidate`, and only EFB-copy settings call `ClearRenderTargets`. On savestate load, `InvalidateHashes()` sets every entry's `hash = TEXHASH_INVALID` instead of deleting all textures. 8d30ac462a 2012-05 “Instead of invalidating texcache whenever the graphics configuration dialog gets opened, clean up textures on configuration changes” · ffddfd8662 2012-12 “do only invalid hashes instead of all textures on loading” • Also seen in Flycast (flycast-133): Switching renderers called `SwitchRenderApi()` every time, which tore down the graphics context. Now it does so only when the change crosses OpenGL↔Vulkan (`settings.pvr.IsOpenGL() != ...`). Otherwise it just sets `settings.pvr.rend`. The extra clear and swap calls in GL term were removed as well. bb5b5d451 2020-12 “only switch vulkan/opengl when needed”

### dolphin-127 — Reuse a cached resource that is a superset of the request
Lens: laziness · Source: Dolphin
Principle: Requiring an exact match on every attribute causes needless re-creation when an existing entry can already serve the request. One example is a texture with more mip levels than asked for, where the extra levels can be clamped by render state. Match on "satisfies" rather than "equals" for attributes that can be restricted at use time.
Why it is faster: —
Applies elsewhere: serving a smaller image size from a larger cached rendition, connection pools that accept a connection with a superset of the needed capabilities, buffer pools handing out a larger buffer.
Watch out: —
Seen in Dolphin: `TextureCache::Load` (`TextureCacheBase.cpp`) reused an entry only if `entry->num_mipmaps == maxlevel`. It now accepts `entry->num_mipmaps > maxlevel` and stores `maxlevel + 1`, because the maximum level is applied through render states. c859aaae84 2012-08 “[optimization] TextureCacheBase: Slightly loosen the requirements for reusing mipmapped textures”

### dolphin-128 — Rotate through several streaming buffers to avoid CPU–GPU stalls
Lens: gpu-rendering · Source: Dolphin
Principle: If the CPU writes into a buffer the GPU is still reading, the driver must either block or copy. Allocate N buffers and cycle through them, and append within a buffer with no-overwrite semantics. The CPU then fills one buffer while the GPU consumes another, and the two run in parallel.
Why it is faster: —
Applies elsewhere: triple-buffered frame resources in Vulkan/D3D12, double-buffered DMA in audio drivers, ring buffers between pipeline stages.
Watch out: —
Seen in Dolphin: The D3D11 `VertexManager.cpp` went from one `m_vertexBuffer`/`m_indexBuffer` to arrays of `MAXVBUFFER_COUNT = 4`, cycled through with `m_activeVertexBuffer`/`m_activeIndexBuffer`. D3D9 got a similar multi-buffer path that falls back to the old path if the card cannot provide enough buffers. 5230146c73 2012-10 “change a little the way the vertex are send to the gpu … to increase the parallelism between the cpu and gpu”

### dolphin-129 — Order struct fields to remove padding
Lens: memory-layout · Source: Dolphin
Principle: The compiler pads each field to its natural alignment. Mixing pointers, ints and bools in an arbitrary order leaves holes. Group the fields by size, largest first, so the struct shrinks and more of them fit per cache line in a hot array.
Why it is faster: —
Applies elsewhere: Go/Rust struct layout tuning, ECS component structs, network packet headers, rows in in-memory databases.
Watch out: —
Seen in Dolphin: `JitBlock` in `JitCommon/JitCache.h` was rearranged. The pointers `checkedEntry`, `normalEntry` and `exitPtrs[2]` come first, then `int flags`, then the bools `invalid` and `linkStatus[2]` together. This took the struct from 88 to 80 bytes. 00df727d2f 2012-07 “Optimised the JitCache struct size from 88 bytes to 80 bytes”

### dolphin-130 — Back shared memory mappings with a RAM filesystem
Lens: platform · Source: Dolphin
Principle: Mapping a file from a disk-backed directory to get shared memory can cause the kernel to write the pages back to disk. Put the backing file on tmpfs (`/dev/shm`) so it lives only in RAM. Unlinking it right after opening means it disappears when the process exits and doesn't collide with other instances.
Why it is faster: —
Applies elsewhere: POSIX `shm_open` or `memfd_create` for IPC buffers, browser shared-memory transport, database shared buffers.
Watch out: —
Seen in Dolphin: `MemArena.cpp` created the emulated-RAM backing file at `/tmp/gc_mem.tmp`. It is now `/dev/shm/gc_mem.tmp`, and `unlink()` is called right after open. be200074e9 2012-08 “[Linux] Change from using /tmp to /dev/shm in MemArena so we don't cause any disk IO”

### dolphin-131 — Access memory directly and patch the rare faulting sites
Lens: memory-layout · Source: Dolphin · Also: Flycast
Principle: Routing every guest memory access through a checked slow function is safe but slow. Emit a plain load from a reserved address range instead. When an access faults (an MMIO address, for example), the signal handler rewrites that one instruction into a call to the slow path. Common accesses pay nothing extra, and each rare one pays once.
Why it is faster: —
Applies elsewhere: guard pages for stack growth, the JVM's implicit null checks via SIGSEGV, WebAssembly bounds checks done with virtual-memory reservations.
Watch out: —
Seen in Dolphin: In `Core.cpp` and `JitCommon/Jit_Util.cpp`, the fastmem/backpatch path guarded by `#if defined(_WIN32) && defined(_M_X64)` was opened to every x64 platform (`#if defined(_M_X64)`). The commit reports 8% on Linux amd64. `JitBackpatch.cpp` then gained 16-bit load support (`Memory::Read_U16` then `SHL 16`), so `SafeLoadToEAX` uses fast access for every size except 8-bit. eb5e7d44bd 2013-01 “Enable fastmem (aka. backpatch) support on Linux amd64 and OSX x64” · c553c42d63 2013-01 “Implement backpatch support for 16 bit loads” • Also seen in Flycast (flycast-024): The Linux `fault_handler` now calls `ngen_Rewrite(pc, *(esp), eax)` on x86 and `ngen_readm_fail_v2` on ARM. `ngen_Rewrite` matches the faulting call against the `mem_code` fast stubs and overwrites it with `op_call` to the slow stub in `mem_code[2]`, or to the store-queue stub. When `TARGET_NO_NVMEM` is set (no page tricks), `_vmem_reserve` instead uses page-aligned `malloc_pages` for RAM, VRAM and ARAM. 5b609a6be 2015-05 “linux: Sh4 dynarec works” · ea26e2784 2015-07 “Merge windows and linux x86 recompiler paths” · a2831d67a 2015-02 “vmem/alloc: TARGET_NO_NVMEM to not use page tricks”

### dolphin-132 — Keep JIT tables within reach of short addressing
Lens: platform · Source: Dolphin
Principle: On x86-64, code can reach data with a 32-bit absolute or RIP-relative displacement only if the data is within about 2 GB. Allocating lookup tables inside the code region, or mapping executable memory in the low 2 GB, keeps the short encodings working and avoids loading 64-bit addresses into scratch registers.
Why it is faster: —
Applies elsewhere: small-code-model compilation, GOT/PLT placement, LuaJIT's low-memory allocation for its 32-bit pointer tricks.
Watch out: —
Seen in Dolphin: The `pairedLoadQuantized`/`pairedStoreQuantized`/`singleStoreQuantized` dispatch tables were static arrays that could end up above 2 GB. They are now reserved inside the JIT code space with `AlignCode16(); ReserveCodeSpace(…)` (`JitAsmCommon.cpp`), so 32-bit `MDisp`/`MScaled` table accesses stay valid. `MemoryUtil.cpp` also allocates executable memory with `MAP_32BIT` on Linux x86-64. 4e4a613b8b 2011-04 “Put the quantized paired/single load/store tables into the code memory” · d4158f076a 2011-01 “Allocate low executable memory on OS X as on Windows and Linux”

### dolphin-133 — Move a self-contained job onto its own thread
Lens: concurrency · Source: Dolphin
Principle: A substantial job that runs synchronously on the main thread adds its full latency there. Hand it to a dedicated worker through a mutex and condition variable, and let the main thread continue. Remove handshakes that make one thread wait for another when nothing needs them, and detect and recover from underruns instead of locking the two threads together.
Why it is faster: —
Applies elsewhere: audio mixing threads in games, offloading compression in log shippers, Web Workers for parsing.
Watch out: —
Seen in Dolphin: `CUCode_NewAX` (`UCode_NewAX.cpp`) now starts `m_axthread`. It waits on `m_cmdlist_cv`, runs `HandleCommandList` and posts the DSP interrupt, while mail handling waits on `m_processing`. In `OpenALStream.cpp`, the `mainSyncEvent.Set()/Wait()` handshake with the CPU thread was removed, and an `AL_SOURCE_STATE != AL_PLAYING` check restarts playback after an underrun. 9e813502ac 2012-11 “Create a new thread for AX and make it handle the command lists” · 3632ce6df5 2013-01 “Removed the synchronisation between the CPU thread and the audio thread”

### dolphin-134 — Switch a pipeline to float so library SIMD paths turn on
Lens: simd · Source: Dolphin
Principle: Many DSP libraries vectorize only their float path. Converting samples to float once at the input, and handing float buffers to the audio API at the output, lets every stage in between use SSE. It also removes back-and-forth conversions between stages that wanted different formats.
Why it is faster: —
Applies elsewhere: audio plugin hosts working in 32-bit float, image pipelines working in linear float, ML preprocessing done in float32 tensors.
Watch out: —
Seen in Dolphin: SoundTouch was rebuilt with `SOUNDTOUCH_FLOAT_SAMPLES` instead of integer samples (`STTypes.h`). `OpenALStream.cpp` converts the 16-bit mix to float once before `putSamples`. The DPL2 decoder now consumes the float buffer directly, and output uses `AL_FORMAT_STEREO_FLOAT32` where available. 6d4a566bc4 2013-01 “Changed SoundTouch to use float samples, allowing SSE to be used”

### dolphin-135 — Process a bigger chunk per call, or use the batched API
Lens: algorithms · Source: Dolphin · Also: PostgreSQL
Principle: Calling a per-chunk routine three times on 1 ms each pays call overhead, setup and pointer bookkeeping three times. Make the chunk size a parameter and process 3 ms in one call, so per-call costs are amortized and the inner loops run longer. The same applies to driver calls: when an API has a plural form, gather the elements into one contiguous buffer and make one call instead of N.
Why it is faster: —
Applies elsewhere: larger audio callback buffers, SQL multi-row INSERT, `writev`, Redis `MSET`, `glUniform4fv` with count > 1.
Watch out: —
Seen in Dolphin: `CUCode_AXWii` called `Process1ms` three times and advanced the buffer pointers by 32 samples in between. `UCode_AX_Voice.h` now defines `SAMPLES_PER_FRAME` (32 for GC, 96 for Wii), and a single `ProcessVoice` call handles the whole 3 ms frame. In the OpenGL `VertexShaderCache.cpp`, `SetMultiVSConstant4fv`/`3fv` looped over `glProgramEnvParameter4fvARB`. When `GLEW_EXT_gpu_program_parameters` is present they now make a single `glProgramEnvParameters4fvEXT(…, count, f)` call, padding 3-component values into a buffer first. 35249a8576 2013-01 “Handle 3ms at a time in AXWii instead of 3x1ms” · d4ced9e995 2011-01 “Tweaked SetMultiVSConstant functions to prefer glProgramEnvParameters4fvEXT” • Also seen in PostgreSQL (postgresql-062): 9b29f9f3f86 2005-08-06 COPY FROM parses all attributes of a line in one call instead of a function call per character. • 0a5fdb0d914 2006-05-25 COPY OUT sends literal runs between special characters with one call; 011b51cb7e9 combined it with specialized loops after the routine was found at 66% of a COPY OUT profile. • 4d63e267742 2006-05-26 psql writes each row with one `fwrite`, because "fwrite()'s per-call overhead is quite significant". • 0a0727ccfc5 2020-07-25 Binary COPY FROM reads 64 KB at a time and hands out bytes itself, about 30% faster for narrow columns on Linux; a84c95634e2 raised the scanner's read-buffer cap. • 3838fa269c1 2022-07-01 The JSON lexer looks ahead to the next special character and appends the run in one call instead of per byte; JSON parsing had taken about half the time of COPY BINARY with large JSONB values.

### dolphin-136 — Regenerate instead of reading back from write-only memory
Lens: memory-layout · Source: Dolphin
Principle: Memory mapped from a GPU (write-combined or write-only) is extremely slow to read, and some mapping modes forbid reads entirely. If you need to repeat recently written data, recompute it from the source instead of copying it back out of the mapped buffer. The extra CPU work happens rarely, while every frame keeps the faster write-only mapping.
Why it is faster: —
Applies elsewhere: never reading back from mapped Vulkan host-visible memory, recomputing rather than reading framebuffer memory on embedded displays.
Watch out: —
Seen in Dolphin: When a strip or fan is split across vertex-buffer flushes, `VertexLoader::RunVertices` used to `memcpy_gc` the last converted vertices out of `s_pCurBufferPointer`. It now rewinds `g_pVideoData` and calls the new `ConvertVertices(n)` to decode them again into the new buffer, which keeps the buffer write-only for mapping. 7a89faf13f 2013-02 “converting the last vertices again instead of copying from buffer on buffer split”

### dolphin-137 — Give the driver everything it would otherwise compute or infer
Lens: gpu-rendering · Source: Dolphin
Principle: Some API calls make the driver scan your data to find things you already know. One example is the index range of a draw. Passing the extra information directly, through the range variant of the call or an integer vertex attribute in place of a float you convert back, removes hidden per-draw work in the driver or shader.
Why it is faster: —
Applies elsewhere: passing `Content-Length` instead of chunked encoding, hinting array sizes to allocators, typed arrays in place of generic arrays in JS engines.
Watch out: —
Seen in Dolphin: `VertexManager.cpp` (OpenGL) switched `glDrawElements`/`glDrawElementsBaseVertex` to `glDrawRangeElements`/`glDrawRangeElementsBaseVertex` with `0..GetNumTriangles()*3` and similar bounds. On Mali, this removed an internal driver function that had taken 8% of time. The position-matrix index became `ATTRIN int posmtx` fed by `glVertexAttribIPointer`, replacing a `float fposmtx` that every vertex shader converted with `int(fposmtx)`. 6887a0c341 2013-07 “Change from using glDrawElements/glDrawElementsBaseVertex to glDrawRangeElements/glDrawRangeElementsBaseVertex” · 52dacaa3e3 2013-02 “Make the posmtx vertex attribute an int instead of converting to and from a float”

### dolphin-138 — Help tile-based GPUs skip memory traffic
Lens: gpu-rendering · Source: Dolphin
Principle: Mobile tiled GPUs must load the previous contents of every attachment into tile memory unless told otherwise. Clearing all attachments (color, depth and stencil) together tells the driver nothing needs loading. On some drivers, flushing at regular, small batch boundaries stops queued work from piling up.
Why it is faster: —
Applies elsewhere: Vulkan `LOAD_OP_CLEAR`/`DONT_CARE`, `glInvalidateFramebuffer` in WebGL and mobile games, Metal load and store actions.
Watch out: —
Seen in Dolphin: A `glClear(GL_COLOR_BUFFER_BIT)` in the OpenGL `Render.cpp` now also clears depth and stencil, which the commit says improved performance on Mali. A `BUG_MALIBROKENBUFFERS` entry in `DriverDetails` makes `VertexManager` call `glFlush()` after every flush on Mali-T6xx, which was found to be quicker than flushing every n times. 7d187dc597 2013-08 “Change a glClear in the OpenGL renderer to improve performance on Mali chips” · 672871b3be 2013-07 “Add in the Mali driver bug so we can call glFlush every flush”

### dolphin-139 — Collapse a two-pass effect into one pass, and don't write what isn't tested
Lens: gpu-rendering · Source: Dolphin
Principle: A second geometry pass to produce one more output (such as destination alpha) doubles vertex and raster work. Dual-source blending outputs both values from one pass. Similarly, writing depth from a pixel shader disables early-Z and costs bandwidth, so emit the depth write only when depth testing is enabled.
Why it is faster: —
Applies elsewhere: MRT (multiple render targets) in deferred shading, single-pass order-independent transparency, removing `gl_FragDepth` writes in WebGL shaders to keep early-Z.
Watch out: —
Seen in Dolphin: The D3D9 backend used a separate alpha pass. With dual-source support it now writes `ocol1` in `PixelShaderGen.cpp` and blends with `D3DBLEND_SRCCOLOR2`/`INVSRCCOLOR2` (`Render.cpp`), skipping the extra pass. The `per_pixel_depth` decision gained `&& bpmem.zmode.testenable`, so the shader writes depth only when the depth test is on. 40d919b352 2013-03 “Implement dual source blending to avoid unneeded alpha pass” · 4c22e1264e 2013-08 “PixelShaderGen: Do not write depth in pixel shader if depth testing (and thus writing) is not enabled”

### dolphin-140 — Move containers through queues instead of copying them
Lens: allocation · Source: Dolphin
Principle: Pushing a `std::vector` into a queue by const reference deep-copies its heap buffer, and popping by assignment copies it again. Perfect-forward on push and `std::move` on pop transfer the buffer pointer instead, with no allocation and no memcpy.
Why it is faster: —
Applies elsewhere: message queues between threads in any C++ server, Rust's default move semantics, transferable ArrayBuffers in `postMessage`.
Watch out: —
Seen in Dolphin: `Common::FifoQueue::Push(const T&)` became `template<typename Arg> Push(Arg&& t)` with `new T(std::forward<Arg>(t))`, and `Pop` uses `t = std::move(Front())`. Real-Wiimote reports, which are `std::vector<u8>`, are now moved through the queue. 71f4bf25a7 2013-04 “Make FifoQueue take advantage of rvalue references to avoid std::vector copies”

### dolphin-141 — Drop outgoing messages that wouldn't change the remote state
Lens: io · Source: Dolphin
Principle: If a peer's state is fully determined by a value you send, remember the last value sent and suppress any message that repeats it. That removes a steady stream of no-op traffic on a slow link.
Why it is faster: —
Applies elsewhere: MQTT retained-state dedup, suppressing unchanged telemetry samples, coalescing identical UI state pushes over WebSockets.
Watch out: —
Seen in Dolphin: A new `Wiimote::WriteReport` (`WiimoteReal.cpp`) tracks `m_rumble_state` and drops `WM_RUMBLE` reports whose rumble bit equals the current state. That removed the constant report streams that games such as Just Dance 2 and DKCR generate, since their speaker data is turned into rumble reports. 018282c2b9 2013-04 “Track the real wiimote rumble state to drop outgoing rumble reports with no effect”

### dolphin-142 — Let the hardware fault handler take the slow path
Lens: jit-interpreters · Source: Dolphin
Principle: If an access is almost always to plain memory but occasionally hits something special, emit only the fast direct access. Reserve address space so that the rare cases fault, and patch or redirect the faulting site in the signal handler. The common path then has no bounds check and no branch.
Why it is faster: —
Applies elsewhere: guard pages for stack growth, Wasm runtimes that drop bounds checks by reserving 8 GB of virtual address space, copy-on-write `fork`, and memory-mapped files that page in on demand.
Watch out: —
Seen in Dolphin: Before 2013 the x86-64 JIT used fastmem for loads only. `2a339c926e` extended it to stores: `SafeWriteRegToReg` emits a raw `MOV` into the 4 GB view, and `JitBackpatch.cpp` rewrites the faulting instruction into a call to a trampoline. Later commits made it work in MMU mode, first for loads (`43d56febc4`, “loads outside the area specified by the BAT are rare”), then for stores (`190312e1a6`). `0ff6ad5734` let the fault path raise the DSI exception, so the per-op `TEST Exceptions` check after a fastmem access could go. JitIL had fastmem stubbed out. Re-basing it on `Jitx86Base` cut the POV-Ray benchmark from 1985 s to 1316 s. 2a339c926e 2013-09 “Fastmem writes for x86-64” · 43d56febc4 2014-12 “JIT: use fastmem loads in MMU mode.” · 190312e1a6 2015-01 “MMU: enable fastmem stores” · 0ff6ad5734 2015-01 “MMU: handle exception checks in fastmem” · 6956df79b2 2014-09 “Fix Fastmem in JitIL for massive speed gains.”

### dolphin-143 — Record bookkeeping in a side table, materialize it only on the slow path
Lens: laziness · Source: Dolphin
Principle: Some state is needed only when something goes wrong, such as which instruction caused a fault. Don't write it on every operation. Keep a compile-time map from code address to that state, and have the rare slow path look it up and write it. The hot path loses one store per operation.
Why it is faster: —
Applies elsewhere: zero-cost exception tables, debug line tables in place of runtime line counters, and source maps in place of instrumented JavaScript.
Watch out: —
Seen in Dolphin: Every JIT store used to run `MOV [PC], compilerPC` first, so that a write that went to MMIO could report the right PC. Now `SafeWriteRegToReg` records `pcAtLoc[mov] = compilerPC`, and `TrampolineCache::GetWriteTrampoline` emits the PC store only inside the backpatched slow path. The commit measured about 3.5% overall on POV-Ray. a1655a0e3c 2014-08 “JIT: avoid saving the PC on every store”

### dolphin-144 — Move cold paths out of line, away from hot code
Lens: jit-interpreters · Source: Dolphin
Principle: Rarely taken branches such as exception exits and slow memory calls take up instruction-cache lines and decoder bandwidth even when they never run. Emit them into a separate code region and leave only one conditional jump in the hot stream. Hot code stays dense, so more of it fits in the L1I cache.
Why it is faster: —
Applies elsewhere: `__builtin_expect` and `[[unlikely]]`, GCC's `.text.unlikely` hot/cold splitting, PGO function layout, and moving error handling into `noinline` helpers.
Watch out: —
Seen in Dolphin: `FarCodeCache` (“a place to throw blocks of code we don't want polluting the cache”) was added to `Jit_Util.h`. In `Jit64::DoJit`, the external-exception and DSI checks now do `J_CC(CC_NZ)` into far code, which flushes registers with `FLUSH_MAINTAIN_STATE` and exits, then `SwitchToNearCode()`. The follow-up moved the MMIO and slow-memory fallbacks of `SafeLoadToReg` there too. Later, fastmem backpatching started `JMP`ing to freshly generated trampolines instead of `CALL`ing cached ones. That removed the return-address push and the NOP padding on the way back. c5381bae66 2014-08 “JIT: add "far code emitter" to reduce L1I cache pressure” · 54e26f64c6 2014-08 “JIT: use far code emitter in more places” · 2a8936312e 2015-01 “Fastmem: jump to trampolines instead of calling them”

### dolphin-145 — Pair calls with returns so the CPU's return predictor works
Lens: branching-fast-paths · Source: Dolphin
Principle: CPUs predict `RET` targets with a hardware return-stack buffer, but only when every `RET` matches a `CALL`. If generated code mirrors guest calls with real `CALL`/`RET` pairs and checks that the return address is correct, function returns are predicted almost perfectly. Without that, they go through an indirect lookup that mispredicts.
Why it is faster: —
Applies elsewhere: interpreters and coroutine libraries that keep call/return balanced, and the advice to avoid `push addr; ret` tricks, which desynchronize the RSB.
Watch out: —
Seen in Dolphin: For `bl`, `Jit64::WriteExit` now pushes the guest return address and `CALL`s the linked block or dispatcher. For `blr`, it compares `[rsp+8]` with LR and executes `RET` if they match. On a mismatch it resets RSP. This “saves a trip through the dispatcher and improves branch prediction.” Since only 4-byte-aligned addresses are ever pushed, a later commit dropped the `AND LR, ~3` on that path. b597ec3e08 2014-09 “Opportunistically predict BLR destinations using RET.” · 7388c62439 2014-10 “JIT: use BLR optimization to avoid anding LR with 0xFFFFFFFC”

### dolphin-146 — Chain compiled blocks directly, and check the cheap flag first
Lens: jit-interpreters · Source: Dolphin
Principle: When a compiled block exits to a known address that is already compiled, patch in a direct jump. Control then skips the central dispatcher's hash lookup and indirect branch. Check whether linking is enabled before doing the lookup, so the disabled case costs nothing.
Why it is faster: —
Applies elsewhere: trace linking in LuaJIT and PyPy, inline caches that patch call sites, and threaded-code interpreters.
Watch out: —
Seen in Dolphin: `Jit64::WriteExit` called `GetBlockNumberFromStartAddress` before checking `jo.enableBlocklink`. The order is now reversed. Later, block linking was enabled in MMU mode. Only blocks whose instruction fetch raised an ISI are marked `memoryException` and left unlinked. d5e40bfb01 2013-10 “Don't waste time looking up the block if enableBlocklink is off anyway.” · 110d079c1e 2014-12 “Enable block-linking with MMU. Blocks that cause an ISI exception are not linked.”

### dolphin-147 — Specialize dispatch at code-generation time when the target is constant
Lens: jit-interpreters · Source: Dolphin · Also: Flycast
Principle: If an address or key is known when code is generated, resolve the handler then. Emit a constant, a direct masked load, or a direct call in place of a runtime table walk. A visitor over handler kinds (constant / direct / complex) makes this systematic.
Why it is faster: —
Applies elsewhere: compile-time route tables in web frameworks, devirtualization when the concrete type is known, and generated register-access code in embedded HALs.
Watch out: —
Seen in Dolphin: MMIO handlers were reworked into typed `ReadHandlingMethod`s. `MMIOReadCodeGenerator` visits the handler for a constant-propagated address. `VisitConstant` emits a `MOV imm`, `VisitDirect` emits a `MOVZX/MOVSX` from the backing variable (with an `AND` only if the mask isn't all-ones), and only `VisitComplex` calls the lambda. 8802770a85 2014-02 “JitUtil: Optimize MMIO loads by generating code to directly access a given MMIO.” • Also seen in Flycast (flycast-104): The ARM64 backend extended immediate-address `readm` to 64-bit reads. It checks the address up front and either emits `Ldr(x0, ptr)` straight from host memory or calls the resolved handler directly, twice for 64-bit when it is an I/O handler. 419785b92 2019-09 “arm64: support for 64b immediate memory reads”

### dolphin-148 — Store the result itself, not the flags derived from it
Lens: branching-fast-paths · Source: Dolphin
Principle: If several flags are cheap to derive from a value (zero, positive, negative), store the value lazily in a form where each flag test is a single instruction, rather than computing and packing flags with branches. Choose the encoding so that the setters are also branch-free.
Why it is faster: —
Applies elsewhere: lazy condition-code evaluation in interpreters, storing a comparison result as a signed int (`strcmp`-style) rather than three booleans, and sign-bit tricks in sort keys.
Watch out: —
Seen in Dolphin: Each PPC CR field became a 64-bit `cr_val` holding the sign-extended result. EQ means the low 32 bits are 0, GT means `(s64)v > 0`, and LT means bit 62 is set. `Helper_UpdateCR0` went from an if/else chain to one sign-extend. Later, `GetCRFieldBit`/`SetCRFieldBit` became branchless with `BT`/`BTR`/`SETcc`, and `mfcr` assembles the SO and LT bits from an 8-entry byte table: `MOVZX EAX, m_flagTable[cr_val>>61]`. 0ff1481494 2014-05 “Optimize PPC CR emulation by using magic 64 bit values” · c936984c96 2014-08 “JIT64: faster + branchless crxxx” · 8c71703098 2014-08 “JIT64: faster implementation of mfcr”

### dolphin-149 — Skip computing outputs that are overwritten before anyone reads them
Lens: compiler-build · Source: Dolphin
Principle: Run a backward liveness pass over a block. A side effect that a later instruction overwrites before any read is dead, so don't generate it. If the very next consumer can read the value straight from a host register or flag, hand it over there rather than via memory.
Why it is faster: —
Applies elsewhere: dead-store elimination in compilers, skipping intermediate DOM writes when a later write replaces them, and coalescing redundant state updates in a reducer.
Watch out: —
Seen in Dolphin: `PPCAnalyst` now computes `wantsCA` per op (instruction tables gained `FL_SET_CA`/`FL_READ_CA`), and `JitSetCAIf` calls are guarded by it. The commit calls this “very common in the case of srawix”. The follow-up added `wantsCAInFlags`. When the next op consumes carry, `FinalizeCarry` leaves it in the x86 carry flag (`js.carryFlagSet`) and doesn't write XER. An earlier commit replaced branchy carry setting with `JitSetCAIf(cc)` (`SETcc`), because carries are “common and unpredictable”. 805be80f12 2014-08 “JIT64: Optimize carry handling” · bea2504a51 2014-08 “JIT64: optimize carry calculations” · 45d84605a9 2014-09 “JIT64: optimize carry calculations further”

### dolphin-150 — Fuse common instruction pairs into one host operation
Lens: compiler-build · Source: Dolphin
Principle: Look one instruction ahead for idioms the source ISA spells in two steps but the target does in one. Emit the fused form and skip the second instruction. Reordering harmless instructions next to their partner creates more of these pairs.
Why it is faster: —
Applies elsewhere: peephole passes in compilers, fused multiply-add, SQL planners combining filter and scan, and bytecode superinstructions.
Watch out: —
Seen in Dolphin: `lbz` followed by `extsb` on the same register becomes one `MOVSX` (“PPC has no 8-bit sign-extended load”). Two adjacent `mftb` reads of TU/TL share one timebase computation (with a related change, “~1% faster overall on F-Zero GX”). `fcmpx` followed by `cror` is folded into the compare's output table. `PPCAnalyst` gained `REORDER_CROR`, which moves `cror` up next to the `fcmp`. de86d2003a 2014-09 “JIT: merge lbz + extsb” · 6875d911f1 2014-08 “JIT: merge paired timebase reads where possible” · 97fba41860 2014-10 “JIT: merge fcmpx and cror”

### dolphin-151 — Strength-reduce arithmetic on known constants
Lens: algorithms · Source: Dolphin
Principle: When one operand is a compile-time constant, replace the general operation with a cheaper special case: shift or LEA for powers of two and small multipliers, XOR/NEG for 0 and −1, zero-extending moves for masks of 0xFF/0xFFFF, and a 64-bit multiply plus shift for an unsigned high-half multiply.
Why it is faster: —
Applies elsewhere: what every optimizing compiler does for `x*8` and `x/16`, by hand in shader generators and SIMD kernels, and division by a constant via a multiply by a magic reciprocal.
Watch out: —
Seen in Dolphin: Factoring out `Jit64::MultiplyImmediate` covered 0, −1, powers of two (`SHL` or `LEA [a*2^n]`) and 3/5/9 (`LEA [a+a*k]`). `AndWithMask` and `rlwinmx` use `MOVZX` for byte/half extraction. `mulhwu` uses `IMUL r64; SHR 32`. In generated pixel shaders, the TEV scale table went from `*2`, `*4`, `/2` to `<<1`, `<<2`, `>>1`. 58dc802ce2 2014-08 “JIT64: optimize multiplication by immediate constants” · 41c3dde737 2014-08 “JIT64: optimize rlwinmx/rlwinix and friends” · a666bb6bf6 2014-10 “JIT: optimize mulhwu” · 8b8bb04fd3 2014-02 “PixelShaderGen: Use bit shifts instead of multiplications as a small optimization.”

### dolphin-152 — Pick the shortest encoding and the forms the decoder fuses
Lens: jit-interpreters · Source: Dolphin
Principle: Code density matters for the I-cache and the decoders. Use sign-extended 8-bit immediates when the value fits, `XOR r,r` to zero a register, `TEST r,r` rather than `CMP r,0`, and multi-byte NOPs instead of runs of single-byte ones. Put the flag-setting instruction directly before its branch so the pair can macro-fuse.
Why it is faster: —
Applies elsewhere: assembler relaxation passes, size-optimized firmware (`-Os`), and the alignment-padding choices in LLVM's MC layer.
Watch out: —
Seen in Dolphin: `x64Emitter`'s `WriteNormalOp` now emits the `simm8` opcode whenever a 16- or 32-bit immediate fits (“a nice alternative than trying to do it throughout the JIT”). `XEmitter::NOP(size)` emits the recommended `0F 1F` forms up to 11 bytes. `JitRegCache` loads a zero immediate with `XOR`, and `cmpXX` against 0 uses `TEST`. The downcount and BLR compares were reordered so that the `SUB`/`CMP` sits right before the `J_CC`. 75b3e425fd 2014-08 “x64Emitter: optimize immediate sizes” · 957649b7af 2014-04 “Optimize NOPs” · ad51fc7c4b 2014-08 “JIT64: use xor instead of mov for loading a zero regcache immediate” · 61af91ff16 2014-08 “JIT64: Optimize cmpXX” · 5b5e462200 2014-10 “JIT: reorder blr comparisons”

### dolphin-153 — Fold address arithmetic into the addressing mode
Lens: jit-interpreters · Source: Dolphin
Principle: The x86 `[base + index*scale + disp]` form, and `LEA`, compute an add-and-scale in one instruction without touching flags. Use them instead of separate `MOV`+`ADD`. Treat an operand that is a known constant as a displacement.
Why it is faster: —
Applies elsewhere: compilers lower `a[i+1]` this way. In hand-written SIMD loops, keep pointer+offset in the addressing mode instead of incrementing several pointers.
Watch out: —
Seen in Dolphin: In `Jit64::lXXx`, a reg+reg load whose `b` is an immediate is treated as constant-offset mode. The address is computed with `LEA ABI_PARAM1, [ra+offset]` or `LEA [ra+rb]`, or the offset is folded straight into the load. Update forms with a zero offset no longer write back. `psq_l`/`psq_st` got the same change. 043256449e 2014-08 “Jit64: some load/store optimizations” · 7ab820c6f8 2014-07 “JIT: Various JitAsmCommon optimizations”

### dolphin-154 — Avoid false dependencies and domain-crossing penalties
Lens: simd · Source: Dolphin
Principle: Some instructions merge into the destination register, such as `MOVSD reg,reg`, which keeps the upper half. They create a dependency on the register's old value even when you don't care about it. Use full-register moves like `MOVAPD` instead. Keep float data in float-domain instructions to avoid the bypass delay between the integer and float SIMD units.
Why it is faster: —
Applies elsewhere: zeroing a register before `cvtsi2sd` or `popcnt`, `vzeroupper`, and choosing `andps` over `pand` for float masks.
Watch out: —
Seen in Dolphin: In `Jit_FloatingPoint.cpp`, `MOVSD XMM0, fpr.R(b)` was replaced by `MOVAPD` (“Avoids false dependencies”). In `JitAsmCommon`, quantized load/store routines use float-domain instructions “where appropriate (it's a cycle faster on CPUs with arithmetic unit forwarding penalties)”, plus SSE4 instructions when available. Paired-single loads use `CVTSI2SS` in place of `MOVD`+`CVTDQ2PS`. 8d039a77af 2014-10 “JIT: minor cleanup, avoid MOVSD where MOVAPD will do” · 7ab820c6f8 2014-07 “JIT: Various JitAsmCommon optimizations” · 2ae6f13d22 2014-09 “JIT: use cvtsi2ss in paired singles”

### dolphin-155 — Use non-destructive three-operand forms to drop copies
Lens: simd · Source: Dolphin
Principle: Two-operand SSE overwrites a source register, so keeping both inputs costs a `MOV` first. AVX's VEX encoding takes a separate destination. Emitting it when available removes those copies and the temporary registers they need.
Why it is faster: —
Applies elsewhere: compiling with `-mavx` so compilers emit VEX forms, and ARM/RISC-V three-operand ALU ops, which make register allocation easier.
Watch out: —
Seen in Dolphin: `fp_tri_op` and `tri_op` now take both an `avxOp` and an `sseOp`, routed through a new `avx_op` helper in `Jit_Util.cpp`, which picks `VADDSD d,a,b` etc. when the CPU has AVX. The commit reports a 10–20% smaller compiled block size for float-heavy blocks. An earlier change had already removed a dead branch that needed two scratch XMM registers. bf014636c8 2014-10 “JIT: add more AVX support, reduce redundant XMM moves” · c234dc97c0 2013-10 “Jit64: optimize floating-point/paired-single arith”

### dolphin-156 — Track value properties so you can skip normalization
Lens: compiler-build · Source: Dolphin
Principle: If analysis can prove a value is already in a canonical form (already rounded, top half equal to bottom half, high bit clear), the normalizing step is a no-op and can be dropped. Track these facts per register through the block.
Why it is faster: —
Applies elsewhere: skipping re-escaping of strings already marked safe in a template engine, not re-validating input that came out of a validated type, and nullability tracking that elides checks.
Watch out: —
Seen in Dolphin: `PPCAnalyst` gained `fprIsSingle` and `fprIsDuplicated`. `fmul` skips `Force25BitPrecision` when its input is known single. When both inputs are duplicated singles, `fp_arith` uses packed `ADDPD/MULPD` and drops the trailing `MOVDDUP`, except for divides and on Atom CPUs, where packed is slower. Stores skip the double-to-single conversion if the value came from a single op. `frsp` got the same packed path later. `andi_rc` skips sign-extending the CR value when the mask clears bit 31. 7df50b0710 2014-10 “JIT: skip weird fmul rounding if the input is known to be single precision” · 72c96c20d3 2014-10 “JIT: more optimizing of float ops based on known input characteristics” · ece0ef4ca8 2015-05 “Jit64: add packed optimization to frsp” · af471d0a84 2014-09 “JIT: optimize andi(s)_rc”

### dolphin-157 — Speculate on observed state, guard it, recompile on failure
Lens: jit-interpreters · Source: Dolphin
Principle: If a configuration value is almost always the same at runtime, generate code specialized for that value. Add one cheap check at entry. If the check fails, invalidate and recompile the general version, and remember not to speculate there again.
Why it is faster: —
Applies elsewhere: V8/HotSpot type speculation with deoptimization, shape checks in inline caches, and query plans cached under parameter assumptions.
Watch out: —
Seen in Dolphin: Paired-single loads/stores are dequantized according to a GQR register. If a block uses only one GQR and it is zero at compile time, `Jit64::DoJit` sets `assumeNoPairedQuantize`, emits a check that the GQR is still zero, and inlines plain float loads/stores, which can use fastmem. On a mismatch, the block's address goes into `pairedQuantizeAddresses` and the block is recompiled without the assumption. 8237004448 2015-01 “JIT: optimize for the common case of unquantized psq_l/st”

### dolphin-158 — Narrow the slow path to exactly the inputs that need it
Lens: branching-fast-paths · Source: Dolphin
Principle: If an expensive fallback is needed only for a small, easily identified input class, test for that class with a couple of integer ops and send everything else down the fast path. A slightly wider test is fine if it saves an instruction. Spurious diversions are cheap when they are rare.
Why it is faster: —
Applies elsewhere: ASCII fast paths in UTF-8 decoders, small-integer fast paths in bignum libraries, and handling denormals or NaNs out of line in math kernels.
Watch out: —
Seen in Dolphin: The double-to-single conversion in `ConvertDoubleToSingle` had several SSE paths (`PTEST` or an emulated version) to get NaN bits right. The commit notes that “the only values that need to be sent through the slow x87 path are denormals.” The new code does `MOVQ rax,xmm; SHR rax,55; SUB al,0x6D; CMP al,3; JBE x87` and otherwise runs a single `CVTSD2SS`. POV-Ray went from 708 s to 678 s. 59c1a46ab1 2014-08 “JIT: faster PPC_FP code”

### dolphin-159 — Let liveness drive register-cache spills and loads
Lens: compiler-build · Source: Dolphin
Principle: A register allocator that knows which values are still needed can write a dead value back once, at its last use, rather than at every branch. It can also skip loading a register whose old value is about to be overwritten. Both shrink code and free registers.
Why it is faster: —
Applies elsewhere: linear-scan register allocation, and in any write-back cache, discarding a cached entry once you know it won't be read again.
Watch out: —
Seen in Dolphin: `PPCAnalyst` computes `gprInUse`/`fprInUse` for the rest of the block. `Jit64::DoJit` now calls `StoreFromRegister` for registers outside that set, which “should dramatically reduce code size in the case of blocks with lots of branches.” Branch merging flushes unused compare operands before the branch, so they aren't flushed on both sides. Merged int+branch code no longer forces a full flush (`FLUSH_MAINTAIN_STATE`). `BindToRegister(d, false)` skips loading a destination that is only written. f103234e2b 2014-09 “JIT: flush a register if it won't be used for the rest of the block” · fb0960f0ee 2014-10 “JIT: flush unused registers during branch merges” · 97beea1963 2014-08 “JIT: Don't needlessly flush register state in merged integer+branch code” · 710a0ff435 2013-11 “Jit64: don't unnecessarily load registers”

### dolphin-160 — Use combined load-and-transform instructions when the CPU has them
Lens: simd · Source: Dolphin
Principle: Detect optional ISA extensions at startup, and when present emit the single instruction that replaces a common pair: `MOVBE` for load+byteswap, `PDEP`/`ANDN` for bit scatter, `BLENDPD` for a lane merge. Keep the generic sequence as a fallback.
Why it is faster: —
Applies elsewhere: network stacks using `movbe` for big-endian headers, function multiversioning (`target_clones`), and runtime dispatch in codecs and hash libraries.
Watch out: —
Seen in Dolphin: `EmuCodeBlock::LoadAndSwap`/`SwapAndStore` emit `MOVBE` if `cpu_info.bMOVBE` is set, otherwise `MOV`+`BSWAP`. They replaced the hand-written swaps in `JitAsmCommon`. The x64 vertex loader converts RGB565 and RGBA4444 colors with `PDEP` into immediate masks on BMI1+BMI2 CPUs, which replaced a shift/AND/OR chain. `ps_sum1` uses `BLENDPD` on SSE4.1. fcbe265adc 2014-03 “Jit_Util: Provide two util functions to load/store and swap values, using MOVBE internally when possible.” · 6cb42859d4 2014-03 “JitAsmCommon: Use MOVBE everywhere it matters (!x86 only code, !old CPU support code).” · bc5cf10ad5 2015-01 “VertexLoaderX64: optimize color conversions” · 05a55de08f 2015-05 “Jit64: optimize ps_sum”

### dolphin-161 — Generate a decoder specialized to each input format
Lens: jit-interpreters · Source: Dolphin
Principle: If data arrives in one of many layouts selected by a descriptor, compile a straight-line decoder for each descriptor seen and cache it. The per-record loop then has no format dispatch, no function-pointer calls, and every offset is a constant.
Why it is faster: —
Applies elsewhere: generated parsers in protobuf and Cap'n Proto, JIT-compiled regexes, serializer codegen (simdjson-style or `System.Text.Json` source generators), and pixel-format converters in graphics drivers.
Watch out: —
Seen in Dolphin: `VertexLoaderX64` replaced the old chain of `WriteCall(PosMtx_Write)`-style function pointers. For each vertex format it emits code. `GetVertexAddr` handles index8/index16 with `MOVZX`+`ROR`, then `IMUL` by the array stride, and branches out for skipped (index −1) positions. `ReadVertex` runs a `MOVD/MOVQ/MOVDQU` sized to the attribute, then `PSHUFB`, then `PSRAD`, `CVTDQ2PS` and `MULPS`, all with constant source and destination offsets. e82e9f97f5 2015-01 “VertexLoader: add new JIT”

### dolphin-162 — Table-driven byte shuffles for byte-swap and widening in one step
Lens: simd · Source: Dolphin · Also: ares
Principle: A single `PSHUFB` with a precomputed control mask can byte-swap, reorder, and place narrow fields in the high bytes of 32-bit lanes all at once. An arithmetic right shift then sign-extends. Index the masks by (format, count) so one code path covers every integer width.
Why it is faster: —
Applies elsewhere: endian conversion of packed arrays, unpacking RGB24 to RGBA32, base64 and UTF-8 SIMD codecs, and network-order field extraction.
Watch out: —
Seen in Dolphin: An earlier commit added SSE paths in `VertexLoader_Position/TextCoord/Normal.cpp` for more formats, with the scale factors duplicated into aligned `posScale[4]` and `tcScale[8][2]` so that one `MULPS` handles all components. The commit reports “~35-45% faster NFS:HP2.” The JIT then generalized this into `shuffle_lut[5][3]` and `scale_factors[32]`. Because SIMD loads overread, the FIFO buffer is now padded by 4 bytes. 3ddf82a318 2014-11 “Vertex Loader: SSE implementations of more position/texcoord/normal formats” · e82e9f97f5 2015-01 “VertexLoader: add new JIT” · 3effd08843 2014-12 “FIFO: allocate slightly more for SIMD vertex loader overreads” • Also seen in ares (ares-135): LUV/LPV/SUV/SPV dropped their `E==0` special cases and always call SIMD helpers with a pointer into `alignas(16) simdLHVSource[16][8]`, `simdSHVSource[16][32]`. A new `mov128` encoder helper copies 16 bytes with one SIMD load/store pair. 59a0a898e 2026-04 “n64: add LHV/SHV in RSP JIT, and refactor LUV/LPV/SUV/SPV to always use SIMD” · e3f4d6dc0 2026-04 “nall: add mov128 and mem0 for absolute addressing” • Also seen in ares (ares-157): `r128::operator()(index)` holds a static array of 16 `_mm_set_epi8` masks (01234567, 00224466, ..., 77777777) and returns `_mm_shuffle_epi8(v128, shuffle[index])`. The SISD fallback is a 16-case switch. current code: ares/n64/rsp/interpreter-vpu.cpp

### dolphin-163 — Copy globals into locals inside inner loops
Lens: compiler-build · Source: Dolphin · Also: Flycast
Principle: A global or member that is read in a loop which also writes through pointers must be reloaded every iteration, because the compiler can't rule out aliasing. Read it into a `const` local before the loop, keep the output pointer in a local, and write it back once.
Why it is faster: —
Applies elsewhere: caching `this.length` or a config field before a hot JS/Java loop, and `restrict` pointers in C image kernels.
Watch out: —
Seen in Dolphin: Vertex loader position and texcoord functions read `posScale`/`tcScale[tcIndex]` and wrote through the global `VertexManager::s_pCurBufferPointer` for each component. They now use `auto const scale = posScale;` and a local `dst`, with a new `DataWrite(u8*& dst, T)` overload, and store the pointer back after the loop. Later, the `DataReader` rewrite put the read cursor in a `__forceinline` class held in a local. c613868f57 2014-01 “VertexLoader: load scale factor as const, this will save some assembler instructions” · 0b97b33ceb 2014-01 “VertexLoader: inline destionation buffer” · 27f984c02e 2014-11 “VideoCommon: rewrite DataReader” • Also seen in Flycast (flycast-116): In the Vulkan drawers, `pvrrc.verts.head()` and `pvrrc.idx.head()` are now read once into `vtx_base`/`idx_base`. Loops became `for (const PolyParam *pp = list.head(); pp != pp_end; pp++)`, which needed `LastPtr()` to be made `const`. 6bfde7651 2020-02 “vk: avoid calling List::head() in loops”

### dolphin-164 — Force inlining, or turn runtime flags into template parameters
Lens: compiler-build · Source: Dolphin
Principle: Small accessors and hot lookup helpers only pay off once inlined, and some compilers silently refuse. Mark them force-inline. If such a helper branches on a mode argument that is constant at every call site, make the mode a template parameter so each instantiation compiles without the branch.
Why it is faster: —
Applies elsewhere: `__attribute__((always_inline))` on SIMD wrappers, C++ policy templates, and Rust const generics for mode flags.
Watch out: —
Seen in Dolphin: `BitField<>`'s `operator=`, `operator T` and `GetMask` got `__forceinline`. MSVC wasn't inlining `ReadFromHardware`, `WriteToHardware`, `LookupTLBPageAddress` or `TranslatePageAddress`, so they were force-inlined. Then `TranslateAddress<XCheckTLBFlag>` and `ReadFromHardware<flag, T>` were templated on the lookup type, and an early segment check was added for the MMU path. 12841928df 2014-04 “BitField: Optimize generated assembly by forcing inlining.” · 92a0716f65 2014-12 “Memory: forceinline read/write hardware functions, TLB/page functions” · c2ed29fe0d 2014-12 “MemmapFunctions: various MMU optimizations”

### dolphin-165 — Make always-constant parameters actual constants
Lens: compiler-build · Source: Dolphin
Principle: If a function is only ever called with one value for some argument, remove the argument and use a local constant. The compiler can then fold size checks and unroll copies of a known length.
Why it is faster: —
Applies elsewhere: fixed-size `memcpy` becoming plain moves, specializing a packet handler for a fixed MTU, and `constexpr` buffer sizes.
Watch out: —
Seen in Dolphin: `ReadDataFromFifo(u8*, u32 len)` and `ReadDataFromFifoOnCPU` were always called with 32. The parameter became `size_t len = 32;` “to enable optimizations.” 87a95727cd 2014-09 “ReadDataFromFifo is always called with len = 32. Remove the parameter to enable optimizations.”

### dolphin-166 — Break a serial dependency chain with independent accumulators
Lens: simd · Source: Dolphin
Principle: If each step of a loop depends on the previous one, as with a running CRC, throughput is limited by the instruction's latency, not its throughput. Keep N independent accumulators over interleaved data and combine them at the end, so N operations are in flight at once.
Why it is faster: —
Applies elsewhere: multiple partial sums in dot products, multi-lane hashes such as xxHash, and parallel CRC folding.
Watch out: —
Seen in Dolphin: `GetCRC32` in `Hash.cpp` went from a single `h = _mm_crc32_u64(h, data[0])` chain to `h[4]`, consuming four words per iteration and combining with `_mm_crc32_u64(h[0]+h[1]+h[2]+h[3], tail)`. The commit reports “20-30% faster texture cache hashing.” 15a4bccb73 2014-10 “Hash: unroll CRC loop, since CRC32 typically has nontrivial latency”

### dolphin-167 — Use a fast non-cryptographic hash for content keys
Lens: algorithms · Source: Dolphin · Also: ares, Flycast
Principle: To identify blobs for caching, use a hash designed for throughput, such as a 64-bit xxHash, rather than a homemade or cryptographic one. Collision resistance against attackers isn't needed, and these hashes run near memory bandwidth.
Why it is faster: —
Applies elsewhere: build-system content hashing, deduplication in backup tools, asset caches, and the hashing in Zstandard and LZ4 frames.
Watch out: —
Seen in Dolphin: Custom texture lookup in `HiresTextures.cpp` replaced `GetHashHiresTexture` with `XXH64(texture, size, 0)` for both texture and TLUT, after xxhash was vendored into Externals. 1d0557a5e6 2015-01 “CustomTexture: use xxhash” • Also seen in ares (ares-050): The RSP replaced a `for(offset : range(256)) crc.input(imem.read<Byte>(...))` loop with `XXH3_64bits(self.imem.data + address, 256)`, using `#define XXH_INLINE_ALL` in `n64.hpp`. The commit calls it "significantly faster and stronger" than CRC32. 9985c693b 2022-08 “n64: use xxh3 instead of crc32 for rsp cache” • Also seen in Flycast (flycast-098): The idle-loop block detector compared a SHA-1 of the block's code (the `idle_hash` strings) against known BIOS and game loops. It now uses xxhash, and the signature strings got shorter. Earlier, the hash was skipped altogether when the MMU is on (`if (!mmu_enabled() && strstr(idle_hash, blk->hash(...)))`). 0595bae85 2019-09 “dyna: use xxhash instead of sha1 to hash code blocks” · 2434d0183 2019-05 “wince: better sh4 instr cycle counting. Don't hash blocks if mmu on” • Also seen in Flycast (flycast-192): The bundled xxHash was updated to 0.8.3, and the achievements image-cache key went from `XXH64(url, strlen(url), 13)` to `XXH3_64bits(url, strlen(url))`. ea6d5f873 2025-03 “Update xxHash to 0.8.3 + Use faster algorithm”

### dolphin-168 — Replace linear scans and ordered maps with hash lookups
Lens: algorithms · Source: Dolphin
Principle: If you only need lookup by key and never order, a hash map is O(1) and iterates faster than a red-black tree. A pool searched linearly for a matching config should be a hash multimap keyed by that config.
Why it is faster: —
Applies elsewhere: object pools keyed by size class, connection pools keyed by host, and swapping `TreeMap` for `HashMap` in Java when order is irrelevant.
Watch out: —
Seen in Dolphin: The display-list cache maps (`DLMap`, `VDLMap`) and `s_native_vertex_map` switched from `std::map` to `std::unordered_map`. The latter got a `std::hash<PortableVertexDeclaration>` using `HashFletcher`. The texture pool became `std::unordered_multimap<TCacheEntryConfig, ...>`, so `AllocateTexture` does a `find(config)` rather than looping over a vector comparing configs. 390760bd75 2013-09 “Perf: Use unordered_map instead of map in very slightly hot function.” · 90638c6806 2014-08 “Switch to an unordered_map as a micro-optimization.” · 8565f02699 2015-01 “TexCache: use an unordered_multimap for the tex pool”

### dolphin-169 — Pool expensive resources and expire them by age
Lens: allocation · Source: Dolphin
Principle: Creating and destroying GPU textures or other driver objects is costly and often repeats every frame. Put freed objects in a pool keyed by their parameters, reuse a match on allocation, and delete entries unused for N frames. Once pooling makes reallocation cheap, the complex reuse-in-place logic can go.
Why it is faster: —
Applies elsewhere: database connection pools, `ArrayPool<T>`, recycled views in mobile list UIs, and buffer pools in network servers.
Watch out: —
Seen in Dolphin: `TextureCache::AllocateRenderTarget`/`FreeRenderTarget` reuse EFB-copy render targets and destroy them after `RENDER_TARGET_KILL_THRESHOLD = 3` frames (“avoids creating and destroying render targets every frame, which is a significant CPU overhead”). The pool later covered all textures (`AllocateTexture`/`FreeTexture`). `Load` then dropped its in-place reuse checks: “As we pool them now, freeing and reallocating them is quite fast.” d8d9bc8c6c 2014-06 “Render: Implemented simple render target pool” · 4639d3b1bc 2015-01 “TexCache: also incude textures within the render target pool” · 9f13a77799 2015-01 “TexCache: don't try to aggressive reuse the entry”

### dolphin-170 — Cache derived objects next to the cache entry that produces them
Lens: laziness · Source: Dolphin
Principle: If every cache hit is followed by a second lookup for a derived object, store the derived object in the same entry. One lookup then returns both, and the hot path skips the second map search.
Why it is faster: —
Applies elsewhere: memoizing a compiled regex together with its source pattern, and storing an ETag next to a cached HTTP response.
Watch out: —
Seen in Dolphin: `VertexLoaderManager`'s cache changed from `VertexLoader*` to `std::pair<VertexLoader*, NativeVertexFormat*>`, both in the per-attribute-group slots `s_VertexLoaders[8]` and in `s_VertexLoaderMap`. This fixed a performance regression. a64b0bf499 2014-08 “VertexLoader: cache NativeVertexFormat”

### dolphin-171 — Set-associative indexing instead of scanning a history
Lens: algorithms · Source: Dolphin
Principle: A lookup cache that scans the last N entries costs O(N) on a miss. Index it by the low bits of the key into a small set (2-way here) and keep an LRU bit, as hardware caches do. Then every lookup is a constant amount of work.
Why it is faster: —
Applies elsewhere: software TLBs in hypervisors, CPU-style direct-mapped caches for memoizing hot functions, and the hashed and bucketed slots in a CDN edge cache.
Watch out: —
Seen in Dolphin: `LookupTLBPageAddress` in `MemmapFunctions.cpp` had two variants. The default looped up to 128 entries back from `itlb_last`/`dtlb_last`. The disabled `FAST_TLB_CACHE` indexed a 2-way set and flipped `TLB_FLAG_MOST_RECENT`. The fast variant is now the only one, and it updates LRU only for game accesses, not `FLAG_NO_EXCEPTION` debugger probes. 7b44e79356 2014-10 “Changed the FAST_TLB_CACHE to update its state only if the TLB cache is accessed by the game (not by Dolphin).”

### dolphin-172 — Transform the search key once, not every element
Lens: algorithms · Source: Dolphin
Principle: If the data is stored in a foreign encoding (byte-swapped, shifted, packed), convert the search key into that encoding once and compare raw words, rather than decoding each element. Combining several field tests into one precomputed word turns multiple compares into one.
Why it is faster: —
Applies elsewhere: searching network-order buffers with an `htonl`'d constant, case-folding the needle once in a case-insensitive search, and comparing packed composite keys as one integer.
Watch out: —
Seen in Dolphin: `IsWiiElf` byte-swapped every code word before testing it against the HID4 pattern. Now it swaps `HID4_pattern`/`HID4_mask` once and tests raw words. The MMU's page-table walk does the same. It builds `pte1 = bswap((VSID<<7) | api | PTE1_V)`, ORs in `PTE1_H` for the secondary hash, and compares each PTEG slot with a single `pte1 == *(u32*)&base_mem[addr]`, instead of decoding the V, H, VSID and API fields separately. b7d4ff679a 2014-11 “IsWiiElf: Optimise inner loop.” · 1ee83e332e 2015-01 “MMU: optimize PTE lookup code”

### dolphin-173 — Bitsets with popcount and count-trailing-zeros iteration
Lens: algorithms · Source: Dolphin · Also: PostgreSQL
Principle: Represent small sets as an integer bitmask. Count members with `popcount`, and iterate over only the set bits with `ctz` plus clearing the lowest bit. This replaces bool arrays and 0..N loops, uses less memory, and costs time proportional to the number of members.
Why it is faster: —
Applies elsewhere: chess bitboards, entity-component signatures in game engines, free-slot maps in allocators, and the fd sets in event loops.
Watch out: —
Seen in Dolphin: `Common/BitSet.h` added `BitSet32/64` with `Count()` (`__builtin_popcount`, or a bit-hack on MSVC) and a range-for iterator over set bits (`__builtin_ctz`/`_BitScanForward`). It replaced bool arrays in `PPCAnalyst` and the register-cache masks. `ABI_PushRegistersAndAdjustStack` now iterates `for (int r : mask & ABI_ALL_GPRS)` rather than testing all 16 bits twice. b6a7438053 2014-10 “Add BitSet and, as a test, convert some JitRegCache stuff to it.” · eb7f4dac50 2014-10 “Convert registersInUse to BitSet.” • Also seen in PostgreSQL (postgresql-009): c15a4c2aef3 2003-02-08 Planner relation sets moved from integer Lists to Bitmapsets, "faster as well as less error-prone"; 145014f8115 did the same for executor param lists and bb34970f918 for duplicate-column checks, avoiding O(N²). • be86e3dd5b4 2012-07-19 The checkpointer's fsync-request table keeps one entry per relation with a bitmap of segments instead of one entry per segment, so forgetting a relation is O(1); this fixed O(N²) under many TRUNCATE/DROP operations. • 3373c715535 2019-07-21 `RelOptInfo.eclass_indexes` stores which EquivalenceClasses mention each relation, so planning no longer scans every EC per relation; fa86238f1ea added an `ec_relids` prefilter that saved about 40% of planner runtime on a real query. • 475dbd0b718 2021-08-03 A Bitmapset of live partitions lets planning skip pruned ones. • e9e26b5e716 2022-11-16 "Multibitmapsets" (a list of Bitmapsets, effectively a 2-D bit array) replace `list_intersect` over Var lists in outer-join reduction. • a8c09daa8bb 2023-07-04 `bms_is_subset` and `bms_equal` fast-path on word count because sets never carry trailing zero words; significant planner speedups at about 100 or more partitions. 00b41463c21 made the empty set canonically NULL. • 0f4c170cf3b 2026-03-30 PlannedStmt carries bitmapsets of result-relation and row-mark indexes so "is this relation modified?" is a bit test.

### dolphin-174 — Upload shader constants as one buffer, only when dirty
Lens: gpu-rendering · Source: Dolphin
Principle: Setting uniforms one by one per draw costs a driver call each. Mirror all constants in a CPU-side struct, set a dirty flag when any of them changes, and when it's set, stream the whole struct into a ring buffer and bind a range. Draws with no constant changes then cost nothing.
Why it is faster: —
Applies elsewhere: UBOs and constant buffers in game engines, WebGL2 uniform blocks, and batching state changes into one command-buffer write.
Watch out: —
Seen in Dolphin: `ProgramShaderCache::UploadConstants` now checks `PixelShaderManager::dirty || VertexShaderManager::dirty`. If either is set, it uploads both constant structs through the stream buffer and calls `glBindBufferRange` for each. The per-constant `glUniform4fv` path is kept only for drivers without UBOs. `PixelShaderManager`'s setters each mark the struct dirty when they change it. D3D moved to the same shared VideoCommon handling. 7c14463d11 2013-10 “ogl: implement useful constant buffer upload” · 2754c1132e 2013-10 “VideoCommon: reimplement Dirty() in PixelShaderManager”

### dolphin-175 — Stream dynamic data through pinned or persistently mapped memory
Lens: gpu-rendering · Source: Dolphin
Principle: For data rewritten every frame, a buffer the GPU reads directly from client memory avoids the driver's staging copy and the map/unmap round trips. Enable it per buffer type, and disable it only where a known driver bug exists rather than globally.
Why it is faster: —
Applies elsewhere: `GL_ARB_buffer_storage` persistent mapping, Vulkan host-visible ring buffers, and zero-copy DMA buffers in video pipelines.
Watch out: —
Seen in Dolphin: `StreamBuffer` had excluded index buffers from `PINNED_MEMORY` on all AMD drivers. The condition became `!(HasBug(BUG_BROKENPINNEDMEMORY) && type == GL_ELEMENT_ARRAY_BUFFER)`, and the bug entry no longer applies to current drivers. The commit reports a “big-ish speedup on AMD GPUs for streaming intensive games.” 672fa65ee7 2013-11 “OpenGL: Enable pinned memory even for index buffers (works for me). Big-ish speedup on AMD GPUs for streaming intensive games.”

### dolphin-176 — Choose cheaper shader primitives and integer arithmetic
Lens: gpu-rendering · Source: Dolphin
Principle: In generated shaders, call the specialized built-in (`exp2(x)` rather than `pow(2.0, x)`), because not every driver rewrites it. Do fixed-point color math in integers with shifts instead of float divides by 255. A lerp written as `(a*256 + (b-a)*(c + (c>>7))) >> 8` replaces two multiplies and a divide.
Why it is faster: —
Applies elsewhere: mobile GLSL and Metal shaders, fixed-point DSP code, and blending integer pixels in software renderers.
Watch out: —
Seen in Dolphin: `PixelShaderGen`'s fog table switched from `pow(2.0, ...)` to `exp2(...)`. The commit cites Haswell numbers: 24 cycles for POW, 14 for EXP2. `LightingShaderGen` accumulates `lacc` as `int4` in 0..255. The TEV combiner, formerly `(a*(255-c) + b*c)/255`, became the shift-based lerp above, which the commit says also matches hardware exactly. b55a4bb087 2014-01 “Slight optimization in the pixel shader. We are using pow(2.0, X) in place of exp2(X).” · 387b9bf3c2 2013-10 “LightingShaderGen: Perform some lighting calculations with integers.” · 4f82d6f7af 2014-03 “PixelShaderGen: Implement tev combiner lerping in a faster way which also reproduces hardware behavior perfectly.”

### dolphin-177 — Reject discarded work before preparing for it
Lens: laziness · Source: Dolphin
Principle: If an input will be thrown away, such as primitives that are fully culled or vertices flagged as skipped, test for that before setting up state for it. Skipping the setup is cheaper than doing it and discarding the result.
Why it is faster: —
Applies elsewhere: frustum culling before the draw call, short-circuiting middleware for requests that will be rejected anyway, and predicate pushdown in query engines.
Watch out: —
Seen in Dolphin: The `CULL_ALL` check, which skips triangles and quads with `DataSkip`, moved from `VertexLoader::RunVertices` (after the loader was prepared) into `VertexLoaderManager`, before any preparation. Vertices with position index −1 now back up the output pointer and are subtracted from the count, so they are never sent to the GPU. 16f180524c 2014-08 “VertexLoader: do not prepare for vertices if we need to skip them” · 1efd00227d 2014-12 “VertexLoader: Skip vertices with position index = -1”

### dolphin-178 — Track "already clear" to skip redundant resets, and use memset when you do clear
Lens: memory-layout · Source: Dolphin
Principle: A cache-invalidate that runs often but usually finds the cache already empty should check a cheap flag first. When it does clear, one `memset` over the whole array beats element loops the compiler may not vectorize.
Why it is faster: —
Applies elsewhere: generation counters instead of clearing visited arrays, lazily clearing per-frame scratch buffers, and dirty-tracking before resetting a canvas.
Watch out: —
Seen in Dolphin: In OGL `Renderer::ClearEFBCache`, two loops over `s_efbCacheValid` (“gcc doesn't optimize this loops with -O2”) became one `memset`. It is guarded by `s_efbCacheIsCleared`, which is reset whenever an entry is filled. 9566dcf0da 2014-06 “OGL: speed up the EFB cache”

### dolphin-179 — Don't zero buffers you will overwrite, and reset reusable state objects
Lens: io · Source: Dolphin
Principle: Filling a buffer with zeros right before a read overwrites it doubles the memory traffic. Pad only the bytes the read didn't fill. In the same loop, reset an expensive codec context instead of creating and destroying it per block.
Why it is faster: —
Applies elsewhere: `deflateReset`/`inflateReset` in HTTP compression middleware, `Vec::with_capacity` plus `set_len` after a read, and reusing `MessageDigest` instances.
Watch out: —
Seen in Dolphin: The GCZ compressor in `CompressedBlob.cpp` did `std::fill(in_buf, ..., 0)` before every block read, and `deflateInit`/`deflateEnd` per block. Now `ReadArray` and `DiscScrubber::GetNextBlock` return the bytes read, only the tail gets zeroed, and one `z_stream` is `deflateReset` per block. The commit reports 6% saved. f54bf81520 2014-11 “DiscIO: Avoid zeroing buffer when compressing gcz.”

### dolphin-180 — Cache the last decoded block and decrypt in place
Lens: io · Source: Dolphin · Also: Flycast
Principle: Sequential small reads from a block-encrypted source usually hit the same block repeatedly. Check whether the requested block is the one already decoded before reading and decrypting it again, and decrypt straight from the read buffer with no intermediate copies.
Why it is faster: —
Applies elsewhere: block caches in encrypted filesystems, decompressed-chunk caches in archive readers, and page caches in SQLite VFS layers.
Watch out: —
Seen in Dolphin: `CVolumeWiiCrypted::Read` read the 0x8000-byte block from disc on every call, even when only the already-decrypted `m_LastDecryptedBlock` was needed. The read now happens inside the cache-miss branch. AES-CBC uses `m_pBuffer + 0x3D0` directly as the IV, which drops the `static IV[16]` copy, and the cache buffer shrank to 0x7C00. 521ccd7b44 2015-01 “VolumeWiiCrypted: Decryption optimization” • Also seen in Flycast (flycast-086): `CHDTrack` (`core/imgread/chd.cpp`) never stored `disc->old_hunk = hunk`, so every sector read decompressed the hunk again. The bundled `chd.c` now returns early for uncompressed hunks. The GD-ROM DMA cache in `gdromv3.cpp` grew from 32 to 8192 sectors (`u8 cache[2352 * 8192]`). 0f1c63b44 2018-10 “CHD: Use the hunk caching” · 6a8eafbc5 2018-09 “CHD: If uncompressed we don't need to check compression” · 4aee2eb5f 2018-05 “Increase gdrom dma buffer size to 8192 sectors” · f76df2da6 2018-08 “GDROM: Increase dma buffer cache sectors”

### dolphin-181 — Prefetch assets on a background thread
Lens: io · Source: Dolphin
Principle: Loading large assets from disk on first use stalls the frame that needs them. A background thread can load everything into a shared, mutex-guarded map at startup. The render thread takes from the map if the asset is there and falls back to a synchronous load if not. Add an abort flag and stop when RAM runs low.
Why it is faster: —
Applies elsewhere: `<link rel=preload>`, game level streaming, and warming caches at service startup.
Watch out: —
Seen in Dolphin: `HiresTexture::Update` starts `s_prefetcher`, which fills `s_textureCache` (an `unordered_map<string, shared_ptr<HiresTexture>>`) under `s_textureCacheMutex`. `s_textureCacheAquireMutex` gives the render thread priority, and `s_textureCacheAbortLoading` stops the prefetch on shutdown. It is controlled by the new “Prefetch Custom Textures” option. A follow-up aborts prefetching on low RAM. 242f7d964d 2015-03 “CustomTexture: prefetch all available textures”

### dolphin-182 — Single-producer/single-consumer lock-free queue for cross-thread events
Lens: concurrency · Source: Dolphin
Principle: If one thread posts work and one thread drains it, a linked FIFO where the producer only writes the tail and the consumer only reads the head needs no lock. Store elements inline in the nodes, and drop the shared size counter when no one needs it, so the queue has no atomic read-modify-write at all.
Why it is faster: —
Applies elsewhere: audio callback ring buffers, log shippers, and the Disruptor pattern.
Watch out: —
Seen in Dolphin: `CoreTiming`'s thread-safe events had used a mutex-protected linked list with its own free pool (`GetNewTsEvent`/`FreeTsEvent`). They now go into a `Common::FifoQueue<BaseEvent, false>`, where `NeedSize=false` compiles out the atomic size counter. Elements are stored by value in nodes (`T current` rather than `T*`), and `Empty()` is `!m_read_ptr->next`. Producers still serialize with `tsWriteLock`, but the consumer takes no lock. c3b9f3556f 2013-08 “Make CoreTiming's threadsafe events lock-free.”

### dolphin-183 — Atomics with a lock-free fast path instead of mutexes for simple shared state
Lens: concurrency · Source: Dolphin
Principle: A counter or flag shared between threads doesn't need a mutex. Use `fetch_add`, `exchange` and `load`. For a signal object, check an atomic flag first and take the mutex and condition variable only when a waiter might actually be asleep. Merging two flags into one atomic state means the common no-op wakeup is a single load.
Why it is faster: —
Applies elsewhere: futex-based mutexes, request counters in metrics libraries, `std::latch`, and Go's `sync.Once` fast path.
Watch out: —
Seen in Dolphin: DSP LLE's `m_cycle_count` became `std::atomic<u32>`, which removed `m_csDSPCycleCountActive`. `Common::Event::Set` locks only if `m_flag.TestAndSet()` succeeds, and `Wait` returns early on `TestAndClear()`. `BlockingLoop` merged `m_is_running`/`m_is_pending` into one `atomic<int> m_running_state`. `Wakeup` returns immediately when the state is already `>= STATE_NEED_EXECUTION` (“the common case”), and sets the event only if the `exchange` shows the worker was sleeping. 4051da75e4 2015-03 “Made cycle count atomic to avoid using a mutex” · 48bd904028 2014-04 “Common::Event: Implement in terms of Common::Flag to get rid of a volatile and optimize with early returns and atomic swaps” · bfa61105d5 2015-05 “Common: Update BlockingLoop to only use one atomic.”

### dolphin-184 — Swap in a native primitive when the portable one is slow on a platform
Lens: platform · Source: Dolphin
Principle: Standard-library primitives vary widely in quality between implementations. When profiling shows one is slow on a platform, wrap it and use the OS- or runtime-native equivalent there. The interface stays the same everywhere.
Why it is faster: —
Applies elsewhere: `SRWLOCK` instead of an old MSVC `std::mutex`, `os_unfair_lock` on Apple, and `eventfd` instead of pipes for wakeups on Linux.
Watch out: —
Seen in Dolphin: On Windows, `Common::Event` is now a thin wrapper over `concurrency::event` (`set`/`wait`/`reset`), because “std::condition_variable has terrible performance for this kind of workload with MSVC++ 2013.” Other platforms keep the flag+condvar version. 7074feacbe 2014-04 “Common::Event: Add a faster Windows specific implementation based on the concurrency runtime.”

### dolphin-185 — Answer "which thread am I?" with a thread-local flag
Lens: concurrency · Source: Dolphin
Principle: Comparing `std::this_thread::get_id()` against a stored id, plus a joinable check, costs more than reading one thread-local bool set when the thread starts. For checks called on hot paths, use TLS.
Why it is faster: —
Applies elsewhere: "am I on the UI thread" asserts, per-thread allocator caches, and reentrancy guards in logging.
Watch out: —
Seen in Dolphin: `Core::IsCPUThread()` returned `s_cpu_thread.joinable() ? get_id()==... : !s_is_started`. It now returns a `__declspec(thread)`/`__thread bool tls_is_cpu_thread`, set by `DeclareAsCPUThread()`. Android and macOS use a `pthread_getspecific` fallback. 568f0dca78 2015-02 “In hope of speeding up IsCPUThread() got rid of id checking and instead implemented a TLS boolean...”

### dolphin-186 — Sleep on a condition variable instead of spinning when cores are scarce
Lens: concurrency · Source: Dolphin
Principle: Busy-waiting cuts wakeup latency but takes a whole core. On machines with few cores, or on CPUs that do background work in idle cycles, it starves other threads. Have the worker wait on an event with a timeout, and have producers signal it. Keep an "is running" flag so a signal that arrives mid-iteration forces one more pass.
Why it is faster: —
Applies elsewhere: worker pools blocking on a queue, game render threads on laptops, and adaptive spin-then-park mutexes.
Watch out: —
Seen in Dolphin: The GPU thread loop in `Fifo.cpp` busy-looped. It first yielded when `cpu_info.num_cores <= 2`, and on dual-core Nvidia Denver that went from 35–40 FPS to 50–55. Later the yield was replaced by `s_gpu_new_work_event.WaitFor(100ms)`, with `s_gpu_is_running` forcing a rerun when the CPU thread pushed data mid-loop. Producers call `RunGpu()` to set the event. d58bba2160 2014-12 “Idle loop in GPU loop on dual core systems.” · 279c657cda 2015-03 “Fifo: Replace busy loop with condition variable”

### dolphin-187 — One request queue instead of per-feature polled flags
Lens: concurrency · Source: Dolphin
Principle: Polling separate "is there a request?" flags each loop iteration grows with every feature and adds latency. Use one mutex-protected queue with an atomic `empty` flag. The consumer checks the flag cheaply and drains only when it's set. Blocking requests set a "wake me" bit and wait on a condition variable.
Why it is faster: —
Applies elsewhere: message queues in actor systems, a UI thread's single `postMessage` mailbox, and work queues in game engines.
Watch out: —
Seen in Dolphin: The new `AsyncRequests` class (`PushEvent(e, blocking)`, and `PullEvents` called from the FIFO loop) replaced `VideoFifo_CheckAsyncRequest`'s separate EFB-access and perf-query flag/event pairs. `PullEventsInternal` unlocks while handling each event and notifies only if `m_wake_me_up_again` is set. bc248f8941 2015-01 “VideoCommon: use a new async event system for efb access” · b35fa222f5 2015-01 “VideoCommon: perf querys by async events”

### dolphin-188 — Schedule events instead of polling every time slice
Lens: laziness · Source: Dolphin
Principle: If a subsystem checks a "something changed?" condition on every scheduler tick, put that work on the scheduler's event queue at the moment the change happens. The per-tick check disappears, and the work runs only when it's needed.
Why it is faster: —
Applies elsewhere: timer wheels instead of scanning all connections for timeouts, `setTimeout` instead of polling in `requestAnimationFrame`, and interrupt-driven I/O instead of polling loops in firmware.
Watch out: —
Seen in Dolphin: EXI had its own `UpdateInterrupts` check, which `PowerPC.cpp` ran in the CPU loop. Devices now call `ExpansionInterface::UpdateInterrupts()` directly, or `ScheduleUpdateInterrupts(_Threadsafe)` via CoreTiming, and the special check was removed (“instead of having its own slow special check”). 9f683f353b 2014-11 “Make EXI use CoreTiming events like everything else instead of having its own slow special check.”

### dolphin-189 — Hoist per-iteration setup and teardown out of the loop
Lens: other · Source: Dolphin
Principle: If a loop body saves and restores some state, or takes a lock and checks status, on every iteration, and nothing between iterations needs the restored state, do it once around the loop. One commit also halved a fixed-size scan that covered empty "future use" slots.
Why it is faster: —
Applies elsewhere: one transaction around a batch of DB writes instead of one per row, setting GL state once per batch, and reading controller state once per frame.
Watch out: —
Seen in Dolphin: The single-core FIFO loop called `FPURoundMode::SaveSIMDState`/`LoadDefaultSIMDState`/`LoadSIMDState` on each iteration. Now a `reset_simd_state` flag saves once and restores once after the loop. In `HotkeyManager`, the GUI `try_to_lock` moved into `IsReady()`, the controller status is read once per frame, the hotkey arrays shrank from 6 to 3 words, and the redundant memsets were removed. b1ffd32f5f 2015-03 “Fifo: only touch the SIMD state once in the single core loop” · a83d27b9c2 2015-03 “Increased the performance of the hotkey code by:”

### dolphin-190 — Retire hand-tuned routines once the platform library is better
Lens: platform · Source: Dolphin
Principle: Copy and compare routines tuned for one old CPU family end up slower than the toolchain's `memcpy`/`memcmp`, which modern compilers inline or dispatch per CPU. Delete them. Likewise, use the cheapest clock API that gives the precision you need.
Why it is faster: —
Applies elsewhere: removing custom `strlen`/`memcpy` from legacy C codebases, and using `CLOCK_MONOTONIC_COARSE` or vDSO clocks for timestamps.
Watch out: —
Seen in Dolphin: `memcpy_amd.cpp` (Athlon/Duron-era MMX `memcpy_amd`/`memcmp_mmx`, used through `memcpy_gc` on 32-bit MSVC) was deleted, and the BP/XF loaders call `memcpy` directly. On non-Apple Unix, `Common::Timer` uses `clock_gettime(CLOCK_MONOTONIC)` instead of `gettimeofday` (“more efficient on low power systems”). a4bb0dafb4 2014-05 “Removes ZeroFrog's "optimized" memcpy and memcmp functions.” · 797fc14414 2015-01 “Use clock_gettime instead of gettimeofday, it's more effiency on low power system”

### dolphin-191 — Specialize on values that rarely change, behind an entry guard
Lens: jit-interpreters · Source: Dolphin
Principle: If a configuration value is almost always the same at runtime, generate code that assumes its current value and add one cheap check at entry that falls back when the assumption breaks. The common path loses all the generic dispatch, and the guard costs one compare.
Why it is faster: —
Applies elsewhere: JS engines' inline caches keyed on object shape, a database plan cached for one parameter value and re-planned on change, image filters compiled for the current pixel format.
Watch out: —
Seen in Dolphin: Paired-single loads and stores used to read the GQR register at runtime and make an indirect call through a table of quantize routines. JitArm64 (and later Jit64) now checks the GQR values at block entry. If they still match the compile-time values, the loads are inlined as plain float loads (`LD1` + `REV32`). On a mismatch the block records its address in `pairedQuantizeAddresses` and is recompiled without the assumption. Commit 4aa5291f54 extended this to any constant GQR value and reports a 5–10% speedup. The same pattern added "speculative constants" for registers that look like gather-pipe or MMIO addresses (`IntializeSpeculativeConstants`), which lets FIFO writes be optimized in more places. 2c68f6bfc5 2015-09 “[AArch64] Implement Fiora's preemptive paired loadstore optimization.” · 4aa5291f54 2016-02 “Refactor the paired load/store code …” · 789975e350 2016-06 “Jit: FIFO optimization improvements”

### dolphin-192 — Move cold code out of the hot instruction stream
Lens: jit-interpreters · Source: Dolphin
Principle: Rarely taken paths such as exception exits and slow fallbacks still use instruction-cache lines and branch-predictor entries when they sit inline. Emit them into a separate "far" region and leave only a conditional jump inline. The hot loop then stays dense.
Why it is faster: —
Applies elsewhere: `__builtin_expect`/`[[unlikely]]` together with `__attribute__((cold))` so the compiler moves error handlers into `.text.unlikely`, or putting error formatting in a separate non-inlined function.
Watch out: —
Seen in Dolphin: JitArm64 gained a `FarCodeCacheArm64` with `SwitchToFarCode()`/`SwitchToNearCode()`, described as "a place to throw blocks of code we don't want polluting the cache, e.g. rarely taken exception branches". Later `bclrx` stopped detouring through far code in the unconditional case, which is the common one. b8dd68beef 2015-07 “JitArm64: Far Code Cache” · 637851fce0 2016-03 “JitArm64: optimize bclrx”

### dolphin-193 — Fall through to adjacent code instead of jumping to it
Lens: jit-interpreters · Source: Dolphin
Principle: When the jump target is the next instruction, a jump only costs a fetch redirect. Pad the gap with NOPs, or lay out code so it falls through. If the jump can go straight to a target's fast entry, also skip that target's redundant checks.
Why it is faster: —
Applies elsewhere: basic-block ordering in compilers (PGO layout puts the likely successor next), placing a `switch`'s default case right after the dispatch.
Watch out: —
Seen in Dolphin: Blocks are compiled in execution order, so a block's exit link often points at the very next block. `JitBaseBlockCache` link writing now emits `NOP(offset)` when the distance is at most 8 bytes, instead of `JMP`. On ARM64 the linker emits `B(CC_LE, dest->normalEntry)`, folding the downcount check into the link so the target skips its own checked entry. 78aa01e06e 2015-08 “Jit64: Faster linking of continuous blocks” · b6f16352e4 2016-05 “JitArm64: Check downcount on block linking.”

### dolphin-194 — Range check with one unsigned compare
Lens: branching-fast-paths · Source: Dolphin
Principle: `lo <= x && x < hi` needs two compares and two branches. `(unsigned)(x - lo) < (hi - lo)` needs one, because values below `lo` wrap around to huge unsigned numbers. Fold several outlier classes into one test the same way and sort them out later on the slow path.
Why it is faster: —
Applies elsewhere: `isdigit` as `(unsigned)(c - '0') < 10`, array bounds checks for signed indices, HTTP status class tests.
Watch out: —
Seen in Dolphin: In `GenFres` and `ConvertDoubleToSingle` (Jit64), two compares on the exponent were replaced by `SUB` + one `CMP` + `J_CC(CC_AE)`. `frsqrte` now sends negatives, zeros, denormals, infinities and NaNs to the slow path with a single `LEA exp-1; CMP 0x7FE; JAE`, and the slow path classifies them. ce10525056 2015-07 “Eliminate branch in GenFres” · 78bc9690e2 2018-06 “Eliminate branch in ConvertDoubleToSingle” · 89a0b0de32 2018-09 “Jit64AsmCommon: Reduce branches in fast-path and inline most behavior”

### dolphin-195 — Keep a frequently re-read flag in a host flag register
Lens: jit-interpreters · Source: Dolphin
Principle: If one operation produces a carry or condition that the next operation consumes, leave it in the CPU flags. Store it to memory only when something else is about to clobber the flags. This removes a store, a load and a compare per pair.
Why it is faster: —
Applies elsewhere: bignum add loops that chain `adc` instead of materializing the carry, and the compiler idea of fusing a compare with the branch that follows it.
Watch out: —
Seen in Dolphin: JitArm64 sets `js.carryFlagSet` after carry-producing instructions. If the analyzer reports that the next instruction `wantsCAInFlags`, the store to `xer_ca` is skipped. Consumers such as `addex`/`subfex` then use `ADCS` directly, where they previously had to run `LDRB xer_ca; CMP 1`. A `FlushCarry()` runs before anything that clobbers the flags. 4d88f5410e 2016-10 “JitArm64: Store carry within the host carry flag.” · f449541e65 2016-10 “JitArm64: Implement derefered carry usage.”

### dolphin-196 — Don't compute side results that nothing reads
Lens: jit-interpreters · Source: Dolphin
Principle: A quick liveness pass that marks which outputs are actually used later lets the code generator drop work whose result is overwritten before it is read. This matters most for secondary outputs, such as flags, that are expensive to produce and rarely consumed.
Why it is faster: —
Applies elsewhere: dead-store elimination in compilers, skipping optional response fields that the client did not request (GraphQL field selection).
Watch out: —
Seen in Dolphin: PPCAnalyst marks `wantsCA` per instruction. JitArm64's `ComputeCarry` returns early when `!js.op->wantsCA`, and `srawix` skips its `ORR/ANDS/CSINC/STRB` carry sequence in that case. 68defb72a3 2016-02 “JitArm64: Don't generate the carry if it's not used any more.” · 3405f5ac1a 2016-10 “JitArm64: Optimize carry behavior of srawix.”

### dolphin-197 — Track a value's current representation and convert only on demand
Lens: jit-interpreters · Source: Dolphin
Principle: If a value can exist in several encodings (single or double, low lane only or both lanes), record which one each register currently holds and convert lazily when a consumer needs a different one. Chains of operations in the same format then do no conversions at all.
Why it is faster: —
Applies elsewhere: keeping strings as UTF-8 or UTF-16 and transcoding only at an API boundary (as V8 does), and tracking whether a GPU texture is in linear or sRGB layout.
Watch out: —
Seen in Dolphin: The ARM64 FPR cache gained the states `REG_REG_SINGLE`, `REG_DUP_SINGLE` and `REG_LOWER_PAIR`. `Arm64FPRCache::R()` returns the host register unchanged when the caller asks for the format already held, and emits `FCVTL`/`FCVT` only when a double is needed. Scalar ops on registers that hold only the low lane write directly, without a temp and `INS`. 7ce4c3138e 2015-08 “[AArch64] Optimize cases when an FPR is only used for non-paired ops.” · 1e59dc1025 2016-02 “JitArm64: Single precision tracking.”

### dolphin-198 — Keep hot state in registers instead of memory
Lens: jit-interpreters · Source: Dolphin
Principle: Every load from or store to a state struct adds latency and an extra instruction. Pin long-lived values, such as a base pointer, in a register once at the dispatcher. Add frequently written fields to the register allocator so they are written back only when flushed.
Why it is faster: —
Applies elsewhere: interpreters that keep `pc`, `sp` and a frame pointer in locals (registers) instead of in the VM struct, and global register variables in embedded firmware.
Watch out: —
Seen in Dolphin: JitArm64 used to emit `MOVI2R(X28, base)` at the start of every block. The memory base is now loaded once in the dispatcher. Condition-register fields (`cr_val`) became cacheable via `gpr.BindCRToRegister`/`gpr.CR(crf)`, replacing a `STR` per compare. 55b9ce9b5b 2016-02 “JitArm64: Store memory pointer once, not per block.” · d4e2529f13 2017-04 “JitArm64: Use CR cache”

### dolphin-199 — Inline helpers once they are only a few instructions
Lens: jit-interpreters · Source: Dolphin
Principle: A call costs a push of the return address, a jump, a return and register saves. Once a helper has been trimmed down to a few instructions, inlining it is cheaper than calling it, and the code-size cost is small.
Why it is faster: —
Applies elsewhere: moving a tiny accessor into a header so it can be inlined across translation units, or turning a ring-buffer push into an inline function.
Watch out: —
Seen in Dolphin: Jit64 removed the `fifoDirectWrite8/16/32/64` asm routines. `UnsafeWriteGatherPipe` was a `CALL` and is now an inline byteswapped store through `gather_pipe_ptr`, using `MOVBE` in place when the CPU has it ("As we're down to 4 instructions now, it is always worth to inline those writes"). JitArm64 likewise addresses `gather_pipe_ptr` off the state register instead of materializing its absolute address. 36ad887a19 2017-11 “Jit64: Inline GP writes.” · 6c9bb67ca0 2017-11 “JitArm64: Optimize gather pipe writes.” · 46c3a11d01 2015-06 “[AArch64] Optimize gatherpipe stores”

### dolphin-200 — Merge adjacent loads and stores into wide or paired ones
Lens: memory-layout · Source: Dolphin
Principle: If consecutive fields are read or written together, one wide or paired memory operation replaces several narrow ones. That means fewer instructions, fewer issue slots and fewer address computations.
Why it is faster: —
Applies elsewhere: copying a struct with 64-bit words instead of bytes, and SQL batched inserts as the database-level equivalent.
Watch out: —
Seen in Dolphin: `Arm64GPRCache::FlushRegisters` stores two consecutive dirty guest registers with one `STP` ("Typically cuts 16 STR instructions down to 8 STP"). `lmw` loads four registers with one 64-bit `LDP` plus `REV32`/shift. Ikaruga had a block taking about 20% of CPU time that did exactly this. `ABI_PushRegisters` now uses `STP` pairs and updates `SP` once instead of building a `std::vector` of pairs. 8bf332cf08 2015-08 “[AArch64] Optimize GPR cache flushing.” · 8d61706440 2015-08 “[AArch64] Optimize lmw.” · 6aa54a029e 2017-02 “JitArm64: Optimize GPR register push/pop.”

### dolphin-201 — Pick the shortest encoding for constants and idioms
Lens: platform · Source: Dolphin
Principle: Code generators should choose the cheapest form for each operation: a 32-bit move when the upper half is zero, a sign-extended 32-bit immediate, `xor r,r` for zero, `lea` for add-and-move, and the negated immediate with the opposite operation when only that fits. Smaller code means better i-cache density and fewer µops.
Why it is faster: —
Applies elsewhere: the same peephole rules apply in any assembler, WASM codegen or bytecode encoder (for example, short opcodes for small integer literals).
Watch out: —
Seen in Dolphin: The x64 emitter now emits `mov r32, imm32` (5 bytes) or `mov r64, simm32` (7 bytes) instead of 10-byte `movabs`. `MOV` of zero becomes `XOR`, and `MOV_sum` centralizes the zero-operand shortcuts. The DSP JIT uses `LEA`/`TEST`/32-bit ops in place of `MOV`+`ADD`, `CMP 0` and 64-bit forms. The ARM64 `ADDI2R`/`SUBI2R` try `SUB`/`ADD` with the negated immediate before falling back to a temp register, and `addx` uses an immediate `ADD` when one operand is a known constant below 4096. 575f1b309a 2018-09 “x64Emitter: short MOV for 64bit immediates (1)” · 53a947749a 2018-09 “… (2)” · 1e08ad0d97 2016-06 “Add MOV optimization and MOV_sum” · ef94fca504 2018-09 “DSPJit: various micro-optimizations” · 1df694626d 2016-10 “JitArm64: Optimize addi2r & subi2r.” · 7e8cfff3c0 2015-07 “Optimized addx()”

### dolphin-202 — Use bitfield and combined-logic instructions
Lens: platform · Source: Dolphin
Principle: Modern ISAs have single instructions for patterns that otherwise take three or four: and-not, bitfield extract/insert, logical ops with encoded immediates, scaled LEA for shift-and-add. Recognize the pattern and emit the one-cycle form.
Why it is faster: —
Applies elsewhere: write `x & ~y` and `(x >> s) & mask` in shapes compilers map to ANDN/UBFX, packet-header field extraction, and pixel-format repacking.
Watch out: —
Seen in Dolphin: Jit64 `andcx` uses BMI1 `ANDN` where it previously did `MOV`/`NOT`/`AND`. JitArm64 `rlwinmx` picks `ANDI2R`, `UBFX` or `UBFIZ` for the common shapes, and logic-immediate ops try the encoded-immediate form. The vertex loader's 4444/6666/565 colour unpacking was rewritten with `PDEP` and `LEA`-scaled shuffles that roughly halve the instruction count. The same applies to SIMD: ARM64 quantized paired stores replaced an `FMIN`/`FMAX` clamp against a range table plus two `XTN` with two saturating `UQXTN`/`SQXTN`. 5ea66369f8 2015-06 “[AArch64] Optimize paired quantized stores.” ·f9129c0e08 2015-08 “Jit64: use BMI1's ANDN for andcx” · 5ee7f86199 2017-08 “JitArm64: Optimize rlwinmx.” · 1f94abea18 2016-09 “JitArm64: Optimize logic immediate instructions.” · d80124ab59 2015-10 “VertexLoaderX64: optimize 4444 color conversion” · b0d5b4483e 2015-10 “VertexLoaderX64: optimize more color swizzles”

### dolphin-203 — Hardware CRC32 as a fast hash, with independent lanes
Lens: simd · Source: Dolphin
Principle: When a hash only needs to detect changes rather than resist attackers, the CPU's CRC32 instruction is much cheaper than a general hash. Run four independent accumulators so the instruction's latency overlaps, and select the routine at runtime from CPU feature bits.
Why it is faster: —
Applies elsewhere: cache keys for asset change detection, dedup chunking, and hash tables for trusted data (RocksDB and many storage engines use crc32c).
Watch out: —
Seen in Dolphin: Texture hashing on ARM64 gained a `crc32x` path with `h[0..3]` lanes. In one Crazy Taxi scene it took 1.86% of CPU time where Murmur3 took 3.11%. The path is selected via `cpu_info.bCRC32`. The x86 SSE4.2 version moved to `FUNCTION_TARGET_SSE42` so generic builds still include it, and the ARM version later switched from inline asm to the `__crc32d` intrinsic. ffe085f5ea 2015-06 “[AArch64] Implement CRC32 texture hashing.” · 0f978227f5 2016-12 “Common: Use function-level targeting for CRC32 (SSE4.2)” · 25f61f0329 2017-05 “Common/Hash: use __crc32d() intrinsic on ARM64”

### dolphin-204 — Fill memory with wide stores of a repeating pattern
Lens: simd · Source: Dolphin
Principle: A byte-at-a-time fill loop with a parity test does one store and one branch per byte. Splat the repeating pattern into a vector register and store 16 bytes per iteration, then finish the tail with scalar code.
Why it is faster: —
Applies elsewhere: clearing a framebuffer to a colour, initializing arrays of structs with a non-zero default, and `memset_pattern16` on macOS.
Watch out: —
Seen in Dolphin: `UninitializeXFBMemory` wrote alternating 1/254 bytes per offset. It now writes `_mm_set1_epi16(0xFE01)` with `_mm_storeu_si128`, which the author measured as 10x faster (1% of CPU time down to 0.09%). 6cbb716f60 2018-03 “SSE-optimize UninitializeXFBMemory”

### dolphin-205 — Test for rare inputs only when the result is rare
Lens: branching-fast-paths · Source: Dolphin
Principle: If an exceptional input always produces an exceptional output (NaN in gives NaN out), compute first and check the inputs only when the output is exceptional. The common path keeps a single check.
Why it is faster: —
Applies elsewhere: overflow checking by testing the result's sign instead of pre-checking operands, and validating a decoded value only when the checksum fails.
Watch out: —
Seen in Dolphin: The interpreter's `NI_add/sub/mul/madd` used to test `a != a` and `b != b` before every operation. Those checks now run only inside `if (result != result)`, mirroring `Jit64::HandleNaNs()`. c7544719e2 2015-06 “Interpreter: optimize NaN checks”

### dolphin-206 — Cheap inline check before an expensive call
Lens: branching-fast-paths · Source: Dolphin
Principle: If a function usually does nothing, for example "handle pending events" when there are none, test its trigger condition inline and call it only when the test passes. One load and branch replaces a full call with register saves.
Why it is faster: —
Applies elsewhere: `if (log_enabled) log(...)` guards, and checking an atomic "pending" flag before taking a lock.
Watch out: —
Seen in Dolphin: JitArm64 exception exits used to call `PowerPC::CheckExceptions` unconditionally. They now emit `LDR Exceptions; CBZ` and branch over the call when no exception bits are set. 56c8f65bc2 2016-02 “JitArm64: Precheck for exceptions before calling PowerPC::CheckExceptions.”

### dolphin-207 — Detect spin loops and jump time forward
Lens: jit-interpreters · Source: Dolphin · Also: ares
Principle: A loop that only waits for an external event burns host CPU while simulating nothing. Recognize the pattern, such as a branch to itself, and skip directly to the next scheduled event instead of iterating.
Why it is faster: —
Applies elsewhere: discrete-event simulators that advance the clock to the next event, game servers that sleep until the next tick instead of polling, and OS idle (`hlt`) in place of spinning.
Watch out: —
Seen in Dolphin: In Jit64 `bx`, a branch whose destination is its own address used to add 8 to the downcount. It now calls `CoreTiming::Idle` and exits, and JitArm64 skips the normal block exit for the same case. c657134c86 2016-11 “Jit64: Treat branch-to-self instruction as an idle loop” · e83bf5705a 2016-11 “JitArm64: Don't emit normal block exit for branch-to-self instructions” • Also seen in ares (ares-057): The N64 recompiler's emit loop checks `if(unlikely(instruction == 0x1000'ffff))` (`beq zero,zero,-1`). In that case it adds 64 to the clock, first as inline `mov/add/mov` and later as `mov(ra1d, imm32(64)); call(&CPU::step);`. 25771ba20 2021-05 “Update to ares v119r15 release.” · d31d5a931 2021-10 “recompiler: avoid encoding absolute data addresses” • Also seen in ares (ares-084): The N64 recompiler already treated `beq 0,0,<pc>` (`0x1000ffff`) as an idle loop and emitted `step(64*2)`. It now also matches `j <pc>`, built as `2<<26 | vaddr>>2`. The commit reports that about half of roughly 60 games used beq and a quarter used j. The match had to use the virtual address, not the physical one, and `emit` now advances `vaddr` along with `address`. 0130187f8 2023-08 “n64: detect idle loops using jump instruction” · 3fd03e7a3 2023-08 “n64: improve idle loop detection” · 8574a5879 2023-08 “n64: advance virtual address in recompiler”

### dolphin-208 — Follow unconditional branches to build longer traces
Lens: jit-interpreters · Source: Dolphin
Principle: Ending a compiled unit at every unconditional branch means a dispatch per branch and no optimization across it. Follow such branches during analysis so the target code joins the same unit. Where a call is inlined, keep the return-address bookkeeping valid with a fake call frame.
Why it is faster: —
Applies elsewhere: trace-based JITs (LuaJIT, TraceMonkey), and function inlining plus jump threading in AOT compilers.
Watch out: —
Seen in Dolphin: `PPCAnalyst` enabled `OPTION_BRANCH_FOLLOW` for Jit64. Inlined `bl` uses `FakeBLCall`, which pushes the return address and emits a `CALL` so the BLR-stack optimization still works. The commit calls this "a big overhead, but still better than calling the dispatcher". f31b25fe39 2017-01 “Jit64: Enable branch following.”

### dolphin-209 — Bucketed range index for overlap invalidation
Lens: algorithms · Source: Dolphin
Principle: To find every item overlapping an address range without scanning all items, register each item in coarse fixed-size buckets covering its span, keyed in an ordered map. An invalidation then visits only the buckets that intersect the range.
Why it is faster: —
Applies elsewhere: spatial hashing in game engines, page-granular dirty tracking, and the cache-invalidation index of a CDN purge-by-prefix.
Watch out: —
Seen in Dolphin: `JitBaseBlockCache::InvalidateICache` used to iterate the whole `block_map` checking `Overlap`. It now uses `block_range_map`, a map from a `BLOCK_RANGE_MAP_ELEMENTS`-aligned address to the set of blocks touching it, searched with `lower_bound`. This also removed the restriction that made block inlining impossible. f3ed993747 2017-01 “JitCache: Use a map with macro blocks for the occupied memory regions.”

### dolphin-210 — Answer "which interval contains x" with an ordered map
Lens: algorithms · Source: Dolphin
Principle: Key a sorted map by interval end (or start) and use `upper_bound`/`lower_bound` to find the single candidate, then check one boundary. That is O(log n) instead of a linear scan over all intervals.
Why it is faster: —
Applies elsewhere: mapping an instruction pointer to its function in a profiler, IP-to-ASN lookup, and timestamp-to-segment lookup in log storage.
Watch out: —
Seen in Dolphin: `PPCSymbolDB::GetSymbolFromAddr` fell back to looping over every function. It now does `lower_bound(addr)` and steps back one. `FileSystemGCWii::FindFileInfo(disc_offset)`, called on every disc read by FileMonitor, builds a `std::map` from file end offset to index once and uses `upper_bound`. cc40931d64 2017-09 “PPCSymbolDB: Optimise symbol lookups” · ee27be06d7 2015-08 “Filesystem: Add a cache for finding file info by disc offset”

### dolphin-211 — Walk a hierarchy by path components instead of flattening it
Lens: algorithms · Source: Dolphin
Principle: Looking up "a/b/c" by building the full path of every node and comparing strings is O(n) string work. Split the path and descend one directory at a time, comparing only names. Only the siblings along the path are visited.
Why it is faster: —
Applies elsewhere: trie or radix-tree URL routers, filesystem path resolution, and JSON-pointer lookup.
Watch out: —
Seen in Dolphin: `FileSystemGCWii::FindFileInfo(path)` called `GetPathFromFSTOffset(i)` for every entry. It now recurses with `search_start_offset`, matching one component per level and skipping whole subdirectories via their stored size. The commit notes that looking up "opening.bnr" now scans only root-level entries. f49b64caff 2015-07 “Filesystem: Rewrite finding file info by path for performance”

### dolphin-212 — Skip expensive work whose consumer is disabled
Lens: laziness · Source: Dolphin · Also: Flycast
Principle: Before computing something only an optional feature uses, check whether that feature is on. Debug-only metadata, unused GPU resources and descriptive names are often computed unconditionally and cost real time.
Why it is faster: —
Applies elsewhere: building log-message strings only when the level is enabled, and creating tracing spans only when sampling picks the request.
Watch out: —
Seen in Dolphin: Every new JIT block looked up its symbol name, causing "noticeable stuttering" with large symbol maps. The lookup now runs only if `JitRegister::IsEnabled()`. OGL creates texture-decoding compute resources only when `bSupportsGPUTextureDecoding` is set. evdev calls `GetName()`, which opens the device and issues many ioctls, only when the name is needed, halving `PopulateDevices()`. 8cd8e9d905 2017-08 “JIT: Don't always look up symbols for blocks” · 82129d30c9 2017-10 “TextureCache: don't create texture decoding resources if not enabled” · 57a005ff3e 2016-12 “evdev: reduce startup time” • Also seen in Flycast (flycast-147): `Drawer::Draw` in `core/rend/vulkan/drawer.cpp` always called `SortTriangles()`, which sorts every translucent triangle, even when per-strip sorting meant the result would not be used. It became `if (!config::PerStripSorting) SortTriangles();`. 98123b921 2022-12 “vk: don't sort triangles in per-strip mode”

### dolphin-213 — Defer construction until first use
Lens: laziness · Source: Dolphin
Principle: Wrap expensive-to-build members in a lazy holder (value or thunk) that computes on first access. Callers that never touch the value, which is common in list or enumeration paths, never pay for it. This also applies to large buffers that only one feature uses.
Why it is faster: —
Applies elsewhere: `std::call_once`, `Lazy<T>` in .NET, Python's `functools.cached_property`, and lazy-loaded routes in SPAs.
Watch out: —
Seen in Dolphin: `Common::Lazy<T>` wraps a `std::variant<T, std::function<T()>>`. `VolumeWii` uses it so tickets, TMDs and AES keys for non-game partitions are not read while building the game list, and DiscIO uses it for filesystems and banners. FileMonitor creates the disc filesystem object only once a read is logged. `FifoRecorder` no longer allocates its 100 MB of RAM mirrors at startup: release-build idle memory dropped from about 101 MB to 5.7 MB. 2c7e93f3b8 2017-06 “Common: Add a Lazy type” · 78217532e5 2017-06 “VolumeWii: Defer loading tickets, TMDs and keys until when needed” · 98b4ff12f4 2017-02 “FileMonitor: Move file system parsing to DVD thread” · 82c66e2bab 2017-06 “FifoRecorder: Don't allocate ~100.7MB on program startup”

### dolphin-214 — Keep handles open and query results resident
Lens: io · Source: Dolphin
Principle: Opening a file per read, re-parsing a header on every query, or asking a driver for a value that never changes adds syscalls, synchronization and allocation for nothing. Open once per logical session, and keep parsed metadata and constant query results in members that are returned by reference.
Why it is faster: —
Applies elsewhere: connection pools, keeping a SQLite handle open, caching parsed config or JWT keys, and caching `sysconf` or WebGL `getParameter` limits.
Watch out: —
Seen in Dolphin: `CNANDContentDataFile` constructed an `IOFile` on every `Get`/`GetRange`. It now keeps `m_file` open between `ES_OPENCONTENT` and `ES_CLOSECONTENT`. `VolumeWiiCrypted::GetTicket/GetTMD` returned freshly read copies each time. They now return `const&` to values read once at construction. Likewise, `glGetIntegerv(GL_MAX_TEXTURE_SIZE)` was called several times a frame and "showed up really high" in profiles. It is now read once into `s_max_texture_size`. a449ef4e11 2016-03 “properly open/close the file to avoid rapid open/close cycles” · 5c68a03ae4 2017-05 “VolumeWiiCrypted: Cache TMDs and tickets” · 066b6b0bcb 2016-03 “OpenGL: Cache query to max texture size.”

### dolphin-215 — Push blocking I/O to a dedicated worker thread
Lens: concurrency · Source: Dolphin · Also: Flycast
Principle: If a latency-critical thread performs blocking reads or device writes, it stalls for the full I/O time. Hand the request to a worker thread with a queue and wait for the result only when it is actually needed, which is often later or never.
Why it is faster: —
Applies elsewhere: Node's libuv threadpool for file I/O, a game's asset-streaming thread, and an audio thread that hands disk writes to a writer thread.
Watch out: —
Seen in Dolphin: Disc reads moved to `DVDThread`, so the CPU thread only calls `WaitUntilIdle()` when it needs the data ("This gives the CPU thread more time to do CPU things"). Later commits added request/result queues and moved DTK audio reads there. GC adapter rumble writes (`libusb_interrupt_transfer`) moved to an output thread woken by an event. A generic `Common::WorkQueueThread<T>` was added for this pattern. 54ff4bd0cb 2015-04 “Read disc data asynchronously in DVDThread” · aabb17b7c9 2016-09 “DVDThread: Use queues” · 56239d1ae1 2017-07 “Send rumble data to the adapter on a separate thread” · de9378bf63 2017-08 “Common: add WorkQueueThread” • Also seen in Flycast (flycast-195): Achievements used to start a `std::future` per request. All server calls and image downloads now go through `asyncTask([this, user]() { ... })`. The worker swaps the pending vector out under a mutex and runs it without the lock. The pattern was later turned into reusable `WorkerThread`, `PeriodicThread` and `TsQueue` utilities with tests, which also back custom-texture loading. 0a0d30309 2024-05 “achievements: run all http requests on a background thread” · 7478476f9 2025-01 “multithreading utilities (periodic, worker thread)”

### dolphin-216 — Pipeline GPU readback with double-buffered async transfers
Lens: gpu-rendering · Source: Dolphin · Also: Flycast
Principle: A synchronous read from GPU memory stalls the CPU until the GPU has finished. Issue the read into a pixel buffer, consume the previous frame's buffer instead, and do the encoding on another thread. Reallocate the buffer only when its size actually changes.
Why it is faster: —
Applies elsewhere: screen recorders, WebGL `readPixels` with PBOs or fences, and video-capture pipelines.
Watch out: —
Seen in Dolphin: Frame dumping used to call `glReadPixels` into a `std::vector` and encode on the render thread. Now there is an `m_frame_dump_thread`, two PBOs read back one frame late via `glMapBufferRange`, `glBufferData` runs only when width or height differs, and a `==`/`!=` typo that recreated the dump texture every frame was fixed. d79840078f 2016-10 “Renderer: Threaded frame dumping.” · f6a6cc9c67 2016-10 “OGL: Use PBO for framedump, with async readback.” · 741debe229 2016-11 “OGL: Avoid reallocation of frame dumping PBO.” · 2687c55cf6 2018-04 “Renderer: Only recreate frame dump texture if dimensions differ” • Also seen in Flycast (flycast-127): Render-to-texture copies back to emulated VRAM now `glReadPixels` into `gl.rtt.pbo` (`GL_PIXEL_PACK_BUFFER`, `GL_STREAM_READ`). The data is picked up later by `readAsyncPixelBuffer()` through `glMapBufferRange`. 24d172bd5 2021-04 “gl: use async PBO to copy texture back to vram”

### dolphin-217 — Compile specialized code in the background and use a generic fallback meanwhile
Lens: gpu-rendering · Source: Dolphin
Principle: Compiling a specialized variant on the critical path causes a hitch the first time each variant is needed. Queue the compile on worker threads and render with a precompiled general-purpose variant until the specialized one is ready. Prioritize the queue so on-demand work goes first, and use more threads during startup.
Why it is faster: —
Applies elsewhere: tiered JITs (interpret first, optimize in the background), and web apps that show a server-rendered page while the JS bundle hydrates.
Watch out: —
Seen in Dolphin: An `AsyncShaderCompiler` with worker threads was added. "Ubershaders" interpret TEV and lighting state from uniforms. In hybrid mode `ProgramShaderCache::SetShader` marks a new entry `pending`, queues a `ShaderCompileWorkItem` and calls `SetUberShader` until compilation finishes. Later work added background compilation at boot, a `std::multimap<u32, WorkItemPtr>` priority queue so on-demand pipelines go ahead of ubershaders, a separate precompile thread count, and a "skip drawing" mode. 0a9574eaa1 2017-06 “VideoCommon: Add AsyncShaderCompiler class implementation” · cd502990fa 2017-07 “OGL: Uber shader support” · c8f31656cb 2017-07 “VideoBackends: Support a different number of threads for precompiling” · e31cc1f679 2018-03 “ShaderCache: Implement background shader compilation” · 93865b327f 2018-03 “ShaderCache: Implement compile priority” · 2f1a7cbee1 2018-03 “Implement "Skip" ubershader mode”

### dolphin-218 — Compare POD keys with memcmp
Lens: algorithms · Source: Dolphin
Principle: A member-wise `operator<` built with `std::tie` compiles into a long, branchy chain, and it runs on every map lookup. If the key is plain bytes and padding is zeroed in the constructor and copy operations, a single `memcmp` gives a valid total order and runs as a vectorized compare.
Why it is faster: —
Applies elsewhere: hashing and comparing pipeline-state structs in Vulkan/D3D12 renderers, and composite keys in in-memory indexes.
Watch out: —
Seen in Dolphin: `GXPipelineConfig`/`GXUberPipelineConfig`, looked up on every draw, gained a `memset` constructor and `memcpy` copy operations. `operator<`/`==` became `std::memcmp(this, &rhs, sizeof(*this))`. 590307b94c 2018-02 “ShaderCache: Use memcmp for comparing pipeline UIDs”

### dolphin-219 — Canonicalize cache keys by clearing irrelevant bits
Lens: caching · Source: Dolphin · Also: Flycast
Principle: If a key carries fields that don't affect the cached result in the current configuration, equivalent requests miss the cache and get recomputed. Zero those fields before lookup so equivalent inputs share one entry.
Why it is faster: —
Applies elsewhere: stripping tracking query parameters from CDN cache keys, normalizing header case, and sorting JSON keys before hashing.
Watch out: —
Seen in Dolphin: `ShaderCache` now runs `ClearUnusedPixelShaderUidBits(api, host_config, &uid)` before `shader_map.find` and before queuing compiles, so pixel shaders that differ only in unused bits share one entry. 9a5c2119e5 2018-05 “ShaderCache: Remove unused UID bits before inserting into shader map” • Also seen in Flycast (flycast-155): For paletted textures that the GPU shader resolves itself (`IsGpuHandledPaletted`), the stored texture doesn't depend on the palette selection. The key now uses `tcw.full & TCWPalTextureCacheMask.full` (TexAddr, PixelFmt, VQ, MipMap, but not PalSelect). Previously, every palette bank produced its own cache entry and its own upload. 1ab4eb00c 2022-12 “tex cache: minor key mask fix. egl: no need for depth/stencil surface”

### dolphin-220 — Binary-search a dynamic index instead of switching on it
Lens: gpu-rendering · Source: Dolphin
Principle: Some compilers and drivers lower a `switch` on a runtime value into a linear if-else chain. For small dense ranges, an explicit binary tree of comparisons bounds the depth at log2(n).
Why it is faster: —
Applies elsewhere: GPU shaders that index into non-indexable arrays, and interpreters on targets without jump tables.
Watch out: —
Seen in Dolphin: On OpenGL/Vulkan, the pixel ubershader generator emits nested `if (index < 4u) { if (index < 2u) …` trees instead of `switch (index)` for texgen and similar selections. The commit reports up to 20% higher performance on NVIDIA drivers. D3D keeps the `switch`. f8a64ab1fc 2017-08 “UberShaderPixel: Use an if trees instead of switch statements on GL”

### dolphin-221 — Stay in integer arithmetic when the range allows it
Lens: other · Source: Dolphin
Principle: Converting small integers to float to subtract, square and accumulate costs conversions and float latency. If the bound fits (the squared difference of two bytes fits in 16 bits), do the math in integers and convert once at the end.
Why it is faster: —
Applies elsewhere: image-diff and PSNR tools, audio peak meters, and fixed-point DSP on microcontrollers.
Watch out: —
Seen in Dolphin: The mipmap-detection `AverageDiff` in `TextureCacheBase` now computes `int diff`, `diff*diff` and a per-pixel `int` sum, and adds to the float accumulator once per pixel. b30d56ccc0 2018-05 “Texture AverageDiff: Do more in int space and avoid excessive float conversion”

### dolphin-222 — Dirty flags to skip rescans; periodic instead of per-frame checks
Lens: laziness · Source: Dolphin
Principle: When an expensive overlap search or validation only matters after a specific event, set a flag on that event and skip the search when the flag is clear. For checks that rarely change their answer, run them every N frames instead of every frame.
Why it is faster: —
Applies elsewhere: UI frameworks' "needs layout" flags, incremental build systems, and cache garbage collection on a sweep interval.
Watch out: —
Seen in Dolphin: Texture entries carry `may_have_overlapping_textures`, set on creation and for the range affected by each EFB copy. The partial-update scan in the texture cache returns immediately when the flag is clear, and XFB container entries clear it at creation. Stale EFB copies are hashed only when `(frameCount - entry->frameCount) % TEXTURE_KILL_THRESHOLD == 1` ("for performance reasons") before being freed. d0a73c1d90 2016-12 “TextureCache: Add a dirty bit for partial updates on overlapping EFB copies.” · 8678e8ddd3 2017-10 “Texture Cache Base: Optimization to allow stitched or memory xfb textures to be looked up …” · 38f6cf2089 2015-09 “Perform garbage collection for efb copies”

### dolphin-223 — Reuse partial results and skip repeated work by identity
Lens: caching · Source: Dolphin
Principle: If a composite key is built as base ⊕ extra, store the base part too so a later variant can reuse it without rehashing. Give produced objects a monotonically increasing ID so "is this the same as last time?" is one integer compare, and skip redundant state binds the same way.
Why it is faster: —
Applies elsewhere: ETags, React's referential-equality bailouts, and GL state caches that skip a `glBind*` when the object is unchanged.
Watch out: —
Seen in Dolphin: Paletted texture entries store `base_hash` next to `hash = base_hash ^ palette_hash`, so a palette change does not rehash the texture data. Each texture cache entry gets `id = last_entry_id++`, and the renderer skips presenting when `xfb_entry->id == m_last_xfb_id`. The OGL ubershader path tracks `s_last_VAO` and the last uber entry to avoid rebinding. e408e0ab17 2015-09 “Store the base hash for paletted textures in the texture cache entries” · e6d85b0915 2017-08 “Video Common: Avoid 'presenting' duplicate frames …” · cd502990fa 2017-07 “OGL: Uber shader support”

### dolphin-224 — Tune object pools so steady-state work doesn't churn
Lens: allocation · Source: Dolphin
Principle: A pool that refuses to hand out recently used objects, or that discards caches when the configuration flips, forces fresh allocations in a cycle. Relax reuse restrictions for objects that can't be in use, and keep caches across reversible configuration changes.
Why it is faster: —
Applies elsewhere: database connection pools with an overly strict idle policy, and GPU buffer pools in game engines.
Watch out: —
Seen in Dolphin: The texture pool had locked freed textures for a frame. Render targets are now exempt (`iter.first.rendertarget ||` …) because New Super Mario Bros PAL was reallocating half its textures every few frames. Vulkan stopped destroying its `VkPipelineCache` on MSAA changes, since the user may switch back. ca8d9e2215 2017-03 “TextureCache: Don't lock freed rendertargets for one frame.” · 9604b336c8 2016-11 “Vulkan: Don't destroy the device's pipeline cache on MSAA mode change”

### dolphin-225 — Batch small messages and send diagnostics less often
Lens: io · Source: Dolphin
Principle: For tiny payloads, per-packet protocol overhead dominates. Combine several updates into one packet, send non-critical telemetry every N ticks, and optionally lower the poll rate when bandwidth matters more than a frame of latency.
Why it is faster: —
Applies elsewhere: Nagle-style coalescing, batching analytics events, and multiplayer games sending input for all local players in one datagram.
Watch out: —
Seen in Dolphin: NetPlay's `SendPadState` became `AddPadStateToPacket`, so all local pads share one `NP_MSG_PAD_DATA` packet. The receiver loops `while (!packet.endOfPacket())`. The timebase desync packet is sent every 60 frames instead of every frame, and a "Reduce Polling Rate" option polls SI once per frame. f68dbed535 2018-07 “NetPlay: Bundle multiple local pads into one packet” · 98447eae64 2018-07 “NetPlay: Send timebase packet less frequently” · 5adeca4087 2018-06 “Add "Reduce Polling Rate" option to NetPlay”

### dolphin-226 — Block on events instead of polling or spinning
Lens: concurrency · Source: Dolphin · Also: ares, Flycast, PostgreSQL
Principle: A thread that loops checking a flag, or wakes every 20 ms to see whether anything happened, wastes a core and adds latency. Wait on a condition variable or deliver the event directly. Drop helper threads whose only job was polling.
Why it is faster: —
Applies elsewhere: `epoll`/`kqueue` instead of busy loops, `ResizeObserver` instead of polling element size, and RTOS semaphores instead of spin-waits.
Watch out: —
Seen in Dolphin: The ALSA sound loop busy-looped while muted. It now waits on a `std::condition_variable` that `Clear()` notifies. In single-core mode the idle EmuThread that ran `PeekMessages(); sleep(20ms)` was removed, and the emu thread now becomes the CPU thread. Window resizes are pushed to the renderer as `ResizeSurface(w, h)` events instead of being polled. The producer side was fixed too: the HLE INIT ucode posted its ready mail only on the first `Update()`, leaving the emulated CPU busy-looping until then. It now pushes the mail in its constructor. 333f998123 2015-07 “Don't busy wait in the audio thread (ALSA)” · f9053527a9 2018-01 “Core: Don't spawn an extra thread in single-core mode” · de632fc9c8 2018-01 “Renderer: Handle resize events on-demand instead of polling” · d5e177d3f9 2014-11 “INIT UCode: reduce DSP initialization time” • Also seen in ares (ares-103): `Screen::main` did `usleep(1); if(_frame) refresh();`. It now calls `_frameCondition.wait_for(lock, 10ms, [&]{ return _frame.load(); })`, and `Screen::frame()` calls `notify_one()` in threaded mode. b145f0f00 2024-04 “screen: use condition_variable rather than polling for next frame” • Also seen in Flycast (flycast-094): The DirectSound backend got a dedicated audio thread with an atomic-cursor `RingBuffer`. It waits on DirectSound position `notificationEvents` instead of a busy loop, and fills with silence on underrun. While the GUI is open, the renderer loop waits on the render-start event with a 17 ms timeout (`if (rs.Wait(17)) rs.Set();`) instead of spinning. 890936803 2021-04 “dsound: dedicated audio thread. configurable latency” · c2916e6f2 2019-03 “Wait between two frames when gui is open” • Also seen in PostgreSQL (postgresql-162): 2746e5f21d 2010-09-11 latches (self-pipe) wake walsender as soon as a transaction finishes; 723d0184e2 2010-09-15 the startup process wakes immediately on new WAL; a180776f7a 2011-08-11 uses poll() instead of select(); 08882ce74c 2010-03-24 fixes a ms/µs bug that made walsender poll 1000x too often. • 5461564a9d 2012-05-08, 49340037ee 2012-05-08, 398b240151 2012-05-12 and 6308ba05a7 2012-05-09 let walwriter, checkpointer, stats collector, syslogger and bgwriter hibernate; a64ca63e59 2013-06-15 uses WaitLatch in pg_sleep. • 98a64d0bd7 2016-03-21 WaitEventSet with an epoll backend and a long-lived set, fixing a nearly 3x regression. • 05a7be9355 2022-11-08 walreceiver tracks per-reason wakeup times instead of waking 10 times a second.

### dolphin-227 — Derive counters from timestamps instead of ticking them
Lens: laziness · Source: Dolphin · Also: Seastar
Principle: A counter that advances at a known rate does not need a periodic event to update it. Store a base value and a base time, compute `base + (now - t0) / period` on read, and schedule wakeups only for the moments that matter, such as interrupts.
Why it is faster: —
Applies elsewhere: token-bucket rate limiters that refill on read, lazy TTL expiry, and game cooldowns computed from timestamps.
Watch out: —
Seen in Dolphin: The audio interface's sample counter was updated by an event every half period. Reads now return `m_SampleCounter + (GetTicks() - g_LastCPUTime) / g_CPUCyclesPerSample`. The event is scheduled for the next interrupt point, or one second at most, so it fires far less often. 0e0fec3075 2015-09 “AI: reduce overhead of sample counter” • Also seen in Seastar (seastar-032): 6e1d7193d1 2022-01-10 5025cd44ea 2022-01-11 `fair_queue::maybe_replenish_capacity` replaced the 500 µs `_replenisher` timer with lazy replenishment from elapsed time: no user time while idle and no single-shard dependency.

### dolphin-228 — Let the GPU consume formats it supports natively
Lens: gpu-rendering · Source: Dolphin
Principle: Decompressing or reformatting data on the CPU before upload costs CPU time and bandwidth. Upload block-compressed textures as-is when the driver supports the format, decode the rest in a compute shader, and skip intermediate copies by writing straight into the buffer the shader reads.
Why it is faster: —
Applies elsewhere: shipping BCn/ASTC/KTX2 textures in web and mobile games, and video decoded to GPU surfaces without a CPU round trip.
Watch out: —
Seen in Dolphin: The texture cache gained `DecodeTextureOnGPU` for supported formats. OGL, D3D and Vulkan upload DXT1/3/5 hi-res textures natively (`GL_COMPRESSED_RGBA_S3TC_*`). Vulkan XFB decoding copies into a texel buffer instead of going through an intermediate texture. 79ba946d70 2016-11 “VideoCommon: Changes to TextureCache to support decoding in backend” · f5d95dcc86 2017-04 “OGL: Use native compressed textures on supported drivers” · 58978c1440 2016-11 “Vulkan: Faster path for decoding XFB data”

### dolphin-229 — Choose upload paths with the least driver overhead
Lens: gpu-rendering · Source: Dolphin
Principle: Persistent coherent mappings avoid per-write flush calls, and default-usage resources updated through the driver's copy path avoid map/unmap synchronization. Explicit flushing or dynamic resources can be much slower on some vendors.
Why it is faster: —
Applies elsewhere: WebGPU `writeBuffer` versus mapping, Vulkan host-coherent staging rings, and D3D11 `UpdateSubresource` versus `Map(WRITE_DISCARD)`.
Watch out: —
Seen in Dolphin: The OGL `BufferStorage` stream buffer now defaults to coherent mapping for all drivers. The commit notes that "Qualcomm and ARM performs very bad with explicit flushing". D3D11 texture-cache textures all use `D3D11_USAGE_DEFAULT` plus `UpdateSubresource`, where single-level textures had been `DYNAMIC`. fdfd8b19d6 2018-05 “OGL/Streambuffer: Use coherent mapping by default.” · 526fa9bc85 2017-03 “D3D11: Use D3D11_USAGE_DEFAULT for all TextureCache textures”

### dolphin-230 — Recognize operations that are no-ops and emit nothing
Lens: other · Source: Dolphin
Principle: Some general operations reduce to nothing, or to something trivial, for particular operands: OR with zero into itself, a blend that keeps the destination, a filter tap whose weight is zero. Detect those cases and skip the work or switch to the trivial form.
Why it is faster: —
Applies elsewhere: algebraic simplification in query planners (`WHERE 1=1`), skipping zero-weight kernel taps in image convolution, and short-circuiting `x * 1` in tensor graphs.
Watch out: —
Seen in Dolphin: The `ori`/`oris`/`xori` NOP check now covers `a == s && imm == 0`, not only r0. Blend logic op `NOOP` becomes `colorupdate = false` (a write mask) instead of enabling logic ops. EFB-copy shaders skip sampling adjacent rows when `NeedsCopyFilterInShader` finds the top and bottom coefficients are zero. Redundant JIT instructions whose results were immediately overwritten were deleted, and the dispatcher reuses a register copy of the PC instead of reloading it. 42fce74f39 2018-03 “Jit_Integer: Handle NOP case where RA == RS for ori” · f6c7a258ed 2016-12 “BlendState: Use masking as a fastpath for the logic op NO_OP.” · 4faac3a627 2018-05 “TextureConversionShader: Don't sample from adjacent rows when not needed” · e3b424facd 2018-09 “EmuCodeBlock: remove redundant instructions” · 1737f1bee8 2018-04 “Jit64: avoid an unnecessary load” · fbdc20ac37 2015-08 “Jit64: load PC only once after fallbacks”

### dolphin-231 — Hand data to the consumer in the format it already accepts
Lens: memory-layout · Source: Dolphin
Principle: A per-pixel copy loop that converts RGBA to BGR, only for the encoder to convert again, doubles memory traffic. Pass the mapped buffer directly with its native format tag and let the single real conversion happen once.
Why it is faster: —
Applies elsewhere: passing `ImageBitmap` instead of re-encoding canvases, and zero-copy buffers between decoders and renderers.
Watch out: —
Seen in Dolphin: D3D/D3D12 frame dumping removed `formatBufferDump`, which copied 4-byte pixels to 3-byte BGR with a vertical flip. The renderer now calls `DumpFrameData(map.pData, …, FORMAT_RGBA)`. 1ef5ba0c53 2016-10 “D3D: Skip redundant format convertions.”

### dolphin-232 — Restrict slow paths to the pages that need them
Lens: memory-layout · Source: Dolphin
Principle: If a debugging or safety feature forces the slow path globally, everything gets slower. Keep the fast path everywhere and divert only the address ranges that actually need checking, using the page-table or translation mechanism that already exists.
Why it is faster: —
Applies elsewhere: write-protecting only watched pages (`mprotect` plus fault handler) instead of instrumenting every store, and copy-on-write on specific pages.
Watch out: —
Seen in Dolphin: Any memory breakpoint used to set `jo.alwaysUseMemFuncs` and disable fastmem entirely. Now `PowerPC::DBATUpdated()` leaves pages overlapping a memcheck unmapped from the fast BAT table (`overlaps_memcheck`), so only those accesses take slowmem. 52fe05af6b 2017-02 “Make memory breakpoint faster”

### dolphin-233 — Allocate related code regions contiguously to keep jumps short
Lens: jit-interpreters · Source: Dolphin
Principle: If generated code, helper routines and trampolines are allocated separately, they can land too far apart for short relative branches. Generated code then falls back to absolute indirect calls. One allocation split into child regions guarantees every cross-reference fits in a rel32.
Why it is faster: —
Applies elsewhere: linker sections kept within ±2 GB for RIP-relative addressing, and arena allocation for related objects.
Watch out: —
Seen in Dolphin: Jit64 now calls `AllocCodeSpace(CODE_SIZE + routines + trampolines + farcode + constpool)` once and carves it up with `AddChildCodeSpace`, "This ensures that relative memory offsets remain within architecture limits". cb7d9556bf 2017-03 “Jit64: Merge memory allocations into a single allocation”

### dolphin-234 — Fire change notifications once per transaction
Lens: other · Source: Dolphin
Principle: If a save touches N sub-stores and each one notifies listeners, the listeners run N times for one logical change. Raise the notification once at the outer level.
Why it is faster: —
Applies elsewhere: batching React state updates, wrapping DOM mutations so a single reflow occurs, and emitting a single "settings changed" event after a bulk import.
Watch out: —
Seen in Dolphin: `InvokeConfigChangedCallbacks()` moved from `Layer::Save`, which ran up to 7 times per save, to `Config::Save`/`Load`. 93f49b1ca4 2017-08 “Config: Make Load() and Save() slightly faster”

### dolphin-235 — Avoid incidental copies and heap allocations of static data
Lens: allocation · Source: Dolphin
Principle: Iterating a map by value copies every key and string. Static tables built with `new` and `std::string` pay for an allocation per entry at startup. Iterate by reference and keep fixed metadata in plain structs with string literals.
Why it is faster: —
Applies elsewhere: `for (const auto& [k, v] : m)`, `constexpr` lookup tables, and `string_view` for constant names.
Watch out: —
Seen in Dolphin: `GameFile::GetLanguages` iterated `std::pair<Language, std::string>` by value and now uses `const auto&`. `LogManager` replaced roughly 50 heap-allocated `LogContainer` objects holding `std::string` names with a POD array of `const char*`. d3ed750c9d 2018-06 “GameFile: Avoid copying map pairs in GetLanguages()” · 8ff8dfcdf2 2017-07 “LogManager: make LogContainer a POD and don't allocate”

### dolphin-236 — Defer device-to-host readbacks until a synchronization point
Lens: gpu-rendering · Source: Dolphin
Principle: Reading results back from an asynchronous device (GPU, DMA engine, remote service) forces the host to wait until the device drains its queue. If the consumer can't observe the data until a later sync event anyway, queue the copy and complete all pending copies together at that event. Many stalls become one.
Why it is faster: —
Applies elsewhere: WebGL `readPixels` into a PBO with a fence instead of a blocking read; batching DB reads until a transaction commit; flushing write-behind caches at `fsync` only.
Watch out: —
Seen in Dolphin: EFB-to-RAM copies used to encode, flush, map and read back immediately in each backend's encoder (e.g. `PSTextureEncoder::Encode` called `Flush()`/`Map()`/`ReadTexels`). `TextureCacheBase` now has the encoder write into an `AbstractStagingTexture` and keeps a list of pending copies. `FlushEFBCopies()` is called from `BPWritten` on draw-done and PE-token writes, which are the points where the game synchronizes with the GPU. The setting defaults to on. 8e2c063d62 2018-11 “TextureCache: Implement deferred/batched EFB copies”

### dolphin-237 — Submit work to the device only once a batch is big enough
Lens: gpu-rendering · Source: Dolphin
Principle: Every submission has a fixed cost: driver overhead, a kernel transition, lost overlap. When a submission is only "nice to have" early, skip it unless enough work has piled up since the last one. That keeps the number of submissions proportional to the work done rather than to the events that happen.
Why it is faster: —
Applies elsewhere: Nagle-style coalescing of small socket writes; flushing a log buffer only after N records; Vulkan/Metal apps grouping draws into fewer command buffers.
Watch out: —
Seen in Dolphin: In deferred-copy mode, the Vulkan `StateTracker::OnEFBCopyToRAM()` calls `ExecuteCurrentCommandsAndRestoreState` only when at least `MINIMUM_DRAW_CALLS_PER_COMMAND_BUFFER_FOR_READBACK` draws have happened since the last EFB copy. Otherwise the copy stays in the current command buffer. 66b6e72c5e 2018-11 “Vulkan: Submit fewer command buffers in deferred EFB copies mode”

### dolphin-238 — Flag a cache stale, and drop it only at the next sync point
Lens: laziness · Source: Dolphin
Principle: Invalidating a cache on every write means you pay to rebuild it again and again, even when nobody reads it in between. Instead set an "out of date" flag on write and actually discard the cache at the next point where a reader could notice the difference. Writes in between cost nothing.
Why it is faster: —
Applies elsewhere: UI frameworks marking layout dirty and recomputing once per frame; CDN purges batched per deploy; memoized selectors recomputed only when read.
Watch out: —
Seen in Dolphin: `FramebufferManager::FlagPeekCacheAsOutOfDate()` replaced direct `InvalidatePeekCache()` calls in `ClearScreen` and draw paths. With "Defer EFB Cache Invalidation" enabled, the real `InvalidatePeekCache(false)` happens only on draw-done and PE-token writes in `BPStructs.cpp`. Resizes and format reinterpretation still force it with `InvalidatePeekCache(true)`. a218a794cb 2019-03 “FramebufferManager: Implement deferred EFB cache invalidation”

### dolphin-239 — Cache remote data in tiles and fetch only the tile that was asked for
Lens: memory-layout · Source: Dolphin
Principle: If callers read single elements out of a large remote resource, copying the whole resource on the first miss wastes bandwidth and adds latency. Split it into tiles, keep a valid bit per tile, and fetch only the tile that holds the requested element.
Why it is faster: —
Applies elsewhere: map tile servers; page-granular mmap of remote files; chunked range requests for large blobs.
Watch out: —
Seen in Dolphin: CPU peeks into the emulated framebuffer used to read back the whole EFB. `FramebufferManager` gained a tile cache with a configurable size (`EFBAccessTileSize`, default 64). `IsEFBCacheTilePresent` picks the tile and `PopulateEFBCache(depth, tile_index)` copies only that rectangle. A `tiles` bitmap tracks validity. 6bc4bfd26a 2019-03 “FramebufferManager: Implement EFB tile cache”

### dolphin-240 — Move slow or blocking side work off the latency-critical thread
Lens: concurrency · Source: Dolphin · Also: Flycast
Principle: A slow driver call, DNS lookup or text formatting step on the main loop stalls everything behind it. Hand it to a worker thread or queue, and have the hot thread only signal or enqueue. The hot path's cost becomes one cheap handoff.
Why it is faster: —
Applies elsewhere: async logging libraries (spdlog async, log4j AsyncAppender); browsers resolving DNS off the main thread; game engines streaming audio and haptics on separate threads.
Watch out: —
Seen in Dolphin: DirectInput force-feedback updates now run on a dedicated thread in `ForceFeedbackDevice::ThreadFunc`. The input path just calls `SignalUpdateThread()`. The Qt `LogWidget::Log` copies the string and does its HTML formatting inside `QueueOnObject`, on the UI thread. `IOCTL_SO_GETHOSTBYNAME`/`GETADDRINFO` go through `LaunchAsyncTask` on a work queue, and their replies are drained in `NetIPTopDevice::Update()`. 0f19c4a40f 2018-12 “DInput: Update force feedback effects in a thread” · 0abb1c6ea6 2019-02 “Qt/LogWidget: Make logging asynchronous” · 0553aeb0e0 2020-06 “IOS/Network: Make name resolution asynchronous” • Also seen in Flycast (flycast-093): On Android, every touch-control press called `Vibrator.vibrate()` directly, which lagged when buttons were mashed. A `VibratorThread` now waits on its monitor, and `vibrate()` just sets `vibrate = true; notify()`. Game-list scanning moved to a background `GameScanner` (`core/rend/game_scanner.h`). Game loading now runs `dc_run` on its own thread, behind a loading screen with a cancel button, so the UI doesn't freeze. 2d03662a9 2019-04 “android: run the vibrator in a separate thread” · 381f0f0f9 2020-04 “ui: scan games in a background thread...” · 17a2c3644 2020-04 “ui: add loading screen with cancel button. don't freeze ui on load”

### dolphin-241 — Keep a handle you already know instead of asking the OS again on every poll
Lens: io · Source: Dolphin
Principle: OS queries such as "which window is under this point" and "which process owns it" are system calls. Run on every input poll, they add up. If the answer is known when the object is created, store it once and use it directly.
Why it is faster: —
Applies elsewhere: caching a resolved file descriptor or socket instead of reopening by path; storing `document.getElementById` results instead of querying in a `requestAnimationFrame` loop.
Watch out: —
Seen in Dolphin: On every update, the DInput `GetMousePos` called `WindowFromPoint` and `GetWindowThreadProcessId` and compared the result with the current process ID. `KeyboardMouse` now receives the render `HWND` in its constructor, and `UpdateCursorInput()` calls only `ScreenToClient(m_hwnd)` and `GetClientRect(m_hwnd)`. 8c1310d1d1 2019-04 “ControllerInterface/DInput: Optimize cursor position updating.”

### dolphin-242 — Detect spin-wait loops statically and fast-forward time
Lens: algorithms · Source: Dolphin
Principle: A loop that only polls a value nothing else in the loop changes does no useful work until an external event fires. If static analysis can prove the loop is side-effect-free, replace each pass with a jump straight to the next event. Hours of spinning collapse into one scheduler call.
Why it is faster: —
Applies elsewhere: virtual machines trapping x86 `PAUSE`/`HLT` loops; discrete-event simulators jumping to the next scheduled event instead of ticking.
Watch out: —
Seen in Dolphin: Idle-skip detection used to be a hard-coded opcode pattern inside `Jit64::lXXx` and the ARM64 equivalent. `PPCAnalyst` now analyses blocks generically and sets `branchIsIdleLoop` on a branch. `bx`, `bcx` and `DoMergedBranch` then emit `ABI_CallFunction(CoreTiming::Idle)` followed by an exception exit, for any loop shape that qualifies. 55db7c7a05 2018-07 “Jit64: Optimized idle skipping detection.”

### dolphin-243 — Parse the query once, then scan with a prebuilt predicate
Lens: algorithms · Source: Dolphin
Principle: If a search re-parses its input for every element it tests, most of the work is redundant parsing. Convert the input once, capture the parsed value and operator in a closure, and run that closure over the data.
Why it is faster: —
Applies elsewhere: compiling a regex or filter expression once before a scan; building a SQL prepared statement; turning a user filter into a JS function before filtering a big array.
Watch out: —
Seen in Dolphin: `CheatsManager::MatchesSearch(addr)` called `text.toUShort()`/`toFloat()` once per memory address. `CreateMatchFunction()` now converts the text once, validates it, and returns a `std::function<bool(u32)>` lambda holding `comparison_value` and `op`. 4ce7079098 2018-10 “CheatsManager: Improve performance of searching & add input validation”

### dolphin-244 — Match the container's exact element type when iterating by reference
Lens: compiler-build · Source: Dolphin
Principle: A `std::map` stores `pair<const K, V>`. Binding `const pair<K, V>&` to that silently builds a converted temporary copy of every element. Use `auto&` or structured bindings so the reference binds directly.
Why it is faster: —
Applies elsewhere: any C++ loop over `unordered_map` with an explicit pair type; Rust `for (k, v) in &map` instead of `.clone()`.
Watch out: —
Seen in Dolphin: `HttpRequest::Impl::Fetch` looped with `const std::pair<std::string, std::optional<std::string>>& header`, which copied both strings for every header. It now uses `for (const auto& [name, value] : headers)`. b15f595130 2019-05 “Common/HttpRequest: Avoid unnecessary copies in loop in Fetch()”

### dolphin-245 — Simplify operations whose operands are the same
Lens: algorithms · Source: Dolphin
Principle: When both inputs of a binary operation are the same value, many operations reduce to a constant or to a copy: `x ^ x = 0`, `x & x = x`, `~(x | x) = ~x`. Check for equal operands first and emit the reduced form. That avoids a second load and the ALU work.
Why it is faster: —
Applies elsewhere: query planners folding `WHERE a = a`; compiler peephole passes; expression simplifiers in spreadsheets and CAS tools.
Watch out: —
Seen in Dolphin: `Jit64::crXXX` handled only `crclr`/`crset` when `CRBA == CRBB`. It now handles all eight ops: crandc and crxor clear the bit, creqv and crorc set it, crand and cror copy A, and crnor and crnand copy ~A. Each loads the source bit only once. 174d2b0230 2018-12 “crXXX: Avoid loading twice when CRBA == CRBB”

### dolphin-246 — Choose the shortest instruction encoding for the same operation
Lens: jit-interpreters · Source: Dolphin
Principle: Smaller machine code means fewer instruction-cache misses and fewer bytes for the decoder. Many operations have several encodings: `test r,r` instead of `bt r,63`, `add r,r` instead of `lea r,[r*2]`, no displacement byte when the offset is zero, and sign-extended 8-bit immediates where they fit. A code generator should pick the shortest one.
Why it is faster: —
Applies elsewhere: assembler relaxation passes; WebAssembly LEB128 constants; V8/LuaJIT backends picking short forms; firmware size tuning.
Watch out: —
Seen in Dolphin: `OpArg::WriteRest` stopped emitting a zero 8-bit displacement unless the base register is RBP/R13. `GenFrsqrte` uses `TEST` plus `J_CC(CC_S)`. The DSP JIT replaced same-register `LEA`s with `ADD`/`SHL` and `[rax*2]` with `[rax+rax]`, and turned `MOV`+`ADD 1` into one `LEA`. `boolX` xor uses `MOV imm; XOR reg` when the immediate doesn't fit in a signed byte. cde3a3b448 2020-01 “x64Emitter: Avoid 8-bit displacement when possible” · e9d4869965 2019-06 “Micro-optimize GenFrsqrte” · 49a4e09719 2020-01 “DSPJit: Replace/optimize LEAs” · 9b6f2859a4 2019-06 “Micro-optimize increment_addr_reg” · 845d7cd51f 2021-01 “boolX - Optimize xor for size”

### dolphin-247 — Fold known values into the generated code
Lens: jit-interpreters · Source: Dolphin
Principle: If an input is known when code is generated, compute whatever depends on it right then: shifts by a known amount, complements of constants, sign extension of immediates, and adds of zero. The output then contains only the instructions that remain. Removing variable shifts also frees the fixed shift-count register and cuts spills.
Why it is faster: —
Applies elsewhere: template engines precompiling static fragments; shader specialization constants; SQL engines folding `WHERE 1=1`; C++ `constexpr`.
Watch out: —
Seen in Dolphin: `Jit64::srwx`/`slwx`/`srawx`/`rlwnmx` emit `SHR r, imm8` when `gpr.IsImm(b)`, or a constant zero for amounts of 32 and up, instead of loading ECX. `boolX` computes `~imm` for andc/orc at JIT time. DSP `addi` becomes `MOV(64, RDX, Imm32(imm << 16))` instead of MOV, MOVSX and SHL. `addx` skips `ADD 0`. JitArm64 got matching ports. 17db359979 2020-11 “Jit64: srwx - Optimize shift by constant” · 26f70657bc 2021-01 “boolX - Precompute complement of b” · 3951d238d9 2019-06 “Precompute addi immediate” · 89646c898f 2020-04 “addx - Skip ADD after MOV when possible” · efeda3b759 2020-12 “JitArm64: More constant propagation optimizations”

### dolphin-248 — Replace division by a constant with multiply and shift
Lens: algorithms · Source: Dolphin
Principle: Integer division takes tens of cycles. Dividing by a known constant d equals multiplying by a precomputed "magic" reciprocal and then shifting, with a small sign fix-up. Powers of two need only a bias, conditional move and arithmetic shift. Special-case d = 0, 1 and −1, where the general method doesn't work.
Why it is faster: —
Applies elsewhere: what GCC and LLVM do for `x / 7`; hash-table modulo by a fixed prime (libdivide, fastmod); fixed-point DSP code.
Watch out: —
Seen in Dolphin: `Jit64::divwx` with `gpr.IsImm(b)` calls `JitCommon::SignedDivisionConstants(divisor)` and emits `IMUL`, `SHR`/`SAR` and `ADD` in place of the guarded `IDIV` sequence. For power-of-two divisors it emits `TEST; LEA r,[x+d-1]; CMOVNS; SAR log2(d)`. Division by 1 compiles to a single `MOV`. 95698c5ae1 2021-03 “Jit64: divwx - Optimize constant divisor” · 0637a7ec59 2021-03 “Jit64: divwx - Optimize power-of-two divisors”

### dolphin-249 — Choose code paths by measured speed, not only by whether a feature exists
Lens: platform · Source: Dolphin
Principle: "The CPU supports instruction X" doesn't mean X is fast. Some microarchitectures microcode an instruction that others run in one cycle. Keep a separate "fast" flag next to the "supported" flag and branch on that. Apply the same idea to choosing default algorithms: turn on by default only the ones with hardware acceleration.
Why it is faster: —
Applies elsewhere: simdjson/zstd runtime dispatch tables per microarchitecture; OpenSSL capability masks; video codecs disabling AVX-512 paths on CPUs that downclock.
Watch out: —
Seen in Dolphin: `CPUInfo` gained `bFastBMI2 = bBMI2 && !bZen`, since PDEP/PEXT are very slow on early Zen. `VolumeVerifier::GetDefaultHashesToCalculate()` turns on CRC32 only when `cpu_info.bCRC32` is set, keeps SHA-1 on, and turns MD5 off by default when any hardware acceleration is present. `SHA1::Context` exposes this through `HwAccelerated()`. 185e49d2a9 2020-01 “x64CPUDetect: Add flag for slow PDEP/PEXT on AMD Zen” · d71797154a 2022-07 “VolumeVerifier: enable fast hash functions by default”

### dolphin-250 — Make the callable or flag a template parameter instead of a runtime value
Lens: compiler-build · Source: Dolphin · Also: Flycast
Principle: Type-erased callables (`std::function`) and runtime bool flags block inlining and add indirect calls or branches. When the concrete type or flag is known at compile time, make it a template parameter. The compiler then generates a specialized copy with the dead side removed.
Why it is faster: —
Applies elsewhere: Rust generics compared with `dyn Fn`; C++ `std::sort` taking a comparator by template; Go generics for hot inner loops.
Watch out: —
Seen in Dolphin: `Common::ScopeGuard` changed from holding `std::function<void()>` to `template <typename Callable>` with `std::optional<Callable>`, and CTAD keeps call sites unchanged. `VertexLoaderManager::RunVertices<bool IsPreprocess>` and `GetOrCreateLoader<IsPreprocess>` now pick the state, dirty bits and loader array with `constexpr` instead of runtime ternaries. 44b4c2db49 2020-02 “Common: Avoid std::function overhead in ScopeGuard” · a6c6ec012c 2022-09 “VertexLoaderManager: Clean up and slightly speed up with templates” • Also seen in Flycast (flycast-019): `shil_canonical` in `shil_canonical.h` now wraps each op as `struct name { static rv impl args {...} };`. `rec_cpp.cpp` adds `opex2<T>` variants that call `&T::impl` directly, through `createType2<shilop, CTR>`, and keeps the indirect handlers only as a fallback. `sync_sr`/`sync_fpscr` got wrapper impls because MSVC was otherwise optimizing the functions away. 7303a046b 2015-07 “rec-cpp: Direct calls to shil_opcl_* handlers for many opcodes” · b1f7015a9 2015-07 “rec/cpp: Mostly fully direct specialized dispatchers” · fb55e7f6f 2015-07 “sh4/canonical: Provide impls for sync_sr and sync_fpscr”

### dolphin-251 — Use non-destructive three-operand SIMD forms to drop register copies
Lens: simd · Source: Dolphin
Principle: Legacy two-operand SIMD instructions overwrite one input, so you need a `MOVAPS` copy whenever that input is still needed. VEX/AVX encodings take a separate destination register, so the result can go straight where it belongs and the copy disappears.
Why it is faster: —
Applies elsewhere: compiling with `-mavx` so compilers emit `vaddps dst, a, b`; hand-written codecs and BLAS kernels; ARM NEON, which is already three-operand.
Watch out: —
Seen in Dolphin: `Jit64::fselx` has an AVX path that uses `VBLENDVPD(XMM1, src1, Rb, XMM0)` with the cached register as `src1`, and the packed variant writes directly to `Rd`. That removes a `MOVAPS` and a `MOVSD`. `GenConvertDoubleToSingle` uses `avx_op(VPAND, PAND, XMM1, R(XMM0), …)` instead of `MOVAPD` followed by `PAND`. a52774ca63 2020-07 “Jit64: fselx - Add AVX path” · afb86a12ab 2020-07 “fselx - Optimize AVX packed” · 08bdeefe05 2020-08 “Use AVX in ConvertDoubleToSingle”

### dolphin-252 — Look a key up once instead of find-then-get, and never insert on a read
Lens: algorithms · Source: Dolphin
Principle: "Find which layer has the key" followed by "get the key from that layer" runs the same search twice. Merge them into one pass that returns the value. And on a read, use `find` rather than `operator[]`: `operator[]` inserts an empty entry on a miss, which makes the map grow and slows later lookups.
Why it is faster: —
Applies elsewhere: Python `d.get(k)` instead of `if k in d: d[k]`; Java `computeIfAbsent`; layered config systems (env, file, defaults) resolved in one walk.
Watch out: —
Seen in Dolphin: `Config::Get(info)` called `GetActiveLayerForConfig` and then `Layer::Get`, searching twice. The new `GetAsString` walks `SEARCH_ORDER` once and returns the first value it finds. Earlier, `Layer::Get` and `DeleteKey` switched from `m_map[location]` to `m_map.find`, so reads stopped creating empty `std::optional` entries. 2f264c6448 2020-11 “Common: Optimize Config::Get” · 48a4b62125 2019-07 “Change Layer code not to create superfluous std::optional entries in LayerMap”

### dolphin-253 — Read settings once when they change, not on every hot-path call
Lens: laziness · Source: Dolphin
Principle: A settings lookup that takes a lock, searches maps and parses a string is cheap on its own but costly inside a per-tick function. Keep the derived value in a plain variable, update it from a change callback, and read only that variable on the hot path. Inside a loop, also hoist singleton accessors out of the loop body.
Why it is faster: —
Applies elsewhere: feature-flag SDKs caching evaluated flags locally with push updates; reading `process.env` once at startup; caching `getComputedStyle` outside an animation loop.
Watch out: —
Seen in Dolphin: `CoreTiming::Advance()` called `Config::Get(MAIN_OVERCLOCK…)` on every slice. `RefreshConfig()` now caches `s_config_OC_factor` and its inverse, and runs from `AddConfigChangedCallback`. `VertexShaderManager::SetConstants` stopped calling `Config::Get(GFX_FREE_LOOK_CONTROL_TYPE)` on every call; `Renderer::CheckForConfigChanges` applies it instead. `RunGpuLoop` calls `SConfig::GetInstance()` once, outside the loop lambda. 7625cb7aca 2021-12 “CoreTiming: Cache the overclock factor instead of re-querying it every Advance().” · 0441d6aa5e 2020-06 “VertexShaderManager: Avoid accessing freelook control type configuration each call” · 1183fb3d2d 2021-06 “Move SConfig::GetInstance() outside the GPU loop.”

### dolphin-254 — Turn a byte- or halfword-aligned bit insert into a narrow store
Lens: jit-interpreters · Source: Dolphin
Principle: The general way to insert bits under a mask is AND-NOT, AND, OR. When the mask covers exactly the low 8 or 16 bits, a single partial-width move writes those bits and leaves the rest untouched.
Why it is faster: —
Applies elsewhere: packing RGBA channels with byte stores instead of shift/mask/or; updating one field of a packed header struct; SIMD byte-insert instructions (`pinsrb`).
Watch out: —
Seen in Dolphin: `Jit64::rlwimix` now checks for `mask == 0xFF || mask == 0xFFFF` and emits `MOV(8 or 16, Ra, src)` in place of the XOR/AND/XOR or AND/OR sequences. It sets `needs_test` so the flags are still computed correctly. cdcca72b61 2020-12 “Jit_Integer: Optimize common rlwimix cases”

### dolphin-255 — Use liveness to skip computing results nobody reads
Lens: laziness · Source: Dolphin
Principle: Side outputs such as carry flags, exact rounding and NaN canonicalization cost extra instructions. Run a backward pass that records which outputs are consumed downstream, then emit the cheap variant wherever the expensive one's extra output is dead or can't be observed.
Why it is faster: —
Applies elsewhere: compilers' dead-flag elimination; GraphQL resolvers computing only requested fields; database projections that never materialize unread columns.
Watch out: —
Seen in Dolphin: JitArm64's `CARRY_IF_NEEDED(ADD, ADDS, …)` picks the flag-setting form only when `js.op->wantsCA`. `PPCAnalyst` tracks `fprIsStoreSafeBeforeInst`/`AfterInst`, and `ConvertDoubleToSingleLower` emits a single `FCVT` for store-safe registers instead of calling the bit-exact routine. Register-cache flushes then pick whichever conversion is cheaper for that case. `Jit64::fp_arith` skips `HandleNaNs` for add and subtract, because x86 already propagates NaNs correctly for them. baecddd262 2021-03 “JitArm64: Skip calculating carry flag when not needed” · 2a9d88739c 2021-02 “JitArm64: Skip accurate single/double conversion if store-safe” · 54451ac731 2021-02 “Use ConvertSingleToDoubleLower in RW when faster” · 2f1a8ee1b9 2022-11 “Jit64: Skip HandleNaNs for operations that can't generate NaN”

### dolphin-256 — Track where a value currently lives, including when it is a known constant
Lens: jit-interpreters · Source: Dolphin
Principle: If the generator records only "in memory" or "in a register", it has to reload or rematerialize values it already knows. Use a richer state: in memory, in the host carry flag, in the inverted host carry flag, or a known constant. Each consumer then picks the cheapest way to use the value, and a known constant gets baked in as an immediate.
Why it is faster: —
Applies elsewhere: SSA constant/copy propagation lattices; register allocators that track rematerializable values; interpreters caching the top of the stack in a register.
Watch out: —
Seen in Dolphin: `js.carryFlagSet`/`carryFlagInverted` became `enum CarryFlag { InPPCState, InHostCarry, InHostCarryInverted, ConstantTrue, ConstantFalse }`. JitArm64 `subfex`/`addzex`/`subfzex` now branch on it. For a constant carry they emit a plain `SUBS`/`NEGS` or `ADD` and skip the `LDRB xer_ca`. 4c2cdb61df 2021-03 “JitArm64: Constant carry flag optimizations”

### dolphin-257 — Don't load a value you are about to overwrite
Lens: jit-interpreters · Source: Dolphin
Principle: If a register-cache bind can't tell reads from writes, it loads or materializes the old value even when the very next instruction overwrites it. Pass a write-only mode down to the bind and skip the load. Likewise, if the final value is known, write it directly instead of loading the old value and adjusting it.
Why it is faster: —
Applies elsewhere: `O_TRUNC` instead of reading a file you'll rewrite; GPU render passes with `LOAD_OP_DONT_CARE`; `calloc`/`mmap` zero pages instead of copying.
Watch out: —
Seen in Dolphin: `Arm64GPRCache::BindToRegister` gained an `Immediate` case that emits `MOVI2R` only `if (do_load)`. This removed sequences like `mov w27,#0x18; lsr w27,w22,#24`. In `Jit64::stX`/`stfXXX` with an immediate base, the update of `Ra` binds as `RCMode::Write` and emits `MOV(Ra, Imm32(addr))` instead of `ADD(Ra, offset)`. 9d73583ea3 2022-09 “JitArm64: Skip redundant imm to register writes” · 4b6bad2f9e 2022-11 “Jit64: Use MOV instead of ADD for updating imm Ra”

### dolphin-258 — Use the target's fused operand forms and break false dependencies
Lens: jit-interpreters · Source: Dolphin
Principle: Many ISAs can extend, shift or permute an operand as part of another instruction. Use those forms to save instructions and scratch registers. When a two-instruction sequence partially writes its destination, start with a full-width move, so the CPU doesn't have to wait on the register's stale old value.
Why it is faster: —
Applies elsewhere: ARM `add x0, x1, w2, sxtw` in compiled code; x86 `xor eax,eax` before `setcc`; NEON `EXT` for lane rotations in codecs.
Watch out: —
Seen in Dolphin: JitArm64 `cmp` now emits `SXTW` followed by `SUB(CR, CR, RB, ArithOption(RB, SXTW))`, which drops one instruction and the temporary register. `ps_merge10` became one `EXT`, and `ps_merge01` starts with a `MOV` to avoid a false dependency. `FloatCompare` produces each CR result with one `MOVI2R` instead of an initial set followed by ORRs. d0de68c41b 2022-11 “JitArm64: cmp - Optimize general case” · f45d3a6a2c 2022-11 “JitArm64: Optimize ps_mergeXX” · 3af21d3d22 2021-07 “Optimize FloatCompare's CR value emitting”

### dolphin-259 — Bulk-copy contiguous bytes instead of one call per byte
Lens: memory-layout · Source: Dolphin
Principle: A loop that writes one byte per call pays call overhead and bounds logic on every byte, and the compiler can't vectorize it. When the destination is a plain contiguous buffer, use `memcpy`/`std::copy` and advance the pointer once.
Why it is faster: —
Applies elsewhere: `ByteBuffer.put(byte[])` instead of per-byte puts; `TypedArray.set` in JS; `write(fd, buf, n)` instead of `putc` loops.
Watch out: —
Seen in Dolphin: `FifoPlayer::WriteFifo` called `GPFifo::FastWrite8(data[written++])` in a loop. It now does `std::copy(data + written, data + burstEnd, ppcState.gather_pipe_ptr)` and advances `gather_pipe_ptr` by the burst length. cc32fa91af 2021-04 “FifoPlayer: Copy data with memcpy instead of one byte at a time”

### dolphin-260 — Write straight into the output buffer, and skip buffers for data you'll reuse
Lens: allocation · Source: Dolphin
Principle: Building data in a growing temporary vector and then copying it into the final output means repeated reallocations plus one extra full copy. Measure the total size first, reserve that region in the output, and fill it in place. Likewise, check whether a chunk will be deduplicated before allocating space for it.
Why it is faster: —
Applies elsewhere: two-pass serializers (FlatBuffers, Cap'n Proto) that size first and then write; `StringBuilder` with capacity; content-addressed stores checking the hash before buffering a blob.
Watch out: —
Seen in Dolphin: `TextureCacheBase::SerializeTexture` used to append every mip and layer to a `std::vector<u8>` with `resize` and then `p.Do(texture_data)`. It now adds up `total_size`, gets a pointer from the new `PointerWrap::DoExternal(total_size)`, and calls `ReadTexels` straight into the savestate buffer. Deserialization reads from that pointer the same way. The WIA/RVZ writer calls `TryReuse()` before allocating `main_data` for a chunk. 43ceba4fef 2021-04 “optimize TextureCacheBase::SerializeTexture, ::DeserializeTexture” · 1e92b54bf5 2020-05 “WIA/RVZ: Skip some memory allocations when reusing chunks”

### dolphin-261 — Use a hash map of sets for a reverse index with frequent deletes
Lens: algorithms · Source: Dolphin
Principle: A `multimap` (a red-black tree) makes "remove this one value under key K" an O(n) scan over K's range, plus tree rebalancing. Store `unordered_map<K, unordered_set<V>>` instead. Lookup, insert and erase of a specific value all become expected O(1).
Why it is faster: —
Applies elsewhere: reverse dependency graphs in build systems, pub/sub subscriber maps, followers indexes, and DOM event-listener registries.
Watch out: —
Seen in Dolphin: `JitBaseBlockCache::links_to` changed from `std::multimap<u32, JitBlock*>` to `std::unordered_map<u32, std::unordered_set<JitBlock*>>`. `DestroyBlock` now does `it->second.erase(&block)` instead of an `equal_range` erase loop. Profiling had shown `DestroyBlock` above 50% of CPU time. According to the commit message, 7-second stutters in an invalidation-heavy game dropped to about 0.7 seconds. c812ab6a63 2021-04 “Jit: Optimize block link queries by using hash tables”

### dolphin-262 — Call hot helpers through a custom, minimal calling convention
Lens: jit-interpreters · Source: Dolphin · Also: Flycast
Principle: The standard ABI forces the caller to save every caller-saved register it has in use. If you write the helper yourself in generated assembly and document exactly which registers it clobbers, call sites need to save only those few. Call it with a direct branch-and-link instead of loading a function pointer.
Why it is faster: —
Applies elsewhere: runtime write barriers and allocation fast paths in GC'd VMs (V8, HotSpot stubs); compiler-rt builtins; the Linux kernel's `__preserve_most`-style helpers.
Watch out: —
Seen in Dolphin: JitArm64 `ConvertDoubleToSingleLower` called the C++ `ConvertToSingle` through `QuickCallFunction` and saved every used caller-saved GPR. `GenerateConvertDoubleToSingle()` now emits a routine that documents "Input in X0, output in W1, clobbers X0-X3". Call sites do `BL(cdts)` and save only `gpr.GetCallerSavedUsed() & BitSet32{0,1,2,3,30}`. Single-to-double conversion got the same treatment. 28e4869c43 2021-01 “JitArm64: Optimize ConvertDoubleToSingle” · 018e247624 2021-01 “JitArm64: Optimize ConvertSingleToDouble, part 1” • Also seen in Flycast (flycast-103): `GenCall(fn, skip_floats)` in `rec_x64.cpp` stopped spilling xmm8–11 around `ReadMemNoEx`/`WriteMemNoEx`. Later, the remaining xmm saves moved from `sub rsp` stack frames to `movd [rip + xmmSave[i]]` on non-Windows hosts. Returning memory reads in `eax`, the ABI return register, instead of copying to `ecx` removed a move per read. 714de0303 2019-05 “rec-x64: don't save fp regs when calling mem access handlers” · cfa21934b 2021-07 “x64: fix stack unwinding issue. don't save xmm regs on stack” · 2c3b51634 2019-06 “x64: use eax instead of ecx for mem reads return value”

### dolphin-263 — Recognize a whole loop idiom and run it in bulk
Lens: jit-interpreters · Source: Dolphin
Principle: A tight loop that applies the same operation to consecutive addresses pays its per-iteration cost (dispatch, checks, counters) N times. Pattern-match the full loop and compute how many iterations are allowed, here limited by the time budget. Then run them in one native loop and fix up the counters afterwards.
Why it is faster: —
Applies elsewhere: compilers turning byte loops into `memset`/`memcpy` calls; Python replacing a loop with `bytes.translate`; database batch executors.
Watch out: —
Seen in Dolphin: `Jit64::dcbx` checks whether the next instructions are `addi rX,rX,32; bdnz -8`. If so, it computes `min(downcount / cycles_per_loop, CTR - 1)` with a `DIV`, adjusts CTR, downcount and Rb by that amount, and invalidates that many cache lines in one native loop. The interpreted loop then runs only its final iteration. JitArm64 has the same optimization. 8b2f5d5006 2021-08 “Jit64: Optimize dcbx being called in a loop over a large memory region.”

### dolphin-264 — Limit spill and flush sets to registers the instruction actually touches
Lens: jit-interpreters · Source: Dolphin
Principle: "Flush everything not needed later" makes the generator scan, and sometimes emit stores for, registers this instruction never used. Intersect the dead set with the instruction's inputs and outputs. Only registers that just stopped being live get flushed.
Why it is faster: —
Applies elsewhere: incremental GC write barriers that track only touched cards; React reconciling only changed subtrees; dirty-region repaint.
Watch out: —
Seen in Dolphin: In `Jit64::DoJit` and `JitArm64::DoJit`, `gpr.Flush(~op.gprInUse)` became `gpr.Flush(~op.gprInUse & (op.regsIn | op.regsOut))`, and the same for FPRs. The commit message estimates 10–20% faster code generation on Jit64. fd511a689f 2022-02 “Jit: Skip redundant flushes”

### dolphin-265 — Pick the lightest synchronization the access pattern allows
Lens: concurrency · Source: Dolphin · Also: Tokio
Principle: A variable that is always accessed under a mutex doesn't also need to be atomic, because the mutex already orders the accesses. For data that is read often and written rarely, a reader-writer lock lets readers run in parallel instead of queueing behind each other.
Why it is faster: —
Applies elsewhere: Go `sync.RWMutex` for config maps; Java `ReadWriteLock` on caches; dropping `volatile`/atomics inside `synchronized` blocks.
Watch out: —
Seen in Dolphin: In `GCAdapter.cpp`, `s_controller_payload_size` went from `std::atomic<int>` to a plain `int` marked "Only access with s_mutex held". In `PerformanceTracker`, `m_mutex` became a `shared_mutex`: getters such as `GetHzAvg` and `ImPlotPlotLines` take `std::shared_lock`, while `Count`, `Reset` and `SetPaused` take `std::unique_lock`. 871b01a5d9 2022-03 “Remove unnecessary atomic usage in GCAdapter.cpp” · bc46089ab0 2022-12 “PerformanceTracker: Use shared_mutex instead of mutex” • Also seen in Tokio (tokio-079): aef2d64b0a 2021-07-16 `JoinError`'s panic payload: `Mutex<Box<dyn Any + Send>>` became `SyncWrapper`, which is `Sync` because it only allows access through ownership or `&mut`. • 37ca2f049c 2025-09-03 `SetOnce` dropped its own `lock: Mutex<()>`. `Notify` gained `lock_waiter_list()`, so `set` serializes on Notify's waiter-list lock and calls `notify_waiters` with it held. • d6bf379eba 2026-09-08 Broadcast `Slot::rem`: `AtomicUsize` became `usize` because the slot is already inside a `Mutex`. `fetch_sub(1, SeqCst)` became `rem -= 1`.

### dolphin-266 — Re-run a consistency check only after its inputs change
Lens: laziness · Source: Dolphin
Principle: A validation that runs on every operation repeats the same answer until one of its inputs changes. Set a dirty flag in the writers of those inputs and return early from the check when the flag is clear. A `[[likely]]` hint keeps the fast path on the fall-through side of the branch.
Why it is faster: —
Applies elsewhere: form validators that run only on changed fields; build systems that skip unchanged targets; ORMs that validate only dirty attributes.
Watch out: —
Seen in Dolphin: `VertexLoaderManager::CheckCPConfiguration` counted CP and XF colors, normals and texcoords on every draw. A new `g_needs_cp_xf_consistency_check` flag is set in `OpcodeDecoding` on MATINDEX, VCD and VAT writes and in `XFStructs`. The check returns early otherwise. According to the commit message, it now runs 350 times instead of 35,000. f179fd9c33 2022-11 “VideoCommon:VertexLoaderManager: Reduce CPConfig checks”

### dolphin-267 — Replace per-field conditionals with bit arithmetic
Lens: branching-fast-paths · Source: Dolphin
Principle: "If bit i is set, add 1" over many flags is just a popcount of a mask. "If bit 16 is set, add 0x8000, else add 0x7FFF" is "add 0x7FFF plus carry", where the carry comes from a bit-test instruction. Likewise, an OR of five flag-and-enable pairs is one shift-and-mask when the two groups have the same layout. Each rewrite removes several data-dependent branches.
Why it is faster: —
Applies elsewhere: counting enabled features with `popcount(mask)`; branch-free rounding in fixed-point DSP; checking a permission mask with one AND instead of a chain of `if`s.
Watch out: —
Seen in Dolphin: `VertexLoaderBase::GetVertexSize` replaced its per-index `if (TexMatIdx) f(1)` checks with `Common::CountSetBits(vtx_desc.low.Hex & 0x1FF)` and dropped the `if (size != 0)` guards. DSP `round_long_acc` became `BT 16; ADC 0x7FFF; XOR low16`. The interpreter computes FEX as `((Hex >> 22) & (Hex & FPSCR_ANY_E)) != 0`. 8aa214453a 2022-09 “VertexLoader: Optimize GetVertexSize” · 832e320f08 2019-06 “DSPJitUtil: Make round_long_acc branchless” · 89a464dafa 2021-08 “Interpreter: Optimize FEX calculation”

### dolphin-268 — Wait on all sources at once with one readiness call
Lens: io · Source: Dolphin
Principle: Polling N sockets one after another, each with its own timeout, adds up their latencies, and a busy socket waits behind an idle one. Register all of them with one selector, block once, and serve only the ones that are ready.
Why it is faster: —
Applies elsewhere: `epoll`/`kqueue`/`select` event loops; Go `select` over channels; Node's libuv.
Watch out: —
Seen in Dolphin: The DSU client's `HotplugThreadFunc` called `ReceiveWithTimeout` on each server socket in turn, building a new `SocketSelector` each time. It now adds every `server.m_socket` to one `sf::SocketSelector`, calls `selector.wait` once, and receives only from sockets where `isReady` is true. dbb0b72cc5 2021-03 “InputCommon: instead of blocking on individual DSU server sockets, block on a selector”

### dolphin-269 — Write persistent state once per batch, and only when it changed
Lens: io · Source: Dolphin · Also: Flycast
Principle: Saving a cache or metadata file after every item in a bulk operation makes the operation's I/O grow with the number of items. Suppress saves while a batch is running and save once at the end. Also compare old and new values first, and skip the write when nothing changed.
Why it is faster: —
Applies elsewhere: package managers writing the lockfile once after resolving everything; IDE indexers committing at the end of a scan; `if (newVal !== oldVal) save()` in settings UIs.
Watch out: —
Seen in Dolphin: `GameTracker` sets `m_refresh_in_progress` on `BeginRefresh`, skips `m_cache.Save()` in `LoadGame` while it is set, and saves once on `EndRefresh`. The commit subject says this gave a large speedup. `HostFileSystem::SetMetadata` calls `SaveFst()` only if gid, uid, attribute or modes actually changed. `ARCodeWidget` no longer re-saves when an edit dialog is cancelled. 7c3cdeda2b 2022-08 “qt: defer writing gamelist cache until EndRefresh large speedup” · 0d39c4a78a 2022-08 “IOS/FS: Skip writing FST if SetMetadata() changes nothing.” · 4d8d2acae7 2019-07 “ARCodeWidget: Avoid unnecessary disk operations” • Also seen in Flycast (flycast-118): `cfgSetAutoSave(false)` now wraps the settings save loop, and `cfgSetAutoSave(true)` calls `savecfgf()` once at the end. Before, each `cfgSaveStr` rewrote `emu.cfg`. 4cc199b2b 2019-10 “minimize writes to config file”

### dolphin-270 — Batch per-item network exchanges, and make the common message tiny
Lens: io · Source: Dolphin · Also: Flycast
Principle: When each item does its own request and wait, the round trips add up. Gather all items for a tick and send them in one batch. Shrink the payload too: a presence bitmap in the header lets fields with default values take no space, so an idle update fits in a single byte.
Why it is faster: —
Applies elsewhere: GraphQL/DataLoader batching; protobuf's omission of default fields; game netcode delta-compressing input frames.
Watch out: —
Seen in Dolphin: `BluetoothEmuDevice::Update` now builds an array of `WiimoteDataBatchEntry` and calls `NetPlay_GetWiimoteData(std::span)` once, instead of once per Wii Remote. `SerializeDesiredState` writes one header byte with has_buttons, has_accel, has_camera and has_motion_plus flags plus a 4-bit extension ID, followed only by the fields that are present. a1563f2def 2022-09 “Netplay: Implement batching for Wiimotes.” · e137f7f878 2022-09 “WiimoteEmu: Rewrite serialization of DesiredWiimoteState to optimize for network bandwidth.” • Also seen in Flycast (flycast-167): `Boxart::getBoxart` used to start one `std::async` scrape per game. It now appends to `toFetch`. `fetchBoxart()` starts at most one `fetching` future, which takes `std::min(toFetch.size(), (size_t)10)` entries, calls `scraper->scrape(boxart)` on the vector, and reinserts the entries if an exception is thrown. The TheGamesDB client also gained a blackout period after errors. 5f5a74803 2022-06 “boxart: batch uid searches. Bios image. More arcade rom name changes”

### dolphin-271 — Specialize code on a guessed constant, guarded by a cheap runtime check
Lens: jit-interpreters · Source: Dolphin
Principle: If a value is almost always the same but can't be proven constant, generate code that assumes it is, and put one compare at entry that falls back to generic code on a mismatch. Every operation downstream gets the constant-folded fast path, and the guard costs one compare and branch.
Why it is faster: —
Applies elsewhere: JS engines' type-feedback specialization with deopt guards; PGO-based devirtualization with a type check; branch-on-common-case in hand-tuned parsers.
Watch out: —
Seen in Dolphin: `JitArm64::IntializeSpeculativeConstants()` checks each block input GPR whose compile-time value looks like a gather-pipe or MMIO address (`IsOptimizableGatherPipeWrite`, `0xCC000000`). For each one it emits `CMP` against that value, branching to far code that recompiles without speculation, and then calls `gpr.SetImmediate(i, value)`. Addresses that fail are recorded in `noSpeculativeConstantsAddresses`. This is a port of an existing Jit64 feature. dbf5dca11c 2021-08 “JitArm64: FIFO optimization improvements”

### dolphin-272 — Persist the driver's compiled pipeline blobs between runs
Lens: gpu-rendering · Source: Dolphin
Principle: Shader source caches still leave the driver to compile and link every pipeline at startup or on first use, which causes stutter. Where the API can return its compiled binary, store that blob next to your own cache key and hand it back on the next run. The driver can then skip the compile.
Why it is faster: —
Applies elsewhere: `VkPipelineCache` serialization; `glGetProgramBinary`; D3D12 `CachedPSO`; V8 code cache for scripts.
Watch out: —
Seen in Dolphin: Backends gained support for returning "cache data" from `AbstractPipeline`. `ShaderCache` now serializes `SerializedGXPipelineUid`/`SerializedGXUberPipelineUid` keys with the pipeline data in `LoadCaches()` and passes the data back when creating pipelines, alongside the existing shader-source caches. 00b83b4196 2019-04 “ShaderCache: Utilize pipeline cache data where possible”

### dolphin-273 — Remove dimensions from the specialization key to cut variant count
Lens: gpu-rendering · Source: Dolphin
Principle: Each field in a compiled-variant cache key multiplies the number of variants to compile and keep. If a field can be handled at runtime inside the variant, or a code path is unused for a given input, drop it from the key or normalize it to one value. You compile fewer, more widely shared variants and stutter less.
Why it is faster: —
Applies elsewhere: Unity/Unreal shader keyword stripping; JIT inline caches going megamorphic only when needed; template bloat reduction by hoisting type-independent code.
Watch out: —
Seen in Dolphin: With a dynamic vertex loader in the ubershader, `ApplyDriverBugs(GXUberPipelineUid)` sets `out.vertex_format = nullptr`, so vertex layout no longer splits the ubershader pipelines. With framebuffer fetch available, ubershaders always blend in the shader: `blending_state.hex` is zeroed except for the write masks, and `no_dual_src` is forced. Texture-decode compute shaders for non-palette formats now share one key, using `palette_format.value_or(IA8)` together with `#if defined(HAS_PALETTE)`. 4c629c2bee 2022-06 “VideoCommon: Add dynamic vertex loader to ubershaders” · 991024173e 2022-06 “VideoCommon: Always use fbfetch in ubershaders if available” · 23c1721fbd 2022-06 “Reduce duplicates of non-palette-requiring texture decode shaders”

### dolphin-274 — Overlap reading the next chunk with processing the current one
Lens: concurrency · Source: Dolphin
Principle: If several independent digests (or other passes) run serially over one data stream, the stream is limited by the slowest pass plus the read time. Launch each pass as an async task on the current chunk, read the next chunk meanwhile, and wait for the tasks only before swapping buffers.
Why it is faster: —
Applies elsewhere: `sha256sum`/`md5sum` in parallel via `tee`; double-buffered DMA in embedded systems; backup tools hashing and compressing in a pipeline.
Watch out: —
Seen in Dolphin: `VolumeVerifier::Process` computed CRC32, MD5, SHA-1 and content/block checks inline after each read. Each now runs through `std::async(std::launch::async, …)` on `m_data`. `ReadChunkAndWaitForAsyncOperations` reads the next chunk into a fresh vector first, then calls `WaitForAsyncOperations()` and moves the vector into `m_data`. 34fb608dd6 2019-08 “VolumeVerifier: Multithreading”

### dolphin-275 — Build dispatch and size tables at compile time, and inline the lookups
Lens: lookup-tables · Source: Dolphin
Principle: A table built by an init function costs startup time, needs ordering care, and stops the compiler from treating its entries as constants. A `constexpr` table that replaces nested switches is fixed at build time and can live in read-only memory. Defining tiny accessors in the header lets callers inline the lookup.
Why it is faster: —
Applies elsewhere: `constexpr` CRC tables; generated opcode tables in interpreters; Rust `const` arrays; perfect-hash keyword tables from gperf.
Watch out: —
Seen in Dolphin: `Jit64_Tables.cpp` dropped `InitializeInstructionTables()`. Each table is now built by an immediately invoked `constexpr` lambda. `VertexLoader::CompileVertexTranslator` replaced a two-level switch on component format and color format with `VertexLoader_Color::GetFunction(type, format)` table lookups. `GetSize` for color, normal, position and texcoord moved into headers as `DOLPHIN_FORCE_INLINE` lookups into `EnumMap` tables. ee200d09eb 2019-05 “Jit64/Jit64_Tables: Construct tables at compile-time” · bc7da726a5 2021-03 “Convert VertexLoader_Color to a table” · fdcd2b7d00 2022-09 “VertexLoader: Inline GetSize”

### dolphin-276 — Shrink hot structs: order fields by size and use the narrowest type
Lens: memory-layout · Source: Dolphin · Also: PostgreSQL
Principle: Padding between mixed-size fields and over-wide integers make structs bigger, so fewer fit per cache line and return values may no longer fit in registers. Put the widest fields first, and store small values (here, 8-way bitmasks) in `u8` rather than `u32`.
Why it is faster: —
Applies elsewhere: Go/Rust struct field reordering for alignment; packing ECS components; `-Wpadded` audits in embedded firmware.
Watch out: —
Seen in Dolphin: `TranslateAddressResult` moved `u32 address` ahead of the `u8` enum and the `bool`, which reduces its padding. In `PPCCache::Cache`, `plru`, `valid` and `modified` changed from `std::array<u32, CACHE_SETS>` to `std::array<u8, CACHE_SETS>`. Each holds an 8-way bitmask. 3296d2fc1f 2021-08 “PowerPC: Reorder members of TranslateAddressResult to reduce struct size.” · 825bacde43 2023-01 “PPCCache: Change u32 to u8 for plru, valid, modified” • Also seen in PostgreSQL (postgresql-066): dcbbdb1b3ee 2001-03-30 On ARM, the 6-byte `ItemPointerData` was laid out as 8 bytes. Forcing 6 "avoids wasting four bytes of pad space in every tuple header". • d5db3abfb61 2006-01-18 Stats file entries dropped struct padding when written. • d4b538ea367 2015-05-21 `ItemPointerData` uses `packed` plus `aligned(2)`, with a static assertion: no padding, but still 2-byte aligned access. 44ed65a5459 2015-08-23 used short ints instead of floats in bipartite matching: "smaller, faster". • 28d750c0cd5 2018-10-16 Reordered `FmgrBuiltin` from 32 to 24 bytes (25% smaller); it is one of the largest constant tables in a backend. • f5024d8d7b0 2021-05-23 Moved `attcompression` into the char-width group of `pg_attribute`, saving 4 bytes per catalog row and per TupleDesc entry. 90189eefc1e 2023-03-28 narrowed three int32 columns to int16 for another 4 bytes. 036297cf1b8 2023-09-25 shrank `ParsedWord` from 40 to 24 bytes. • 908a968612f 2024-09-05 Repacked WindowAgg's bool fields into existing padding so a new flag did not grow the struct and regress another workload. • 5865150b6d5 2025-08-27 Replaced enum bitfields in the AIO handle with `uint8` fields because "gcc and clang generate rather terrible code for the bitfield access".

### dolphin-277 — Delegate checksums to a SIMD-tuned library
Lens: simd · Source: Dolphin
Principle: A textbook byte-at-a-time checksum loop runs far slower than implementations that use SIMD or CRC instructions. If a dependency already ships a vectorized version, call it and delete the hand-rolled loop.
Why it is faster: —
Applies elsewhere: zlib-ng or ISA-L instead of stock zlib; xxHash/BLAKE3 in place of custom hashes; the JDK's intrinsic `CRC32C`.
Watch out: —
Seen in Dolphin: In `Common/Hash.cpp`, `HashAdler32` was a Wikipedia-style loop over `a += *data++; b += a`, and an unused Fletcher implementation sat beside it. Adler-32 now calls zlib-ng. 17c554c165 2022-07 “Common/Hash: use zlib-ng for adler32. small cleanups.”

### dolphin-278 — Clear coverage bits for fully handled ranges so later calls return early
Lens: laziness · Source: Dolphin
Principle: A per-line "may contain something" bitmap lets invalidation return immediately for lines that are already clean. A slow path that handles a big range must also clear the bits for every line it fully covered. Otherwise the next call over the same range repeats all the work.
Why it is faster: —
Applies elsewhere: card-marking GCs clearing cards after a scan; dirty-page bitmaps in live migration; tile invalidation masks in renderers.
Watch out: —
Seen in Dolphin: `JitBaseBlockCache::InvalidateICacheInternal` cleared `valid_block` only for the single-line case. For `length > 32` it now clears the bit for every 32-byte line fully inside the range. 34992f52c7 2021-09 “PowerPC/JitCache: Clear valid_block bits for long invalidations.”

### dolphin-279 — Compare strings in place instead of building converted copies
Lens: allocation · Source: Dolphin · Also: ares
Principle: Converting every candidate name to a new `std::string` (decoding, lowercasing) just to compare it allocates once per comparison. Walk the raw bytes with a `string_view`, compare as you go, and allocate only in the rare case that really needs a full conversion.
Why it is faster: —
Applies elsewhere: `strcasecmp` instead of `tolower(copy)`; Java `regionMatches(true, …)`; HTTP header matching without allocating.
Watch out: —
Seen in Dolphin: `FileInfoGCWii::NameCaseInsensitiveEquals(std::string_view)` walks the FST bytes directly and compares with `std::tolower(…, std::locale::classic())`. It converts to UTF-8 with `SHIFTJISToUTF8` only when both sides hold non-ASCII bytes at the same position. f25367a7a2 2019-05 “DiscIO: Avoid copies when comparing FileSystemGCWii file names” • Also seen in ares (ares-144): The Sufami Turbo signature check changed from `string(romSignature) == sufamiSignature` to `romSignature == sufamiSignature`. 3a46652b1 2025-04 “[SNES] Remove unnecessary additional string allocation”

### dolphin-280 — Use the lightweight UI component for append-only text, and bound it
Lens: other · Source: Dolphin
Principle: Rich-text widgets keep a full document model and re-layout costs grow as content grows. For logs, a plain-text widget plus a cap on the number of blocks keeps append and scroll costs bounded.
Why it is faster: —
Applies elsewhere: virtualized log viewers in web consoles; terminal emulators with a scrollback limit; `<textarea>` instead of `contenteditable` for large logs.
Watch out: —
Seen in Dolphin: `LogWidget` replaced `QTextEdit` with `QPlainTextEdit`, uses `appendHtml`, and sets a maximum of `MAX_LOG_LINES = 5000`. It still processes at most `MAX_LOG_LINES_TO_UPDATE = 200` queued lines per timer tick. eb7d42dfe8 2019-08 “LogWidget: Replace QTextEdit with QPlainTextEdit for better performance”

### dolphin-281 — Coalesce duplicate cross-thread notifications and shared work
Lens: concurrency · Source: Dolphin
Principle: If an event can fire many times before the receiving thread runs, queuing one task per event floods that thread with redundant work. Keep an atomic "already queued" flag so only one task is pending at a time. In the same spirit, run per-consumer generation work once for all consumers instead of once per consumer.
Why it is faster: —
Applies elsewhere: `requestAnimationFrame` debouncing; a "dirty and scheduled" bit in reactive frameworks; one shared fetch serving many subscribers.
Watch out: —
Seen in Dolphin: The Qt `Settings` config callback does `do_once.exchange(false)` on an atomic flag before calling `QueueOnObject`, and the queued lambda sets the flag back. Dynamic input textures used to be regenerated inside every controller's `LoadConfig`/`SaveConfig`. `InputConfig` now calls `DynamicInputTextureManager::GenerateTextures(file, controller_names)` once for all controllers, which reduces image loads and texture-cache invalidations. e6057c5f62 2021-03 “Qt: avoid queuing ConfigChanged() more than once” · 32d584a0f5 2021-02 “InputCommon: reduce number of image loads and texture cache invalidations”

### dolphin-282 — Run inline when you are already on the target thread
Lens: concurrency · Source: Dolphin
Principle: A thread-safe "schedule this on thread T" helper adds an event, a delay and a context switch even when the caller is already T. Check the current thread first and run the work directly in that case.
Why it is faster: —
Applies elsewhere: Android `runOnUiThread` and Qt's `DirectConnection` when sender and receiver share a thread; executors that run inline on the owning loop.
Watch out: —
Seen in Dolphin: `PowerPC::ScheduleInvalidateCacheThreadSafe` queued a CoreTiming event whenever the CPU was running. It now does that only if `!Core::IsCPUThread()`, and otherwise invalidates immediately. 3081a781fd 2022-12 “Core: when scheduling an invalidate cpu cache, no need to schedule if we're already on the cpu thread”

### dolphin-283 — Turn on the optimizer for generated GPU code in release builds
Lens: gpu-rendering · Source: Dolphin
Principle: Shader translators often default to unoptimized output with debug info kept, so validation and debugging tools are easier to use. When validation is off, enable the optimizer and strip debug info. The driver gets smaller, cleaner IR.
Why it is faster: —
Applies elsewhere: `glslangValidator -Os`/`spirv-opt`; `dxc -O3`; release-mode WASM via `wasm-opt`.
Watch out: —
Seen in Dolphin: `CompileShaderToSPV` in `Spirv.cpp` keeps `options.validate = true` when validation layers are on. Otherwise it now sets `options.disableOptimizer = false` and `options.stripDebugInfo = true` before calling `glslang::GlslangToSpv`. a8b5266347 2022-05 “VideoCommon: enable optimizer and remove debug details when converting GLSL to SPIR-V with validation layers turned off”

### dolphin-284 — Direct-mapped table over a sparse key space, backed by lazily committed memory
Lens: lookup-tables · Source: Dolphin
Principle: If keys are bounded integers, a table indexed by the whole key needs no tag compare, no masking and no collision handling: a hit is one load. Reserve the full table as virtual memory and let the OS commit pages only when they are written, so physical memory grows with the keys actually used rather than the size of the key space.
Why it is faster: —
Applies elsewhere: handle-to-object tables in a kernel or game engine, opcode or syscall dispatch tables, sparse ID-indexed arrays in an ECS.
Watch out: —
Seen in Dolphin: The JIT dispatcher used to hash the guest PC into a small array of block pointers, then compare the block's address and MSR bits. The table is now an mmap-ed segment indexed by `(msrBits << 26) | (address >> 2)`, and it stores `normalEntry` directly. The dispatcher does one load and an indirect jump, and on a miss it skips the C++ slow lookup, because that lookup could not succeed either. MSR bits occupy the high index bits because they change less often than the address, which keeps nearby entries on the same pages. 859da32a6c 2023-04 “Jit: Improve block lookup performance through a shm memory segment.” · fd9c970621 2023-07 “Extend the fast lookup mmap-ed segment further to avoid needing to check the msr bits.”

### dolphin-285 — Reserve big, commit on touch, and reset by remapping
Lens: memory-layout · Source: Dolphin
Principle: A zero-initialised region that is large but mostly untouched can be reserved as anonymous virtual memory: the OS hands out zero pages on first access. To clear it, map fresh pages over the range (or decommit it) instead of writing zeros, which also returns the physical memory.
Why it is faster: —
Applies elsewhere: arena allocators in browsers and databases, sparse hash tables or bitmaps over a 32-bit address space, and quickly resetting a large scratch buffer between requests.
Watch out: —
Seen in Dolphin: `Common::LazyMemoryRegion` wraps `mmap(..., MAP_PRIVATE | MAP_ANONYMOUS)`, with `MAP_NORESERVE` on Linux and manual block mapping on Windows. `Clear()` re-maps the range with `MAP_FIXED` instead of calling memset. The JIT's fast block map and its valid-block bitsets are built on it. 5bd7756064 2023-09 “Common/MemArena: Add LazyMemoryRegion to represent a zero-initialized memory region whose pages are only allocated on first access.”

### dolphin-286 — Replace hash sets of addresses with a flat bitmap and word-wide range ops
Lens: algorithms · Source: Dolphin
Principle: For integer keys from a dense, bounded domain, a bitmap indexed by key makes membership a shift and an AND, with no hashing, no node allocation and no pointer chasing. To set or clear a range, mask the first and last partial words and memset the whole words in between, instead of looping bit by bit.
Why it is faster: —
Applies elsewhere: page-dirty tracking, free-block maps in filesystems and allocators, "seen" sets in graph traversals over integer IDs.
Watch out: —
Seen in Dolphin: `LazyMemoryRegionBitSet` replaced three `std::unordered_set`s in `JitBase` that track instruction addresses, and it replaced the old valid-block bitset as well. `SetBits` ORs the edge words and handles the middle with `std::memset(m_pointer + start_index, 0xFF, ...)`. The commit reports about a 1% gain in an NBA Live 2005 menu. 9407047330 2026-08 “Jit: Use LazyMemoryRegion-backed bit sets”

### dolphin-287 — Use a fixed stack buffer when the size has a small known upper bound
Lens: allocation · Source: Dolphin
Principle: If a per-call temporary buffer can never exceed a small constant size, allocate it on the stack at that maximum and assert the bound. You avoid an allocator call and a free on every invocation, and you avoid touching fresh cache lines.
Why it is faster: —
Applies elsewhere: audio callbacks and real-time loops where malloc is forbidden, packet parsers with a fixed MTU, and small-string or small-vector optimisations.
Watch out: —
Seen in Dolphin: The DVD streaming-audio callback built a `std::vector<s16>` on every tick. It now uses `std::array<s16, MAX_POSSIBLE_SAMPLES * 2>` with the bound 168 (the 48 kHz case) and zero-fills the unused tail. Wiimote speaker decoding replaced `new s16[length * 2]` with a `std::array` sized from the 20-byte report. 66b6a60afe 2023-03 “HW/DVDInterface: Avoid heap allocation in DTK callback.” · 3bf2de369a 2025-02 “WiimoteEmu: Eliminate dynamic memory allocation in speaker data decoding.”

### dolphin-288 — After a local change, re-check locally instead of rescanning from the start
Lens: algorithms · Source: Dolphin
Principle: If swapping two adjacent elements can only create a new opportunity involving one of them, step back one position after each swap rather than starting another full pass. This turns a repeat-until-stable loop of O(n) passes into one sweep with bounded backtracking, the same idea as insertion sort.
Why it is faster: —
Applies elsewhere: peephole optimisers, bubble-style reordering of scheduled jobs, and incremental re-layout in text editors.
Watch out: —
Seen in Dolphin: `PPCAnalyst::ReorderInstructionsCore` bubbles compares toward branches and carry ops toward each other. It used to repeat whole passes until nothing changed. Now, after each swap it sets `go_backwards` and retests only the previous pair. 061ec365a8 2023-06 “PPCAnalyst: Reduce number of iterations in ReorderInstructionsCore”

### dolphin-289 — Don't re-check on entry what the caller already checked
Lens: branching-fast-paths · Source: Dolphin
Principle: If every path into a routine has already tested a condition, a second test at the callee's entry is pure overhead. Have the failing caller jump straight to the handler instead.
Why it is faster: —
Applies elsewhere: validation layers where both the API boundary and the internal function bounds-check, and coroutine schedulers that check a budget at both yield and resume.
Watch out: —
Seen in Dolphin: On JitArm64, a block exit checked the downcount, and when it was exhausted it jumped to the target's `checkedEntry`, which checked the downcount again before going to `do_timing`. Exits now branch straight to `do_timing`, every entry is `normalEntry`, and the per-block entry check was deleted. f78ba9ac55 2022-10 “JitArm64: Never check downcount on block entry”

### dolphin-290 — Put only genuinely rare paths in the cold section
Lens: jit-interpreters · Source: Dolphin
Principle: Splitting code into hot and cold areas keeps the instruction cache dense, but only if the cold side is really cold. A path taken often, such as a taken conditional branch, belongs inline; moving it out just adds a jump and an i-cache miss. Conversely, moving rare exit code out shortens the hot path.
Why it is faster: —
Applies elsewhere: `__builtin_expect`/`[[unlikely]]`, `__attribute__((cold))` error handlers, and profile-guided hot/cold function splitting.
Watch out: —
Seen in Dolphin: JitArm64 stopped emitting `SwitchToFarCode()` around conditional-branch exits, because taking a branch is common. Separately, the rarely executed "downcount exhausted, go to `do_timing`" tail of block links moved into farcode, which brought the nearcode link back to three instructions. 2f407fc854 2023-11 “JitArm64: Don't use farcode for conditional branch instructions” · d2c5d79614 2022-10 “JitArm64: Use farcode in WriteLinkBlock”

### dolphin-291 — Iterate over set bits, not over all bit positions
Lens: algorithms · Source: Dolphin
Principle: If a bitmask is usually sparse, walk it with count-trailing-zeros and clear-lowest-bit. The loop then runs once per member instead of once per possible member.
Why it is faster: —
Applies elsewhere: event-mask dispatch in poll loops, interrupt-pending registers in firmware, and chess bitboards.
Watch out: —
Seen in Dolphin: `Arm64GPRCache::FlushRegisters` went from `for (i = 0; i < GUEST_GPR_COUNT; ++i) if (regs[i])` to `for (int i : regs)`. `BitSet`'s iterator uses `std::countr_zero`. 6a9f565ac4 2023-11 “JitArm64: Optimize Arm64GPRCache::FlushRegisters/FlushCRRegisters”

### dolphin-292 — Merge adjacent narrow stores into one wider store
Lens: memory-layout · Source: Dolphin
Principle: If two values go to neighbouring memory locations, write them with one wider or paired store. That halves the store instructions and store-buffer entries. Use a `static_assert` to pin the field layout this depends on.
Why it is faster: —
Applies elsewhere: zeroing two adjacent struct flags with one 16-bit write, and serialising header fields with one 64-bit store.
Watch out: —
Seen in Dolphin: Register-cache flushes write two dirty guest registers with one ARM64 `STP`, and that now covers known immediates too, using `WZR` for zero. `mcrxr` clears `xer_ca` and `xer_so_ov` with one `MOV(16, ...)`/`STRH` behind `static_assert(PPCSTATE_OFF(xer_ca) + 1 == PPCSTATE_OFF(xer_so_ov))`. 383cbffdec 2024-10 “JitArm64: Allow ppcState STP optimization for imm” · c70dcf99dd 2023-08 “Jit: Some mcrxr optimizations”

### dolphin-293 — Emit nothing for identity operations
Lens: compiler-build · Source: Dolphin
Principle: A code generator should recognise operands that make an operation a no-op, such as AND with all ones or OR/XOR with zero, and emit nothing. These cases show up more often than expected once constants have been propagated.
Why it is faster: —
Applies elsewhere: SQL planners dropping `WHERE TRUE`, shader compilers removing multiply-by-one, and query builders skipping empty filters.
Watch out: —
Seen in Dolphin: The ARM64 emitter helpers `ANDI2R`/`ORRI2R`/`EORI2R` now return early for `imm == ~0` (AND) or `imm == 0` (ORR/EOR). That makes `rlwnmx` with a full mask free. 6303416201 2023-12 “JitArm64: Optimize ANDI2R and friends to no-ops when possible”

### dolphin-294 — Encode small constants in the instruction instead of materialising them
Lens: compiler-build · Source: Dolphin
Principle: Many ISAs can encode a small immediate, or a hard-wired zero register, directly in the instruction. Checking for these cases avoids loading the constant into a register first, and that saves both an instruction and register pressure.
Why it is faster: —
Applies elsewhere: what every native compiler backend does, and worth remembering when writing assembly or a bytecode-to-native translator.
Watch out: —
Seen in Dolphin: JitArm64 `cmp` now handles constants that fit 12 bits with `SUB(CR, CR, imm)`, so they are no longer moved into a register and sign-extended. Stores of a known zero use `WZR` and skip the byteswap, because a byteswapped zero is still zero. 4a29e0e4f4 2025-02 “JitArm64_Integer: cmp - Subtract 12-bit constant” · a307d9d9b8 2024-11 “JitArm64_LoadStore: Optimize zero stores in stX”

### dolphin-295 — Specialise generated code on a compact "mode word" that is part of the cache key
Lens: jit-interpreters · Source: Dolphin · Also: ares
Principle: When generated code depends on a few slow-changing runtime modes, pack them into one feature word. Read it at compile time to choose code paths, and include it in the cache key, so a mode change selects different code instead of the code testing modes at run time.
Why it is faster: —
Applies elsewhere: JS engines keying inline caches on hidden-class shape, and shader permutation keys in renderers.
Watch out: —
Seen in Dolphin: JIT code now consults `m_ppc_state.feature_flags & FEATURE_FLAG_MSR_DR` (and the IR and PERFMON flags) at compile time, in place of raw MSR/MMCR reads. The same flags become the high bits of the block-map index and of the BLR return-stack key. This makes explicit why ppcState is safe to read at JIT time. 94b31eb4f4 2023-08 “Jit: Replace MSR/MMCR access with feature_flags access” • Also seen in ares (ares-125): The CPU JIT replaced 256-byte "pools" (and whole-pool eviction) with 4 KiB `Section`s of per-word `Block*` chains. `block()` walks the chain for a matching `stateKey` and `vaddrPage`. The load emitter reads `emitStateKey` to choose 32-bit or 64-bit address checks during compilation. 6d6c5d5fa 2026-04 “n64: refactor CPU JIT to introduce 4K sections and state key” · 9e6be60b9 2026-04 “n64: implement all aligned load opcodes in JIT”

### dolphin-296 — Pick the codec for the direction that is latency-critical
Lens: io · Source: Dolphin
Principle: When data is compressed rarely but decompressed while a user waits, choose an algorithm with very fast decompression, even if its ratio is slightly worse.
Why it is faster: —
Applies elsewhere: LZ4 or zstd at low levels for caches, game asset packs and kernel images, and Brotli for static web assets that are compressed once and served many times.
Watch out: —
Seen in Dolphin: Savestates switched from LZO (with a static work buffer) to `lz4.h`, adding a versioned extended header to describe the compressed data. 1eaa9380dd 2023-10 “Savestates: Use LZ4 algorithm for faster decompression”

### dolphin-297 — Use a spinlock for tiny, almost never contended critical sections
Lens: concurrency · Source: Dolphin
Principle: A general-purpose mutex or shared mutex pays for features such as reader counts and kernel parking even when uncontended. If the lock is held for a few instructions and contention is rare, an atomic-exchange spinlock (optionally backed by `atomic::wait`) is several times cheaper.
Why it is faster: —
Applies elsewhere: per-object caches of computed values, reference-count side tables, and allocator free-list heads.
Watch out: —
Seen in Dolphin: `Common::AtomicMutex`/`SpinMutex` lock with `m_lock.exchange(true, std::memory_order_acquire)`. `Config::Info`'s cached value replaced `std::shared_mutex` with it, and the commit notes it is about 3x faster uncontended, which makes `Config::Get` cheaper. 46369ef1b1 2025-10 “Common: Add AtomicMutex and SpinMutex classes as faster alternatives to std::mutex.” · 999333c0a1 2025-10 “Config: Make ConfigInfo use Common::SpinLock...”

### dolphin-298 — Replace a mutex-guarded queue with a single-producer/single-consumer ring
Lens: concurrency · Source: Dolphin
Principle: If exactly one thread produces and one consumes, a lock-free SPSC queue built on atomic indices removes the mutex entirely. Pair it with a wait or notify primitive only for the empty case.
Why it is faster: —
Applies elsewhere: audio threads handing buffers to a mixer, logging back-ends, and network receive rings between IRQ and task.
Watch out: —
Seen in Dolphin: `WorkQueueThread` was reimplemented over `WaitableSPSCQueue` and gained a single-producer variant, `WorkQueueThreadSP`, which DVD, network and Vulkan swap threads adopted. `PerformanceTracker` dropped its mutex in favour of an `SPSCQueue` plus atomics. a149b9d62d 2025-03 “WorkQueueThread: Cleanups. Implement in terms of WaitableSPSCQueue. Add single producer WorkQueueThreadSP.” · 46e0952e97 2025-03 “PerformanceTracker: Use SPSCQueue and atomic to eliminate need for a mutex.”

### dolphin-299 — Track known constants separately from where values live
Lens: compiler-build · Source: Dolphin
Principle: Keep "value is a known constant" as its own fact, apart from "value is in a host register" and "value is in memory". Then putting a constant in a register no longer forgets the constant, and an instruction whose result is fully known can be evaluated at compile time with no code emitted.
Why it is faster: —
Applies elsewhere: SSA-based compilers (sparse conditional constant propagation), and partial evaluators in template engines and query compilers.
Watch out: —
Seen in Dolphin: A shared `JitCommon::ConstantPropagation` class now runs `EvaluateInstruction(op.inst)` before codegen. If `instruction_fully_executed` is set, nothing is emitted. The Jit64 register cache replaced its five-state `LocationType` enum with independent "in ppcState" and "in host register" booleans, so a value can be both a known immediate and held in a register. f9601dc38c 2023-08 “Jit: Extract immediate handling to separate ConstantPropagation class” · 817bb9d94c 2023-08 “Jit64: Don't store immediate values in register cache”

### dolphin-300 — Coalesce adjacent OS mappings into one call
Lens: platform · Source: Dolphin
Principle: Each mmap or MapViewOfFile call has a fixed kernel cost, and that cost is large on some OSes. When consecutive logical pages map to consecutive physical offsets, extend a single mapping instead of creating one per page.
Why it is faster: —
Applies elsewhere: merging adjacent I/O requests (scatter/gather), batching syscalls, and coalescing GPU buffer uploads.
Watch out: —
Seen in Dolphin: `UpdateLogicalMemory` used to create one 128 KiB view per BAT entry. It now keeps extending `logical_size` while the next entry's translated address is contiguous. The commit reports Windows boot times about 200 ms faster on GameCube and 60 ms faster on Wii, and faster savestate loads. ac84ea17a6 2025-06 “Memmap: Optimize UpdateLogicalMemory by merging mappings”

### dolphin-301 — One lookup, not two: use the result of find or insert
Lens: algorithms · Source: Dolphin
Principle: The patterns `contains()` then `at()`, or `contains()` then `insert()`, hash or walk the tree twice. Dereference the iterator that `find` returns, or call `insert`/`try_emplace` and check its `.second`.
Why it is faster: —
Applies elsewhere: `dict.setdefault` or `get` in Python, `Map.get` then a null check in JS, and Rust's `entry()` API.
Watch out: —
Seen in Dolphin: Code across the tree was rewritten, for example `if (const bool inserted = m_symset.insert(name_str).second; !inserted)` and `while (!m_ios_ids.emplace(new_id, id).second)`. f2392e4048 2025-07 “Avoid map/set double lookups”

### dolphin-302 — Iterate the small collection and probe the large one
Lens: algorithms · Source: Dolphin
Principle: To remove N keys from a container of M entries, where N is much smaller than M, look up each of the N keys in the large container. That costs O(N log M), compared with O(M log N) for scanning the large container and testing each entry against the small set.
Why it is faster: —
Applies elsewhere: hash-join build/probe side selection, removing a handful of IDs from a large cache, and set intersection by probing the larger set.
Watch out: —
Seen in Dolphin: `RemoveHostPageTableMappings` ran `std::erase_if` over every mapped entry and checked `mappings.contains()`. It now loops over `mappings` and does a `find`/`erase` per address. The commit reports roughly 3% in Rogue Squadron 3. a3b37c041a 2026-03 “Core: Optimize RemoveHostPageTableMappings”

### dolphin-303 — Store contiguous runs as intervals, not individual elements
Lens: algorithms · Source: Dolphin
Principle: If a set mostly consists of contiguous runs, store it as a sorted map of [from, to) intervals. Insertions merge with neighbours, and an overlap query is one `lower_bound` plus a step back.
Why it is faster: —
Applies elsewhere: free-space maps in filesystems, IP range blocklists, and text editor selection and dirty-range tracking.
Watch out: —
Seen in Dolphin: The JIT block cache replaced a per-address set of physical addresses with `Common::RangeSet` and added `overlaps(from, to)`. The commit reports `ErasePhysicalRange` about 50% faster, `PPCAnalyzer::Analyze` about 40% faster, and Rogue Squadron 2's cockpit switch 20–30% faster. c1a26808ce 2026-02 “Jit: Use RangeSet for physical_addresses”

### dolphin-304 — Null object instead of null checks
Lens: branching-fast-paths · Source: Dolphin
Principle: If a pointer may be "nothing", install a do-nothing implementation instead of null. Every call site loses its branch, and the invariant becomes "always valid".
Why it is faster: —
Applies elsewhere: no-op loggers and metrics sinks, and default strategy objects in UI frameworks.
Watch out: —
Seen in Dolphin: `HSPManager` now always holds an `IHSPDevice`. `Read` became just `return m_device->Read(address);`, and `AddDevice`/`RemoveDevice` became `SetDevice`. 187fcfa7dc 2026-03 “HW/HSP: Ensure an IHSPDevice object always exists to remove null checks as a micro optimization.”

### dolphin-305 — Shrink small value types to 16 bytes so they travel in registers
Lens: memory-layout · Source: Dolphin
Principle: On common 64-bit ABIs, a trivially copyable struct of 16 bytes or less is passed and returned in two registers, while a larger one goes through memory. Order fields from largest to smallest to remove padding, and encode less eagerly, until the struct fits.
Why it is faster: —
Applies elsewhere: `Result`/`Option` types, span or slice structs, and small geometry types passed by value in hot math code.
Watch out: —
Seen in Dolphin: The x64 emitter's `OpArg` moved `u64 offset` first and `u8 scale` last, which brings it to 16 bytes. The ARM64 `FixupBranch` now stores a pre-encoded `instruction` word and patches only the distance in `SetJumpTarget`, which removed the type, condition and register fields. 283218bc29 2026-01 “x64Emitter: Reduce padding in OpArg” · 819553ea4a 2026-08 “Arm64Emitter: Make FixupBranch smaller”

### dolphin-306 — Bit-pack cache keys
Lens: memory-layout · Source: Dolphin
Principle: Keys that get hashed, compared and stored by the thousand should hold only information that is needed and cannot be derived, packed into the smallest bitfields. That cuts hashing, compare and memory cost per lookup.
Why it is faster: —
Applies elsewhere: pipeline-state keys in renderers, flow keys in packet classifiers, and compact tuple keys in database indexes.
Watch out: —
Seen in Dolphin: `vertex_shader_uid_data` went from 41 bytes to 27 plus padding. UV-usage bits were dropped because they are derivable from `texcoord_elem_count`, unused bits were removed, per-texgen info was repacked, and mutually exclusive fields were overlaid in a union. `GX_PIPELINE_UID_VERSION` was bumped. e5eabd08f5 2026-08 “Optimize vertex_shader_uid_data down to 28 bytes”

### dolphin-307 — Process several packed fields with one masked operation (SWAR)
Lens: simd · Source: Dolphin
Principle: When fields that need the same transform can be lined up in one register, a single AND with a combined mask, plus a shift-or, handles all of them at once. Work out which fields are already in position before reordering anything.
Why it is faster: —
Applies elsewhere: pixel format conversion (RGB565 to RGBA8888) in image libraries and on embedded display drivers, and popcount or byte-search tricks in string libraries.
Watch out: —
Seen in Dolphin: The ARM64 vertex loader's RGB565 path used to byteswap first and then extract and expand each channel separately. It now uses `BFI` to line up red and blue, isolates both with one `AND(..., LogicalImm(0xF800F8))`, and expands them from 5 to 8 bits together. Red is already in position before the swap. RGBA6666 got a similar rewrite. 9c60ba938c 2026-10 “VertexLoaderARM64: Optimize RGB565”

### dolphin-308 — Collapse a strided loop to one bulk call when the stride equals the row size
Lens: simd · Source: Dolphin
Principle: A per-row fill or copy over a 2-D buffer is a single contiguous operation when the row pitch equals the row width, so detect that case and issue one memset or memcpy. For patterned fills, write code that compilers reliably auto-vectorise, such as constant-size memcpy from a static pattern, rather than intrinsics for a single architecture.
Why it is faster: —
Applies elsewhere: image buffer clears, framebuffer blits, and matrix row copies in numeric code.
Watch out: —
Seen in Dolphin: Clearing a skipped EFB copy now does `std::memset(dst, clear_byte, bytes_per_row * num_blocks_y)` when `bytes_per_row == stride`. The XFB clear dropped x86-only SSE intrinsics and memcpy's a `static constexpr` 32-byte pattern, so ARM gets vectorised code too. d74e757bad 2026-09 “TextureCacheBase: Optimize EFB and XFB uninitialization”

### dolphin-309 — Narrow an over-broad invalidation heuristic
Lens: laziness · Source: Dolphin
Principle: Defensive cache invalidation on loosely related events can cost more than it saves, causing stutter from rebuilding the cache. Keep only the triggers that are known to matter.
Why it is faster: —
Applies elsewhere: CDN purges on every deploy compared with targeted keys, and ORMs that clear a whole identity map on any write.
Watch out: —
Seen in Dolphin: Data-cache instructions `dcbst` and `dcbi` no longer invalidate JIT blocks; only `dcbf` does, because one game family depends on it. This fixes heavy stutter in NBA Live 2005/06 and Harry Potter: Prisoner of Azkaban. ffc287903e 2026-10 “Jit: Invalidate instruction block cache only with dcbf”

### dolphin-310 — Zero memory with the widest store the CPU has
Lens: simd · Source: Dolphin
Principle: Clearing a fixed-size aligned block, such as a cache line, takes fewer instructions with 256-bit stores than with two 128-bit ones. Clear the upper vector state afterwards so legacy SSE code doesn't pay a transition penalty.
Why it is faster: —
Applies elsewhere: what `memset` and compilers already do for fixed sizes, and worth copying in hand-written allocators and kernels.
Watch out: —
Seen in Dolphin: The Jit64 `dcbz` fast path writes the 32-byte line with one `VMOVAPS` on a YMM register when AVX is available, following LLVM's output. `VMOVAPS`/`VZEROUPPER` emitters were added for this. d1ba849876 2025-09 “Jit64: dcbz, use VMOVAPS with YMM registers on AVX CPUs”

### dolphin-311 — One container entry per owner, not one per item
Lens: allocation · Source: Dolphin
Principle: If many small records are created and destroyed together (per block, per request), store them as a vector under one map key. Insertion becomes a vector push instead of a tree-node allocation, and deleting the owner removes all its records at once.
Why it is faster: —
Applies elsewhere: per-request log buffers, per-entity component lists, and source maps grouped per function.
Watch out: —
Seen in Dolphin: The x86 JIT's `m_back_patch_info` was keyed per fastmem instruction. It is now keyed per block, holding a `std::vector<TrampolineInfo>`; a fault does `lower_bound` to find the block, then a linear `find_if`. Profiling had shown the map insertion as hot during compilation, and it disappeared from the profile. 4c9019d6fb 2026-09 “Jit: Refactor m_back_patch_info/m_fault_to_handler's maps to store a vector per block”

### dolphin-312 — Pass the cheap handle, build the expensive object only on the slow path
Lens: laziness · Source: Dolphin
Principle: If a function's fast path returns early without using a derived descriptor, pass the raw inputs and construct the descriptor only after the early return.
Why it is faster: —
Applies elsewhere: building error messages or request objects only when needed, and lazily parsing headers in HTTP servers.
Watch out: —
Seen in Dolphin: `TextureCacheBase::Load` now takes a `u32 stage`. The bound-texture fast path checks `m_bound_textures[stage]`, and `TextureInfo::FromStage(stage)` runs only after that path misses. The commit reports roughly 4% in Rogue Squadron 2's Hoth level. fb07406f10 2026-01 “VideoCommon: Defer creating TextureInfo”

### dolphin-313 — Test "is anyone listening?" before formatting output
Lens: laziness · Source: Dolphin
Principle: String formatting and allocation for a diagnostic sink cost the same whether or not the sink is enabled. Check the enable flag first.
Why it is faster: —
Applies elsewhere: `if (logger.isDebugEnabled())` guards, and lazily evaluated log macros in C and Rust.
Watch out: —
Seen in Dolphin: `JitRegister::Register` (perf/VTune symbol export) formatted a name string for every finalised block. It now begins with `if (!IsEnabled()) return;`. c54dc9db60 2026-01 “JitRegister: Check IsEnabled before constructing string”

### dolphin-314 — Short-circuit a pipeline whose output is discarded
Lens: laziness · Source: Dolphin
Principle: If the final consumer is a null sink, skip the upstream processing entirely instead of computing results and throwing them away.
Why it is faster: —
Applies elsewhere: skipping rendering when a window is minimised, and skipping serialisation when no subscriber is attached.
Watch out: —
Seen in Dolphin: `NullSoundStream` sets the mixer's output rate to 0. `Mixer::PushSamples`, its streaming variants and `Mix` now return early when `!IsOutputSampleRateValid()`. ff0560574e 2025-07 “AudioCommon/Mixer: Skip sample processing when NullSoundStream is being used.”

### dolphin-315 — Process whole blocks straight from the input buffer
Lens: memory-layout · Source: Dolphin
Principle: In block-based hashing or cipher code, copy into the staging block only to complete a partial block. While the staging block is empty and the input has whole blocks left, process them in place.
Why it is faster: —
Applies elsewhere: every production SHA, AES-CTR and CRC implementation, and buffered readers that bypass the buffer for large reads.
Watch out: —
Seen in Dolphin: `SHA1::BlockContext::Update` fills a partial block first, then runs `while (len >= m_block.size()) { ProcessBlock(msg); ... }` on the caller's data, and copies only the tail into the block. bb77d61967 2025-05 “Crypto/SHA1: Re-add memcpy avoiding optimization to BlockContext::Update.”

### dolphin-316 — Reuse values already in registers instead of rematerialising or allocating temporaries
Lens: compiler-build · Source: Dolphin
Principle: A code generator should check whether a needed value is already in some register, and whether the destination can double as scratch. If so, reuse it rather than reloading the constant or grabbing a new temporary.
Why it is faster: —
Applies elsewhere: register coalescing in compiler backends, and hand-written inner loops that reuse the output register as an accumulator.
Watch out: —
Seen in Dolphin: JitArm64 `WriteExit`/`FakeLKExit` accept an `exit_address_after_return_reg` and push it directly instead of running `MOVI2R` again. `subfic` and several other instructions allocate a temp only when `d == a`; otherwise they compute in `RD`. 7daa19f40d 2023-08 “JitArm64: Avoid loading compilerPC multiple times if it's already in a register.” · 5d647251f7 2024-05 “JitArm64: subfic - Conditionally skip temp reg allocation”

### dolphin-317 — Allocate without initialising when you'll overwrite everything
Lens: allocation · Source: Dolphin
Principle: The value-initialising forms `vector::resize` and `new T[n]()` zero memory that the next step overwrites completely. Use an allocate-uninitialised buffer such as `make_unique_for_overwrite`, and write by index instead of `push_back`.
Why it is faster: —
Applies elsewhere: image decoders, network receive buffers, and `Vec::with_capacity` followed by `set_len` in Rust (unsafe but common).
Watch out: —
Seen in Dolphin: Texture data moved from `std::vector<u8>` to `Common::UniqueBuffer<u8>`, which deletes `resize` on purpose because it "often unnecessarily copies data about to be overwritten". `RGBAToRGB` writes by `buffer[buffer_index++]`. `LinearDiskCache` uses `std::make_unique_for_overwrite<V[]>`. 5a80105555 2025-05 “Common and VideoCommon: Change texture data from std::vector to Common::UniqueBuffer.” · f78fa2e9f1 2025-04 “LinearDiskCache: Use make_unique_for_overwrite.”

### dolphin-318 — Build off-lock, then swap under the lock
Lens: concurrency · Source: Dolphin
Principle: If a shared structure is rebuilt from scratch, build it in a local variable without holding the lock, then take the lock only to swap it in. Readers are blocked for the duration of a pointer swap rather than the whole build.
Why it is faster: —
Applies elsewhere: RCU-style config reloads, routing-table rebuilds, and double-buffered search indexes.
Watch out: —
Seen in Dolphin: `PPCSymbolDB::LoadMap` fills local `XFuncMap`/checksum maps through new pointer-taking helpers (`AddKnownSymbol(..., &functions, ...)`, static `Index(XFuncMap*)`), then swaps them into the members under `m_mutex`. 803e6b017b 2025-06 “PPCSymbolDB: Reduce lock contention in LoadMap/LoadMapOnBoot”

### dolphin-319 — Paired loads and early-issued loads in a hot dispatcher
Lens: jit-interpreters · Source: Dolphin
Principle: In a tight lookup sequence, fetch adjacent fields with one paired load, and issue every load before the compare that first needs it, so memory latency overlaps with other work. Fold shifts into the addressing mode and the bitfield extract instead of spending separate instructions on them.
Why it is faster: —
Applies elsewhere: interpreter dispatch loops, hash-table probe sequences, and any hand-scheduled assembly on in-order cores.
Watch out: —
Seen in Dolphin: The JitArm64 fallback dispatcher reads `msrBits` and `effectiveAddress` with one `LDP`, guarded by a `static_assert` that they are adjacent. It loads `normalEntry` into a separate register before the compares. It indexes with `UBFX(pc, 2, n)` plus an LSL-3 scaled `LDR`, replacing ORR, AND and LSL. c9347a2a19 2023-06 “JitArm64: Use LDP in slow dispatcher” · 06c7862160 2023-08 “JitArm64: Rearrange dispatcher instructions to improve scheduling” · 9e970bcb30 2023-08 “JitArm64: Optiming shifting and masking PC in slow dispatcher”

### dolphin-320 — Speculative devirtualisation of the dominant callee
Lens: branching-fast-paths · Source: Dolphin
Principle: When an indirect call almost always targets one or two functions, compare the pointer against them and call them directly in the fast path. The direct call can be inlined and predicted, and the generic indirect call remains as the fallback.
Why it is faster: —
Applies elsewhere: what JITs and PGO-driven C++ compilers do, and worth hand-applying in callback-driven hot loops such as virtual-DOM diffing or event dispatch.
Watch out: —
Seen in Dolphin: The Cached Interpreter's block runner now checks `if (callback == CallbackCast(Interpret<false>)) [[likely]]` and the `<true>` variant, calling `Interpret` directly with the inline operand payload. Other callbacks go through the pointer as before. 0cde7ae79b 2025-09 “Cached Interpreter: Speculative devirtualization”

### dolphin-321 — Give a frequently used rare-variant instruction the native path
Lens: jit-interpreters · Source: Dolphin
Principle: If an "exotic" opcode behaves identically to a common one under your current assumptions, route it to the fast implementation. Leaving it on the generic slow path is costly when real workloads hit it in loops.
Why it is faster: —
Applies elsewhere: bytecode VMs with superinstructions, and SQL engines mapping rarely seen function aliases onto optimised kernels.
Watch out: —
Seen in Dolphin: Without cache emulation, `dcbz_l` behaves like `dcbz`. Both JITs' tables now map opcode 1014 to `&Jit64::dcbz`/`&JitArm64::dcbz` instead of `FallBackToInterpreter`. The SDK's `LCEnable` runs it in a loop. 667c523755 2025-09 “Jit: Use dcbz() emitter for dcbz_l instead of interpreter fallback”

### dolphin-322 — Optimise cold code for size, hot code for latency
Lens: jit-interpreters · Source: Dolphin
Principle: For code that rarely runs, instruction fetch and i-cache misses outweigh pipeline latency. Choose the shortest sequence, even if it is a cycle slower, and free up registers in the process. Also lean on hardware behaviour that already matches the spec, so you can skip redundant checks.
Why it is faster: —
Applies elsewhere: `-Os` on cold functions, and compact error-handling stubs in firmware.
Watch out: —
Seen in Dolphin: JitArm64's NaN farcode used to build a quiet-bit constant and ORR it in. It now uses `FADD(VD, input, input)`, one instruction and no temp register. It also skips checking the last input, because AArch64 already propagates a single NaN operand quietly, just as PowerPC does. 4ecdb9e57e 2023-08 “JitArm64: Use one instruction for making NaNs quiet” · 8274dcbfe4 2023-08 “JitArm64: Skip checking last input for NaN for non-SIMD operations”

### dolphin-323 — Classify by decoded fields, not by comparing names
Lens: algorithms · Source: Dolphin
Principle: If a classification can be made from integer fields, test those fields directly instead of comparing strings such as mnemonics or type names, which costs a call and a byte loop per check.
Why it is faster: —
Applies elsewhere: dispatch on enum tags instead of `type.name == "..."`, and HTTP method parsing into an enum once.
Watch out: —
Seen in Dolphin: `PPCAnalyst` replaced `strncmp(op.opinfo->opname, "stfd", 4)` and similar calls with helpers such as `IsStfd(inst)`, which test `inst.OPCD`/`SUBOP10`. The `strncmp` calls had shown up in the profiler. 786d91da28 2026-09 “PPCAnalyst: Avoid using strncmp”

### dolphin-324 — Drain a shared queue into a private buffer in batches
Lens: concurrency · Source: Dolphin
Principle: When a consumer polls a lock-protected queue one item at a time, take the lock only when its private buffer runs dry, and swap the whole shared queue in at once. Replace "is the peer alive?" queries with an atomic flag.
Why it is faster: —
Applies elsewhere: job systems that steal or drain batches, and log consumers that swap double buffers.
Watch out: —
Seen in Dolphin: USB Gecko emulation keeps a CPU-thread-only `m_recv_buffer` and refills it with `m_recv_buffer.swap(recv_fifo)` under `transfer_lock` only when it is empty. Connection status is a `Common::Flag client_connected` instead of calling `getLocalPort()` on the socket. 3ce096543c 2026-09 “Optimize USB Gecko Emulation”

### dolphin-325 — Binary-heap event queue on a plain vector, with a countdown "slice" until the next event
Lens: algorithms · Source: Dolphin · Also: ares
Principle: Keep scheduled events in a vector managed as a min-heap: insertion and pop-min are O(log n), and the vector can still be serialised or edited arbitrarily. Run the main loop against a decrementing budget that ends at the earliest event, so the hot path tests one counter instead of consulting the queue.
Why it is faster: —
Applies elsewhere: timer wheels and heaps in event loops (libuv, Tokio), and discrete-event simulators.
Watch out: —
Seen in Dolphin: `CoreTimingManager` keeps `std::vector<Event> m_event_queue` and manages it with `std::ranges::push_heap/pop_heap(..., std::ranges::greater{})`. A `fifo_order` tie-breaker keeps ordering stable. The comment explains that `priority_queue` was avoided because events must be erased and serialised. JIT code only decrements `ppc_state.downcount`, and `Advance()` runs when it reaches zero. Scheduling an earlier event shrinks the slice in place. current code: `Source/Core/Core/CoreTiming.cpp`, `CoreTiming.h` • Also seen in ares (ares-001): The new speed-oriented S-CPU core uses a `nall::priority_queue<unsigned>` (a binary min-heap array). `CPU::add_clocks` calls `queue.tick(clocks)`, and DRAM refresh and HDMA are enqueued as `QueueEvent`s, e.g. `queue.enqueue(534, QueueEvent::DramRefresh)`. The changelog credits this core with a ~40% speedup. Super Scope/Justifier latching was later added on the same queue, which the changelog says cost nothing. 9000bb408 2010-08 “Update to 20100810-2 release.” · 70429285b 2010-08 “Updated to v067r23 release.” • Also seen in ares (ares-045): v120r03 added `nall/priority-queue.hpp`, with `step(clocks, callback)` popping while `ge(clock, heap[0].clock)`, where `ge` is `x - y < 0x7fffffff`. `n64::CPU::synchronize` uses it for PI and SI DMA completion, and RSP DMA later re-inserts itself one row at a time (`queue.insert(Queue::RSP_DMA, …)`). `remove()` was changed from `for(auto& entry : heap)` to `for(i < size)`. 8c7bad148 2021-05 “Update to ares v120r03 release.” · bcf9cab22 2022-05 “n64: rework RSP DMA double buffering” · a3a777ef9 2022-04 “nall: optimize priority_queue remove function”

### dolphin-326 — Let other threads post to a single-threaded structure through a side queue
Lens: concurrency · Source: Dolphin
Principle: Keep a hot data structure owned by one thread with no locks. Other threads push requests into a separate queue, and the owner merges them at a safe point.
Why it is faster: —
Applies elsewhere: UI-thread message queues, actor mailboxes, and game engines' main-thread command buffers.
Watch out: —
Seen in Dolphin: `ScheduleEvent` from the CPU thread pushes onto the heap directly. Other threads lock only `m_ts_write_lock` to push into `Common::SPSCQueue<Event> m_ts_queue`, and `MoveEvents()` drains that queue into the heap on the CPU thread. current code: `Source/Core/Core/CoreTiming.cpp`

### dolphin-327 — Optimistic unchecked access, patched on fault
Lens: jit-interpreters · Source: Dolphin
Principle: Emit the fast, unchecked memory access inline and pad it to a fixed size. If it ever faults, the signal handler looks up the faulting address and rewrites that site to call a slow-path trampoline, so the common case costs one host load with no bounds test.
Why it is faster: —
Applies elsewhere: WebAssembly runtimes using guard pages to elide bounds checks, and JVM implicit null checks via SIGSEGV.
Watch out: —
Seen in Dolphin: `EmuCodeBlock::SafeLoadToReg` emits `UnsafeLoadToReg` against the fastmem arena, NOP-pads it to `BACKPATCH_SIZE`, and records a `TrampolineInfo` (registers in use, access size, PC). The fault path rewrites that site to call a generated trampoline. When fastmem is off but the arena exists, it emits `CheckIfSafeAddress` plus an inline fast load, with the slow call in farcode. current code: `Source/Core/Core/PowerPC/Jit64Common/EmuCodeBlock.cpp`, `Jit64/JitBackpatch.cpp`

### dolphin-328 — Specialise stores to compile-time-known addresses
Lens: jit-interpreters · Source: Dolphin
Principle: When constant propagation proves the target address of a store, emit code for that specific destination, such as a direct RAM write or a bump-pointer append to a write-combining buffer, instead of the generic address-translation path.
Why it is faster: —
Applies elsewhere: compilers lowering MMIO register writes to direct stores, and kernel ring-buffer producers that write at a cached tail pointer.
Watch out: —
Seen in Dolphin: `WriteToConstAddress` checks `IsOptimizableGatherPipeWrite(address)`. If it matches, it loads `gather_pipe_ptr`, does a `SwapAndStore` (in place with MOVBE when available), bumps the pointer and tracks `fifoBytesSinceCheck`. RAM addresses go to `WriteToConstRamAddress`. current code: `Source/Core/Core/PowerPC/Jit64Common/EmuCodeBlock.cpp`

### dolphin-329 — Mirror guest calls with host call/return so the return predictor works
Lens: branching-fast-paths · Source: Dolphin
Principle: A JIT translating guest calls to plain jumps defeats the host's return-stack predictor. Push a tagged guest return address onto the host stack at call sites, and at returns compare it against the actual target, then use a native RET. Mismatches fall back to the dispatcher, and stack overflow is caught by a guard page.
Why it is faster: —
Applies elsewhere: binary translators and emulators of any ISA, and interpreters that implement guest calls with host recursion for the same reason.
Watch out: —
Seen in Dolphin: `Jit64::WriteExit` with LK pushes `u64(feature_flags) << 32 | after`. `WriteBLRExit` builds the same key from the target, does `CMP(64, R(RSCRATCH), MDisp(RSP, 8))`, jumps to `dispatcher_mispredicted_blr` on mismatch, and otherwise executes `RET()`. Faults in the stack guard region go to `HandleStackFault()`. current code: `Source/Core/Core/PowerPC/Jit64/Jit.cpp`

### dolphin-330 — Spill choice by a cheap cost score with bounded lookahead
Lens: compiler-build · Source: Dolphin
Principle: When a register must be freed, score each candidate: a dirty value costs a writeback, and a value needed soon costs a reload. Cap the lookahead so compile time stays linear on huge blocks.
Why it is faster: —
Applies elsewhere: linear-scan register allocators, and cache eviction that weighs dirty pages against clean ones.
Watch out: —
Seen in Dolphin: `RegCache::ScoreRegister` adds 2 if the value isn't in ppcState (it's dirty), and adds `1 + 2 * (5 - log2f(1 + regs_in_count))` if it is used again, looking at most 64 instructions ahead (`std::min(instructionsLeft, 64)`). The comments record that a bias of 2 was found empirically. current code: `Source/Core/Core/PowerPC/Jit64/RegCache/JitRegCache.cpp`

### dolphin-331 — Backward liveness pass to drop dead registers and flags
Lens: compiler-build · Source: Dolphin
Principle: Scan a block in reverse to learn which outputs are overwritten before they are read. Dead values never need to be written back, and flag computations whose results are unused can be skipped. Wherever control can leave the block, assume everything is live.
Why it is faster: —
Applies elsewhere: dead-store elimination in compilers, and lazy flag evaluation in CPU emulators of every kind.
Watch out: —
Seen in Dolphin: `PPCAnalyzer` walks `for (int i = num_instructions - 1; i >= 0; i--)` to compute `gprDiscardable`, `fprDiscardable`, `crDiscardable`, `wantsCA` and `wantsFPRF`, resetting them at any `may_exit_block` or gather-pipe check. The JIT uses `wantsCAInFlags` to keep the carry in the host carry flag between adjacent ops. current code: `Source/Core/Core/PowerPC/PPCAnalyst.cpp`

### dolphin-332 — Detect busy-wait loops and fast-forward to the next event
Lens: laziness · Source: Dolphin
Principle: A loop that only polls memory and branches back can't make progress until something external changes. Recognise it statically, and when it is reached, skip time forward to the next scheduled event instead of spinning through it.
Why it is faster: —
Applies elsewhere: hypervisors trapping PAUSE or HLT, and tickless kernels sleeping until the next timer.
Watch out: —
Seen in Dolphin: `PPCAnalyzer::IsBusyWaitLoop` checks branches back to the block start that contain only loads, compares and pure calls, and marks them `BranchAction::IdleLoop`. `Jit64::bx`/`bcx` then emit `WriteIdleExit`, and CoreTiming's `Idle()` zeros the downcount and accounts the idled cycles. current code: `Source/Core/Core/PowerPC/PPCAnalyst.cpp`, `Jit64/Jit_Branch.cpp`, `Core/CoreTiming.cpp`

### dolphin-333 — Specialise on known config at compile time, otherwise jump through a routine table
Lens: jit-interpreters · Source: Dolphin
Principle: When a conversion is parameterised by a register that is usually constant, emit an inline specialised conversion when the value is known at compile time. Otherwise compute an index from the runtime value and call through a table of pre-generated routines, never through a branchy generic routine.
Why it is faster: —
Applies elsewhere: pixel-format converters picked once per surface, and codec dispatch tables indexed by sample format.
Watch out: —
Seen in Dolphin: `psq_lXX` checks `js.constantGqrValid[i]` and calls `GenQuantizedLoad(single, type, scale)` inline. Otherwise it masks the GQR with `0x3F07`, ORs it into the base of a 256-byte-aligned `paired_load_quantized` table, and does `CALLptr`. Scale factors come from `alignas(16)` tables such as `m_dequantizeTableS`, and a per-region `ConstantPool` keeps constants within RIP-relative reach. current code: `Source/Core/Core/PowerPC/Jit64/Jit_LoadStorePaired.cpp`, `Jit64Common/Jit64AsmCommon.cpp`, `Jit64Common/ConstantPool.h`

### dolphin-334 — Describe handlers as data so a code generator can inline the simple ones
Lens: jit-interpreters · Source: Dolphin
Principle: Instead of opaque callbacks, describe each device register as "constant", "direct pointer plus mask" or "complex function". A visitor can then emit an immediate or a plain load for the first two kinds, and a call only for the third.
Why it is faster: —
Applies elsewhere: ORM field mappings that compile to direct accessors, and route tables that separate static responses from handlers.
Watch out: —
Seen in Dolphin: The MMIO mapping indexes per-width handler arrays by `UniqueID(addr) = ((addr >> 24) & 1) << 16 | (addr & 0xFFFF)`. For known addresses, Jit64's `MMIOReadCodeGenerator` implements `VisitConstant` (load an immediate), `VisitDirect` (`LoadAddrMaskToReg`) and `VisitComplex` (`CallLambda`). current code: `Source/Core/Core/HW/MMIO.h`, `Source/Core/Core/PowerPC/Jit64Common/EmuCodeBlock.cpp`

### dolphin-335 — Self-replacing function pointer for runtime CPU dispatch, plus a sampled hash
Lens: simd · Source: Dolphin
Principle: Initialise a function pointer to a resolver that detects CPU features, overwrites the pointer with the best implementation and forwards the call. Later calls are a single indirect call. For change detection on large buffers, hash a strided sample with hardware CRC instead of every byte.
Why it is faster: —
Applies elsewhere: glibc IFUNC and `memcpy` dispatch, and sampled ETags or perceptual hashes for big assets.
Watch out: —
Seen in Dolphin: `s_texture_hash_func` starts as `SetHash64Function`, which picks `GetHash64_SSE42_CRC32`, `GetHash64_ARMv8_CRC32` or `GetMurmurHash3`. The CRC variants run four independent `_mm_crc32_u64`/`__crc32d` chains, with a step of `len/8/samples`. The texture cache passes `textureCacheSafetyColorSampleSize` as `samples`. current code: `Source/Core/Common/Hash.cpp`, `Source/Core/VideoCommon/TextureCacheBase.cpp`

### dolphin-336 — Compile one kernel source several times, once per ISA level
Lens: simd · Source: Dolphin
Principle: Write the SIMD kernel once against macros, include it repeatedly with different target macros, and pick the best variant at runtime through a function-pointer table. Skip variants below the compiler's baseline.
Why it is faster: —
Applies elsewhere: Highway, xsimd and simdjson multi-target builds, and image codecs shipping SSE2, AVX2 and NEON variants.
Watch out: —
Seen in Dolphin: `CPUCull.cpp` includes `CPUCullImpl.h` four times with `USE_SSE3`, `USE_SSE41`, `USE_AVX` and `USE_FMA` defined in turn. `GetTransformFunction<PositionHas3Elems, PerVertexPosMtx>` chooses by `MIN_SSE` and `cpu_info`, and fills `m_transform_table[2][2]` and a cull-function table indexed by primitive and cull mode. current code: `Source/Core/VideoCommon/CPUCull.cpp`

### dolphin-337 — JIT-compile format converters, using byte-shuffle lookup tables
Lens: jit-interpreters · Source: Dolphin
Principle: When input layouts come from a large combinatorial format space that changes rarely, generate a straight-line converter per layout. Inside it, use a byte-shuffle instruction driven by a precomputed mask table, followed by a convert and a scale multiply, so a whole attribute is reordered and converted in a few instructions.
Why it is faster: —
Applies elsewhere: SQL engines JIT-compiling row decoders, and network stacks generating parsers per protocol version.
Watch out: —
Seen in Dolphin: `VertexLoaderX64` emits `PSHUFB(coords, MPIC(&shuffle_lut[format][count_in - 1]))`, then `CVTDQ2PS` and `MULPS` with a `scale_factors[32]` table. Loaders are cached in `s_vertex_loader_map` by `VertexLoaderUID`, and a `BitSet8 attr_dirty` per VAT group means a lookup happens only after the format registers change. current code: `Source/Core/VideoCommon/VertexLoaderX64.cpp`, `VertexLoaderManager.cpp`

### dolphin-338 — Never block on a compile: background-compile specialised variants, fall back to a generic one
Lens: gpu-rendering · Source: Dolphin
Principle: Specialised programs are fast but expensive to build. Look them up asynchronously, queue a background compile on a miss, and draw with a precompiled generic interpreter-style variant until the specialised one is ready. Strip irrelevant bits from the key first so equivalent states share a variant.
Why it is faster: —
Applies elsewhere: tiered JITs (an interpreter or baseline tier while the optimising tier compiles), and game engines' async PSO creation with fallback materials.
Watch out: —
Seen in Dolphin: `ShaderCache::GetPipelineForUidAsync` returns the pipeline only if it is present and not pending. Otherwise it calls `QueuePipelineCompile(uid, COMPILE_PRIORITY_ONDEMAND_PIPELINE)` and returns empty, and the caller uses `GetUberPipelineForUid`. Worker thread counts switch between precompile and runtime settings. `ClearUnusedPixelShaderUidBits` canonicalises keys. current code: `Source/Core/VideoCommon/ShaderCache.cpp`

### dolphin-339 — Template the decoder on its callback so handlers inline
Lens: compiler-build · Source: Dolphin
Principle: A command-stream parser that calls virtual handlers pays an indirect call per command. Make the parser a template over the handler type and mark handlers force-inline, so each instantiation becomes one fused loop, and keep inheritance only as an optional check.
Why it is faster: —
Applies elsewhere: SAX-style JSON parsers templated on a handler (RapidJSON), and visitor templates in AST walkers.
Watch out: —
Seen in Dolphin: `OpcodeDecoder::Run`/`RunCommand` are `DOLPHIN_FORCE_INLINE` templates over the callback, and `OPCODE_CALLBACK(sig)` expands to `DOLPHIN_FORCE_INLINE sig`. The comment says this is so "the compiler generates separate versions for each callback". Preprocess and main variants are separate instantiations. current code: `Source/Core/VideoCommon/OpcodeDecoding.h`
