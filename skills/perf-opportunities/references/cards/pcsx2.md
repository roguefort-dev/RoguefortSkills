# PCSX2 — technique cards (298)

Each card is exactly 7 lines. Fetch one with: grep -A 6 "^### pcsx2-NNN " pcsx2.md

### pcsx2-001 — Generate a specialized inner loop per configuration and cache it by key
Lens: jit-interpreters · Source: PCSX2
Principle: When a hot loop's behavior depends on a dozen mode flags, a generic version tests every flag on every pixel or item. Pack the flags into one integer key, generate machine code with only the selected paths for that key at runtime, and keep the result in a map. All the per-item flag tests disappear, and each configuration is compiled only once.
Why it is faster: —
Applies elsewhere: query engines compiling a filter expression to native code once per query shape, regex engines JIT-compiling patterns, shader permutation caches keyed by material flags.
Watch out: —
Seen in PCSX2: The GSdx software rasterizer replaced its template/function-pointer scanline variants with an xbyak code generator. `GSDrawScanline` looks up `m_env.sel` (a bitfield of ztst, fge, tfx, iip, alpha modes and so on) in a map of `GSDrawScanlineCodeGenerator` objects and calls the emitted function through `m_dsf`. A follow-up moved more of the pipeline into generated code and deleted about 2,500 lines of hand-written variants. 9ee9d817c 2009-02 “GSdx software renderer speed-up, using xbyak to JIT compile a few things” · e131e22ea 2009-02 “GSdx: more JIT code and a little clean-up”

### pcsx2-002 — Gate values with an AND mask instead of multiplying by 0 or 1
Lens: branching-fast-paths · Source: PCSX2
Principle: If a per-channel on/off switch is stored as 0 or 1 and applied by multiplication, every sample pays for an integer multiply. Store the switch as an all-zeros or all-ones mask (0 or -1) and apply it with a bitwise AND, which is cheaper, has no branch, and maps directly onto SIMD. Put the mask fields together, and aligned, so that a vector AND can later handle them all at once.
Why it is faster: —
Applies elsewhere: audio mixers muting buses, branchless select in image filters (`(a & m) | (b & ~m)`), SIMD predicate masks in database scans.
Watch out: —
Seen in PCSX2: In the SPU2 mixer, voice and core routing gates such as `DryL`/`WetL` went from `s8` 0/1 values to `s32` masks, and `SDL += VValL * vc.DryL` became `SDL += VValL & vc.DryL`. SPU2-X then regrouped the gates into `V_CoreGates DryGate/WetGate` structs and moved them to the top of `V_Core` so they would be 16-byte aligned for SSE2. 8630529f8 2009-02 “SPU2ghz: … implemented some minor speedups.” · 7d00eb7ec 2009-03 “SPU2-X: Resorted the mixer so that it's a little faster” · ef2e9e209 2009-03 “SPU2-X: Fixes alignment condition for SSE2-dependent members”

### pcsx2-003 — Jump the clock forward when code is only waiting
Lens: laziness · Source: PCSX2
Principle: If a simulated program is spinning on a status register or sitting in a known idle loop, executing each iteration only burns host time until the next event fires. Detect the wait, either a read of the polled register or entry to the idle block, and move the simulated clock straight to the next scheduled event.
Why it is faster: —
Applies elsewhere: discrete-event simulators skipping to the next timestamp, test frameworks with fake timers that advance to the next pending timeout, tickless kernels sleeping until the next timer.
Watch out: —
Seen in PCSX2: The INTC_STAT hack sets `cpuRegs.cycle = g_nextBranchCycle` whenever the EE reads `INTC_STAT`, the register games poll while waiting for interrupts. Later the EE recompiler marked the block at `0x81fc0`, a kernel idle loop, as fast-forward. Its epilogue uses `xCMOVL` to set the cycle counter to `max(cycle+blockCycles, g_nextBranchCycle)` and returns, so the loop never spins. 4bd685485 2009-02 “New Speed Hack! And a good one!” · d7341d5b6 2009-04 “…fast-forward a block starting at 0x81FC0 which some games (FFX) run a lot”

### pcsx2-004 — Test the hottest key before the general dispatch
Lens: branching-fast-paths · Source: PCSX2
Principle: When a single input dominates calls to a multi-way handler, check for it first and return directly. That one compare saves the switch, the nested page decoding and the logging hooks on most calls.
Why it is faster: —
Applies elsewhere: an HTTP router checking `/health` before its route table, a memory allocator with a fast path for its most common size class, a parser special-casing the most frequent token.
Watch out: —
Seen in PCSX2: `hwRead32` gained `if (mem == 0x1000f000) return *((u32*)&PS2MEM_HW[0xF000]);` ahead of its page switch, because games read INTC_STAT constantly. Days later the shortcut moved into the vtlb recompiler's constant-address path, which emits the read directly. 1e4f48437 2009-03 “Shortcut for intc_stat reads” · d60718e79 2009-04 “More vtlb optimizations: … added a shortcut for the INTC_STAT register”

### pcsx2-005 — Resolve address translation at compile time for constant addresses
Lens: jit-interpreters · Source: PCSX2
Principle: When a code generator can prove that an address is constant, it can do the page-table lookup itself and emit either a direct memory access or a direct call to the right handler. The generated code then skips the runtime lookup, the sign test and the branch. This is valid as long as any change to the mapping invalidates the generated code.
Why it is faster: —
Applies elsewhere: link-time resolution of GOT/PLT entries, inline caches that bake a resolved property offset into JIT code, an ORM caching the column offsets for a statement it has already prepared.
Watch out: —
Seen in PCSX2: EE loads and stores check `IS_EECONSTREG(_Rs_)` and call `vtlb_DynGenRead32_Const`/`Write_Const` with the folded address. A second pass reads `vtlbdata.vmap` at recompile time. If `ppf >= 0` it emits a plain `MOV` from the host address, and otherwise a direct call to the handler, on the assumption that a TLB change clears the recompiler. A third pass emits 64/128-bit const reads through a free MMX/XMM register. e20599974 2009-02 “Minor Optimization: Added Const support to Vtlb's Load and Store” · d60718e79 2009-04 “Switched over to full const resolution of the TLB” · a8d9cbc25 2009-04 “Improved the codegen for const-propagated direct reads and writes”

### pcsx2-006 — Fill dispatch tables with a "compile me" stub instead of null
Lens: jit-interpreters · Source: PCSX2
Principle: If an empty slot in a lookup table holds null, every dispatch has to check for null and branch. Initialize every slot with a pointer to a stub that does the slow work (compile, load or resolve) and then overwrites the slot. Dispatch becomes an unconditional indirect jump.
Why it is faster: —
Applies elsewhere: lazy PLT binding in dynamic linkers, vtables whose entries point at a resolver on first call, lazy-initialized function pointers in plugin hosts.
Watch out: —
Seen in PCSX2: The EE and IOP block managers fill `recLUT` (now a static `uptr recLUT[0x10000]`) with the address of `JITCompile`. The dispatcher went from a `PC_GETBLOCK` lookup, a `startpc` compare and a conditional recompile to `mov ecx,[recLUT+esi*4]; jmp [ecx+ebx*2]`. `JITCompile` recompiles the block and jumps into it. 482505d24 2009-03 “Improved block managers for EE/IOP recompilers! … replacing NULL/invalid pointer checks with a direct link to a JIT dispatcher” · 801d71f7f 2009-03 “Fixes for the new block manager optimizations… a bit more speed even”

### pcsx2-007 — Patch direct jumps when targets appear, using a back-reference list
Lens: jit-interpreters · Source: PCSX2
Principle: Generated code that jumps to another block should jump straight there instead of returning to a dispatcher each time. Record every jump site by target address. When the target is compiled, patch all waiting sites at once. Linking then costs nothing at execution time.
Why it is faster: —
Applies elsewhere: incremental linkers fixing up relocations, a bundler patching module references when a chunk loads, trace-linking in tracing JITs.
Watch out: —
Seen in PCSX2: The runtime dispatchers that patched the previous block's `jmp` were removed. `BaseBlocks::Link(pc, jumpptr)` writes either the target's `fnptr` or the recompiler stub and inserts the site into a `std::multimap<u32,uptr> links`. `BaseBlocks::New` walks `links.equal_range(startpc)` and rewrites each rel32 to the new block. The commit deleted 86 lines of assembly from each dispatcher file. c483f1733 2009-03 “Rewrite immediate jumps from the block manager instead of having dispatchers to do this at execution time”

### pcsx2-008 — Choose the shortest equivalent instruction encoding
Lens: jit-interpreters · Source: PCSX2
Principle: When two encodings do the same thing, the shorter one fits more code into the instruction cache and the decoders. Code generators should prefer short forms, such as xor-zeroing, base-plus-displacement addressing instead of absolute addresses, and opcodes without a prefix byte.
Why it is faster: —
Applies elsewhere: compiler backends (`xor eax,eax` instead of `mov eax,0`), WebAssembly and bytecode encoders using varint and short-form opcodes, firmware tuned to fit flash.
Watch out: —
Seen in PCSX2: The x86 emitter has an `AlwaysUseMovaps` constant that routes every MOVDQA through MOVAPS, which is one byte shorter because it has no 0x66 prefix. The yuv2rgb and Freeze*Regs code switched to Mod/RM memory-operand forms. Loads of GPR 0 emit `XOR32RtoR`. microVU keeps the VI register file base in a GPR (`gprR`) and addresses it as `[gprR+disp]` instead of 32-bit absolute addresses. 79cde4c09 2009-03 “x86 emitter … always uses MOVAPS over MOVDQA” · 71e2dc2fb 2009-03 “Optimized yuvrgb_sse2, by using Mod/RM form instructions” · 4c8cf52c9 2009-04 “zero register should use xor” · a8094d578 2009-05 “use a GPR as a reg-offset index to save a few bytes”

### pcsx2-009 — Avoid partial-register writes that create false dependencies
Lens: simd · Source: PCSX2
Principle: Instructions that write only part of a vector register (scalar moves, in-place shuffles) have to merge with the register's old contents. That ties them to the previous writer and limits out-of-order execution. When the upper lanes don't matter, use full-width moves and non-destructive shuffles that write the whole register.
Why it is faster: —
Applies elsewhere: hand-written SIMD in codecs and math libraries, compilers emitting `vxorps` before `cvtsi2ss` to break the dependency, game-engine vector code.
Watch out: —
Seen in PCSX2: In microVU, `SSE_MOVSS_XMM_to_XMM` became `SSE_MOVAPS_XMM_to_XMM` wherever the upper three lanes didn't need preserving, and `SHUFPS reg,reg,imm` became `PSHUFD`, for example when flipping the P/Q instance in `xmmPQ`. b894a7a21 2009-03 “operations that don't modify all the vectors can give false-dependencies… always better(faster) to use movaps”

### pcsx2-010 — Group hot lookup tables into one cache-aligned block
Lens: memory-layout · Source: PCSX2
Principle: Placing related hot tables as separate globals lets the linker scatter them. They can then conflict in the cache, and performance shifts with unrelated code changes. Put them in one struct aligned to a cache line, so their placement is fixed and they don't evict each other.
Why it is faster: —
Applies elsewhere: packet-processing lookup tables in DPDK, per-core counters padded to cache lines, interpreters keeping dispatch and register tables together.
Watch out: —
Seen in PCSX2: The vtlb's `pmap`, `vmap` and `RWFT[5][2][128]` handler table were moved into one `PCSX2_ALIGNED(64, MapData vtlbdata)`, and every access went through it. The commit message blames a 30% slowdown in an earlier revision on “bad cacheline coloring”. c5b366095 2009-03 “Fix for some random slowdown introduced in 815 (bad cacheline coloring caused a random 30% speed drop)”

### pcsx2-011 — Inline the fast path and share out-of-line slow-path stubs
Lens: jit-interpreters · Source: PCSX2
Principle: At every generated memory access, emit only the common case inline: a table lookup and a direct load or store. Send rare cases through a conditional jump to a stub shared by all call sites. Generated code stays small, the hot path has no call setup, and the shared stubs stay warm in cache.
Why it is faster: —
Applies elsewhere: JIT write barriers and allocation fast paths in GCs, bounds checks that branch to a shared trap handler, `__builtin_expect` with `noinline` cold helpers in C.
Watch out: —
Seen in PCSX2: `vtlb_dynarec_init` generates ten tiny indirect dispatchers (read/write × 5 operand sizes), each 64-byte aligned, into one executable page `m_IndirectDispatchers`. Each one does `movzx eax,al; sub ecx,0x80000000; sub ecx,eax; call [RWFT…]; jmp ebx`. The inline code is just the `vmap` lookup, a direct access, and `xJS(GetIndirectDispatcherPtr(mode,szidx))`. The commit reports a 2–3% speedup. 1cf028ccf 2009-05 “Optimized vtlb direct path memory operations using some crafty asm code. Expect 2-3% speedups”

### pcsx2-012 — Track code-page writes with counters and re-protect pages that settle down
Lens: jit-interpreters · Source: PCSX2
Principle: To catch self-modifying code cheaply, write-protect pages that contain compiled code. Pages that are written often fall back to checking their contents on entry. Count how many times each manual page runs, and when the counter overflows, reset the page to protected mode so stable code stops paying for the per-entry check.
Why it is faster: —
Applies elsewhere: JS engines deoptimizing and later reoptimizing, adaptive locks moving between spin and blocking, cache invalidation that switches from per-write to batch mode under heavy churn.
Watch out: —
Seen in PCSX2: Blocks on manually checked pages emit `xADD(ptr16[&manual_page[inpage_ptr>>12]],1)` followed by a carry jump to `dyna_block_reset`. That function clears the 4 KB page and calls `mmap_MarkCountedRamPage` to protect it again. An earlier experiment replaced whole-page invalidation with a 16-byte-granularity dirty bitmap (`vtlb_alloc_bits`) tested with `BTS`/`TEST`. It made some games much faster but was reverted because of complications. d7341d5b6 2009-04 “Protect manual pages again after running code in them for a while” · 3fb09c219 2009-04 “Uses BTS + bit arrays for manual block tracking, instead of full invalidation” · 9513be547 2009-04 “Reverted BTS … retained a minor optimization”

### pcsx2-013 — Remove arbitrary capacity caps that force full flushes
Lens: allocation · Source: PCSX2
Principle: A fixed limit on cache entries, when hitting it means "throw everything away", causes periodic stalls well before memory actually runs out. Let the index grow dynamically, and flush only when the real resource (the code buffer) is full.
Why it is faster: —
Applies elsewhere: connection pools with a hard maximum that drop everything, fixed-size hash tables rebuilt from scratch, translation caches in emulators and binary translators.
Watch out: —
Seen in PCSX2: `BaseBlocks` no longer takes a `size`, and `New()` no longer returns null at `EE_NUMBLOCKS` (1<<15) or `PSX_NUMBLOCKS` (1<<12). The vector just reserves 0x4000 and grows, so the EE and IOP recompilers stop calling `recResetEE`/`recResetIOP` merely because the block count hit the limit. a7cbcae4b 2009-05 “removing the legacy limit on the number of recompiled blocks (was causing recompilers to reset the code buffers way more than necessary)”

### pcsx2-014 — Store entries by value in a sorted array and binary-search it
Lens: memory-layout · Source: PCSX2
Principle: A vector of pointers to separately allocated records costs a pointer chase and a likely cache miss per comparison. Store the records contiguously in address order, and look them up by binary search, including reverse lookups by output address.
Why it is faster: —
Applies elsewhere: symbol tables for profilers that map PCs to functions, interval maps for memory regions, read-mostly sorted arrays replacing `std::map`.
Watch out: —
Seen in PCSX2: `BASEBLOCKS`, a `vector<BASEBLOCKEX*>` with special cases for 0 and 1 elements, became `BaseBlocks` holding `std::vector<BASEBLOCKEX>` with reserved capacity. `GetByX86(ip)` is a binary search over `fnptr` with a bounds check against `x86size`, and the accessors are now `__forceinline`. 801d71f7f 2009-03 “Fixes for the new block manager optimizations” · 4d8769ecc 2009-04 “Implemented another block lookup method” · 9513be547 2009-04 “retained a minor optimization”

### pcsx2-015 — Skip computing outputs that no consumer ever reads
Lens: jit-interpreters · Source: PCSX2
Principle: If analysis shows that nothing downstream reads a result, such as a flag register or a derived field, don't generate the code that computes it. When it is read only sometimes, compute only the part that is read, and only on the instructions whose results can reach a reader.
Why it is faster: —
Applies elsewhere: dead-store elimination in compilers, GraphQL resolvers that compute only the requested fields, lazily evaluated derived columns in dataframes.
Watch out: —
Seen in PCSX2: `mVUcheckSflag` runs the analysis pass over a whole microprogram. If no instruction reads the status flag, `sFlagHack` clears `_doStatus` and the stores to `VI[REG_STATUS_FLAG]` are skipped (8–10% in FFX's intro). A second variant (`CHECK_VU_FLAGHACK2`) works per block. A further change added `doNonSticky`: only the last few instructions before a reader clear and update the O/U/S/Z bits. ef01d581b 2009-05 “optimizes out all status flag updates in a microprogram if the entire program doesn't read the status flags” · 342da0e2a 2009-05 “Added a new status-flag speedhack” · b65b439e6 2009-06 “Further optimized status flag updating”

### pcsx2-016 — Search caches in recency order and age entries in generations
Lens: algorithms · Source: PCSX2
Principle: When a lookup scans a cache of expensive artifacts, check recently used entries first, because hits cluster in time. Rather than evicting everything when full, age entries: young entries are searched before old ones, entries unused for a long time are killed, and only a fraction is evicted when the cache is full.
Why it is faster: —
Applies elsewhere: generational GCs, CLOCK/2Q page replacement, CDN edge caches, prepared-statement caches in database drivers.
Watch out: —
Seen in PCSX2: microVU program search: `mVUcmpProg` first checks programs with `last_used` set. Programs unused for about 7 seconds are killed (`curFrame - frame`). When all slots are full, a quarter are killed instead of all of them. A `progList` keeps search order oldest to newest. Later, `isOld` was added after 30 frames unused: young programs are searched first, then old ones, and death comes after 3,600 frames. 2c3043862 2009-05 “Cached Program search is faster/smarter … search recently used blocks first” · 7f5827cee 2009-07 “Kill programs if they haven't been used in ~7 seconds. Only kill 1/4” · 2bcf704ff 2009-07 “we now have young/old/dead programs”

### pcsx2-017 — Validate a cached artifact against only the input bytes it used
Lens: laziness · Source: PCSX2
Principle: If a cached result (compiled code, a rendered fragment) depends on only part of a large input, record which ranges were actually consumed. On lookup, compare just those ranges. Unrelated changes elsewhere in the input no longer cause misses, and the comparison costs less.
Why it is faster: —
Applies elsewhere: build systems hashing only the headers a file actually includes, React memo comparing selected props, HTTP caches varying only on the headers that matter.
Watch out: —
Seen in PCSX2: Each `microProgram` records `range[0..1]`, the PCs that were actually compiled (`mVUsetupRange`). `mVUcmpProg(..., cmpWholeProg=0)` runs `memcmp_mmx` over only that slice of VU micro memory, which helps games that upload many small programs. The single range later became a list (`ranges.range[j]`, checked by `mVUcmpPartial`), so fewer program slots are used. ac00db5dd 2009-05 “Added partial program searching support (helps to recompile less…)” · 8f74247b4 2009-06 “Smarter microProgram comparison. Games will now use less microProgram 'slots'”

### pcsx2-018 — Compare fixed-size keys with aligned SIMD and order fields for early exit
Lens: simd · Source: PCSX2
Principle: For a fixed-size, aligned struct used as a lookup key, a generic `memcmp` is slower than comparing 16 bytes at a time with vector equality plus a movemask. Put the most discriminating fields first and test the first chunk before the rest, so most mismatches exit after one or two vector compares.
Why it is faster: —
Applies elsewhere: hash-table probes comparing 16-byte keys, SwissTable-style metadata matching, deduplication of fixed-size records.
Watch out: —
Seen in PCSX2: `microRegInfo` was reordered “for Faster Compares” (`needExactMatch` and `VI[16]` first) and padded to 160 bytes, 16-byte aligned. `mVUsearchXMM` does `movaps`+`pcmpeqd`+`pand` on the first 32 bytes, exits if `movmskps != 0xf`, and otherwise checks the remaining eight chunks. The routine was first naked inline asm. Later it was emitted at startup by the project's x86 emitter, so Linux builds could use it too. c4531a5c3 2009-06 “Implemented a faster block-compare algorithm” · 2ca4a603b 2009-06 “Rewrote the custom compare function to use the emitter instead of inline asm”

### pcsx2-019 — Canonicalize cache keys by dropping state that doesn't matter
Lens: laziness · Source: PCSX2
Principle: When compiled code is keyed by the full machine state at entry, irrelevant differences such as already-expired stall counters create needless duplicate entries. Normalize or zero the parts of the key that don't affect the output, and require an exact match only for entries that really need one. Hit rates go up and recompilation goes down.
Why it is faster: —
Applies elsewhere: normalizing URLs (sorting query parameters) before caching, stripping volatile fields before hashing build inputs, shader caches that ignore unused state.
Watch out: —
Seen in PCSX2: microVU's block search does a cheap compare unless `blockList[0].pState.needExactMatch` is set, and in that case it matches q, p and clip. The block-end pipeline state is normalized: `optimizeReg(mVUregs.q/p)`, and `mVUregs.r = 0` because “there are no stalls on the R-reg”. A WIP commit lists “a Pipeline State optimization to remove some unnecessary information. (speedup)”. 912185348 2009-05 “Fixed up some code so blocks are recompiled less often” · d9b5baf8f 2009-07 “Added a Pipeline State optimization to remove some unnecessary information” · 3f2f8571d 2009-08 “microVU: some optimizations…”

### pcsx2-020 — Propagate constants to turn indirect jumps into direct branches
Lens: jit-interpreters · Source: PCSX2
Principle: If simple constant tracking shows that the register used by an indirect jump holds a known value, the compiler can treat the jump as a direct branch: link to the target block, or compile it, at compile time. This removes the runtime lookup by address.
Why it is faster: —
Applies elsewhere: devirtualization in C++/Java compilers, inline caching for monomorphic call sites, constant folding of computed gotos.
Watch out: —
Seen in PCSX2: `mVUconstReg[]` tracks VI registers set by immediate ops (`setConstReg`), and vi15's constness is carried across blocks in `mVUregs.vi15`. `mVUanalyzeJump` sets `mVUlow.constJump`, and the compiler then searches or compiles `mVUblocks[jumpAddr/8]` directly, as for a normal branch. d9b5baf8f 2009-07 “Added Simple Constant Propagation to detect Constant Indirect Jump addresses allowing them to act as normal branches. (speedup)”

### pcsx2-021 — Use one shared arena instead of fixed per-tenant partitions
Lens: allocation · Source: PCSX2
Principle: Dividing a buffer evenly among slots wastes space in idle slots and forces overflow in busy ones. Let all tenants allocate from one shared arena, and size the slot count to the workload, so capacity goes where demand is.
Why it is faster: —
Applies elsewhere: a shared buffer pool in a database instead of per-table pools, a unified GPU memory heap, one JIT code cache shared across modules.
Watch out: —
Seen in PCSX2: microVU used to give each cached program `cacheSize/(prog.max+1)` of the code buffer. It now emits all programs into one global `prog.x86start..x86end` region, and VU1's program slots went from 64 to 400. Earlier the slot counts were rebalanced to 64 for VU1 and 8 for VU0, which sped up FFXII. d4132141d 2009-06 “changed VU1's cached program amount from 32 to 64, and vu0's from 32 to 8” · d9b5baf8f 2009-07 “Made microPrograms recompile to a global rec-cache instead of per-program cache”

### pcsx2-022 — Deduplicate constants in a pool using a small hash
Lens: memory-layout · Source: PCSX2
Principle: Appending a fresh copy of every constant that generated code references fills the constant area quickly and forces resets. Look the value up in a small direct-mapped cache of recently pooled constants first, and reuse the existing slot on a hit.
Why it is faster: —
Applies elsewhere: string interning, literal pools in assemblers, deduplicating uniform buffers in renderers.
Watch out: —
Seen in PCSX2: `recAllocStackMem(8,4)` was replaced with `recGetImm64(hi, lo)`. It checks `imm64_cache[lo % 509]` and returns the existing pointer if both words match. Otherwise it appends to `recConstBuf`. The commit says this means fewer recompiler resets, for example in Espgaluda, which had been resetting every couple of seconds. 5bd531e21 2009-07 “reuse recent constants rather than duplicating for every instance, resulting in less recompiler resets”

### pcsx2-023 — On reset, clear only what is actually stale
Lens: memory-layout · Source: PCSX2
Principle: A reset routine that wipes every buffer, including large regions that are rebuilt or never read before being overwritten, turns each reset into a visible stall. Clear only the metadata that encodes validity, and keep debug-pattern fills for debug builds.
Why it is faster: —
Applies elsewhere: arena allocators resetting a bump pointer instead of zeroing, GPU resource pools invalidating handles instead of clearing memory, an HTTP server reusing request buffers without memset.
Watch out: —
Seen in PCSX2: `recResetEE` now skips the 32 MB RAM copy inside `m_recBlockAlloc` (`m_recBlockAllocSize - Ps2MemSize::Base`), and the `0xcc` fill of `recMem` runs only when `IsDevBuild`. This was tried first as a separate `recResetEE_Fast`, then made the only path. ae85ad0f3 2009-07 “Tweak for recompiler resets, helps a bit against the stalls” · 21dac5015 2009-07 “Cleaner version of the EE reset optimisation”

### pcsx2-024 — Don't spill or flush registers the operation leaves unchanged
Lens: jit-interpreters · Source: PCSX2
Principle: A register cache should write back and invalidate only the values an operation actually modifies. Flushing an input that is merely read costs a store now and a reload later.
Why it is faster: —
Applies elsewhere: compiler register allocators treating operands as read-only, write-back CPU caches, ORMs that track dirty fields and update only those columns.
Watch out: —
Seen in PCSX2: EE loads and stores called `_deleteEEreg(_Rs_, 1)`, which flushes the base-address register, even on the constant-address path, where Rs isn't needed at all. The call moved into the non-const branch. A later pass replaced `_deleteEEreg`+`MOV32MtoR` with `_eeMoveGPRtoR(ECX,_Rs_)`, which copies from wherever the value is cached. 43c0d895a 2009-03 “Removed unnecessary rs flushes for loads and stores” · 4c8cf52c9 2009-04 “Regalloc should be working a bit better now”

### pcsx2-025 — Merge partial writes in a vector register cache instead of writing back
Lens: jit-interpreters · Source: PCSX2
Principle: When a cached vector register is only partly modified, writing it to memory and reloading it to combine lanes costs two memory round trips. If another register already holds the same variable, blend the modified lanes into it in registers, and load known-zero values with a register XOR instead of from memory.
Why it is faster: —
Applies elsewhere: SIMD code generators for shader compilers, SSA-based vector lane tracking, game-engine math libraries with masked updates.
Watch out: —
Seen in PCSX2: microVU's regAlloc `clearNeeded` used to write back every partial write (`xyzw < 0xf`). Now, if a read-only copy of the same VF register is cached, it calls `mVUmergeRegs(i, reg, xyzw)` and drops the partial register. Loads of VF0 without the w lane emit `PXOR`. Finishing regAlloc removed the per-op `flushAll` “for Testing” calls, for a 0–2% gain. 98c185591 2009-07 “More regalloc work/fixes” · 37675b6c4 2009-07 “Finished implementing regAlloc”

### pcsx2-026 — Keep diagnostics and safety checks out of the release hot path
Lens: other · Source: PCSX2
Principle: Infinite-loop detectors, debug fills and trace logging are useful during development, but in release builds they cost branches, calls and stores on every execution. Compile them in for dev builds only, so release code runs only the real work.
Why it is faster: —
Applies elsewhere: `assert` versus `NDEBUG`, log levels checked at compile time, React development-only warnings stripped from production bundles.
Watch out: —
Seen in PCSX2: microVU now emits the “Exiting from Possible Infinite Loop” cycle test only for dev builds or VU0. An earlier change commented out `mVUtestCycles` (“costs a few fps”). `UpdateCP0Status` lost its unimplemented kernel/user memory-mode switch “given this function is called so much”. In the bundled wxWidgets, Windows-message trace logging in `window.cpp` was disabled. fa0e10557 2009-05 “Disabled breaking execution on bad programs since it's not needed anymore and costs speed” · 5025b3886 2009-08 “Removed the "Exiting from Possible Infinite Loop" logic from Release builds” · 149f84b00 2009-03 “A few tweaks to more often called functions = general speedup” · 7b800cf29 2009-09 “Disable debug trace logigng of Windows Messages, because it's really! slow”

### pcsx2-027 — Force-inline tiny hot functions and use register calling conventions
Lens: compiler-build · Source: PCSX2
Principle: Small functions called from very hot loops pay call overhead that can exceed their bodies, and stack-based argument passing adds memory traffic. Mark them for forced inlining, or use a register-argument convention where they can't be inlined. Uninline large, rarely called helpers.
Why it is faster: —
Applies elsewhere: `static inline` accessors in C hot paths, `[MethodImpl(AggressiveInlining)]` in .NET, keeping cold error paths `noinline`.
Watch out: —
Seen in PCSX2: `cpuSetNextBranch`, `cpuTestCycle` and `cpuException` became `__fastcall`, and `cpuTestHwInts` became `__forceinline`. The counter functions (`rcntStartGate`, `rcntWmode`, `rcntRcount` and others) went from `__fastcall` to `__forceinline`, while one-time `vSyncInfoCalc` lost `__forceinline`. SPU2-X force-inlined `MixCoreVoices` and `TimeUpdate`. The block-manager accessors `Index`/`Get`/`operator[]` became `__forceinline`. 149f84b00 2009-03 “A few tweaks to more often called functions = general speedup” · 1d9adee46 2009-04 “couple of changes, very minor speedup” · b14881769 2009-04 “SPU2-X: Small cleanup and made it a bit faster”

### pcsx2-028 — Turn on optimization flags for bundled third-party libraries
Lens: compiler-build · Source: PCSX2
Principle: Vendored libraries often build with default or debug-leaning settings even in release builds. Give them the same speed-oriented flags as the main code: inlining, favor speed, frame-pointer omission and no buffer-security checks.
Why it is faster: —
Applies elsewhere: CMake subprojects that inherit no `CMAKE_BUILD_TYPE`, Rust `[profile.release.package."*"]` overrides, npm native addons built without `-O2`.
Watch out: —
Seen in PCSX2: The zlib and bzip2 vcproj files gained `InlineFunctionExpansion="2"`, `FavorSizeOrSpeed="1"`, `OmitFramePointers`, `StringPooling` and `BufferSecurityCheck="false"` in their Release and Devel configurations. The same change made the emitter always use MOVAPS. 79cde4c09 2009-03 “Enabled optimization stuff for zlib and bzip2 projects (Release/Devel build targets)”

### pcsx2-029 — Replace heavy template instantiation with runtime dispatch where it isn't hot
Lens: compiler-build · Source: PCSX2
Principle: Templating whole class hierarchies or hundreds of functions on a parameter multiplies code size and link time. Where that parameter isn't on a per-item hot path, pass it as a function argument or use a virtual interface. The binary shrinks, the instruction cache benefits, and builds get faster.
Why it is faster: —
Applies elsewhere: type-erased `std::function` or interfaces at module boundaries, Rust `dyn Trait` instead of generics in cold code, Java/Kotlin avoiding inline-class explosion.
Watch out: —
Seen in PCSX2: GSdx converted `GPURendererSW<GSDevice7/9/10>` and similar classes to a single class taking a `GSDevice*`, and the DLL shrank by more than 100 KB. microVU turned its `recPass` template parameter into a function argument (`mVUopU<vuIndex>(pass)`), which halved release link time. 3b22adc4b 2009-05 “GSdx: converted a few template classes to pointers and virtual functions, the dll got more than 100k smaller” · 6d6e7fba3 2009-05 “Changed 'recPass' from a template param to a function param”

### pcsx2-030 — Specialize the dominant format pair in a generic copy loop
Lens: algorithms · Source: PCSX2
Principle: A generic converter that dispatches read and write functions per pixel or element through pointers is slow for the common case where source and destination formats match. Add a dedicated loop for the dominant pair, and hoist per-row address computation out of the inner loop with precomputed offsets.
Why it is faster: —
Applies elsewhere: image libraries with RGBA8 to RGBA8 fast paths, serialization with a memcpy path for identical layouts, database bulk-copy when schemas match.
Watch out: —
Seen in PCSX2: GSdx's local-to-local transfer used `rp`/`wp` function pointers per pixel. It now computes `sbase = spsm.pa(0,sy,…)` and `soffset = spsm.rowOffset[sy&7]` once per row and, when both formats are `PSM_PSMCT32`, calls `WritePixel32(dbase+doffset[dx], ReadPixel32(sbase+soffset[sx]))` directly. This fixed BoF5's speed, and a further rework boosted FFXII and ICO. 35655edb4 2009-04 “GSdx: the BoF5 speed fix” · 09ee76d9e 2009-05 “GSdx: optimized local-local transfer a bit”

### pcsx2-031 — Precompute offset tables for swizzled or tiled addressing
Lens: lookup-tables · Source: PCSX2
Principle: Addressing tiled or swizzled memory involves bit shuffling for every pixel. Precompute per-row and per-column offset tables, and per-block tables for coarser loops, so the inner loop becomes base plus a table lookup.
Why it is faster: —
Applies elsewhere: Morton/Z-order texture addressing, tiled framebuffer drivers, column-major to row-major transposes with index tables.
Watch out: —
Seen in PCSX2: GSdx added `blockOffset32/16/8/4[256]` tables beside the existing `rowOffset` tables to support block-granular texture-cache tracking. CLUT reads went from `ReadPixel32(base + offset[i])` to `s[o[i]]`, using a cached `PixelOffset` with `row[]`/`col[]` arrays obtained through `GetPixelOffset`. 3202a4257 2009-05 “GSdx: Reworked the sw renderer texture cache a bit … keeping track of blocks” · f9f056d58 2009-07 “GSdx: fixing/breaking things again…”

### pcsx2-032 — Write uploads straight into mapped destination memory
Lens: gpu-rendering · Source: PCSX2
Principle: Decoding into a static staging buffer and then copying into a GPU resource touches every byte twice. Map or lock just the destination sub-rectangle and decode directly into it. If a full-surface lock is unavoidable, restrict the work to the dirty region.
Why it is faster: —
Applies elsewhere: `glMapBufferRange` with exact ranges, zero-copy network receive into application buffers, decoding images directly into canvas pixel buffers.
Watch out: —
Seen in PCSX2: `GSTexture::Map` gained an optional `GSVector4i* r`. On DX9 it passes the rectangle to `LockRect`, and `ReadTextureNP` writes into the locked bits instead of a static 4 MB `_aligned_malloc` buffer (Odin Sphere went back to 40–50 fps). The texture cache then tracks valid blocks per page with counts (`m_blocks`) and returns early when the texture is fully valid. 4bc698d69 2009-05 “GSdx: just fixing slow texture uploads with dx9” · 7c2e51157 2009-06 “GSdx: dx9 texture uploads should be at least as fast as before”

### pcsx2-033 — Skip redundant graphics API state calls
Lens: gpu-rendering · Source: PCSX2
Principle: Each driver call has fixed overhead, so setting the same state, or opening and closing a scene, for every draw adds up quickly. Bracket the whole frame once, compare new state with what is already bound before calling the API, and don't unbind resources between draws when the next draw rebinds them.
Why it is faster: —
Applies elsewhere: WebGL state-caching wrappers, Vulkan/Metal command encoders that diff pipeline state, React reconciliation skipping unchanged DOM properties.
Watch out: —
Seen in PCSX2: DX9 `BeginScene`/`EndScene` moved from every `SetupDATE`/draw into `Flip`, once per frame (Issue 256). `OMSetDepthStencilState` folds the stencil ref into the state object, so its `m_dss != dss` check now covers both. DX10/11 no longer call `PSSetShaderResources(NULL,NULL)` on every draw. db8feac8c 2009-06 “GSdx: the BeginScene/EndScene optimization for dx9” · 8e979f1b9 2009-06 “GSdx: moved 16 bit texture processing from gpu to cpu” · a0828bc8f 2009-07 “Don't clear some shaders at each drawcall (in dx10)”

### pcsx2-034 — Recycle expensive resources through a larger pool
Lens: allocation · Source: PCSX2
Principle: Creating and destroying GPU textures, or any costly object, every frame wastes driver time. Return them to a pool keyed by size and format instead of deleting them, and make the pool big enough for the real working set so live objects aren't evicted and recreated.
Why it is faster: —
Applies elsewhere: object pools in game engines, database connection pools, `sync.Pool` in Go, slab caches in kernels.
Watch out: —
Seen in PCSX2: `GSDevice`'s texture pool limit was raised from 200 to 600 (“many games constantly recreated them”). The `m_merge` render target is now handed to `Recycle()` instead of being `delete`d when it is resized. a0828bc8f 2009-07 “Trippled the number of cached textures, many games constantly recreated them” · 6447ef282 2009-10 “GSdx: Recycle the m_merge texture instead of deleting it”

### pcsx2-035 — Size producer–consumer queues for bursty workloads
Lens: concurrency · Source: PCSX2
Principle: If a ring buffer between threads is smaller than the largest burst, the producer stalls waiting for the consumer, and the two threads end up running in lockstep. Make the buffer big enough to absorb the biggest bursts, and the threads overlap again.
Why it is faster: —
Applies elsewhere: audio callback buffers, Kafka/LMAX disruptor ring sizing, GPU command-buffer queues, log shippers.
Watch out: —
Seen in PCSX2: The MTGS ring buffer between the EE thread and the GS thread grew from 2 MB to 8 MB (`m_RingBufferSizeFactor` 17 → 19), which fixed slowness in data-heavy games such as the Xenosaga series. a0e6c8c26 2009-03 “Increased the MTGS ringbuffer from 2 to 8mb. Fixes slowness in games with lots of data moving”

### pcsx2-036 — Drop thread-safety the program doesn't need
Lens: concurrency · Source: PCSX2
Principle: Libraries and drivers often lock on every call when asked for multithreaded mode. If the program already guarantees single-threaded access to an object, request the single-threaded variant and skip the per-call locking.
Why it is faster: —
Applies elsewhere: SQLite `SQLITE_OPEN_NOMUTEX`, `StringBuilder` instead of `StringBuffer`, `Rc` instead of `Arc` in Rust, non-atomic refcounts in single-threaded runtimes.
Watch out: —
Seen in PCSX2: GSdx stopped passing `D3DCREATE_MULTITHREADED` to DX9 and now sets `D3D10/11_CREATE_DEVICE_SINGLETHREADED`, since only the GS thread uses the device. 0b6ed4bc1 2009-07 “GSdx: Disabled requesting of thread safe D3D devices. This is a speedup”

### pcsx2-037 — Block on events instead of sleep-polling, and use fine timer resolution
Lens: concurrency · Source: PCSX2
Principle: Sleep-and-recheck loops either waste CPU when they spin or add latency when they oversleep, and coarse OS timer granularity makes it worse. Wait on an event that the producer signals, put idle threads to sleep when there's nothing to do, and raise the timer resolution so the remaining short sleeps are accurate.
Why it is faster: —
Applies elsewhere: condition variables instead of `while(!ready) sleep(1)`, epoll/kqueue instead of polling sockets, frame limiters in games and media players.
Watch out: —
Seen in PCSX2: The cdvdGigaherz read thread now waits on `hRequestComplete` instead of sleeping, and sleeps 10 ms when the tray is empty or open instead of spinning. The framelimiter's sleep mode was reworked, and `EnableHiresScheduler` calls `timeBeginPeriod` on Windows. f64876203 2009-10 “Replaced a thread sleep with an Event when waiting on pending events (speedup)” · 27174c3f5 2009-10 “Added timeBeginPeriod() to improve the Win32 kernel scheduler resolution”

### pcsx2-038 — Leave thread placement to the OS after a pinned measurement
Lens: platform · Source: PCSX2
Principle: Pinning a thread to one core makes sense for a timing measurement, such as calibrating a cycle counter. Leaving it pinned afterwards stops the scheduler from spreading load. Restore the full affinity mask once the measurement is done.
Why it is faster: —
Applies elsewhere: benchmark harnesses that pin during warm-up, RDTSC calibration in game engines, test runners that inherit a restricted CPU mask.
Watch out: —
Seen in PCSX2: CPU detection pinned the thread with `SetSingleAffinity()` to time the CPU and never undid it. `UnSetSingleAffinity()` now restores `0xFFFFFFFF`, which the commit puts at up to 25% faster on an Athlon X2 and about 5% on a Core 2. Calibration also switched from `timeGetTime` to `GetCPUTicks`, making startup “near instantaneous”. 62cc3cfbf 2009-08 “Workaround a problem with thread affinities. Should fix some slowness issues, mainly on AMD cpus” · 074a87ead 2009-06 “Improved the cpu speed/ghz detection … and is faster”

### pcsx2-039 — Use compiler-native thread-local variables instead of TLS API calls
Lens: platform · Source: PCSX2
Principle: OS TLS APIs such as `TlsGetValue` and `pthread_getspecific` make a function call and an index lookup on every access. A compiler `thread`-qualified variable compiles to a segment-relative load.
Why it is faster: —
Applies elsewhere: `thread_local` in C++11, `__thread` in GCC, per-thread allocator caches, request-scoped context in native servers.
Watch out: —
Seen in PCSX2: In w32pthreads, `pthread_getspecific(ptw32_selfThreadKey)` was replaced with `__declspec(thread) ptw32_thread_t* ptw32_selfThread` in `pthread_self`, `pthread_exit` and `pthread_setspecific`. 6c0afe7c4 2009-09 “w32pthreads: minor optimization using _declspec(thread) for internal thread handles, instead of TlsAlloc”

### pcsx2-040 — Route a subsystem's churny allocations to a private heap
Lens: allocation · Source: PCSX2
Principle: Many small, short-lived allocations, such as strings and UI objects, fragment the shared process heap and contend with other threads on its lock. Give that subsystem its own heap through class-level `operator new/delete`. Its churn stays isolated and its lock is separate.
Why it is faster: —
Applies elsewhere: per-subsystem arenas in game engines, jemalloc arenas per thread, Windows `HeapCreate` per module, request-scoped pools in Apache/nginx.
Watch out: —
Seen in PCSX2: wxWidgets on Win32 (`wxUSE_PRIVATE_HEAP`): `wxObject` gets `operator new/new[]/delete` that call `_allocHeap_wxObject`/`_freeHeap_wxObject`, and `wxString` has its own heap as well (`HeapAllocator.cpp`). 67957b13b 2009-10 “Added a private heap allocation feature … should reduce fragmentation and multithreaded contention”

### pcsx2-041 — Don't shrink short-lived buffers after use
Lens: allocation · Source: PCSX2
Principle: Calling shrink-to-fit on a temporary that will soon be freed costs an extra allocation and copy for no lasting benefit. It also defeats any over-allocation done up front to avoid reallocating. Shrink only long-lived objects with significant slack.
Why it is faster: —
Applies elsewhere: `std::string::shrink_to_fit` in formatting helpers, Java `trimToSize` on temporary lists, Go slices re-sliced instead of copied.
Watch out: —
Seen in PCSX2: In the bundled wxWidgets `string.cpp`, the `Shrink()` after printf-style formatting was commented out. That function starts with a 1 KB buffer, which the shrink immediately threw away. 4047e0ee9 2009-10 “Cure wxWidgets of some over-joyous re-allocation of formatted strings”

### pcsx2-042 — Append in bulk and use shifts for power-of-two radixes
Lens: algorithms · Source: PCSX2
Principle: Padding a string one character at a time in a loop calls append repeatedly, each time checking capacity. A single `append(n, c)` does the same work in one call. For base-16 output, `& 0xf` and `>> 4` replace a general division and modulo.
Why it is faster: —
Applies elsewhere: JSON/CSV writers, hex dump utilities, logging formatters, `String.prototype.padStart` versus loops.
Watch out: —
Seen in PCSX2: In `vssprintf.cpp`, `while (size-- > 0) dest += c;` became `dest.append(size, c)`, digit copies became `dest.append(tmp, n)`, and the number conversion got a `base == 16` branch using `num & 0xf` / `num >> 4`. d1ae92e20 2009-03 “Minor optimizations to vssprintf”

### pcsx2-043 — Pad a handler table to the full index range to drop the range check
Lens: lookup-tables · Source: PCSX2
Principle: Dispatching through a small table means bounds-checking the index first. Extend the table to cover every possible index, filling unused slots with a default handler, and the check becomes an unconditional indexed call.
Why it is faster: —
Applies elsewhere: 256-entry byte-dispatch tables in interpreters and tokenizers, syscall tables padded with `ENOSYS` stubs, HTTP method lookup by first byte.
Watch out: —
Seen in PCSX2: The GIF path's `Handlers[3]` for registers 0x60–0x62, guarded by `handler >= 0x60 && handler < 0x63`, became `Handlers[0x100-0x60]` filled with `RegHandlerUNMAPPED` through macros, guarded only by `handler >= 0x60`. The handlers became `__fastcall`, and the path-1 check in `incTag` was dropped. 32550ef6a 2009-09 “Removed two redundant/unneeded conditional checks from GIFpath processing”

### pcsx2-044 — Skip a whole run of no-op iterations arithmetically
Lens: algorithms · Source: PCSX2
Principle: When a loop's iterations only advance pointers and counters unless some rare item is present, check up front that the item is absent and the data is all there. Then advance by the total in one step instead of iterating.
Why it is faster: —
Applies elsewhere: protocol parsers skipping fixed-size payloads without inspecting them, run-length fast-forwarding in decoders, a tokenizer bulk-skipping whitespace with `memchr`.
Watch out: —
Seen in PCSX2: The MTGS dummy GIF transfer (`gifTransferDummy`) walked every register of every loop to find A+D writes that might raise GS interrupts. The tag now records `numregs` and `hasADreg`. If there's no A+D register and `numregs*nloop <= size`, `nloopOpt` calls `incTag` twice to skip all remaining loops and sets `nloop = 0`. A follow-up reverted this because it broke games. 8234604dc 2009-09 “MTGS: Optimization for gsTransferDummy to reduce idle loops”

### pcsx2-045 — Pack and unpack bitfields with lane-parallel shifts
Lens: simd · Source: PCSX2
Principle: Converting packed formats (5:5:5:1 to 8:8:8:8 and back) element by element in scalar code needs a dozen shifts and masks per element. Do the same shift/mask/OR sequence on whole vector registers, so every lane converts at once with no branches.
Why it is faster: —
Applies elsewhere: RGB565/RGBA5551 conversion in image libraries, bit-packed column decoding in Parquet/ORC readers, network header field extraction with SIMD.
Watch out: —
Seen in PCSX2: The EE MMI instructions PPAC5/PEXT5 used to fall back to the interpreter. Now they are recompiled as SSE2 sequences of `PSLLD`/`PSRLD`/`PSRLW`/`POR` on two temporary XMM registers that pack or unpack the 1:5:5:5 fields of four words at once. The same series added recompiled versions of other MMI opcodes and SSSE3 detection. 33d5c66ac 2009-02 “This update adds recompiling for several MMI opcodes … and adds SSSE3 detection” · b6f3f8e41 2009-03 “MMI: Added recompiled version of PPAC5/PEXT5, and optimized PADDUW a bit”

### pcsx2-046 — Evaluate transcendentals with SIMD polynomials instead of x87
Lens: simd · Source: PCSX2
Principle: Switching between legacy x87 and SIMD state is costly, and x87 instructions are slow. Approximate sine, exp, atan and similar functions with fixed polynomial coefficients evaluated in SSE registers, using aligned broadcast constant vectors.
Why it is faster: —
Applies elsewhere: fast-math libraries (SLEEF, Cephes-style polynomials), audio DSP oscillators, game-engine approximations of `sin`/`exp`.
Watch out: —
Seen in PCSX2: microVU's EFU ops (EATAN, EEXP, ESIN and others) use `PCSX2_ALIGNED16` coefficient vectors such as `mVU_S2..S5` and `mVU_E1..E6` with SSE multiply-add chains. The commit says they were written this way mainly to avoid x87 instructions, because MMX registers were in use for storage. Loads from VF0 emit `XORPS` instead of a memory load. 93b85af33 2009-03 “implemented all EFU opcodes … mainly did it to avoid using x87 FPU instructions”

### pcsx2-047 — Put expensive accuracy work behind tiered settings
Lens: other · Source: PCSX2
Principle: Some correctness work, such as extra clamping, exact cycle accounting or register caching that was assumed to be faster, costs a lot and matters only for a few workloads. Measure it, turn it off by default when it is slower, and offer graded levels so users can trade accuracy for speed.
Why it is faster: —
Applies elsewhere: video encoder presets, database isolation levels, floating-point strict versus fast modes, image-resampling quality settings.
Watch out: —
Seen in PCSX2: Clamping added to `iVUmicroUpper.cpp` runs only under `CHECK_VU_EXTRA_OVERFLOW` because it slowed games down. superVU's `SUPERVU_X86CACHING`, labeled “faster”, was turned off after measurement showed it was much slower (Persona 4, Tekken 5). A five-level VU cycle-stealing slider (`VUCycleHack`) speeds up 3D geometry at increasing risk. af89808f8 2009-02 “Disabled a superVU optimization that slowed down VU intense games lots” · 1f01b4583 2009-02 “calmping code added in r579 is enabled only in "extra" setting or above, because it slows down some games” · 4d2adcae9 2009-04 “New speed hack mainly targeting 3D geometry”

### pcsx2-048 — Pick the fastest compression level for interactive snapshots
Lens: io · Source: PCSX2
Principle: When the user is waiting for a save to finish, compression time matters far more than a few percent of file size. Set the compressor to its fastest level.
Why it is faster: —
Applies elsewhere: `gzip -1`/zstd level 1 for logs and checkpoints, LZ4 for VM snapshots, fast compression for HTTP on dynamic responses.
Watch out: —
Seen in PCSX2: Savestate writing calls `gzsetparams(m_file, Z_BEST_SPEED, Z_DEFAULT_STRATEGY)` after opening the gzip stream, a “big speedup for when making savestates”. e9d722ba1 2009-03 “Set zlib to use fast compression for savestate”

### pcsx2-049 — Park idle worker threads on a semaphore instead of spinning
Lens: concurrency · Source: PCSX2
Principle: A thread that spins while waiting for work burns a whole core and competes with the threads that are doing useful work, especially when there are fewer cores than threads. Blocking on a semaphore hands the core back to the OS until work arrives. A single short spin is still worth keeping on the coordinating thread when the wait is nearly always tiny.
Why it is faster: —
Applies elsewhere: thread-pool workers that block on a condition variable instead of polling a queue; embedded RTOS tasks that pend on a semaphore rather than loop on a flag.
Watch out: —
Seen in PCSX2: The GSdx software rasterizer used to have 2–6 worker threads spin-waiting for scanline jobs. `GSRasterizerMT` now waits on `sem_t m_semaphore` and signals completion through a shared `m_finished` semaphore. The author kept one spinning finish on the main (MTGS) thread behind `UseSpinningFinish`, noting it "leaves one spinwait in the rasterizer, but that's still worlds better than 2-6 spinning threads like before." 97041701a 2009-12 “GSdx: SW rasterizer converted to use pthreads semaphores in the place of spinwaits…”

### pcsx2-050 — Wake a blocked producer only after a useful amount of room frees up
Lens: concurrency · Source: PCSX2
Principle: If a producer outruns its consumer, it should sleep. But waking it for every freed slot means a context switch per item. Have the producer record how much room it needs. The consumer counts down and signals once, when that much is free. For very short waits, spinning is cheaper than a sleep/wake round trip.
Why it is faster: —
Applies elsewhere: a bounded queue with a low-water-mark wakeup; a network send path that unblocks a writer once the socket buffer has drained a set number of bytes, not on every ACK.
Watch out: —
Seen in PCSX2: In the MTGS ring buffer, the EE (producer) thread sets `m_SignalRingPosition` to the amount it needs and `m_SignalRingEnable`, then waits on `m_sem_OnRingReset`. The GS thread does `AtomicExchangeSub(m_SignalRingPosition, ringposinc)` and posts the semaphore once the count reaches zero. In `RestartRingbuffer`, if the shortfall is ≤ 0x80 quadwords, the EE just `SpinWait()`s instead of sleeping. 2f99d8c51 2009-11 “MTGS Optimization! Implemented EEcore thread sleeping and signaling…” · b288e82b9 2009-11 “Couple more MTGS sync fixes, and some final optimizations.”

### pcsx2-051 — Tune how far a pipelined producer may run ahead
Lens: concurrency · Source: PCSX2
Principle: A producer/consumer pipeline needs enough queued work to hide jitter on both sides. Too little lookahead and the stages take turns waiting for each other. Too much and latency grows and the working set leaves the cache. Measure, and pick the smallest depth that keeps both sides busy.
Why it is faster: —
Applies elsewhere: how many frames a game or browser compositor buffers ahead (input lag vs. stutter); prefetch depth in a streaming decoder; the number of in-flight requests in an HTTP client pipeline.
Watch out: —
Seen in PCSX2: The MTGS ring buffer once let the EE queue up to 8 frames, which caused input lag. The redesign capped it at 2. A later fix found an off-by-one that allowed only 1 frame of lookahead, so the two threads were stalling on each other: `if( AtomicIncrement(m_QueuedFrameCount) == 0 ) return;` became `< 2`. 0433a3872 2009-11 “(experimental) Partial implementation of a new model MTGS…” · ae4248131 2009-11 “More MTGS optimizations.” · a6b3acb5d 2010-07 “ReorderingMTGS: Minor performance tweak. MTGS was only queuing 1 frame ahead instead of 2.”

### pcsx2-052 — Power-of-two ring buffer with masked indices
Lens: memory-layout · Source: PCSX2
Principle: If a ring buffer's size is a power of two, wrapping an index is just `(pos + n) & mask`. There is no compare-and-branch, no special restart packet, and no stall while the reader catches up to the start. The buffer stays continuously in use.
Why it is faster: —
Applies elsewhere: lock-free SPSC queues (LMAX Disruptor, Linux kfifo), audio callback buffers, UART receive buffers in firmware.
Watch out: —
Seen in PCSX2: The first MTGS redesign forced a wrap ("restart") back to the start every other frame, partly so that the front of the buffer, likely still in L2 cache, got reused. The ReorderingMTGS replaced the restart mechanism with "a truly free-flowing wrapping mechanism" over `RingBufferSize = 1<<RingBufferSizeFactor` (8 MB) and `RingBufferMask = RingBufferSize - 1`. This removed overhead from `PrepDataPacket`. 0433a3872 2009-11 “(experimental) Partial implementation of a new model MTGS…” · a9084741b 2010-07 “ReorderingMTGS: * Implemented GIFPath_CopyTag…”

### pcsx2-053 — Copy while you parse, in one pass
Lens: memory-layout · Source: PCSX2
Principle: If data must be both validated and moved, a parse pass followed by a copy pass reads the input twice. Do the copy inside the parse loop instead. Each cache line is touched once, and the copy can use wide moves because the parser already knows the record boundaries.
Why it is faster: —
Applies elsewhere: a network stack that computes the checksum while copying the packet; a JSON or CSV reader that unescapes into the output buffer as it tokenizes; memcpy-with-CRC in DMA drivers.
Watch out: —
Seen in PCSX2: GIF packets were run through `GIFPath_ParseTag` to find their size and then `memcpy`'d into the MTGS ring. `GIFPath_CopyTag(pathidx, const u128* pMem, size)` now writes each 128-bit quadword into `RingBuffer.m_Ring[ringpos]` with `_mm_store_ps(_mm_loadu_ps(...))` during the parse. The commit calls this a "big speedup over the old parse-then-copy strategy". a9084741b 2010-07 “ReorderingMTGS: * Implemented GIFPath_CopyTag, which performs a "copy-in-place" while parsing tags…”

### pcsx2-054 — Stream large one-shot writes past the cache, then fence once
Lens: simd · Source: PCSX2
Principle: When you write a large buffer that this core won't read again soon, non-temporal stores avoid evicting useful cache lines and skip the read-for-ownership. Small copies should still use normal stores, because the consumer will probably read them while they are still in cache. One store fence after a batch of streaming copies is enough before handing the data to another thread.
Why it is faster: —
Applies elsewhere: framebuffer or texture uploads; zero-filling large allocations; logging into a big ring that only a background thread reads.
Watch out: —
Seen in PCSX2: `memcpy_amd_qwc` keeps cached moves for small copies ("small copies should be cached (definite speedup)") and uses `prefetchnta` + `movntq` for 32-byte blocks. It deliberately leaves out the fence and adds `Threading::StoreFence()` (`sfence`) "so that many memcpys can be issued in a row more efficiently". In zzogl, the 16→32-bit texture resolve was ported to intrinsics with `_mm_stream_si128` to "reduce cache pollution", and an `_mm_sfence()` was added afterwards. 934578c8f 2010-07 “ReorderingMTGS: Added a new optimized memcpy_amd_qwc, for use by GIFpath copies…” · 3a1ef55cb 2010-09 “GregMiscellaneous: zzogl-pg: * Port more ASM to intrinsics. Note use non-cacheable store…” · 703927056 2010-09 “… use a fence after non temporal move…”

### pcsx2-055 — Prefer inlinable intrinsics over opaque asm or generated routines
Lens: compiler-build · Source: PCSX2
Principle: The compiler can't inline a hand-written assembly function or a pointer to runtime-generated code. Every call pays for the call/return and for spilling registers around it, and constants can't be propagated in. The same logic written with intrinsics can be inlined, specialized for known sizes and register-allocated with the caller.
Why it is faster: —
Applies elsewhere: replacing an `extern "C"` asm memcmp with SSE/NEON intrinsics in a hash table probe; using `__builtin_memcpy` with constant sizes so the compiler can emit inline moves.
Watch out: —
Seen in PCSX2: The newVif block cache compared keys through a JIT-generated function, `nVifMemCmp`. It switched to an inline `_mm_movemask_ps(_mm_cmpeq_epi32(_mm_load_si128(a), _mm_load_si128(b))) & 0x7` ("about 1-2% faster"). `memcpy_vibes` also got an SSE intrinsic version, a Duff's-device `switch` over `_mm_store_ps/_mm_load_ps`, that "is able to inline fully (no call/ret overhead)". c5b47530f 2009-12 “newVif: minor optimizations…” · 383c58ba3 2010-07 “Added a third alternative for memcpy_vibes. This one uses SSE intrinsics and is able to inline fully…”

### pcsx2-056 — Turn runtime mode flags into template parameters
Lens: compiler-build · Source: PCSX2
Principle: When a hot loop checks a few configuration values on every iteration, and those values don't change during the loop, make them template parameters. Dispatch once through a table of instantiations. Each instance then has its dead branches removed, its constants folded, and often fully unrolled inner loops.
Why it is faster: —
Applies elsewhere: image filters specialized per pixel format; a renderer's shader-permutation table; a codec with separate kernels for each bit depth, chosen once per stream.
Watch out: —
Seen in PCSX2: The VIF unpack interpreter became `template<bool doMask, int size, class T>` routines reached through `VIFfuncTable[v->cmd & 0x1f]`. SPU2-X merged its linear and cubic voice paths into `template<int InterpType> GetVoiceValues`, picked by a switch. The GIFpath copy became `template<CpuExtType CpuExt, int pathidx> CopyTag`, so the aligned SSE loads are chosen at compile time. Later, the VIF globals `vif`/`vifRegs` were replaced by a templated `vifIdx` ("simpler and faster!"). zzogl's `Resolve_32b_to_32b` is templated on its page table and conversion function, with the inner loop unrolled. f34f3ac0c 2009-12 “* Significant optimizations to the VIFunpack interpreter (employs templated maskmode and cyclesize constants)…” · cc4b78d7a 2010-03 “SPU2-X: Optimized interpolation to use a switch-template-based lookup table.” · 43cd55980 2010-07 “ReorderingMTGS: Templated the GIFPath parsers…” · 2f8f86a3e 2010-09 “Significant VIFunpack retooling…” · 28819c209 2010-09 “… speed resolve 32bits functions…”

### pcsx2-057 — Search outward from the most likely position, and cache the answer
Lens: algorithms · Source: PCSX2
Principle: When a brute-force search target usually sits near a known point, start there and move outward in both directions, instead of sweeping from one end. Expected cost drops from proportional to the range to proportional to the typical offset. Persist the result under a compact key so the search only ever runs once per input.
Why it is faster: —
Applies elsewhere: finding a sync marker in a media file near its expected offset; bisect-style probing around the last known good value; caching a font or ICC lookup by a hash of the file path.
Watch out: —
Seen in PCSX2: `FindLayer1Start` looked for the DVD layer-break "CD001" descriptor by scanning from 25% to 75% of the image. It now begins at `midsector = (iso->blocks/2) & ~0xf` and reads `midsector-deviation` and `midsector+deviation` on each step, because "most images have the layer break quite close to the middle". The result is cached in `LayerBreakCache.ini` under `HashTools::Hash(filename)`. c2368da81 2009-11 “Layer break search works on an inside->out path now (should be a good bit faster for most images).” · fedbb42a9 2009-11 “* Cdvd Layer break cache uses hashed strings instead of the literal filename…”

### pcsx2-058 — Emit trivial operations inline instead of calling a generic handler
Lens: jit-interpreters · Source: PCSX2
Principle: If a frequently executed operation is only a few loads, tests and stores, generating that code directly avoids flushing cached registers, the function call itself, and any forced exit from the fast path. Keep the generic call only for the rare complex cases.
Why it is faster: —
Applies elsewhere: a bytecode VM that inlines `GETFIELD` on monomorphic shapes; a template engine that compiles simple property lookups to direct accesses instead of going through a resolver.
Watch out: —
Seen in PCSX2: `recDI` (Disable Interrupts) used to `iFlushCall(0)`, end the block's cycle window and call `Interp::DI`. It now emits `xMOV eax, Status; xTEST eax, 0x20006; …; xAND eax, ~0x10000; xMOV Status, eax` inline. The commit reports "a significant speedup in Suikoden 3 menu's." e2933b842 2009-11 “Disable Interrupt is now recompiled, thanks to pseudonym :p This is a significant speedup in Suikoden 3 menu's…”

### pcsx2-059 — Generate nothing for operations that do nothing
Lens: jit-interpreters · Source: PCSX2
Principle: A code generator that faithfully translates no-ops still pays for register flushes, calls and the instruction bytes. Spot instructions whose effect is empty, such as a canonical NOP or a call to an empty function, and emit nothing. Account only for their cost in the timing model.
Why it is faster: —
Applies elsewhere: a query planner that drops `WHERE 1=1`; a shader compiler that removes multiply-by-one; a JS engine that compiles away calls to empty functions after inlining.
Watch out: —
Seen in PCSX2: `recCACHE` flushed every register and called the interpreter's `CACHE`, which is an empty function. That code is now commented out ("Suikoden 3 uses it a lot"). The EE recompiler also checks `cpuRegs.code == 0x00000000` and only adds cycles, skipping `opcode.recompile()` for NOPs, which share an opcode table entry with `SLL`. 7bf9a0d99 2010-01 “Disable recCACHE, since it calls an empty interpreter function only. Small speedup in Suikoden 3.” · 2edc49552 2010-02 “Added a check for NOPs in the eerec…”

### pcsx2-060 — Detect provably idle loops and fast-forward time
Lens: jit-interpreters · Source: PCSX2
Principle: A polling loop whose body leaves the machine state unchanged each time round will keep spinning until some external event happens. So after one iteration you can jump the clock straight to the next scheduled event instead of simulating every iteration. The check is static: the body reads only registers it doesn't write (other than constants or loads) and has no side effects.
Why it is faster: —
Applies elsewhere: discrete-event simulators that skip idle intervals; an OS tickless idle that sleeps until the next timer; game servers that skip ticks for zones with no players.
Watch out: —
Seen in PCSX2: When a block branches back to its own start, the IOP recompiler's block epilogue (with `s_nBlockFF` set) clamps `psxRegs.cycle` to `g_psxNextBranchCycle` using `xCMOVNS`, instead of adding just the block's cycles. A later commit replaced the hard-coded 0x81FC0 kernel address check with general "constant loop detection": the loop is scanned so that "as long as a loop doesn't write to a register it's already read … it will do the same thing on every iteration". 7c817d97e 2010-04 “Extended "BIFCO" speed hack to the typical IOP idle loop…” · 8da2dc7df 2010-04 “R5900: Replaced 0x81FC0 address check with constant loop detection logic…”

### pcsx2-061 — Tune the time-slice and event granularity of cooperative units
Lens: scheduling · Source: PCSX2
Principle: When several simulated or cooperative components share one thread, the slice length sets the trade-off. Long slices waste work, because a unit keeps running after it should have yielded or synced. Very short timers make the scheduler's own overhead dominate. Profile the real workload and set each quantum on its own.
Why it is faster: —
Applies elsewhere: coroutine schedulers choosing how many steps to run before yielding; game loops deciding physics substep sizes; an interrupt-coalescing delay in a NIC driver.
Watch out: —
Seen in PCSX2: In `ExecuteBlockJIT`, a VU kicked from recompiled EE code now runs 128 cycles per call instead of 1024 ("a big speedup in Ratchet and Clank"). In the other direction, several DMA and GIF interrupts that used to fire after 8 or 16 cycles were moved to 128 (`CPU_INT(channel, 128)`) "so they don't interrupt so often (when they don't really need to)". 05ce8afd6 2010-02 “Comment by ramapcsx2: In ExecuteBlockJIT(), set the running time lower for a big speedup in Ratchet and Clank…” · c59022cc6 2010-06 “- Optimized the ParseTag function a little… - Also adjusted some internal interrupt timers…”

### pcsx2-062 — Complete trivially short asynchronous work synchronously
Lens: scheduling · Source: PCSX2
Principle: Putting an event on a timed queue costs an insertion, a later dispatch and extra bookkeeping. When nothing can observe the delay, do the work or raise the completion at once, and the scheduler has less to do.
Why it is faster: —
Applies elsewhere: a promise library that resolves already-available values synchronously; a disk driver that completes cache-hit reads inline instead of posting a completion; HTTP handlers that answer cached responses without going through the job queue.
Watch out: —
Seen in PCSX2: Defining `SIO_INLINE_IRQS` makes pad and memory-card serial interrupts fire inline (`SIO_FORCEINLINE __forceinline`) rather than as timed IOP events, "to free the IOP event handler of some considerable load". The comment says games are highly unlikely to need timed IRQs there. 9d2699529 2010-02 “Enable the SIO_INLINE_IRQS define, to free some load from the IOP event handler.”

### pcsx2-063 — Advance unobserved state with a cheaper stand-in, in coarse steps, or not at all
Lens: laziness · Source: PCSX2
Principle: If an entity's output is discarded but some of its side effects are still visible, write a stripped-down update that keeps only those effects and skips the expensive computation. Step it at the coarsest granularity that still produces the visible effects, and skip it completely when you can prove it is stuck in a steady state.
Why it is faster: —
Applies elsewhere: off-screen game entities that tick at reduced rate with simplified AI; muted audio tracks that still advance their play position; background tabs that throttle timers.
Watch out: —
Seen in PCSX2: SPU2-X's "free-running" muted voices still have to move their address, loop flags and ENDX bits. `GetNextDataDummy` does that without decoding ADPCM samples. A follow-up advances 4 samples at a time (`vc.SCurrent += 4; vc.SP -= 16384`), and the last change skips the update entirely when `(vc.LoopFlags & 3) == 3` and the voice loops in place. The author notes that speed was then "within measuring error" of the version before the feature. 155c96c0a 2010-02 “SPU2-X: - Pseudonym made the voice free-run system a bit faster…” · a564d4bba 2010-02 “Runaway voices should be faster with this…” · ea766db4c 2010-02 “I guess I should've tried this from the beginning…”

### pcsx2-064 — Size a translation cache to the working set, and let it grow
Lens: jit-interpreters · Source: PCSX2
Principle: If a code or result cache is smaller than the hot working set, it thrashes: entries are evicted and rebuilt over and over, and rebuilding costs far more than looking up. Make the default large enough for real workloads. Then grow on demand up to a cap, rather than flushing everything when it fills.
Why it is faster: —
Applies elsewhere: JVM/V8 code-cache sizing; database buffer pools; a CDN edge cache sized to the hot-object set.
Watch out: —
Seen in PCSX2: VU0's microVU cache went from 5 MB to 8 MB because it "fixes constant recompilation problems in Ratchet and Clank". Soon after, `mVUresizeCache` was added. When `x86end` is reached, mVU `SysMmapEx`s a bigger block, up to `mVUcacheMaxSize` (100 MB for VU1, 50 MB for VU0). The dispatchers moved into their own `dispCache` so they survive a resize. e8f546849 2010-06 “Increase vu0 dynarec cache to 8mb, fixes constant recompilation problems in Ratchet and Clank…” · 8ae1e91f7 2010-06 “microVU: Now supports growable rec-cache…”

### pcsx2-065 — Bucket a cache by its lookup key and keep a last-hit shortcut
Lens: algorithms · Source: PCSX2
Principle: One global pool of cached entries means every lookup scans unrelated candidates. Split the pool by the key you already have, such as an entry address, and keep a direct pointer to the most recent hit in each bucket. Repeated lookups become O(1), and misses scan only the right bucket.
Why it is faster: —
Applies elsewhere: inline caches in dynamic-language VMs; per-route memoization in a web router; a TLB-style "last translation" register in front of a page-table walk.
Watch out: —
Seen in PCSX2: microVU's cache of compiled micro-programs was one fixed array searched as a whole, with a `mVUsortProg` step that moved the hit to the front. It became `mVU->prog.prog[startPC/8]` (a `deque<microProgram*>` per start PC), plus `mVU->prog.quick[startPC/8]`, which holds the last matching `{block, prog}`. Hits are `push_front`'d. The commit says God of War "don't constantly recompile", and memory use fell by about 170 MB. fe915cfa6 2010-03 “mVU: work-in-progress cache and memory model rewrite…” · 69fd9fb39 2010-03 “microVU: Finished most of the caching/memory model rewrite…”

### pcsx2-066 — Canonicalize don't-care fields before hashing or comparing keys
Lens: algorithms · Source: PCSX2
Principle: If a cache key contains fields that don't matter in some configurations, or uninitialized padding, logically equal keys hash and compare as different. You get false misses and duplicate entries. Zero the irrelevant fields, and zero-initialize key structs, so equal meaning gives equal bytes.
Why it is faster: —
Applies elsewhere: normalizing query-string order before HTTP caching; `memset` on structs used as hash-map keys in C; stripping volatile headers before computing an ETag.
Watch out: —
Seen in PCSX2: The newVif block key stored `vifRegs->mask` even when masking was off, and "games leave random junk values here which cause false recblock cache misses". It now stores `doMask ? vifRegs->mask : 0`. In microVU, a new `microBlockLink` is now `memzero`'d on creation "to avoid false cache misses", because blocks are matched with raw word compares. c5b47530f 2009-12 “newVif: minor optimizations. * Improved hashing slightly by ignoring the garbage values in the 'mask' parameter…” · 12dbebcfd 2010-09 “microVU: clear contents of microBlock on creation to avoid false cache misses.”

### pcsx2-067 — Compare fixed-layout keys as a few wide words
Lens: simd · Source: PCSX2
Principle: Comparing a key byte by byte, field by field, or with a general `memcmp` costs many branches and length handling. Lay the key out as an aligned block (pad it if you must), and compare it with one or two 32/64-bit compares or a single SIMD compare plus a mask.
Why it is faster: —
Applies elsewhere: Swiss-table style hash maps comparing 16 control bytes at once; IPv6 address matching in a router; packed tuple keys in a columnar database.
Watch out: —
Seen in PCSX2: `nVifBlock` was rearranged into a 32-byte `__aligned16`, padded struct, so the hash probe can compare its first 16 bytes with one SSE compare, first through generated code and then inline. microVU's `microRegInfo` became a union with `u32 simple32[2]`, so block lookup uses "two 32 bit compares for fast block compares, instead of 6-7 individual u8 compares". 7b2c66e54 2009-12 “newVif: created a custom memcmp for the vif dynarec block finder…” · c5b47530f 2009-12 “newVif: minor optimizations…” · 76d639ff3 2010-09 “microVU: * Remove need for packed structs through use of unions…”

### pcsx2-068 — Don't save and restore registers nobody keeps live
Lens: jit-interpreters · Source: PCSX2
Principle: Defensive save/restore around calls, such as freezing whole register files, adds memory traffic to every crossing between generated and native code. Once the generator guarantees it flushes its state before such calls, the saves are dead weight and can go.
Why it is faster: —
Applies elsewhere: trimming callee-saved register pushes in hand-written ISR prologues; leaving FPU/SIMD context out of a lightweight context switch when the thread never used it (lazy FPU switching).
Watch out: —
Seen in PCSX2: Plugin and GS calls used to `MMXRegisters::Freeze()`/`Thaw()`. Once the recompilers no longer kept EE data in MMX registers, the freezes were dropped, taking Soul Calibur 3 from 40 to 48 fps. A week later, COP0/COP2 calls got proper `iFlushCall(FLUSH_INTERPRETER)`, and all XMM freezes were removed too. `memcpy_amd_` also stopped backing up MMX/XMM registers to static buffers. 1c898ad5f 2010-06 “Since we're not using MMX registers to hold EE data in the recompilers anymore…” · c06cb7b12 2010-07 “(speedup!) Add missing flushes to COP0 and COP2… and subsequently disable *all* XMM freezes…” · 402f19413 2010-07 “Remove MMX register freezes from MemcpyFast.”

### pcsx2-069 — Emit tighter instruction sequences in generated code
Lens: jit-interpreters · Source: PCSX2
Principle: Generated code runs millions of times, so a few saved instructions per operation add up. Shorter code also leaves more of the instruction cache for everything else. Look for cheaper identities, such as XOR-based sign tests and arithmetic shifts in place of compare-and-mask, short jumps when targets are near, and base+displacement addressing instead of recomputing pointers.
Why it is faster: —
Applies elsewhere: peephole passes in compilers; hand-tuning a hot hash function's inner loop; a regex JIT using short branches for local jumps.
Watch out: —
Seen in PCSX2: PADDSW's saturation went from a PSRLD/PCMPEQD/PANDN sequence to PXOR sign tests plus `PSRAD 31`, and QFSRV was compacted ("Big difference in the opening fatal frame fmv … probably cache related"). SLT replaced an `xSmartJump` with `xForwardJump8`. MFIR lost an instruction. The VPU unpack recompiler switched to "displaced memory offsets (1-2% speedup)". e27208035 2010-05 “R5900 recompiler: more compact code generation for QFSRV and PADDSW…” · 5a7508d43 2010-07 “* Removed a completely unnecessary use of xSmartJump in the SLT instruction generation…” · 9b896dda0 2010-07 “microVU: … one less x86 instruction for MFIR.” · b27b89b16 2009-12 “newVif: … Optimized codegen of the VPU recompiler using displaced memory offsets…”

### pcsx2-070 — Use newer ISA instructions when the CPU has them, with fallbacks
Lens: simd · Source: PCSX2
Principle: Newer instruction sets often replace a shuffle-then-store sequence, or a blend built from AND/ANDN/OR, with one instruction. Check CPU features once at startup, emit or call the better path where it exists, and keep the baseline path for older hardware.
Why it is faster: —
Applies elsewhere: glibc's ifunc-selected `memcpy`/`strlen`; image libraries with AVX2/NEON kernels; database scans that use popcount/BMI2 when present.
Watch out: —
Seen in PCSX2: microVU's partial-vector store `mVUsaveReg` uses `SSE4_EXTRACTPS_XMM_to_M32` to store a single lane directly when `x86caps.hasStreamingSIMD4Extensions` is set, and otherwise falls back to `PSHUFD` + `MOVSS`. The VIF unpack recompiler and interpreter also gained "a lot of sse4.1 optimizations". 1d338c3a6 2010-06 “microVU: untested sse4 optimizations…” · a5272f8dc 2009-12 “Wrote a vif 'unpack' packet recompiler…”

### pcsx2-071 — Compile hot data-transform configurations into native kernels
Lens: jit-interpreters · Source: PCSX2
Principle: A generic data-conversion routine branches on format, mask and stride for every element. When the same few configurations come up again and again, generate a native kernel for each one the first time it is seen, cache it under a key built from the configuration, and from then on just call it.
Why it is faster: —
Applies elsewhere: JIT-compiled pixel-format converters (Pixman, Skia); compiled regexes; serialization libraries that generate a codec per schema at runtime.
Watch out: —
Seen in PCSX2: VIF "unpack" commands expand packed data into VU memory under many combinations of type, mask, mode and cycle settings. `dVifUnpack` builds an `nVifBlock` key from `{num, upkType, mode, scl, cl, wl, mask}`, looks it up in a `HashBucket<_hSize = 0x4000>`, and compiles a new SSE routine on a miss. The routines use fixed register roles such as `xmmRow` and `xmmCol0..3`. a5272f8dc 2009-12 “Wrote a vif 'unpack' packet recompiler…” · b27b89b16 2009-12 “newVif: I'm a terrible person…”

### pcsx2-072 — Skip preserving state that is almost never read again
Lens: jit-interpreters · Source: PCSX2
Principle: When a translator can't see what a successor will need, such as after an indirect jump, the safe choice is to materialize all pending state, and that can be expensive. Where the chance of that state being read is negligible, for example two blocks later, skip the materialization behind an opt-in switch.
Why it is faster: —
Applies elsewhere: speculative optimizations in JS engines guarded by deopt; relaxed consistency for analytics counters; skipping a full cache flush when the next consumer is known not to read it.
Watch out: —
Seen in PCSX2: A VU's status/MAC/clip flags can have up to 4 live instances. For a block ending in `JR/JALR`, microVU sets `needExactMatch |= 7` and shuffles all instances into place. The optional "mVU Block Hack" (`CHECK_VU_BLOCKHACK`) skips that shuffle when the indirect jump comes in the block after a branch. Together with a flag-propagation fix, the commit reports 66→73 fps on a God of War title screen. 25e63c243 2010-06 “microVU: - Fixed some logic for flag propagation. Its a 3% speedup… - Added a new mVU speedhack…”

### pcsx2-073 — Stop compiling at an invalid instruction to avoid recompile loops
Lens: jit-interpreters · Source: PCSX2
Principle: If translating a region fails or gets thrown away, every visit triggers another translation attempt. A path that is decoded often but never taken can then cost a full compile each time. Compile the region up to the bad point, emit a stub that reports the error only if it actually runs, and cache that result like any other.
Why it is faster: —
Applies elsewhere: negative caching of failed DNS or template lookups; a JIT that caches a "bailout" stub instead of retrying a method that failed to compile.
Watch out: —
Seen in PCSX2: In Dawn of Mana, a branch led into illegal opcodes and the block kept being recompiled. `mVUcheckBadOp` now ends the block (`isEOB`) when its first instruction is illegal, and `handleBadOp` emits a call to `mVUbadOp0/1` that prints an error and exits only if the block runs. This "solves the constant recompilation". 856968812 2010-06 “Better fix for Dawn of Mana problem; solves the constant recompilation as well…”

### pcsx2-074 — Give high-churn small objects their own heap
Lens: allocation · Source: PCSX2
Principle: Many small, short-lived allocations, such as strings and list nodes, fragment a shared process heap, and the heap lock becomes a point of contention between threads. Sending each category to its own private heap or allocator keeps their churn apart and spreads the lock traffic.
Why it is faster: —
Applies elsewhere: Windows `HeapCreate` per subsystem; jemalloc arenas per thread; a std::allocator that routes a container type to a pooled allocator.
Watch out: —
Seen in PCSX2: wxWidgets string buffers (`wxCharBuffer`, `wxMemoryBuffer`), `std::vector`-backed arrays and `std::list`-backed lists were switched to `_allocHeap_wxString`/`_allocHeap_wxObject` and `wxStringAllocator<T>`/`wxObjectAllocator<T>`. PCSX2's event handler queues were then moved to the wxObject allocator too, citing reduced heap fragmentation and "multithread performance". 6ec36fa37 2010-05 “wxWidgets/MSW: Applied my custom string/object heap allocators to several more wxWidgets containers…” · 9e7394aff 2010-05 “* Set the PCSX2 event handler queues to use the wxObject allocator…”

### pcsx2-075 — Bulk-load records into preallocated blocks with a pre-sized index, off the startup path
Lens: allocation · Source: PCSX2
Principle: Loading thousands of records with one growing vector means repeated reallocate-and-copy. Each element also lands at an index that must be stored and resolved again later. Allocate fixed-size blocks of records, keep stable pointers into them, and size the hash index up front so it never rehashes during the load. If the data isn't needed straight away, load it on a worker thread so the load overlaps UI startup.
Why it is faster: —
Applies elsewhere: arena-allocated AST nodes in a parser; game asset tables loaded into slabs; `reserve()` on an `unordered_map` before importing a CSV; IDEs that index the project after the window opens.
Watch out: —
Seen in PCSX2: GameDB stored `gList.push_back(game)` with `gHash[id] = index`. `BaseGameDatabaseImpl` now `malloc`s blocks of `m_GamesPerBlock = 256` `Game_Data`, constructs records in place via `createNewGame(id)`, and keeps a `gHash(9400)` that maps directly to `Game_Data*`. That hash is a `google::dense_hash_map`-based `pxDictionary` introduced just before. The commit fixes "long loading times of the database", and also "added a thread at startup to load the gamedb in the background while PCSX2 is opening windows and printing to the console". 24ae022a7 2010-06 “Fully implement hash-based database lookup…” · 00ec82019 2010-06 “GameDB: Fix long loading times of the database, by using a very efficient block-based allocation system…”

### pcsx2-076 — Reuse a per-thread scratch buffer for string formatting
Lens: allocation · Source: PCSX2
Principle: If each formatted message allocates and frees a temporary string, the formatting cost is mostly in the heap, and threads contend on it. A thread-local buffer that grows only when needed removes nearly all allocations and needs no locking.
Why it is faster: —
Applies elsewhere: logging libraries such as spdlog/fmt using per-thread memory buffers; a JSON serializer reusing a `thread_local` output buffer; Go's `sync.Pool` of byte buffers.
Watch out: —
Seen in PCSX2: `FastFormatAscii`/`FastFormatUnicode` (used through `pxsFmt`) became classes whose workspaces come from TLS `SafeArray` buffers. The commit calls the result "a fully concurrent printf with zero malloc/free overhead", applied first to console logging. e5f87e068 2010-08 “Changed the FastFormatString functions into nifty little classes that use TLS for their buffer workspaces…”

### pcsx2-077 — Large write buffers and the fastest compression level for snapshots
Lens: io · Source: PCSX2
Principle: Many small writes cost a syscall each and can fragment the file on disk. A large user-space buffer sends far fewer, larger writes. For data that is saved often and is rarely space-critical, the fastest compression level gives up a little size for a lot of CPU time.
Why it is faster: —
Applies elsewhere: `setvbuf` with a big buffer for log files; zstd level 1 for checkpoints; a database WAL writing aligned large chunks.
Watch out: —
Seen in PCSX2: The gzip savestate writer now calls `gzsetparams(m_gzfp, Z_BEST_SPEED, Z_FILTERED)` and `gzbuffer(m_gzfp, 0x100000)`, a 1 MB buffer. This fixed heavy NTFS fragmentation and was also "a small speedup". Loading uses the same 1 MB buffer. f2c11e23a 2010-03 “Tweaks to the savestate system that fix some heavy file fragmentation on NTFS in windows…”

### pcsx2-078 — Make disabled logging cost nothing, arguments included
Lens: other · Source: PCSX2
Principle: A log call that is disabled at runtime still evaluates its arguments, which may format strings or dump structures, before it bails out. Have the compiler remove the whole expression with a constant-false short circuit, and order runtime checks so the cheapest master switch is tested first.
Why it is faster: —
Applies elsewhere: `if (log.isDebugEnabled())` guards in Java; Rust's `log!` macros checking the level before formatting; `#define DBG(...) (0 && printf(__VA_ARGS__))` in firmware.
Watch out: —
Seen in PCSX2: `IConsoleWriter::Write*` were changed to return `bool`, so release builds can define `DevCon` as `0&&ConsoleWriter_Null` and the call disappears entirely. A later regression came from `GIF_LOG(... tag.DumpRegsToString() ...)` still evaluating its arguments in Release. It was fixed with `macTrace(trace) = SysTraceActive(trace) && SysTrace.trace.Write`. Register trace hooks now test `EmuConfig.Trace.Enabled` and `m_EnableAll` before the per-category flag. Per-block `PreBlockCheck` calls in debug builds became opt-in through the ini. 04c86ea6d 2009-11 “Console logging performance tweaks…” · 1dd9f8d0d 2010-10 “Fix for some general slowness in Release builds…” · 14513cecb 2010-09 “Fix for some major slowdowns present in dev/debug modes when using trace logging features.” · 4352a3927 2010-07 “… Also sped up debug mode execution a bit by removing some per-block recompiled code overhead…”

### pcsx2-079 — Place fixed-size memory at static addresses the compiler can see
Lens: memory-layout · Source: PCSX2
Principle: Reaching emulated or pooled memory through separately allocated global pointers costs an extra load per access. It also hides sizes and aliasing from the optimizer. Put fixed-size regions in one struct at a static or once-set address, and bind hardware views as references to fixed offsets. Addresses then become constants, and templates can work out buffer sizes for themselves.
Why it is faster: —
Applies elsewhere: embedded firmware mapping peripherals as `volatile` structs at fixed addresses; a game engine's single static arena for subsystem state; replacing many `malloc`'d globals with one aligned struct.
Watch out: —
Seen in PCSX2: The EE's `psM`/`psR`/`psS`/`psH` pointers were merged into one `eeMem` struct (`eeMem->HW[0xD000]` and so on), described as "improves compiler optimization … allows templated code to deduce the size of memory buffers automatically." The IOP got the same treatment: `psxM/psxH` became `iopMem`/`iopHw`, and DMA registers became `static dma_mbc& hw_dma0 = (dma_mbc&) iopHw[0x1080];` in place of pointer-deref macros. a8e406523 2010-08 “Improved EE/VTLB memory management: Removes various psM/psR/psS/psH pointers…” · 36d150358 2010-09 “Converted IOP to use a static/global hardware register allocation…”

### pcsx2-080 — Use whole bytes and words instead of bitfields for hot fields
Lens: memory-layout · Source: PCSX2
Principle: Every read of a bitfield member compiles to a load, shift and mask, and every write to a read-modify-write. If a field is already byte- or word-aligned, declaring it as a plain `uint8_t`/`uint32_t` lets the compiler use a single move.
Why it is faster: —
Applies elsewhere: network header structs where fields are naturally byte-aligned; ECS component layouts; GPU constant buffers.
Watch out: —
Seen in PCSX2: GSdx's GS register unions declared fields like `uint32 R:8; uint32 G:8; uint32 B:8;`, `uint32 SIGID:32` and `uint64 REGS:64`. These became `uint8 R, G, B; uint8 _PAD1[5];` and plain `uint32`/`uint64`, as part of a "minor optimization" to VertexKick/DrawingKick and the packed-register handlers. 645188e0e 2010-08 “GSdx: Minor optimization and some code simplifications relating to VertexKick/DrawingKick…”

### pcsx2-081 — Reserve address space for sparse tables and commit pages on first touch
Lens: memory-layout · Source: PCSX2
Principle: A huge, sparsely used lookup table doesn't need physical memory for the parts nobody uses. Reserve the whole virtual range so indexing stays a plain array access. Commit pages only when they are touched, either in a page-fault handler or with an explicit commit, and keep a bitmap of committed blocks.
Why it is faster: —
Applies elsewhere: sparse arrays in V8/JVM heaps; databases mapping a large file with demand paging; hash tables with lazily committed bucket arrays.
Watch out: —
Seen in PCSX2: `SpatialArrayReserve` builds on the new `BaseVirtualMemoryReserve`. Its `DoCommitAndProtect` is called from `OnPageFaultEvent`, it tracks committed blocks in a bit array, and `TryResize` can grow or shrink the reservation. It was introduced for "recompiler lookup tables (large portions of these tables are never used by many games)". Applying it to the R5900 recompiler "reduces the recompiler's lookup tables from ~40mb to a mere 1-3mb." dc7f00d05 2010-11 “newHostVM: * Preliminary implementation for the SpatialArrayReserve class…” · d6de2e394 2010-11 “newHostVM: Made use of the new SpatialArrayReserve for the EE/R5900 recompiler…”

### pcsx2-082 — Age out idle pooled resources
Lens: gpu-rendering · Source: PCSX2
Principle: An object pool avoids the cost of creating objects, but without a limit it keeps every object ever made and pins memory (or VRAM) after a spike. Stamp each pooled object with when it was last used, and each frame or tick free the oldest ones once the pool is over a size threshold and they have sat idle for N ticks.
Why it is faster: —
Applies elsewhere: database connection pools with idle timeouts; GPU staging-buffer pools in game engines; thread pools that let surplus workers exit.
Watch out: —
Seen in PCSX2: `GSDevice::AgePool()` runs once per frame: `m_frame++; while (m_pool.size() > 20 && m_frame - m_pool.back()->last_frame_used > 10) { delete m_pool.back(); m_pool.pop_back(); }`. Each texture records `last_frame_used` when it is recycled into the pool. ab77eb464 2010-05 “GSDx: Free textures that have been in the unused texture pool for a few frames…”

### pcsx2-083 — Size buffers to the content actually seen, not the worst case
Lens: gpu-rendering · Source: PCSX2
Principle: Allocating every buffer at the largest size you might ever need multiplies fill, clear and bandwidth costs for the common smaller case. Start from a typical size and grow only when real input needs more.
Why it is faster: —
Applies elsewhere: canvases sized to the visible viewport times the device pixel ratio; dynamic arrays that start small; video decoders allocating surfaces at the stream's real resolution.
Watch out: —
Seen in PCSX2: GSdx's upscaler created render targets at `1024 * m_upscale_multiplier` in both dimensions. It now starts at `640x512 * multiplier` and raises `m_width/m_height` only when `GetDisplayRect() * multiplier` is larger, which "reduced the overhead of the new scaler". efdeabd86 2009-11 “GSdx: - Reduced the overhead of the new scaler. Instead of just using 1024*1024 blindly, it tries to adapt to games native resolutions…”

### pcsx2-084 — Pass non-trivial objects by const reference and pre-increment iterators
Lens: compiler-build · Source: PCSX2
Principle: Passing a class with a constructor by value, or post-incrementing an iterator, can create temporaries that a weak optimizer won't remove. Passing `const&` and using `++it` express the same intent without the copies.
Why it is faster: —
Applies elsewhere: C++ code taking `const std::string&` parameters; passing large structs by pointer in C; Rust borrowing instead of cloning.
Watch out: —
Seen in PCSX2: microVU helpers such as `mVUallocSFLAGa(x32 reg, …)` became `(const x32& reg, …)`, and `xRegisterSSE::GetInstance(id)` returns a reference from a static table. The commit says MSVC 2008 "can't optimize constructors to save its life". mVU's deque loops changed to `++it` "in-case the compiler doesn't already optimize these out". zzogl's `SetKickVertex(VertexGPU*, Vertex v, …)` now takes `Vertex&` to "avoid useless copy on the stack". 641c3b56c 2010-07 “microVU: avoid making unnecessary copies or instances of xRegister classes…” · b0ec1394d 2010-07 “microVU: minor changes to list traversing…” · 02efc340e 2010-10 “GregMiscellaneous: zzogl-pg: * copy some vertex instead of re compute them twice * pass vertex as reference…”

### pcsx2-085 — Power-of-two sizes turn modulo into a mask
Lens: algorithms · Source: PCSX2
Principle: Integer division and modulo by a value only known at runtime take tens of cycles. If the divisor is a compile-time power of two, `i % N` becomes `i & (N-1)`. Round circular buffers and work-partition counts up to a power of two when you can.
Why it is faster: —
Applies elsewhere: hash table capacities that are powers of two; striping work across 2ⁿ threads by `id & mask`; audio delay lines.
Watch out: —
Seen in PCSX2: zzogl's `gsvertex` circular buffer grew from 3 to 4 entries, with the note "ARRAY_SIZE(gsvertex) == 2^n => modulo is replaced by an and instruction". GSdx's rasterizer added an `IsOneOfMyScanlines` option, `UseConstThreadCount`, which uses `(scanline & ThreadMaskConst) == m_id` to remove "a lot of non-const div/modulus ops" when the thread count is 1, 2, 4 or 8. 3c6dd04c1 2010-10 “GregMiscellaneous: zzogl-pg: * use a size of 4 for the gsvertex array (allow compiler to optimize a % into an and)…” · 97041701a 2009-12 “GSdx: SW rasterizer converted to use pthreads semaphores…”

### pcsx2-086 — Derive repeated outputs by copying, not recomputing
Lens: algorithms · Source: PCSX2
Principle: If several outputs share most of their values with ones already computed, such as corners of a quad or shared vertices of a strip, copy the finished results and patch the few fields that differ. Don't run the full transform again for each.
Why it is faster: —
Applies elsewhere: mesh builders reusing shared vertices; spreadsheet engines copying a computed row and adjusting one cell; CSS layout reusing measurements for identical siblings.
Watch out: —
Seen in PCSX2: zzogl's sprite kick used to call `SetKickVertex` six times. It now converts two vertices and sets `p[3] = p[0]; p[2] = p[0]; p[4] = p[1]; p[5] = p[1];` before adjusting x coordinates. A follow-up keeps `gsTriFanVertex` so that strips and fans reuse the previously computed vertex. 02efc340e 2010-10 “GregMiscellaneous: zzogl-pg: * copy some vertex instead of re compute them twice…” · 3c6dd04c1 2010-10 “… Reuse previous computed vertex for _strip & _fan primitives”

### pcsx2-087 — Move 128-bit data with 128-bit loads and stores
Lens: simd · Source: PCSX2
Principle: Copying 16-byte records as four 32-bit words, or as two 64-bit halves, costs four (or two) times the instructions of one SSE move, and can cause partial-register stalls. Align the data and use a single 128-bit load/store per record.
Why it is faster: —
Applies elsewhere: copying UUIDs or 128-bit hashes with SSE/NEON; packet descriptors in DPDK; pixel blocks in image codecs.
Watch out: —
Seen in PCSX2: IPU FIFO reads and writes replaced `for (i=0;i<=3;i++) data[writepos+i] = pMem[i];` with `CopyQWC(&data[writepos], pMem)`. The MFIFO hardware-register handlers changed from two `psHu64` copies to `CopyQWC(out, &psHu128(VIF0_FIFO))`. zzogl's `update_4pixels_sse2` was redone to "Do 128 bits transfer instead of 32bits." 1c75440a6 2010-09 “IPU optimizations -- use SSE for FIFO reads/writes…” · 20adde44a 2010-08 “PCSX2/EEcore: * Now using SSE for all hardware register reads and writes…” · 28d04d4fb 2010-09 “GregMiscellaneous: zzogl-pg: * Redo update_4pixels_sse2. Do 128 bits transfer instead of 32bits…”

### pcsx2-088 — Fold a scalar fix-up pass into the SIMD pass
Lens: simd · Source: PCSX2
Principle: If a vectorized kernel is followed by a scalar loop that sets a default value in every element, the data is walked twice and the second walk is slow. Build the constant in a vector register, in-register if possible, and merge it in the SIMD pass. Keep scalar code only for the exceptions.
Why it is faster: —
Applies elsewhere: setting alpha to 0xFF during an RGB→RGBA SIMD conversion; zero-extending while widening; OR-ing flag bits into packed records as they are stored.
Watch out: —
Seen in PCSX2: The IPU's colour conversion ran a separate loop `p[3] = p[7] = p[11] = p[15] = 0x80` after the SSE yuv2rgb. The SSE path now generates `0x80` in `xmm7` (`pcmpeqb; psllw 15; psrlw 8; packuswb`) and writes the alpha there. The scalar loop only sets the `0x40` threshold exceptions. a98072a3c 2010-05 “IPU: include default alpha in SSE code for maybe a small speed gain.”

### pcsx2-089 — Precompute derived values at decode time
Lens: other · Source: PCSX2
Principle: If a hot loop keeps re-deriving a value from a header field (with a test, a mask or a count), compute it once when the header is decoded and store it beside the header. The same applies to expensive constant resources: build them once.
Why it is faster: —
Applies elsewhere: storing parsed content-length beside a request header; caching the computed stride of an image row; creating placeholder bitmaps once at startup.
Watch out: —
Seen in PCSX2: GIFpath stored `numregs = ((tag.NREG-1)&0xf) + 1` once per tag and now uses that "precached value for numregs in place of a convoluted test against NREG". `SetTag` also counts `DetectE` (A+D register entries) once per tag, so the REGLIST/PACKED paths only take the slow path when it is non-zero. In GSDumpGUI, the "No Image" bitmap "is created once". 56d3982dc 2010-07 “* Minor optimization to GIFpath, by utilizing the precached value for numregs…” · c59022cc6 2010-06 “- Optimized the ParseTag function a little…” · b7f3b93ec 2010-03 “GSDXGSDumpGUI : Big optimization… Now the Bitmap of "No Image" is created once.”

### pcsx2-090 — Drop redundant masking when the address width already bounds it
Lens: branching-fast-paths · Source: PCSX2
Principle: If an address or index is already limited by its type or by the lookup that follows, an extra `& mask` with a conditional choice of mask adds a compare, a select and an AND to every access for nothing. Remove it, and mark small hot helpers to be inlined in release builds.
Why it is faster: —
Applies elsewhere: removing double bounds checks in a bytecode interpreter after verification; letting a `uint8_t` index wrap naturally rather than masking.
Watch out: —
Seen in PCSX2: The VU interpreter's `_vuLQ/_vuSQ/_vuLQI/…` computed `addr = (...)*16 & (VU == &VU1 ? 0x3fff : 0xfff)` before calling `GET_VU_MEM`, which masks anyway. The ternary mask was removed, which also fixed VU0's view of the VU1 registers, and the functions became `__ri` (release-inline). 17df6e218 2010-08 “VU interpreters: removed redundant VU memory masking… and improved inlining a little bit.”

### pcsx2-091 — Tell the OS scheduler that you are a real-time multimedia app
Lens: platform · Source: PCSX2
Principle: Desktop compositors and schedulers treat ordinary processes as background work. Opting into multimedia-class scheduling, where the platform offers it, can cut presentation jitter for apps that render every frame.
Why it is faster: —
Applies elsewhere: `AvSetMmThreadCharacteristics("Games")` for audio/render threads on Windows; `SCHED_FIFO` or rtkit on Linux audio; QoS classes such as `userInteractive` on macOS.
Watch out: —
Seen in PCSX2: On Windows Vista and later, `SetupDwmStuff` loads `dwmapi.dll` and calls `DwmEnableMMCSS(TRUE)` at startup, "telling the DWM the application requires multimedia-class scheduling for smooth display". The goal was less windowed-mode slowdown under Aero, and the author left the measuring to users. ce31ac361 2009-11 “Implemented a feature which could potentially reduce the slight slowdowns present when running in windowed mode inside Aero…”

### pcsx2-092 — Don't build position-independent code where nothing needs it
Lens: compiler-build · Source: PCSX2
Principle: On 32-bit x86, PIC reserves a register for the GOT pointer and sends global accesses through an extra indirection. On a register-starved ISA that is a real cost. If the modules never need to be relocatable shared objects, leave PIC off.
Why it is faster: —
Applies elsewhere: building static, non-PIE binaries for hot-path tools on i386; using `-fvisibility=hidden` to avoid PLT/GOT indirection in shared libraries.
Watch out: —
Seen in PCSX2: The CMake build removed `-fPIC` from all plugins and cleared `CMAKE_SHARED_LIBRARY_C/CXX_FLAGS`, with the comment "we only support x86 architecture. And last but not least it impact the performance." Some `__forceinline` variadic functions had to drop the qualifier as a result. f8163336b 2010-07 “[cmake] remove -fPIC. Expect a little speed up…”

### pcsx2-093 — Forward-declare instead of including heavy headers
Lens: compiler-build · Source: PCSX2
Principle: Every header pulled into a widely included file gets parsed again in every translation unit. Replacing includes with forward declarations and `extern` declarations, wherever only pointers or references are used, cuts rebuild times.
Why it is faster: —
Applies elsewhere: the pimpl idiom in C++ libraries; `<iosfwd>` instead of `<iostream>`; TypeScript `import type` to trim emitted dependencies.
Watch out: —
Seen in PCSX2: A header cleanup removed `<wx/gdicmn.h>`, `<algorithm>` and `<string>` from common headers. They were replaced with `class wxPoint; class wxRect; class wxSize;` and `extern const wxSize wxDefaultSize;`, which "should speed up recompilation times by a few percents". e96dbe443 2010-06 “Major header file house cleaning! Should speed up recompilation times by a few percents…”

### pcsx2-094 — Per-site cache for indirect jump targets
Lens: jit-interpreters · Source: PCSX2
Principle: An indirect jump has to look up where its target lives on every execution. Most indirect jumps go to the same few targets over and over. Give each jump site a small table, indexed by target address, that remembers the resolved entry point, and check it against a cheap validity token before using it. The full search then runs only on a miss or after an invalidation.
Why it is faster: —
Applies elsewhere: inline caches for virtual or dynamic method dispatch in JS and Smalltalk VMs, per-route handler caches in a web router, memoised `dlsym` lookups.
Watch out: —
Seen in PCSX2: Before this change, microVU's JR/JALR called `mVUsearchProg` on every execution. Each block that ends in an indirect jump now allocates `microJumpCache[mProgSize/2]`. `mVUcompileJIT` returns `jc.x86ptrStart` immediately when `jc.prog == mVU->prog.quick[startPC/8].prog`, so the cached entry is used only if the micro-program that owns it is still current. The check stays valid until VIF writes micro memory again. 835127b69 2010-12 “microVU: Implemented indirect jump address caching (speedup)”

### pcsx2-095 — Push rare-case work into the shared slow path
Lens: jit-interpreters · Source: PCSX2
Principle: When code is generated inline at thousands of sites, every byte on the fast path is copied that many times and competes for instruction cache. Work that only the slow path needs, such as fixing up a result after a fallback call, belongs inside the shared out-of-line stub. That keeps every inline copy short.
Why it is faster: —
Applies elsewhere: put error formatting and retry logic in a `noinline` cold function, emit the GC write barrier's slow path as one shared stub, or normalise a value only in the cache-miss branch of a memoising wrapper.
Watch out: —
Seen in PCSX2: For 8- and 16-bit loads, the EE recompiler's `vtlb_DynGenRead32` emitted a `MOVSX`/`MOVZX` after every load. The commit generates sign- and zero-extending variants of the indirect (slow-path) dispatcher in `vtlb_dynarec_init` (7×64-byte slots instead of 5×64), so the extension now happens only after the handler call, inside the shared dispatcher. 3286f0079 2011-07 “EE: very minor VTLB optimisation.”

### pcsx2-096 — Don't spill state for a feature that isn't implemented
Lens: jit-interpreters · Source: PCSX2
Principle: Writing every cached register back to memory before an operation that "might fault" costs stores and later reloads on every memory access. If the fault path doesn't actually exist yet, or never needs that state, flush only what the call really clobbers. Keep values in registers across the access.
Why it is faster: —
Applies elsewhere: skip saving a full context before calls that can't throw, or avoid `volatile` and memory barriers around code that never runs concurrently.
Watch out: —
Seen in PCSX2: `FLUSH_EXCEPTION` in `iCore.h` went from `0x7ff` to `0x1ff` (“not supported yet, so disabled for a small speedup”). A day later `recLoad64`/`recLoad32` in `iR5900LoadStore.cpp` dropped `iFlushCall(FLUSH_EXCEPTION)` and `_deleteEEreg` for non-constant addresses, and added specialised load/store-with-shift sequences. 8e2887343 2011-05 “Small PCSX2 speedup due to less flushing in the eerec.” · 3c51d6655 2011-05 “EERec: Implemented some common cases of Load/Stores with shifts.”

### pcsx2-097 — Compare floats as integers when the ordering matches
Lens: simd · Source: PCSX2
Principle: IEEE floats of the same sign order the same way as their bit patterns read as signed integers. Integer compares are cheaper and run on more ports than FP compares, especially compared with widening to double. With a sign fix-up you get min and max from integer SIMD compares and blends.
Why it is faster: —
Applies elsewhere: radix-sorting floats by flipping bits, branch-free float clamps in shaders or DSP code, ordering float keys inside a database index.
Watch out: —
Seen in PCSX2: microVU's `MIN_MAX_PS` used to widen ZW and XY to doubles (masking and OR-ing in exponent bits) and compare them as doubles. It now uses integer comparisons, which take less code and are faster “especially on AMD cpus”. 996ca8fff 2011-06 “microVU: Reimplemented mini/max opcodes using integer comparison operations instead of double comparison.”

### pcsx2-098 — Sorted flat array with memmove and range erase
Lens: allocation · Source: PCSX2
Principle: A generic container can do far more work than the access pattern needs. Erasing items one by one means O(n) shifts per erase, and growth means element-wise copies. A purpose-built sorted array with binary search, one `memmove` to open a gap, and a single range erase for a contiguous run turns O(k·n) into O(n).
Why it is faster: —
Applies elsewhere: a timer wheel or interval list kept in a sorted vector, an LSM memtable flush, or deleting a contiguous run of rows from an in-memory table with one splice.
Watch out: —
Seen in PCSX2: `BaseBlocks` in `BaseblockEx.h` replaced `std::vector<BASEBLOCKEX>` with `BaseBlockArray`, which does binary-search insert, `memmove` to make a hole, and `erase(first,last)`. `psxRecClearMem` and the EE equivalent now count the blocks to drop and remove the whole range at once instead of calling `Remove(blockidx)` in a loop. Compile time for some blocks fell from nearly 1 s to 0.1–0.3 s. cfe376491 2011-04 “Patch by Firnis: He wrote his own, faster replacement for std::vector and used it for the EE and IOP block managers.”

### pcsx2-099 — Precompute ownership instead of dividing per query
Lens: lookup-tables · Source: PCSX2
Principle: A modulo by a runtime value compiles to a hardware divide, which costs tens of cycles. If the answer depends only on a small index and on parameters that rarely change, fill a byte table whenever the parameters change. Each query then becomes one load.
Why it is faster: —
Applies elsewhere: shard-to-worker maps in a server, round-robin slot tables, and precomputed "which core owns this tile" arrays in a tiled renderer.
Watch out: —
Seen in PCSX2: `GSRasterizer::IsOneOfMyScanlines` used to compute `((scanline >> THREAD_HEIGHT) % m_threads) == m_id`. It now returns `m_myscanline[scanline >> THREAD_HEIGHT] != 0`. `SetThreadId` rebuilds the table only when the id or thread count changes. Multi-threaded mode got 5–10% faster. d20da5f26 2011-03 “GSdx: 5-10% speedup in multi-threaded mode, replaced that modulo operator with a lookup table.”

### pcsx2-100 — Power-of-two ring sizes
Lens: algorithms · Source: PCSX2
Principle: If a circular buffer's capacity is a compile-time power of two, `% N` on an unsigned index compiles to an AND, and indices can wrap without branches. Round the capacity up and keep the logical length separate if needed.
Why it is faster: —
Applies elsewhere: lock-free queues (LMAX Disruptor), moving-average windows, audio delay lines, hash tables masked with `& (cap-1)`.
Watch out: —
Seen in PCSX2: SPU2-X's time-stretcher running average (`addToAvg` in `Timestretcher.cpp`) changed `MAX_STRETCH_AVERAGE_LEN` from 100 to 128 and made `nextAvgPos` unsigned, “so the compiler can optimize 'modulo' as 'and'”. A comment proposing this had gone in the day before. 44d8301bb 2013-09 “spu2x: array out of bond (-1)” · c2acd8023 2013-09 “spu2x: correct the modulo parameter of the Timestretcher.”

### pcsx2-101 — One reciprocal instead of many divides
Lens: algorithms · Source: PCSX2
Principle: Several divisions by the same denominator can become one reciprocal and several multiplies. Several divisions by related denominators can often be rewritten around a single shared quantity, such as a cross product. Horizontal SIMD instructions then collapse the per-lane leftovers without shuffles.
Why it is faster: —
Applies elsewhere: normalising a vector with `inv = 1/len` and three multiplies, perspective divide in a software rasteriser, or dividing a column of values by the same total.
Watch out: —
Seen in PCSX2: `GSRasterizer::DrawTriangle` used to divide each edge delta by its own `dy` (`dv[i] / dv[i].p.yyyy()`) in a three-way switch. It now computes the triangle's cross product once, takes `cross.rcpnr()` (a reciprocal with a Newton–Raphson step), and derives the gradients from it. A follow-up commit uses SSE3 `hsubps` in the same setup. 47713eee8 2011-03 “GSdx: Replaced a few divs with something more obscure in DrawTriangle” · 686b6da8e 2011-03 “GSdx: finally, some use for hsubps (SSE3).”

### pcsx2-102 — Hoist per-draw constants out of per-pixel code
Lens: laziness · Source: PCSX2
Principle: Anything that depends only on per-batch parameters should be computed once when the batch starts, not inside the per-element kernel. The kernel then just loads the precomputed values.
Why it is faster: —
Applies elsewhere: compute shader uniforms on the CPU instead of in every fragment, build a regex once outside a loop, or resolve config lookups before a request loop starts.
Watch out: —
Seen in PCSX2: When mipmapping uses a constant LOD (`sel.mmin && sel.lcm`), `GSDrawScanline::BeginDraw` now computes the clamped `uv_minmax` once per draw. The JIT-generated scanline code only `vmovdqa`-loads it, instead of shifting and unpacking `t.minmax` for every span (a long-standing “TODO: precalc”). cc2824112 2011-04 “GSdx: just a small mipmapping optimization when lod is constant, it was already a TODO.”

### pcsx2-103 — Detect invariants early and pick the cheaper variant
Lens: laziness · Source: PCSX2
Principle: When a check made before dispatch proves a general feature has no effect, such as a value clamped to a constant, flip the selector bits so the specialised kernel without that feature runs. This pays off most when kernels are generated per feature combination.
Why it is faster: —
Applies elsewhere: a query planner skipping a sort when the input is already ordered, an image library using a nearest-neighbour path at scale 1.0, a JSON encoder taking the ASCII fast path when no escapes are present.
Watch out: —
Seen in PCSX2: In `GSRendererSW::GetScanlineGlobalData`, if the vertex trace's minimum LOD is already at or above `TEX1.MXL`, the draw now sets `sel.lcm = 1` (constant LOD) and `sel.mmin = 1`, because tri-linear filtering can't have any effect. This happens before the per-pixel LOD path would be selected. The commit notes it makes those draws “half fps maybe” cheaper. d7a6b484c 2011-03 “GSdx: small mipmapping optimization, half fps maybe”

### pcsx2-104 — Turn runtime mode flags into template parameters
Lens: compiler-build · Source: PCSX2
Principle: A loop that tests a mode flag on every iteration does extra work and leaves the compiler unsure which path to optimise. Make the flag a template or compile-time parameter, instantiate every combination, and pick the instance once through a function-pointer table. Each instance is branch-free and fully specialised.
Why it is faster: —
Applies elsewhere: pixel-format converters templated on source and destination format, serializers specialised per field type, `std::sort` with an inlined comparator instead of `qsort`'s function pointer.
Watch out: —
Seen in PCSX2: `GSBlock::ExpandBlock16` became `template<bool AEM>`, replacing an `if(TEXA.AEM)` around two copies of the loop. `GSRendererSW::ConvertVertexBuffer<primclass, tme, fst>` is instantiated for every combination into `m_cvb[P][tme][fst]` and chosen once per draw. 19be60515 2012-02 “GSdx: this should fix xp/wine crashing when extrathreads > 0, and added a sprite drawing shortcut” · 9b8c753ea 2012-01 “GSdx: broken frame skipping should be fixed, and a few random sw renderer optimizations.”

### pcsx2-105 — Dedicated handler for the most frequent input shape
Lens: branching-fast-paths · Source: PCSX2
Principle: Measure which input format dominates, and give it a straight-line handler that decodes the whole record at once instead of going through per-field generic dispatch. Detect the shape once per batch header so the check is amortised.
Why it is faster: —
Applies elsewhere: a protobuf parser fast path for messages laid out in field order, an HTTP parser specialised for `GET` with no body, or a memcpy path for structs that need no endianness conversion.
Watch out: —
Seen in PCSX2: `GIFPath::SetTag` now recognises the register list `0x00040102` with `nreg == 3` (STQ, RGBA, XYZF2) as `TYPE_STQRGBAXYZF2`. `GSState::Transfer` sends such packets to `GIFPackedRegHandlerSTQRGBAXYZF2<prim>`, which builds each vertex with a few SSE loads and shuffles and calls `VertexKick` directly, instead of three per-register handler calls through member-function pointers. The software renderer got a matching shortcut: sprites with no frame or z test whose bounding box lies inside the scissor take a direct rectangle-fill path. 9aabcc170 2012-01 “GSdx: added a shortcut in GSState::Transfer for the most frequent vertex format I found (helps quite a lot)” · 19be60515 2012-02 “…added a sprite drawing shortcut”

### pcsx2-106 — Memoise expensive derived maps by a packed key
Lens: laziness · Source: PCSX2
Principle: If a costly structure is a pure function of a few small parameters, pack those parameters into one integer key and cache the result in a hash map. The key must cover every input that matters. A key that leaves one out forces worst-case assumptions or gives wrong hits.
Why it is faster: —
Applies elsewhere: caching compiled templates keyed by name and locale, glyph atlases keyed by (font, size, codepoint), or tile-to-page maps in a map renderer.
Watch out: —
Seen in PCSX2: `GSLocalMemory::GetPage2TileMap` rebuilt its page-to-tile list (a `map` of `hash_set`s) on every call. It is now cached in `m_p2tmap` under a hash of `TBP0|TBW|PSM|TW`, which made Dragon Quest 8 “fast again”. A later change widened the key to 34 bits of `TEX0` so that TH is included, which removed the hard-coded 1024-line scan. fd0245d36 2011-04 “GSdx: implemented the caching of the new page/tile map, dq8 should be fast again” · 5325f9b49 2011-12 “GSdx: Small optimizations here and there”

### pcsx2-107 — Drop a cache that grows faster than it saves
Lens: memory-layout · Source: PCSX2
Principle: A cache keyed on something with high cardinality, such as arbitrary rectangles, keeps growing, and its memory and hashing cost can exceed the cost of just recomputing. Measure both. If recomputation is cheap, remove the cache and make the computation itself lean.
Why it is faster: —
Applies elsewhere: memoising on full request URLs that include query strings, caching per-user rendered fragments that are never reused, or HTTP caches that never get a hit.
Watch out: —
Seen in PCSX2: `GSOffset::GetPages` kept a `hash_map<uint64, list<uint32>*>` keyed on the rectangle and “was caching a ridiculous amount of data”. The cache was removed. The function now walks the rectangle, uses a bitset to deduplicate, and pushes pages straight into a pre-`reserve`d `vector`, which drops a second pass with `_BitScanForward`. 6f97ca35e 2011-12 “GSdx: GSOffset::GetPages was caching a ridiculous amount of data, it isn't that much slower without it.”

### pcsx2-108 — Caller-provided scratch buffers instead of per-call heap allocation
Lens: allocation · Source: PCSX2
Principle: Once the real work is cheap, the allocator can become the hot spot in a short function that allocates on every call. Let the caller pass a buffer that lives in the object or on the stack, sized for the worst case, and fall back to allocating only when no buffer is given.
Why it is faster: —
Applies elsewhere: `Read(buf)` APIs in Go, `ArrayPool` or `stackalloc` in .NET, reusing a `StringBuilder`, or `char buf[256]` for short formatting on embedded targets.
Watch out: —
Seen in PCSX2: VTune showed that the slowest part of the uncached `GetPages` was `new uint32[]`. `GetPages(rect, pages, bbox)` now writes into `m_tmp_pages[512+1]` in `GSRendererSW`, or into `m_temp` in the texture cache, and the matching `delete []` calls were removed. In zzogl, `FlushDecodeClut` similarly replaced a heap `vector<char>` with an `__aligned16` stack array. 651196f66 2012-01 “GSdx: vtune tells me GSOffset::GetPages is too slow without the cache and its slowest part is new uint32[], lets use pre-allocated buffers then.” · 2d38e8919 2011-01 “zzogl: align_16 array before sse2 instruction + minor stuff”

### pcsx2-109 — Contiguous vectors instead of linked lists
Lens: memory-layout · Source: PCSX2
Principle: A `std::list` pays for one allocation per node and a pointer chase per element. A vector stores the elements contiguously, so iteration is prefetch-friendly, and it can be sorted in place without a temporary copy.
Why it is faster: —
Applies elsewhere: replace a `LinkedList` with an `ArrayList` in Java, store adjacency as arrays (CSR) instead of node lists, or use flat arrays for ECS components.
Watch out: —
Seen in PCSX2: `GSLocalMemory::GetPage2TileMap` switched from `list<GSVector2i>[MAX_PAGES]` to `vector<GSVector2i>` and sorts each page's vector in place instead of sorting a temp vector and splicing it into a list. The same month, `GSOffset::GetPages` started returning a `vector<uint32>` instead of a `list<uint32>`. 5325f9b49 2011-12 “GSdx: Small optimizations here and there” · 6f97ca35e 2011-12 “GSdx: GSOffset::GetPages was caching a ridiculous amount of data…”

### pcsx2-110 — Direct-indexed arrays for small key spaces
Lens: lookup-tables · Source: PCSX2
Principle: When a key has only a few bits, such as a shader-variant selector, a plain array indexed by the key beats a hash map. There is no hashing, no probing and no node allocation, and the lookup is a single load.
Why it is faster: —
Applies elsewhere: opcode tables in an interpreter, state × event tables in a protocol state machine, caches of pipeline objects keyed by a small bitfield.
Watch out: —
Seen in PCSX2: `GSDeviceOGL` replaced `hash_map<uint32, GLuint>` for vertex shaders, geometry shaders and samplers, and `hash_map<uint32, GSDepthStencilOGL*>` for depth-stencil states, with `GLuint m_vs[1<<5]`, `m_gs[1<<3]`, `m_ps_ss[1<<3]` and `m_om_dss[1<<6]`. ca1edbf2c 2013-06 “gsdx ogl: … replace various hash_map by basic array”

### pcsx2-111 — Offload a subsystem to its own thread through an SPSC ring
Lens: concurrency · Source: PCSX2
Principle: If one component of a serial pipeline only consumes commands and rarely needs to send results back, run it on its own core. The producer writes commands into a single-producer, single-consumer ring buffer. Each side owns its index and only reads the other side's index atomically, so there is no lock on the common path. Fill level is computed from the two positions rather than from a shared counter that both sides would update.
Why it is faster: —
Applies elsewhere: an audio thread fed by a lock-free ring, a logging thread that drains a per-thread buffer, a game render thread that consumes a command list.
Watch out: —
Seen in PCSX2: MTVU (`pcsx2/MTVU.h`) runs VU1 on a `VU_Thread` with a 16 MB power-of-two `u32` ring. `read_pos` is modified only by the VU thread, and `write_pos`/`write_offset` only by the EE thread. `ReserveSpace` writes an `MTVU_NULL_PACKET` to wrap, and `WaitOnSize` blocks on `mtxBusy` only when the buffer is full. The commit reports 10–45% gains on 3+ cores. SPU2-X's `SndBuffer` was reworked along the same lines: `volatile m_rpos`/`m_wpos`, and the shared `m_data` count replaced by `_GetApproximateDataInBuffer()`. ac9bf45f9 2011-08 “pcsx2: Implemented Threaded VU1 :D” · e0bbc1b81 2011-08 “SPU2-X: Reworked the circular buffer functions for audio output, with a thread-safe version.”

### pcsx2-112 — Send only the fields the consumer reads
Lens: concurrency · Source: PCSX2
Principle: Every byte you put in a cross-thread queue costs bandwidth and ring capacity, and a full ring stalls the producer. Define a compact message struct that holds only the fields the consumer actually uses, rather than copying the whole state block.
Why it is faster: —
Applies elsewhere: DTOs instead of whole ORM entities in a message queue, sending deltas instead of full game state over the network, worker `postMessage` payloads trimmed to what the worker needs.
Watch out: —
Seen in PCSX2: MTVU copied the whole `VIFregisters` block (about 400 bytes with padding) into the ring for every unpack. `WriteRegs`/`ReadRegs` now move a `VIFregistersMTVU` containing only cycle, mode, num, mask, itop and top, which saves “at least half of the ring buffer space”. da7324dbf 2012-02 “Not transferring unused vif registers to the MTVU thread can save at least half of the ring buffer space.”

### pcsx2-113 — Self-contained jobs, synchronised only on real hazards
Lens: concurrency · Source: PCSX2
Principle: If every job reads shared mutable state, workers have to stop at every boundary. Copy each job's inputs (parameters, vertices, a palette) into the job object so that workers can run behind the producer. Track which resources in-flight jobs touch, with per-page reference counts, and wait only when a new write overlaps something still being read or written.
Why it is faster: —
Applies elsewhere: immutable snapshots handed to background workers, render graphs that insert barriers only on resource conflicts, or databases using MVCC instead of locking everything.
Watch out: —
Seen in PCSX2: In the software renderer, `GSRendererSW::Draw` now `memcpy`s vertices and global data into a `shared_ptr<GSRasterizerData>` and queues it. `Sync()` happens only when the frame/z-buffer offset changes, a texture can't be updated, or invalidated pages are in use, so threads “must be synchronized lot less, 1/10th in average”. Each rasterizer had already been given its own copy of the per-draw globals (`memcpy(&m_global, param, …)`), which brought about 10% in MT mode. Later, `UsePages` refcounts texture and target pages with `_InterlockedIncrement16`, so `InvalidateVideoMem` syncs only if `m_fzb_pages[page] | m_tex_pages[page]`. f318e84ac 2011-12 “GSdx: Better multi-threading for the sw renderer.” · 257d57ba5 2011-02 “Tweaked the rasterizer to be about 10% faster in multi-threaded mode” · eaf06e8b1 2011-12 “GSdx: bit less idle time by refcouting used texture pages.”

### pcsx2-114 — Bin work to the workers whose region it touches
Lens: concurrency · Source: PCSX2
Principle: If work is split by spatial region but every job is broadcast to every worker, each worker spends time rejecting jobs that aren't its own. Compute the job's bounding box once on the producer side and enqueue it only to the workers whose regions intersect it.
Why it is faster: —
Applies elsewhere: tile binning in GPU and software rasterisers, routing events to the shard that owns the key range, or spatial partitioning in physics broadphase.
Watch out: —
Seen in PCSX2: `GSRasterizerList::Queue` computes `bbox = data->bbox.rintersect(data->scissor)` and calls `r->Queue(data)` only if `r->IsOneOfMyScanlines(bbox.top, bbox.bottom)`. The old per-thread reject check at the top of `DrawTriangle` was removed. b86e3ebd1 2011-12 “GSdx: Polished the recent changes a bit. … everything needed is packed and copied for the worker threads”

### pcsx2-115 — Spin while work is flowing, block when idle
Lens: concurrency · Source: PCSX2
Principle: Waking a sleeping thread through the kernel can cost more than a small job takes to run. Workers that spin briefly with `pause` while jobs keep arriving respond fastest. Fall back to a condition variable or event only when the queue stays empty, and measure the trade-off.
Why it is faster: —
Applies elsewhere: adaptive mutexes, `SpinWait` before `WaitOne` in .NET, busy-polling NIC drivers (DPDK) under load, futex-based locks that spin before parking.
Watch out: —
Seen in PCSX2: The old `GSRasterizerMT::ThreadProc` carried the comment “once we are running it is better to spin, jobs can be smaller than the cost of waking up every time”. The rework added Vista-and-later condition variables and SRW locks (with a `GetProcAddress` fallback). It then moved queue filling to its own thread so that a spin loop could become an event, and measured the cost at “about -5% fps”. 2421c68be 2011-12 “GSdx: Saving the conditional var update (vista or better)…” · d5dbe7e7e 2011-12 “GSdx: Moved filling up rendering threads on a new thread, to not block the main… Using events results in about -5% fps”

### pcsx2-116 — Cap how far the producer can run ahead
Lens: concurrency · Source: PCSX2
Principle: An unbounded queue between a fast producer and a slow consumer improves throughput numbers but makes latency grow without limit, so input feels laggy. Keep a hard cap on queued frames even when other throttles are off, and make the cap configurable for people running benchmarks.
Why it is faster: —
Applies elsewhere: bounded channels in Go and Rust, swap-chain "max frames in flight" in Vulkan and D3D, TCP send buffers sized to limit bufferbloat.
Watch out: —
Seen in PCSX2: `SysMtgsThread::PostVsyncEnd` used to skip the `VsyncQueueSize` limit when both vsync and the frame limiter were off. It now always enforces the limit, “at the cost of a few max FPS in benchmarks”, because with software rendering the game became unresponsive to input. a698e2ff5 2011-03 “Forced MTGS to obey the max queued frames limit even when the frame limiter is disabled.”

### pcsx2-117 — Flush batched work only when someone needs the result
Lens: concurrency · Source: PCSX2
Principle: Batching amortises handoff cost, but a forced flush on every partial batch throws that away. Find the one event that actually requires the consumer to be up to date, such as a read-back, and flush only there.
Why it is faster: —
Applies elsewhere: flushing a write-behind cache only before a read of the same key, Nagle-style coalescing until a reply is expected, or `glFlush` only before a read-back.
Watch out: —
Seen in PCSX2: An earlier fix pushed every incomplete GS packet to the MTGS thread. `Gif_Unit::FlushToMTGS()` is now called only when `GS_BUSDIR` switches to GS→EE (a download), and it sends only completed primitives. The commit message: “doing it this way should be faster than flushing every incomplete gs packet”. c918df60d 2011-07 “newGif: do what r4829 did, except only flush the gs primitives to the mtgs when theres going to be a GS download (busdir=1).”

### pcsx2-118 — Have the blocker wake the waiter instead of polling
Lens: laziness · Source: PCSX2
Principle: A waiter that reschedules itself every few ticks to check a condition can spin through thousands of useless wake-ups, each with its own dispatch overhead. Stop rescheduling while the condition is false, and have whoever clears the condition wake the waiter directly.
Why it is faster: —
Applies elsewhere: promises or condition variables instead of `setInterval` polling, epoll instead of a non-blocking read loop, webhooks instead of polling a status API.
Watch out: —
Seen in PCSX2: When VIF1 is waiting on GIF path 3 (`vif1Regs.stat.VGW && gifPath[GIF_PATH_3].state != GIF_PATH_IDLE`), `vif1Interrupt` and `vifMFIFOInterrupt` no longer re-queue `CPU_INT(DMAC_VIF1, …)`. Instead, `gifInterrupt` fires `CPU_INT(DMAC_VIF1, 1)` when path 3 finishes. The loop had been running “over 1000 times”. Reported results: Outrun 2006 went from 43–46 to 45–47 fps and GTA:SA from 63–67 to 66–70. GIF likewise stops re-scheduling itself while path 3 is masked, so the recompiler exits less often. 489bce662 2012-05 “Path 3 games speed up. Stopping VIF looping when waiting for Path 3 to finish” · 098640b06 2012-05 “Path3: Stopped Path3 looping when MskPath3 is enabled” · 64447dd23 2012-06 “Path3 Masking: Optimization got missed during r5277 revert.”

### pcsx2-119 — Shadow GPU state and skip redundant API calls
Lens: gpu-rendering · Source: PCSX2
Principle: Each graphics API call costs driver validation even when it sets the value that is already there. Keep a CPU-side mirror of the bound state, compare against it, and only issue calls that change something.
Why it is faster: —
Applies elsewhere: state caches in WebGL engines (three.js, Babylon), virtual-DOM diffing before touching the real DOM, and skipping redundant register writes on slow embedded buses.
Watch out: —
Seen in PCSX2: zzogl's GLSL4 path added `g_current_texture_bind[unit]`, `g_current_vs`/`g_current_ps` and dirty flags for its uniform buffers, so that `glActiveTexture`/`glBindTexture`, `glUseProgramStages` and buffer uploads happen only on change. The God of War menu trace dropped from 4600 to 3000 calls. GSdx-OGL later moved FBO, draw buffer, viewport, scissor, blend and depth-stencil micro-state into a `GLState` namespace and saved “10,000 opengl call by frame” in Colin McRae 3. 0154639e1 2012-05 “zzogl glsl: optimize state change.” · a46b489a2 2013-08 “gsdx ogl: various minor optimization.”

### pcsx2-120 — Bind context-global resources once
Lens: gpu-rendering · Source: PCSX2
Principle: Some bindings belong to the context rather than to the object you are drawing with. Binding them again on every draw is wasted work. Bind them once at setup and drop the per-draw parameter.
Why it is faster: —
Applies elsewhere: binding a UBO to a fixed binding index once in WebGL2, setting static descriptor sets once in Vulkan, configuring a peripheral's clock once instead of before each transfer.
Watch out: —
Seen in PCSX2: In `GSDeviceOGL`, `glBindBufferBase(GL_UNIFORM_BUFFER, …)` for the merge and interlace constant buffers moved into `Create()`. `VSSetShader`/`PSSetShader`/`StretchRect` lost their per-call constant-buffer argument and the rebind logic with it: “constant buffer are attached to contexts not programs. So do it only once!” 4029cc519 2011-12 “gsdx-ogl: LINUX-ONLY * constant buffer are attached to contexts not programs. So do it only once!”

### pcsx2-121 — Prefer async-capable upload paths to synchronous mapping
Lens: gpu-rendering · Source: PCSX2
Principle: Mapping a GPU buffer can force the CPU to wait until the GPU has finished with it. A copy-in call such as `glBufferSubData` lets the driver stage the data and return straight away. Which path wins depends on the driver, so keep the choice switchable and benchmark it.
Why it is faster: —
Applies elsewhere: `queue.writeBuffer` instead of `mapAsync` in WebGPU for small updates, staging buffers for Vulkan uploads, and double-buffered DMA instead of waiting on a single buffer.
Watch out: —
Seen in PCSX2: `GSUniformBufferOGL::upload` replaced `glMapBufferRange(…INVALIDATE…)`, `memcpy` and unmap with `gl_BufferSubData`, noting that “the call is synchronous whereas glBufferSubData could be asynchronous” (“More fps :)”). A later commit added an `ogl_vertex_subdata` option for vertex buffers, because subdata was faster on FGLRX. 65b5a376e 2013-06 “gsdx-ogl-wnd: … Use glBufferSubData to upload data into the constant buffer instead of map/unmap. More fps :)” · ca1edbf2c 2013-06 “gsdx ogl: … add the new option "ogl_vertex_subdata"”

### pcsx2-122 — Immutable storage so the driver validates once
Lens: gpu-rendering · Source: PCSX2
Principle: If a resource can change shape or format at any time, the driver has to recheck it whenever it is used. Allocating it with a fixed, immutable layout lets the driver validate once at creation.
Why it is faster: —
Applies elsewhere: `texStorage2D` in WebGL2, `Object.freeze`d config objects that JITs can constant-fold, read-only memory mappings.
Watch out: —
Seen in PCSX2: `GSTextureOGL` replaced its per-format `glTexImage2D` branches with a single `glTexStorage2D(m_texture_target, 1, m_format, w, h)` for render targets, depth-stencil and textures: “use immutable texture, it would avoid multiple validation from the driver”. 15b255617 2013-05 “gsdx ogl: use immutable texture, it would avoid multiple validation from the driver”

### pcsx2-123 — Don't allocate an output nobody reads
Lens: gpu-rendering · Source: PCSX2
Principle: A pass that exists only to write depth or stencil doesn't need a colour target. On APIs that allow it, bind no colour target, which saves both the allocation and the bandwidth.
Why it is faster: —
Applies elsewhere: depth-only prepasses and shadow-map passes, `/dev/null`-style sinks instead of buffering output you discard, and `SELECT 1` instead of fetching rows just to check existence.
Watch out: —
Seen in PCSX2: `GSDevice11::SetupDATE` used to `CreateRenderTarget` a temporary texture just so the stencil-setup draw would have a colour target. It now calls `OMSetRenderTargets(NULL, ds)`, since D3D10+ allows that. The D3D9 path was left as it was. 96c8c13ea 2012-07 “GSDX: don't unnecessarily create and use a render target for the DATE setup stage”

### pcsx2-124 — Index shared vertices instead of duplicating them
Lens: gpu-rendering · Source: PCSX2
Principle: Expanding strips and fans into independent triangles copies each shared vertex up to three times. Store every vertex once and emit 32-bit indices instead. That cuts copying and conversion work, and the buffer can be trimmed of unused trailing vertices.
Why it is faster: —
Applies elsewhere: indexed meshes in any 3D engine, string interning and dictionary encoding in columnar formats, and deduplicated tile references in sprite sheets.
Watch out: —
Seen in PCSX2: GSdx's `GSState` gained `m_vertex` (head/tail/stride) and `m_index` buffers. For `GS_TRIANGLESTRIP`, `VertexKick` now writes `buff[0..2] = head+0..2` and advances `head` by one instead of copying three vertices. The buffers grow by 1.5× with a −100 slack, so that skipped vertices never trigger growth inside the hot `VertexKick`. f68f007f0 2012-01 “GSdx: the promised index buffer update, needed a lot of changes”

### pcsx2-125 — One large read instead of a read and seek per record
Lens: io · Source: PCSX2
Principle: Reading a small header and then seeking past each fixed-size record costs one system call per record. Read big chunks, such as 1 MB, and walk the records inside memory, carrying the offset over to the next chunk.
Why it is faster: —
Applies elsewhere: scanning fixed-width logs, building an index over a tar-like archive, or parsing records from flash on an MCU in page-sized reads.
Watch out: —
Seen in PCSX2: `BlockdumpFileReader::Open` built its sector table with a `Read(4)` followed by `SeekI(m_blocksize)` for every block. It now reads 1 MB at a time and pulls each `u32` out at `off += 4 + m_blocksize`, carrying `off -= has` into the next chunk. 187175073 2012-12 “Attempt at making blockdump loading faster, may go either way.”

### pcsx2-126 — Asynchronous, read-ahead disc I/O
Lens: io · Source: PCSX2
Principle: If emulation or rendering waits on a blocking read, the device latency becomes a frame hitch. Split reads into begin and finish, issue them as overlapped I/O, read several sectors per request, serve later sectors from that buffer, and start the next expected read as soon as the current one finishes.
Why it is faster: —
Applies elsewhere: io_uring or IOCP prefetch in databases, video players reading ahead of the decode position, OS readahead tuned by `posix_fadvise(SEQUENTIAL)`.
Watch out: —
Seen in PCSX2: The async-iso branch added an `AsyncFileReader` interface with `BeginRead`/`FinishRead`. `FlatFileReaderWindows` opens files with `FILE_FLAG_SEQUENTIAL_SCAN | FILE_FLAG_OVERLAPPED`, and Linux got a libaio implementation. `InputIsoFile::BeginRead2` reads `ReadUnit` sectors at once and skips the read if `lsn` is already buffered. A follow-up passes `cdvd.nSectors` through a new `notifyReadLength` plugin hook as a read-ahead hint. 568e74f0e 2012-12 “async-iso: Current state of the code.” · d2f3b4a7f 2012-12 “async-iso: First iteration of the experimental read-ahead system.”

### pcsx2-127 — Rename instead of copy for backups
Lens: io · Source: PCSX2
Principle: If the old file is about to be replaced anyway, renaming it to the backup name is a metadata-only operation. A copy has to read and write every byte. The rename is also atomic, which makes it more robust if the process is interrupted.
Why it is faster: —
Applies elsewhere: write a temp file and `rename()` it over the original for config saves, log rotation by rename, atomic deploys by swapping a symlink.
Watch out: —
Seen in PCSX2: `StateCopy_SaveToSlot` replaced `wxCopyFile(file, copy)` with `wxRenameFile(file, copy)` before writing the new savestate. Because the backup became that cheap, it was also turned on by default. dcb7c08a5 2011-08 “SaveState backup: 1. Now enabled by default. 2. Now uses rename instead of copy (faster, more resilient to corruptions)”

### pcsx2-128 — Yield before low-priority background I/O
Lens: concurrency · Source: PCSX2
Principle: A background thread that starts disk work right away competes with the latency-critical thread for the CPU and the I/O path. A short sleep or yield before its non-urgent work hands the scheduler back to the important thread.
Why it is faster: —
Applies elsewhere: `requestIdleCallback` for analytics in a browser, `ionice`/`nice` for backup jobs, deferring flash writes until after a real-time control loop iteration.
Watch out: —
Seen in PCSX2: `SysExecEvent_ZipToDisk::InvokeEvent` in `SysState.cpp` now calls `Yield(4)` before creating and writing the file. The comment explains that file transactions are low priority compared with the VM thread. bd0990f9c 2011-01 “Minor tweak to savestates to help improve threaded responsiveness.”

### pcsx2-129 — Keep logging off hot paths in release builds
Lens: io · Source: PCSX2
Principle: Console output means formatting, locking and often a synchronous write. Called on every iteration of a stall or retry loop, it can cost more than the work itself. Demote those messages to developer-only logging, and test the enabled flag before doing any formatting.
Why it is faster: —
Applies elsewhere: guard debug logs with `if (log.isDebugEnabled())` in Java, use compile-time log levels on embedded targets, and never log inside a tight retry loop in production.
Watch out: —
Seen in PCSX2: `_dmaSPR0`'s “SPR0 stall” message moved from `Console.WriteLn` to `DevCon.Warning`, which fixed a “huge slowdown” in Star Trek Voyager – Elite Force. IPU stall messages in `IPUdma.cpp` were demoted the same way. In `DEV9ghzdrk`, `__Log` now returns immediately when `Log` is 0, and the flag defaults to off. 00132caca 2011-05 “Disabling some console spam … that was causing a huge slowdown in Star Trek Voyager - Elite Force.” · b9bb76474 2013-05 “Moved some stall related logs to DevCon(sole) as games that trigger them will slow down.” · 7a00d19b5 2013-03 “DEV9ghzdrk: Fixed the bad slowness.”

### pcsx2-130 — Reserve address space up front, commit pages on first touch
Lens: memory-layout · Source: PCSX2
Principle: Reserve one large contiguous virtual range at a fixed address up front, but give it physical memory only for the blocks that are actually touched. A page-fault handler commits each block on demand. Code then gets stable addresses it can embed as constants, without paying memory for unused capacity.
Why it is faster: —
Applies elsewhere: JVM and V8 heaps that reserve and commit lazily, sparse arrays backed by `mmap(PROT_NONE)` plus `mprotect`, growable arenas in game engines.
Watch out: —
Seen in PCSX2: The newHostVM branch added `VirtualMemoryReserve` (Reserve/Commit/Reset) and `BaseVmReserveListener`, whose `OnPageFaultEvent` calls `CommitBlocks` for the faulting block. EE, IOP and VU memory and the recompiler caches map at fixed host addresses. Moving the VIF recompilers to `RecompiledCodeReserve` “saves another 4-8mb of memory, depending on game”. be1a59046 2010-11 “newHostVM: … Applied host virtual memory mapping to the EE/IOP/VU main and on-chip memory banks.” · 63b254c8b 2010-11 “newHostVM: Applied the new RecompiledCodeReserve to the VIF recompilers (saves another 4-8mb of memory…)”

### pcsx2-131 — Replace a counting loop with arithmetic
Lens: algorithms · Source: PCSX2
Principle: A loop that just decrements a counter once per fixed-size chunk can be replaced by one division. Keep the loop only for the irregular case that isn't fully understood yet, and log when it runs.
Why it is faster: —
Applies elsewhere: computing how many pages a byte range spans instead of iterating, or computing elapsed periods in a scheduler with `(now - start) / period`.
Watch out: —
Seen in PCSX2: For partial non-fill VIF unpacks, `nVifUnpack` used to loop with `--vifRegs.num` per vector. It now does `vifRegs.num -= (size / vSize)`. The fill case still loops, but also computes a guessed value and prints it for checking. The commit message notes that Guitar Hero sends big partial chunks. 44a1d17c9 2011-04 “Small optimization for partial unpacks, won't make a noticable difference, but every little helps :P”

### pcsx2-132 — Drop checks that rarely pay off
Lens: laziness · Source: PCSX2
Principle: A "compare before copy" or a full-data analysis only pays if it often lets you skip work. When the expected outcome is almost always "go ahead", the check is pure overhead, so remove it. Likewise, in generated code, emit loads and merges only for the parts a mask actually selects.
Why it is faster: —
Applies elsewhere: skipping an `If-None-Match` round-trip for content that always changes, not diffing before writing a file that is always new, avoiding a full-table min/max scan whose result is rarely used.
Watch out: —
Seen in PCSX2: `_vifCode_MPG` stopped calling `memcmp_mmx` on VU micro memory before copying (“likelyhood is that the program isnt there, thats why its copying it. Faster without”), and switched to `memcpy_aligned`, which measured 1 fps faster in Grandia 3. In the same commit, `VifUnpackSSE_Dynarec::SetMasks`/`doMaskWrite` recompute the row, column and write-protect masks so that `xmmRow` and the destination are loaded only when the mask uses them. Separately, GSdx removed `GSClut::GetAlphaMinMax32`'s per-palette alpha scan, since textures were almost never proven opaque that way. f19c0b7ca 2012-08 “VIF: Some optimizations for the VIF Rec, some small clean-up/optimizations for VIF itself.” · 21522e71b 2012-06 “GSDX: Skip checking each palette entry's expanded alpha when deciding whether a texture is fully opaque.”

### pcsx2-133 — Wider vectors and non-destructive SIMD encodings
Lens: simd · Source: PCSX2
Principle: Three-operand VEX encodings remove the register-copy moves that two-operand SSE needs. 256-bit integer vectors (AVX2) process twice the data per instruction. Unrolling so that several independent vectors are in flight helps keep the wider load and store units busy. Build per ISA level and dispatch at runtime.
Why it is faster: —
Applies elsewhere: multiversioned builds (`-march=haswell` plus a generic fallback), simdjson's runtime kernel selection, AVX2 paths in image codecs and in compression libraries such as zstd.
Watch out: —
Seen in PCSX2: The software renderer's JIT (`GSDrawScanlineCodeGenerator`) was converted to VEX encoding, which made it 3–5% faster. Later, `GSVector8i` and an AVX2 build (`_M_SSE = 0x501`) arrived. They upgraded `GSBlock` column read/write and expand routines, which were unrolled to four 256-bit rows per step, and added `GSDrawScanlineCodeGenerator.x86.avx2.cpp`, measured at about +10% when not EE-limited. ca7abd983 2011-02 “Mostly code cleanups, XBYAK 2.99, VEX conversion for the sw renderer (3-5% faster)” · 43c7235cc 2013-06 “GSdx: Added a vector class for AVX2 and upgraded a few functions to use it” · d20bd4f86 2013-06 “GSdx: The sw renderer now uses avx2, not much faster though, +10% maybe”

### pcsx2-134 — Built-in, switchable profiling and repeatable replay benchmarks
Lens: other · Source: PCSX2
Principle: Optimisation needs numbers that are cheap to get and repeatable. Compile an opcode or event histogram behind a macro, so it costs nothing when off, and add a replay mode that runs a recorded workload several times and reports mean and standard deviation. Decisions then rest on data rather than guesses.
Why it is faster: —
Applies elsewhere: `perf`-style counters behind feature flags, recorded-traffic replay for load tests, and benchmark harnesses that report variance (criterion, JMH).
Watch out: —
Seen in PCSX2: microVU gained a `microProfiler`, enabled with `#define mVUprofileProg`, that prints how often each VU opcode is used. The same commit makes the status-flag conversion use EAX/ECX/EDX when called from the EE recompiler, so it plays nicer with that recompiler's register allocation. GSdx's `GSReplay` gained a `linux_replay` loop count and prints mean, standard deviation and per-frame fps. That data drove choices such as the `ogl_vertex_subdata` switch. 5488e082f 2011-10 “microVU: Implemented an opcode profiler which prints statistics on how often VU opcodes are used” · ca1edbf2c 2013-06 “gsdx ogl: … replay: add "linux_replay" option and compute some nice stat (mean, standard deviation)”

### pcsx2-135 — Build expensive resources only when the feature is on
Lens: laziness · Source: PCSX2
Principle: Startup and mode switches pay for every resource built eagerly, even ones that are never used. Check the setting first, and skip the costly build (compiling, loading, parsing) when the feature is off.
Why it is faster: —
Applies elsewhere: load heavy JS modules with dynamic `import()` only when the user opens the feature. Skip compiling optional regexes or templates when their flag is off.
Watch out: —
Seen in PCSX2: `GSDevice11::Create` and `GSDevice9::Create` used to compile the FXAA pixel shader every time. The compile is now wrapped in `if (Use_FXAA_Shader)`, which reads the `Fxaa` config, so switching between software and hardware modes no longer stalls on a shader compile that is slow and usually not needed. 5e22971db 2014-01 “GSDX: New shader is quite slow to compile, so lets not compile it unless the user is using it”

### pcsx2-136 — Keep diagnostics out of release hot paths
Lens: branching-fast-paths · Source: PCSX2
Principle: A check that runs once per item multiplies its cost by the item count, even when it never fires. Put breakpoint checks, debug dumps, console logging and performance counters behind compile-time switches, or demote them to developer-only logging, so release builds don't run them in tight loops.
Why it is faster: —
Applies elsewhere: guard verbose logs with `if (log.isDebugEnabled())`. Compile tracing out of embedded firmware. Rate-limit log lines that a request handler writes.
Watch out: —
Seen in PCSX2: The EE interpreter's `execI()` runs once per emulated instruction. It used to check breakpoints and memchecks and call `debugI()` each time. That code now sits under `#ifdef EXTRA_DEBUG`, which made the interpreter about 2–2.5× faster. Separately, an unimplemented-path message in `SPR.cpp` that printed every frame caused fps drops, and it became a `DevCon` warning. GSdx's `GSPerfMon` was compiled out of Linux release builds (`DISABLE_PERF_MON`), with an expected 5–10% gain. c9aa04c67 2014-11 “core: EE interpreter major speed boost” · 5c6b5374a 2015-02 “Less verbose SPR stall control logging, avoids fps dives in ST:Voyager” · 0663501d6 2015-08 “gsdx-linux: disable perf monitor on release build”

### pcsx2-137 — Prefer the platform memcpy over hand-rolled copy loops
Lens: compiler-build · Source: PCSX2
Principle: Modern C libraries and compilers pick the best copy strategy for the CPU they run on, and they inline copies of constant size. Old hand-tuned assembly copies fall behind on newer chips. Where the buffers can't overlap, use `memcpy` rather than `memmove`.
Why it is faster: —
Applies elsewhere: remove custom SSE copy loops from legacy C++ code. In JS use `TypedArray.set`, and in Java use `System.arraycopy`, instead of element-by-element loops.
Watch out: —
Seen in PCSX2: The `memcpy_fast`/`memcpy_const`/`memcpy_qwc` macros in `MemcpyFast.h` now map to plain `memcpy`, with the size multiplied by 16 for the quadword variants. The AMD-tuned routines were then removed from the core and the plugins. `BaseBlockArray` growth copies into freshly allocated memory, so it switched from `memmove` to `memcpy`. 9168c344b 2014-08 “Use generic memcpy from now on. Tests show generics are good enough now.” · 3f7cb991e 2014-08 “Remove some slow, redundant memcpy implementations” · 0f2f25845 2014-11 “core: use memcpy and reserve function”

### pcsx2-138 — Shrink records to the fields actually consumed
Lens: memory-layout · Source: PCSX2
Principle: Bandwidth scales with record size. When the receiver ignores some components, or treats them as constants, drop them from the wire format and supply them on the receiving side. Then check the alignment, so the struct really gets smaller.
Why it is faster: —
Applies elsewhere: leave constant fields out of JSON API payloads. Send 2D positions instead of 4D vectors to a WebGL shader. Pack sensor telemetry structs on embedded links.
Watch out: —
Seen in PCSX2: Full-screen copy quads (`StretchRect`, `SetupDATE`) sent `GSVertexPT1`, with a vec4 position whose z and w were always 0.5 and 1.0. A new 16-byte `GSVertexPxyT1` (vec2 position plus vec2 texcoord, explicitly 16-aligned so it doesn't pad to 32) halves the upload, and `convert.glsl` fills in z and w itself. 0d45e6d70 2014-04 “gsdx-ogl: avoid to send constant to the GPU”

### pcsx2-139 — Compare against a shadow copy before re-uploading
Lens: laziness · Source: PCSX2
Principle: Keep a copy of the last state you sent, and compare the new state with it (memcmp or a wide SIMD compare) before transferring. When nothing changed, skip the upload. The compare costs far less than the transfer and the driver work it would trigger.
Why it is faster: —
Applies elsewhere: React-style props equality before a re-render. Skip a DB write when the row hash is unchanged. Avoid re-sending identical MQTT retained messages.
Watch out: —
Seen in PCSX2: `GSUniformBufferOGL::cache_upload()` keeps a 32-byte-aligned `cache` of the previous contents and calls `upload()` only if `memcmp` differs. Earlier, the PS constant-buffer cache was trimmed so it skips fields that derive from others (MinMax follows WH and MskFix), and compares `GSVector4i` lanes in place instead of copying them to the stack first. fa15d7fd2 2016-05 “gsdx ogl: generic code to cache uniform buffer” · 6fe9ee387 2014-10 “gsdx-ogl: optimize the PS cb cache”

### pcsx2-140 — Put frequently compared fields together at the front
Lens: memory-layout · Source: PCSX2
Principle: Order struct members so the fields checked on every call sit next to each other and fill the fewest cache lines. Fields that are rarely checked go at the end. The hot compare then touches one contiguous block, which can be a single cache line.
Why it is faster: —
Applies elsewhere: put the key and hash first in hash-table entries. Order ECS component fields by access frequency. Lay out Go or Rust structs so the hot fields share a cache line.
Watch out: —
Seen in PCSX2: `PSConstantBuffer` moved `HalfTexel` and `MinMax` after `MskFix`, so the dirty-check compares only `a[0..3]` (FogColor_AREF, WH, MinF_TA, MskFix) as four contiguous vectors and then copies them with plain stores. 84f844767 2014-10 “gsdx-ogl: micro optimize PSConstantBuffer cache”

### pcsx2-141 — Keep state objects alive across calls so unchanged fields stay unchanged
Lens: laziness · Source: PCSX2
Principle: If you rebuild a state struct from scratch each call, stale or default fields make the dirty-check see changes that aren't real. Keep the struct as a member, update only the fields that are relevant, and preserve the rest (even inside a SIMD word, by blending lanes). Then "unchanged" really means unchanged, and fewer uploads happen.
Why it is faster: —
Applies elsewhere: mutate a retained render-state object rather than allocating a new one per frame. Keep memoized selector inputs stable in Redux. Avoid rewriting peripheral registers whose value is unchanged.
Watch out: —
Seen in PCSX2: `vs_cb` and `ps_cb` became members of `GSRendererOGL` rather than locals in `DrawPrims`, which cut UBO updates by about 10%. Later, the fog-colour write uses `fc.blend32<8>(ps_cb.FogColor_AREF)` so AREF in lane 3 is left alone, and `MskFix`/`MinMax` are written only in region-clamp modes. The same change was ported to the D3D renderer. 4d1241070 2015-08 “gsdx-ogl: latch constant buffer in rendering object” · 37f9bcf9c 2015-08 “gsdx-ogl: reduce state change” · 11bad7b07 2018-08 “GSdx-d3d: Reduce state change - port from GL.”

### pcsx2-142 — Cache decompressed chunks with a size-bounded LRU
Lens: io · Source: PCSX2
Principle: Random reads from a compressed archive keep paying the decompression cost again. Keep recently extracted chunks in memory, with the total size capped, and serve overlapping reads from them. Memory is traded for CPU and disk time.
Why it is faster: —
Applies elsewhere: block caches for zstd or gzip seekable archives. Tile caches for compressed map data. Decoded-image caches in browsers and mobile apps.
Watch out: —
Seen in PCSX2: `CompressedFileReader.cpp` gained a `ChunksCache`. It started as an array of 50 most-recent entries and became a `std::list` with a byte limit (`SetLimit`/`MatchLimit`, 200 MB by default). `CopyAvailable` serves the part of a read already covered by a cached extraction. 36fe83afc 2014-04 “Gzip ISO: add cache (50 chunks of 4M)” · 86a6fcddc 2014-04 “gzip-iso: Better cache, more cpu for quicker extract”

### pcsx2-143 — Save decoder state at each visited span to resume sequential reads
Lens: io · Source: PCSX2
Principle: Seeking inside a stream-compressed file means decompressing again from the nearest index point. Store the decoder state at the end of every span you have already decoded. Later reads that continue from a visited span then pick up where they left off instead of restarting.
Why it is faster: —
Applies elsewhere: resumable parser checkpoints for large logs. Seekable video decoders that keep decoder state per GOP. Incremental CSV readers.
Watch out: —
Seen in PCSX2: `GzippedFileReader` replaced its single `Zstate m_zstate` with an array `Czstate* m_zstates` sized to `uncompressed_size / span + 2` (`InitZstates`). Most previously visited spans can now continue sequentially, at the cost of about one more index's worth of RAM. d28669761 2014-05 “gzip-iso: Speedup some cases by using more memory.”

### pcsx2-144 — Warm the OS cache with a throwaway async read
Lens: io · Source: PCSX2
Principle: Once you know which disk region comes next, issue an asynchronous read for it and discard the result. The OS pulls the data into its page cache in the background, so the later synchronous read doesn't wait on a seek.
Why it is faster: —
Applies elsewhere: `posix_fadvise(WILLNEED)` or `readahead()` before scanning a file. Prefetching the next media segment. Database read-ahead of the next pages.
Watch out: —
Seen in PCSX2: On Win32, after each extract, `GzippedFileReader::AsyncPrefetchChunk` opens the file with `FILE_FLAG_OVERLAPPED | FILE_FLAG_SEQUENTIAL_SCAN` and issues an overlapped `ReadFile` of the next `READ_CHUNK_SIZE` into a dummy buffer. `AsyncPrefetchCancel` cancels it before the next real read. This often hides seek time on fragmented files. 6b4878470 2015-02 “gzip iso: reduce seek time on fragmented files by OS prefetch on win32”

### pcsx2-145 — Reset and reuse codec state instead of rebuilding it
Lens: allocation · Source: PCSX2
Principle: Setting up and tearing down a decompressor allocates its windows and tables each time. Keep one instance and call its reset function between independent blocks.
Why it is faster: —
Applies elsewhere: reuse `zlib.Inflater` with `reset()` in Java. Pool `gzip.Reader` in Go with `Reset`. Reuse ZSTD_DCtx in servers.
Watch out: —
Seen in PCSX2: `CsoFileReader::ReadFromFrame` used to call `inflateInit2` and `inflateEnd` on a stack `z_stream` for every frame. Now a member `m_z_stream` is initialised once in `Open` and `inflateReset` runs after each frame. 49cfc3424 2015-04 “Keep around z_stream between decompression passes.”

### pcsx2-146 — Keep the last file handle open and skip redundant seeks
Lens: io · Source: PCSX2
Principle: Opening a file, seeking and closing it again for every small access costs system calls and path resolution. Cache the most recently used handle. Reopen only when the target file or mode changes, and seek only when the current offset differs from the one you need.
Why it is faster: —
Applies elsewhere: file-descriptor caches in static file servers. A log appender that keeps its file open. SQLite-style pagers that keep the database file open.
Watch out: —
Seen in PCSX2: `FolderMemoryCard` gained a `FileAccessHelper m_lastAccessedFile` with `ReOpen()`, so consecutive page reads and writes to the same host file reuse one `wxFFile`. Each read and write now checks `file->Tell()` before calling `Seek`. Path building and per-file metadata writes then moved into `Open`, so they happen once per consecutive access instead of once per chunk. f73db1a57 2015-06 “FolderMemoryCard: Optimize file access by keeping a file open between consecutive reads/writes to the same file.” · 385bdfb86 2015-06 “FolderMemoryCard: Further optimize file access times.”

### pcsx2-147 — Build a reverse index instead of walking chains on each lookup
Lens: algorithms · Source: PCSX2
Principle: If finding the owner of a block means walking every file's cluster chain, each lookup costs O(total clusters). Build a map from block to owner (with parent and position) once at load time. Lookups then become a single map access.
Why it is faster: —
Applies elsewhere: an inode-to-path cache in filesystem tools. Reverse maps from page to object in allocators. Child-to-parent maps in tree UIs.
Watch out: —
Seen in PCSX2: `FolderMemoryCard::AddFileEntryToMetadataQuickAccess` walks each file's FAT chain once and fills `m_fileMetadataQuickAccess[cluster]` with the entry, its parent and its consecutive cluster index. Reads and writes look a cluster up directly. The old per-access traversal loop is no longer used. 80feb1087 2015-06 “FolderMemoryCard: Add a helper structure to quickly access a file entry from a file data FAT cluster.”

### pcsx2-148 — Flush only data that actually changed
Lens: io · Source: PCSX2
Principle: Snapshot each page before its first write. At flush time, compare the snapshot with the current page and drop pages whose contents match, then write out only the real changes. This cuts I/O and leaves file timestamps alone.
Why it is faster: —
Applies elsewhere: dirty-page diffing in editors' autosave. Skip rewriting config files whose content is unchanged, as build systems do to keep mtimes stable. Flash wear-levelling layers that skip identical writes.
Watch out: —
Seen in PCSX2: `FolderMemoryCard` copies a page into `m_oldDataCache` on its first write. `RemoveUnchangedDataFromCache` compares old and new pages (`memcmp` over `PageSize`) for files whose metadata is unchanged, and erases matching pages from `m_cache` before the flush. 6bd578ccb 2015-07 “FolderMemoryCard: Reduce unnecessary file I/O by only flushing files that have actually changed”

### pcsx2-149 — Stream through a persistently mapped ring buffer
Lens: gpu-rendering · Source: PCSX2
Principle: Mapping and unmapping a buffer for every upload, or calling sub-data, costs a driver round trip and sometimes a sync. Map one large buffer once with persistent flags, write into it as a ring that wraps at the end, and issue a single barrier before drawing. More buffers in the pool mean less waiting on ones still in use.
Why it is faster: —
Applies elsewhere: Vulkan or D3D12 upload heaps used as ring buffers. Shared-memory ring buffers between processes. Persistent mmap for log writers.
Watch out: —
Seen in PCSX2: `GSVertexArrayOGL` maps its VBO/IBO once with `GL_MAP_PERSISTENT_BIT` (and without COHERENT). Each upload `memcpy`s to `m_buffer_ptr + m_start*m_stride` and wraps to 0 when full. `GSDeviceOGL::BeforeDraw` issues `GL_CLIENT_MAPPED_BUFFER_BARRIER_BIT`. A replay test went from 60 to 76 fps. Separately, `PBO_POOL_SIZE` went from 4 to 8 to reduce texture-upload conflicts. ce9827632 2015-04 “gsdx-ogl: improve speed of vertex streaming” · b32f808fd 2015-04 “gsdx-ogl: increase the number of pbo”

### pcsx2-150 — Don't run a cheap pre-pass under an expensive exact pass
Lens: gpu-rendering · Source: PCSX2
Principle: When an exact method already handles a case, a coarse pre-filter in front of it only adds work. Choose one path per case, not both.
Why it is faster: —
Applies elsewhere: skip a bloom-filter probe when the exact hash set is already in L1. Don't run a regex pre-scan before a full parser that rejects the same inputs.
Watch out: —
Seen in PCSX2: Destination-alpha testing (DATE) in `GSRendererOGL::DrawPrims` used the stencil method and the image-load-store shader method together. The stencil setup now runs only `if (dev->HasStencil() && !advance_DATE)`, and the shader method runs otherwise. bd6ea17bd 2015-04 “gsdx-ogl: speed improvement for DATE”

### pcsx2-151 — Check a precondition that unlocks a single-pass path
Lens: algorithms · Source: PCSX2
Principle: Multi-pass algorithms often exist only to handle overlap or dependencies between items. Test cheaply whether the batch has any. When it doesn't, use the single-pass version. Cap the cost of the test so that large batches fall back to the conservative path.
Why it is faster: —
Applies elsewhere: use `memcpy` instead of `memmove` when ranges are disjoint. Apply a batch of DB updates in parallel when their keys don't collide. Run lock-free when the transaction footprints don't intersect.
Watch out: —
Seen in PCSX2: `GSRendererOGL::PrimitiveOverlap()` checks whether sprites intersect. With no overlap, the shader can read the render target directly behind a texture barrier (SotC at 6× went from 30 to 40 fps), and accurate DATE later became one pass (`DATE_one`: clear stencil to 1, test in the shader). The overlap check is skipped for draws of more than 256 vertices, because the pairwise test got too slow in Virtua Fighter. c207632e4 2015-04 “gsdx-ogl: improve date performance for GL45” · e776118de 2016-05 “gsdx-ogl: implement previous DATE optimization in a single shader pass” · c6ff7531f 2015-08 “gsdx-ogl: performance boost on virtuafighter”

### pcsx2-152 — Offer a cheaper approximation as the default tier
Lens: other · Source: PCSX2
Principle: An all-or-nothing accuracy switch makes users choose between wrong results and slow ones. Add a middle tier that covers the common cases at low cost, and make it the default. Keep the exact mode for the cases that need it.
Why it is faster: —
Applies elsewhere: approximate COUNT DISTINCT (HyperLogLog) by default with an exact mode on request. Image-resizing quality presets. Lossy and lossless toggles in codecs.
Watch out: —
Seen in PCSX2: The `accurate_date` checkbox became Off/Fast/Full, with Fast as the new default. In `GSRendererOGL`, `ACC_DATE_FAST` uses the single-pass `DATE_one` path. `ACC_DATE_FULL` keeps the image-load-store or full-barrier method. 584397a3f 2018-11 “GSdx-ogl: Implement a fast Accurate Date algorithm.”

### pcsx2-153 — Detect degenerate parameters that make an operation trivial
Lens: algorithms · Source: PCSX2
Principle: General operations often reduce to something cheaper for particular parameter values: a subtraction of equal terms is zero, and multiplying by the identity does nothing. Recognise those values on the CPU and send the cheaper variant or skip the correction pass.
Why it is faster: —
Applies elsewhere: skip the matrix multiply when the transform is identity. Bypass gain stages at unity gain in audio DSP. Skip compression when the payload is already compressed.
Watch out: —
Seen in PCSX2: When `ALPHA.A == ALPHA.B` the blend is a plain copy and can't overflow, so the colclip wrap passes are skipped. The same test, plus `C == 2 && afix <= 1`, avoids the slow software blend. A MODULATE texture function with constant vertex colour 128 is replaced by DECAL, which saves 5 fragment-shader instructions. 7979dec5b 2015-05 “gsdx-ogl: optimize colclip 0” · c2e851b3a 2015-06 “gsdx-ogl: don't use slow SW blend for nothing” · cb4af8fe8 2015-07 “gsdx-ogl: small optimization for the GPU”

### pcsx2-154 — Use value-range analysis to narrow the slow path
Lens: algorithms · Source: PCSX2
Principle: A slow exact path is often needed only when values can leave a safe range. Compute the actual min and max of the inputs, which can be lazy and cached, and take the slow path only when the range proves it is needed.
Why it is faster: —
Applies elsewhere: skip overflow-checked arithmetic when bounds analysis shows it is safe. Choose int16 storage when column stats fit. Skip clamping in image filters when the input range is known.
Watch out: —
Seen in PCSX2: In `ACC_BLEND_FULL` mode, the renderer used to switch to software blending for nearly every non-trivial blend. It now calls `GetAlphaMinMax()` when needed and requires software blending only if `m_vt.m_alpha.max > 128` (or `FIX > 128`), because only then can the coefficient exceed 1. The commit reports a massive speed-up at that accuracy level. 9f92f6319 2015-08 “gsdx-ogl: Use GetAlphaMinMax to limit the scope of FULL accurate blending”

### pcsx2-155 — Skip work whose result nobody observes
Lens: laziness · Source: PCSX2
Principle: If a stage's only effect is on an output that is disabled or absent, drop the stage. Check what is consumed downstream before doing the work.
Why it is faster: —
Applies elsewhere: don't compute analytics fields a dashboard doesn't display. Skip depth pre-passes when depth is never read. Dead-store elimination in compilers.
Watch out: —
Seen in PCSX2: In `FB_ONLY` alpha-fail mode the alpha test affects only depth, so with no depth buffer (`ds == nullptr`) the second draw pass and its program switch are skipped (`ate_skip`). `GSRendererHW` also skips looking up a depth buffer when depth always passes and writes are masked (`ZMSK && ZTST_ALWAYS`), which sped up Tekken 5. 5fbf70250 2016-08 “gsdx ogl: new optimization to bypass the alpha test” · 1853d0d16 2016-02 “gsdx: no depth lookup optimization”

### pcsx2-156 — Restrict clears, copies and readbacks to the touched region
Lens: gpu-rendering · Source: PCSX2
Principle: Full-surface operations cost in proportion to the surface size, not the work, and they get worse as resolution scales up. Compute a bounding box for the affected area, then scissor the clear, copy only that rectangle, or read back only those pixels.
Why it is faster: —
Applies elsewhere: dirty-rectangle repaint in UI toolkits. Partial texture updates in WebGL (`texSubImage2D`). Range requests instead of whole-file downloads.
Watch out: —
Seen in PCSX2: DATE setup now keeps `GL_SCISSOR_TEST` on and scissors to the primitive bounding box before the stencil clean. `ComputeBoundingBox` was factored out and reused for the HDR colclip copy. GPU→CPU reads in `GSTextureOGL::Map` use `glReadPixels(r->x, r->y, r->width(), r->height())` instead of the whole texture. With partial invalidation disabled, target reads were limited to the needed rectangle, because one game read a 1280×448 target for 24×12 texels; that gave a 2× speed-up on Champions of Norrath. 6e386df53 2015-04 “gsdx-ogl: avoid to clean fully texture in DATE” · e026f1bac 2015-07 “gsdx-ogl: implement a fast accurate colclip algo” · da741e294 2016-03 “gsdx-ogl: optimize GPU->CPU memory transfer size” · c43b1ac20 2016-03 “gsdx tc: extend the fast invalidation hack to limit the size of target read”

### pcsx2-157 — Set state once instead of toggling and restoring it
Lens: gpu-rendering · Source: PCSX2
Principle: Saving state, changing it and restoring it around each operation doubles the driver calls. A configuration set once at startup and shared by all draws removes per-draw rebinding. Track state on the CPU, set it to the value that the next consumer needs, and make sure "already done" flags are really updated.
Why it is faster: —
Applies elsewhere: state caches in WebGL wrappers. Sort draw calls by material to minimise binds. Avoid redundant `ctx.save()`/`restore()` in Canvas2D.
Watch out: —
Seen in PCSX2: The two vertex formats were merged into one VAO that is bound once at startup, and `IASetVertexState` was removed. DATE setup now passes the cached `GLState::scissor` to `OMSetRenderTargets` instead of setting a scissor and then restoring the old one with `glScissor`. `HasBeenCleaned()` no longer sets the flag as a side effect; `WasCleaned()` sets it explicitly, so repeated cleans of the same texture are really skipped. ccc1137e1 2014-10 “gsdx-ogl: merge the two vertex buffer format” · 7367b22e0 2015-05 “gsdx-ogl: reduce toggling of scissor state for DATE” · a5e424512 2015-05 “gsdx-ogl: really avoid consecutive clean”

### pcsx2-158 — Copy only payload bytes, not the source stride
Lens: memory-layout · Source: PCSX2
Principle: When the source rows are wider than the region you need, a single bulk copy of rows × pitch moves padding too. Copy row by row, taking only `width × bpp` bytes into a tightly packed destination. Fewer bytes move, and the staging buffer is smaller.
Why it is faster: —
Applies elsewhere: crop images before upload. Pack sub-rectangles of a canvas into a WebGL texture. Repack strided NumPy views before sending them over the network.
Watch out: —
Seen in PCSX2: `GSTextureOGL::Update` used to copy `pitch*height` into the PBO and set `GL_UNPACK_ROW_LENGTH`. It now loops over rows copying `row_byte = width << m_int_shift`, which shrinks the PBO footprint and drops the row-length state. f37f3cb3c 2015-05 “gsdx-ogl: improve texture uploading”

### pcsx2-159 — Produce data directly in the destination buffer
Lens: memory-layout · Source: PCSX2
Principle: Writing into a temporary buffer and then copying it into the final buffer moves every byte twice. Give the producer a pointer into the destination (a mapped staging buffer, say) so it writes there directly.
Why it is faster: —
Applies elsewhere: decode images straight into a mapped GPU buffer. Serialize into the socket's send buffer, as zero-copy protocols do. Pass `out` parameters instead of returning temporaries.
Watch out: —
Seen in PCSX2: For textures and render targets, `GSTextureOGL::Map` now returns a pointer from `PboPool::Map(map_size)`, so the texture cache unswizzles GS memory straight into the GL buffer. Previously it went to `m_local_buffer` and was then copied. Measured gains were 5–8% at native resolution (DQ8 255→270, ZoE 50→54). eb6952264 2016-03 “gsdx-ogl: optimize texture memory upload”

### pcsx2-160 — Move work from a stage that needs a barrier into one that doesn't
Lens: gpu-rendering · Source: PCSX2
Principle: If an exact result in a fixed-function stage needs the full feedback path (read back, then barrier), compute the hard part earlier, where it is cheap and exact, and leave the fixed unit only a simple operation it does exactly. To handle overflow, accumulate in a wider format and apply the wrap in one post-pass.
Why it is faster: —
Applies elsewhere: premultiply alpha in the shader so blending is plain addition. Accumulate in float, then quantise once. Do aggregation in SQL instead of row-by-row in client code.
Watch out: —
Seen in PCSX2: For the `Cs*As + Cd` and `Cs*Af + Cd` blends, `PS_BLEND_ACCU` does the multiply in the fragment shader and the blend unit only adds, so accurate blending needs no costly barrier. The fast accurate colclip then accumulates into a half-float target and wraps it in a final pass over the bounding box. Half-float gave about 12 fps more at 4×. ae8df002a 2015-07 “gsdx-ogl: optimize Cs * As + Cd and Cs * Af + Cd blending” · e026f1bac 2015-07 “gsdx-ogl: implement a fast accurate colclip algo”

### pcsx2-161 — Replace per-lane branches with SIMD compare masks and blends
Lens: simd · Source: PCSX2
Principle: When you test several bytes or floats one by one with branches, the CPU mispredicts. Compare them all at once with a SIMD compare, extract a bitmask, and select results with a blend. The result is a few instructions and no jumps.
Why it is faster: —
Applies elsewhere: byte-classification masks in simdjson. Branchless NaN sanitising in numeric kernels. Vectorised min/max for bounding boxes in game engines.
Watch out: —
Seen in PCSX2: The FBMSK setup replaced four per-channel `if`s with `eq8(xffffffff()).mask()` and `eq8(zero()).mask()`. NaN Q values are replaced with `GSVector4::replace_nan`, which is `v.blend32(*this, *this == *this)`, a cmpps used as a blend mask. Sprite min/max sorting in the overlap test uses `upl16` and a `blend` on `all > all.zwxy()`. 717f0fcb4 2015-08 “gsdx-ogl: optimize fbmask setup” · e010004f1 2015-08 “gsdx: _isnan was not reliable, rewrote it as GSVector4::replace_nan” · 37deeb0d5 2016-01 “gsdx-ogl: Optimise overlap detection algorithm”

### pcsx2-162 — Ask the driver for less validation and more threading
Lens: platform · Source: PCSX2
Principle: Graphics drivers spend CPU time on error checks and on serial command processing. Where the API allows, request a no-error context and enable the driver's threaded mode, so command submission overlaps with your own work.
Why it is faster: —
Applies elsewhere: disable debug layers in release Vulkan builds. Set `preserveDrawingBuffer: false` in WebGL. Enable the driver's multi-threaded mode on NVIDIA.
Watch out: —
Seen in PCSX2: The WGL context creation adds `GL_CONTEXT_FLAG_NO_ERROR_BIT_KHR`. On Linux, `GSopen` sets `__GL_THREADED_OPTIMIZATIONS=1` by default (option `enable_nvidia_multi_thread`), and texture storage is turned on with that driver mode. 77f911330 2015-12 “gsdx-gl: use CONTEXT_FLAG_NO_ERROR_BIT_KHR to reduce driver overheard” · 8a7384953 2015-05 “gsdx-ogl: enable multithread driver by default for nvidia”

### pcsx2-163 — Declare only the bindings each program uses
Lens: gpu-rendering · Source: PCSX2
Principle: Some drivers revalidate every active binding when any one of them changes. Declaring all uniform blocks and images in a shared header makes every program pay for all of them. Declare each binding only in the shaders that read it.
Why it is faster: —
Applies elsewhere: smaller descriptor sets per pipeline in Vulkan. Fewer bound uniform blocks in WebGL2 programs. Narrow dependency injection scopes.
Watch out: —
Seen in PCSX2: The cb10, cb11 and cb14 blocks moved out of `common_header.glsl` into `merge.glsl`, `interlace.glsl` and `fxaa.fx`, and unused `cb22` was commented out. The commit notes that Nouveau validates all constant buffers when one is updated. 2b00447a4 2016-06 “glsl: optimize the number of active constant buffer”

### pcsx2-164 — Emit one strip instead of separate primitives
Lens: gpu-rendering · Source: PCSX2
Principle: A quad emitted as two separate triangles writes 6 vertices and ends two primitives. As one triangle strip it writes 4 vertices and ends one. Geometry amplification costs per emitted vertex, so fewer vertices means less work.
Why it is faster: —
Applies elsewhere: sprite batchers that use strips or indexed quads. Instanced quads in particle systems.
Watch out: —
Seen in PCSX2: `tfx_vgs.glsl` changed from `max_vertices = 6` with two `EndPrimitive` calls to `max_vertices = 4` with one strip (lt, lb, rt, rb) for sprites and lines. c5d35d434 2017-07 “glsl: reduce geometry shader complexity”

### pcsx2-165 — Drop status-flag updates that are never read (liveness)
Lens: jit-interpreters · Source: PCSX2
Principle: Emulated or generated code often computes status flags on every operation, although later code reads only a few of them. Scan the block backwards, find which flag writes are later read (keeping the writes still live at the block exit), and emit only those.
Why it is faster: —
Applies elsewhere: dead-flag elimination in x86-to-ARM translators. Skip computing derived fields that no subscriber consumes. Liveness-based dead-code elimination in compilers.
Watch out: —
Seen in PCSX2: The microVU flag speedhack in `microVU_Flags.inl` now covers MAC flags as well as status flags. It walks the block, marks `sFLAG.skipWrite` on flag instances that are overwritten before any read, protects the last few ops for block linking (`aCount`, `writeProtect`), and changes `sFlagCond` so skipped writes emit no code. Gains ranged from nothing to about 30 fps. d8343cea2 2015-05 “microVU: New improved flag speedhack.”

### pcsx2-166 — Replace a large switch with a lookup table built once
Lens: lookup-tables · Source: PCSX2
Principle: A long switch over a dense key range compiles into chains of comparisons or a jump table plus code for each case. When the result is just a value, fill an array once at startup and index into it.
Why it is faster: —
Applies elsewhere: keycode translation tables. HTTP status-to-text arrays. Character-class tables in lexers.
Watch out: —
Seen in PCSX2: `TranslateVKToWXK` in `WinKeyCodes.cpp` was a large `switch`. It is now a lazily initialised `static int vk2wx[256]`, filled by `initTable()` through `addTx(VK_*, WXK_*)`. a76914a4b 2015-10 “gui: kb shortcuts win: faster VK to WX translation with a table”

### pcsx2-167 — Use relaxed and acquire/release atomics for single-writer indices
Lens: concurrency · Source: PCSX2
Principle: In a single-producer, single-consumer ring, each index has exactly one writer. The owner can read its own index with a relaxed load. The other side needs acquire on read and release on publish. Full sequentially consistent operations and exchanges add fences that aren't needed.
Why it is faster: —
Applies elsewhere: SPSC queues in audio threads and network packet rings. Rust's `Ordering::Relaxed`/`Acquire`/`Release` in lock-free crates.
Watch out: —
Seen in PCSX2: In `MTGS.cpp`, `m_ReadPos` and `m_WritePos` became `std::atomic`, replacing the `volatize()` casts. The GS thread reads `m_ReadPos.load(memory_order_relaxed)` and `m_WritePos.load(memory_order_acquire)`. `MTVU.cpp` replaced `AtomicExchange(write_pos, …)` with `write_pos.store(…, memory_order_release)`. The `readAmount` fetch_add and fetch_sub calls use `acq_rel`. 5913ff1bd 2016-07 “MTGS: port read/write pointer to full atomic + optimize atomic access” · 3f0655c82 2016-07 “MTVU: port write pointer to atomic and optimize atomic access” · 24c780c88 2017-01 “MTVU/GS: try to relax the readAmount atomic operation”

### pcsx2-168 — Replace a mutex-guarded deque with a lock-free SPSC ring
Lens: concurrency · Source: PCSX2
Principle: For one producer and one consumer, a bounded ring buffer with atomic head and tail never blocks either side and doesn't allocate per element. A mutex around a `std::deque` pays for locking, and sometimes for heap allocation, on every push and pop.
Why it is faster: —
Applies elsewhere: logging pipelines (producer threads to a writer). Audio callback queues. Disruptor-style event buses.
Watch out: —
Seen in PCSX2: In `Gif_Unit.h`, the MTVU path's `Mutex gsPackMutex` plus `std::deque<GS_Packet> gsPackQueue` became `ringbuffer_base<GS_Packet, RingBufferSize/2>` from `boost_spsc_queue.hpp`. `push`, `front`, `pop` and `size` no longer take a lock. 7b3984059 2017-01 “MTVU/MTGS: use a mutex free ring buffer”

### pcsx2-169 — Put each thread's hot atomic on its own cache line
Lens: concurrency · Source: PCSX2
Principle: When indices written by different threads share a cache line, every write invalidates the other core's copy. This is false sharing. Pad the struct so each thread's hot atomic sits on a separate line.
Why it is faster: —
Applies elsewhere: `alignas(std::hardware_destructive_interference_size)` on queue heads and tails. `@Contended` in Java. Per-CPU counters in kernels.
Watch out: —
Seen in PCSX2: `boost_spsc_queue.hpp` adds `char padding2[padding_size]` to separate the consumer's read index from the producer-side fields. e4f4350bb 2017-01 “boost ring queue: dedicate cache line for atomic variable”

### pcsx2-170 — Spin or yield for short waits instead of blocking
Lens: concurrency · Source: PCSX2
Principle: Taking a lock or sleeping on a semaphore to wait for a short-lived condition costs a context switch, and wake-up latency can be far longer than the wait itself. Yielding or spinning keeps latency low at the cost of CPU time, so make it an option where power matters.
Why it is faster: —
Applies elsewhere: adaptive mutexes that spin before parking. Busy-poll modes in network stacks (`SO_BUSY_POLL`). Game loops that spin-wait for vsync.
Watch out: —
Seen in PCSX2: When the MTVU ring buffer is full, the EE thread now calls `std::this_thread::yield()` instead of taking `mtxBusy`, which the commit says forced a full ring flush. GSdx's rasterizer gained a `spin_thread` option that uses `GSJobQueueSpin` workers. The commit calls it faster, but notes the thread never sleeps, which is bad for laptops. e3d1871f8 2017-01 “MTVU: small thread scheduling improvement” · 0aac47ca5 2015-03 “gsdx-queue: add a new option "spin_thread" to select the queue behavior at runtime”

### pcsx2-171 — Tune work-unit granularity for parallel workers
Lens: concurrency · Source: PCSX2
Principle: Small work units balance load well but add per-unit sync overhead. Large units cut overhead but leave cores idle waiting for stragglers. Size the units from the thread count (or expose the size as a setting) and turn parallelism on by default.
Why it is faster: —
Applies elsewhere: `chunksize` in Python `multiprocessing.map`. OpenMP `schedule(dynamic, N)`. Tile sizes in parallel image filters.
Watch out: —
Seen in PCSX2: `GSRasterizer` replaced `#define THREAD_HEIGHT 4` with `m_thread_height` from `extrathreads_height` (log2 rows per band, 1–8, default 4). The commit notes that small values add overhead and large ones add wait latency. Extra software-rendering threads now default to 2. e642bbc42 2016-07 “gsdx sw: add extrathreads_height to control the quantity of pixels processed by a thread” · 0c4b25bce 2015-12 “gsdx: defaulting extra rendering threads to 2”

### pcsx2-172 — Move slow encoding or I/O to a worker thread
Lens: concurrency · Source: PCSX2
Principle: Compressing and writing files on the main loop stalls it. Copy the data into a job and push it onto a worker queue, so the main thread continues right away. Wait for the queue to drain only at shutdown.
Why it is faster: —
Applies elsewhere: background thumbnail generation. Async log or metrics flushers. Off-main-thread image encoding in mobile apps.
Watch out: —
Seen in PCSX2: On Linux, `GSCapture::DeliverFrame` now pushes a `GSPng::Transaction` (which owns an aligned copy of the frame) to a `GSPng::Worker` instead of calling `GSPng::Save` inline. `EndCapture` calls `m_worker->Wait()`. cff168e00 2015-05 “gsdx-linux-recorder: implement multithread to save the image”

### pcsx2-173 — Commit virtual memory in large blocks and let the OS do lazy paging
Lens: allocation · Source: PCSX2
Principle: Committing memory 4 KB at a time from a fault handler costs a trap per page, plus bookkeeping. Reserve big regions, commit them in large chunks, and where possible rely on the kernel's own demand paging instead of a hand-written fault-driven allocator.
Why it is faster: —
Applies elsewhere: arena allocators that commit memory in MBs. JIT code caches (V8, LuaJIT) sized up front. `mmap` with overcommit for sparse arrays.
Watch out: —
Seen in PCSX2: `RecompiledCodeReserve` sizes grew: R5900 to 16 MB, R3000A to 8 MB, microVU to 16 MB, VIF to 8 MB. Code memory is now handled in big blocks, so lazy commit faults are rarer. `SpatialArrayReserve`, a SIGSEGV-driven block allocator for the EE block array, was removed in favour of kernel lazy allocation. a4a0b42f8 2015-11 “recompilers: handle the memory by big block (instead of 4KB)” · 7565bcc78 2015-11 “ee: drop SpatialArrayReserve allocator”

### pcsx2-174 — End chains with a sentinel instead of storing their length
Lens: algorithms · Source: PCSX2
Principle: If lookups almost always hit, storing and checking a length on every probe is wasted work. Terminate each chain with a sentinel record (for example a null code pointer). The hot find loop then needs only its key compare and the sentinel test, and the rare insert walks the chain to count.
Why it is faster: —
Applies elsewhere: NUL-terminated strings. Sentinel nodes in linked lists. Open-addressing tables with empty-slot markers.
Watch out: —
Seen in PCSX2: The VIF unpack `HashBucket` replaced `SizeChain{Size, Chain}` with `std::array<nVifBlock*, hSize> m_bucket`, where each chain ends in a block whose `startPtr == 0`. `find` loops until it hits that sentinel, and `add` (the cold path) traverses to get the size. 10b3d429f 2016-12 “vif: new implementation of the hash bucket”

### pcsx2-175 — Pack lookup keys and cache derived values in the entry
Lens: memory-layout · Source: PCSX2
Principle: Shrink key fields to their real range so the key fits in fewer machine words. Use the freed space to store values computed from the key, so a hit doesn't recompute them.
Why it is faster: —
Applies elsewhere: cache a string's hash in its header. Store a precomputed bounding box with geometry. Pack flags into spare bits of an ID.
Watch out: —
Seen in PCSX2: `nVifBlock` packed `cl` and `wl` from `u16` to `u8` and added a `u16 length`, the precomputed transfer length, that the dispatch code uses on a hit. Key comparison then became two integer compares (`key0`, `key1`). 1a3206243 2016-12 “vif: repack nVifBlock struct”

### pcsx2-176 — Use scalar compares for short early-exit searches
Lens: branching-fast-paths · Source: PCSX2
Principle: A SIMD compare, movemask and test sequence has latency and blocks compare/branch fusion. For a search that usually ends within one or two entries, plain integer compares on the key words are faster and easier to predict.
Why it is faster: —
Applies elsewhere: small linear lookups in interpreters' inline caches. Short key arrays in B-tree nodes. Tag checks in software caches.
Watch out: —
Seen in PCSX2: `HashBucket::find` replaced `_mm_cmpeq_epi32` + `_mm_movemask_ps` with `if (chainpos->key0 == d.key0 && chainpos->key1 == d.key1)`. The commit message compares the generated assembly for both versions. 7a33cda12 2016-12 “vif: replace sse cmp code with standard cmp”

### pcsx2-177 — Embed fixed-size members instead of heap-allocating them
Lens: memory-layout · Source: PCSX2
Principle: A member that is allocated separately and reached through a pointer costs an extra dependent load on every access, and has to be null-checked or set up. If its size is fixed and its lifetime matches the owner's, store it inline.
Why it is faster: —
Applies elsewhere: value-type members instead of `unique_ptr` in C++. Structs embedded inline in Go. Arrays sized at compile time in firmware.
Watch out: —
Seen in PCSX2: `nVifStruct::vifBlocks` changed from `HashBucket*` (created with `new` in `dVifReserve`) to an embedded `HashBucket`, which removes one indirection from every `find`. 1acc81c25 2016-12 “vif: don't allocate vifblock hash on the heap”

### pcsx2-178 — Inline small helpers so duplicate checks merge
Lens: branching-fast-paths · Source: PCSX2
Principle: A helper that returns a sentinel, followed by a caller that tests the sentinel, does the same check twice. Inlining it, or putting its logic straight into the caller, lets one branch decide the path. Don't search again after a miss you already handled.
Why it is faster: —
Applies elsewhere: mark hot accessors `inline`. Fold `Optional` unwrap chains into a single branch. Reuse the insertion point from a failed lookup, as `try_emplace` does.
Watch out: —
Seen in PCSX2: `dVifsetVUptr` returned NULL when a block would wrap, and the caller tested for it. Its body moved into `dVifUnpack`, so one bounds check picks dynarec or interpreter. Earlier, block compilation stopped re-searching the hash after a miss. The `LogicalOp*` emitter helpers were inlined as direct `xAND`/`xOR`/`xXOR` switches, and single-use renderer methods were marked `inline`. e4c2c53b1 2016-12 “vif: inline dVifsetVUptr function” · 3dc7dc0cd 2016-12 “vif: improve block compilation management” · 357046783 2015-11 “core: inline LogicalOp* functions” · 2fc244a3e 2016-05 “gsdx-ogl: inline function only call once”

### pcsx2-179 — Store the container position so removal needs no search
Lens: algorithms · Source: PCSX2
Principle: Removing an object from a list by scanning for it costs O(n) per removal. Record the iterator (or index) when you insert the object, and erase with it in O(1).
Why it is faster: —
Applies elsewhere: LRU caches that store list nodes in the hash map. Timer wheels that keep a handle for cancellation. Intrusive lists in kernels.
Watch out: —
Seen in PCSX2: `GSTextureCache::Source` gained `std::array<std::list<Source*>::iterator, MAX_PAGES> m_erase_it`. `SourceMap::Add` saves `m_map[page].begin()` after `push_front`, and `RemoveAt` erases directly. In ZoE2, `RemoveAt`'s share of time fell from 17% to 0.5%. d1315b618 2017-01 “gsdx tc: save list iterator to allow fast removal”

### pcsx2-180 — Store scanned collections contiguously, not as node lists
Lens: memory-layout · Source: PCSX2
Principle: Linked lists scatter their nodes across the heap, so each step of a scan risks a cache miss. A vector, or a list whose links are indices into one contiguous array, keeps the nodes together. Linear scans then run at memory bandwidth, and move-to-front stays cheap.
Why it is faster: —
Applies elsewhere: slot maps or arenas with index links in game engines. `Vec` instead of `LinkedList` in Rust. Array-backed LRU lists.
Watch out: —
Seen in PCSX2: `GSDirtyRectList` and `GSCodeBuffer::m_buffers` moved from `std::list` to `std::vector`. A new `GSFastList<T>` stores `Element{data, uint16 next_index, prev_index}` in one buffer, with `MoveFront(index)`. It replaced `std::list` in the hardware and software texture caches and in `GSDevice`. b431d172b 2017-08 “GSdx GSDirtyRect: std::vector instead of std::list, to speedup linear scan.” · 3300470a2 2017-05 “GSdx GSCodeBuffer: std::vector instead of std::list” · 676c7b89c 2017-08 “GSdx GSFastList: Introducing the custom container template class.” · a0aa585af 2017-08 “GSdx GSTextureCache: Using FastList instead of std::list.” · 965334350 2017-08 “GSdx GSTextureCacheSW: Ported erase iterator trick. Using FastList”

### pcsx2-181 — Memoize expensive derived bitmaps by their input key
Lens: caching · Source: PCSX2
Principle: When the same parameters keep producing the same derived structure (such as a coverage bitmap), compute it once per distinct key and keep it in a hash map. Align the result to a cache line so it is read in one fetch.
Why it is faster: —
Applies elsewhere: caching compiled regexes or query plans by their text. Glyph-coverage masks in font renderers. Permission bitsets per role.
Watch out: —
Seen in PCSX2: `GSTextureCache::SourceMap::GetPagesCoverage` looks up a 64-bit key (`TEX0.u64 & 0x3ffffffff`: base, width, format and size) in `m_pages_coverage`. On a miss it builds a 64-byte-aligned page bitmap. `Add` walks the set bits with `_BitScanForward`. The self time of `SourceMap::Add` fell from 5.39% to 0.23%. fa1377a8e 2016-04 “gsdx tc: reduce texture cache overhead”

### pcsx2-182 — Give proxy objects a lightweight constructor
Lens: allocation · Source: PCSX2
Principle: An object that only forwards to another one doesn't need the buffers a full instance allocates. A constructor flag that skips those allocations makes proxies nearly free.
Why it is faster: —
Applies elsewhere: view objects that borrow buffers (`string_view`, slices). Flyweight wrappers. Lazy ORM proxies that allocate nothing until a field is accessed.
Watch out: —
Seen in PCSX2: `GSTextureCache::Source` gained a `dummy_container` argument. Shared-texture sources skip the `m_clut` and `m_write.rect` `_aligned_malloc`s, the `m_valid` memset and the page-to-tile map. The commit says this makes them almost free in CPU terms. 49d175b67 2016-04 “gsdx tc: allow to create a dummy Source without tons of memory allocation”

### pcsx2-183 — Share identical derived resources through a content-keyed pool
Lens: caching · Source: PCSX2
Principle: When many objects each build their own copy of the same derived resource (a GPU texture, a copied table), deduplicate them. Hash the content, look it up in a map of shared, reference-counted instances, and attach the existing one. Allocations, uploads and bus traffic all drop.
Why it is faster: —
Applies elsewhere: string interning. Content-addressed asset caches in game engines. Docker layer deduplication.
Watch out: —
Seen in PCSX2: A `PaletteMap` keyed by CLUT content (`PaletteKeyHash`/`PaletteKeyEqual`) gives each `Source` a `shared_ptr<Palette>` via `AttachPaletteToSource`, instead of creating and updating a 256×1 palette texture per source. ZoE2, which uses many palettes on few textures, ran 2× faster. A follow-up shares the CLUT copies even when 8-bit textures are disabled, for about 10% more. 9fa1b290b 2018-11 “GSdx-TC: Palette management rework. (#2344)” · 42aee3448 2018-11 “GSdx-TC: Use PaletteMap also when 8-bit texture is disabled by caching only clut copies. (#2681)”

### pcsx2-184 — Pick instruction sets at code-generation time, not build time
Lens: jit-interpreters · Source: PCSX2
Principle: A binary built for the lowest common CPU leaves newer instructions unused. A run-time code generator can check CPUID and emit the better instruction sequence, so one portable build still gets most of the speed.
Why it is faster: —
Applies elsewhere: function multiversioning (`target_clones`). Runtime dispatch in BLAS libraries and simdjson. WASM SIMD feature detection with fallback.
Watch out: —
Seen in PCSX2: `GSDrawScanlineCodeGenerator` replaced `#if _M_SSE >= 0x401` blocks with `if (g_cpu.has(util::Cpu::tSSE41))`, so an SSE2 build still emits `pblendw`/`pmovzxbw` on SSE4.1 CPUs. Follow-ups did the same for AVX and for SetupPrim. 6b78b8f9c 2016-11 “gsdx sw JIT: dynamically select SSE41 at runtime even on SSE2 build (scanline)”

### pcsx2-185 — Specialize generated prologues and prefer 32-bit operations
Lens: jit-interpreters · Source: PCSX2
Principle: Generated code knows its configuration, so it can skip saving, loading and restoring registers that this variant never uses. On x86-64, 32-bit operations zero-extend for free and encode shorter than 64-bit ones and sign-extending moves.
Why it is faster: —
Applies elsewhere: shader permutations that leave out unused bindings. Query-specialized code in database JITs. 32-bit indices in x64 hot loops.
Watch out: —
Seen in PCSX2: The x64 AVX scanline generator now saves, loads and restores r14 and r15 only `if (need_clut)` or `if (need_tex)`. Several `mov`/`and`/`shl`/`movsxd` sequences on 64-bit registers became `ebx`/`ebp`/`a1.cvt32()` forms. 141c9e9c8 2016-11 “gsdx sw x64: prefer faster 32 bits operation when possible”

### pcsx2-186 — Share one constant table across threads, allocated low in memory
Lens: memory-layout · Source: PCSX2
Principle: Per-instance copies of constant tables waste cache, because every thread pulls in its own copy. A single shared table stays hot. If it lives in the low 2 GB, generated x64 code can use absolute 32-bit addresses instead of keeping a base pointer in a register.
Why it is faster: —
Applies elsewhere: shared read-only lookup tables in thread pools. `-mcmodel=small` placement for hot data. Constant pools in JITs.
Watch out: —
Seen in PCSX2: `GSScanlineConstantData` holds `m_test_*`, `m_shift_*` and `m_log2_coef_*` as POD arrays in one global `g_const`, allocated on the heap. The comment explains that static storage may be relocated above 2 GB, while a small heap allocation can land below it and allow absolute addressing. 3b5bc9c38 2016-11 “gsdx sw: create a constant buffer”

### pcsx2-187 — Pass helper arguments in registers from generated code
Lens: jit-interpreters · Source: PCSX2
Principle: When generated code calls runtime helpers, a register calling convention avoids stack pushes and pops. A single emitter helper that places the arguments keeps the convention consistent at every call site.
Why it is faster: —
Applies elsewhere: `__fastcall`/`regparm` for hot callbacks. Trampolines in interpreters that pass arguments in registers.
Watch out: —
Seen in PCSX2: `xCALL` sites across iCOP0, iFPU, iR3000A, iR5900 and microVU became `xFastCall(func, arg1, arg2)`, for example `xFastCall(WriteCP0Status, g_cpuConstRegs[_Rt_].UL[0])` in place of a manual `xMOV(ecx, …)` followed by `xCALL`. e3d5eb5a4 2016-01 “core: convert xCALL to xFastCall”

### pcsx2-188 — Let the compiler target the real CPU and assume strict aliasing
Lens: compiler-build · Source: PCSX2
Principle: Default build flags target an old baseline and disable assumptions that enable optimization. Building for the host architecture, doing float math in SIMD registers, and allowing strict-aliasing optimizations all give the compiler more room.
Why it is faster: —
Applies elsewhere: `-march=native` for self-hosted services. `RUSTFLAGS=-C target-cpu=native`. Removing `-fno-strict-aliasing` after cleaning up type punning.
Watch out: —
Seen in PCSX2: CMake defaulted to `-march=native` (with `DISABLE_ADVANCE_SIMD` as an opt-out), added `-mfpmath=sse` to the common flags, and dropped `-fno-strict-aliasing` from GSdx after fixing the aliasing violations. 679fa65b8 2014-10 “cmake: By default optimize for current arch” · 253e801f5 2016-02 “linux: enable float operation on SSE unit” · 0417a1048 2016-07 “gsdx: allow strict aliasing optimization”

### pcsx2-189 — Read configuration once, not on every hot call
Lens: laziness · Source: PCSX2
Principle: Configuration lookups usually involve string keys, parsing or file access. Doing one inside a frequently called function adds that cost on every call. Read the value once at setup and store it in a member.
Why it is faster: —
Applies elsewhere: read feature flags at request start instead of inside loops. Cache `getenv` results. Inject config into constructors.
Watch out: —
Seen in PCSX2: `GSDevice11` read `GetConfigI("upscale_multiplier")` inside the viewport-setting path. The value moved to an `isNative` member set in `Create`, which fixed a slowdown in games that change the viewport constantly. c530858df 2016-06 “GSdx: Move "isNative" check to DX11 context creation. Resolves speed regression in games that constantly change the viewport size.”

### pcsx2-190 — Query device properties once and cache the results
Lens: io · Source: PCSX2
Principle: System calls to a device are slow, and the same static properties often get queried repeatedly. Query everything once when the medium changes, store the values, and leave out handshakes the operation doesn't need.
Why it is faster: —
Applies elsewhere: cache `stat` results in build tools. Cache USB descriptors after enumeration. Read disk geometry once at mount.
Watch out: —
Seen in PCSX2: `IOCtlSrc` (cdvdGigaherz) replaced its scattered `*Cached` flags with `RefreshDiscInfo`/`ReadDVDInfo`/`ReadCDInfo`, which read media type, sector count and layer break once per disc. Later, the `IOCTL_DVD_START_SESSION`/`END_SESSION` pair around the physical-descriptor read was removed as unnecessary. 1f60e3101 2016-10 “cdvdgigaherz: Reduce disc info ioctl calls” · 5c158b4df 2018-01 “cdvdgigaherz:windows: Avoid unneeded DVD ioctls”

### pcsx2-191 — Look up once and use the iterator; reserve before filling
Lens: algorithms · Source: PCSX2
Principle: A `find` followed by `at` hashes and probes twice, so keep the iterator from `find`. When you know the final size of a vector, reserve it before pushing, which avoids repeated reallocation and copying.
Why it is faster: —
Applies elsewhere: `map.get` instead of `has` followed by `get` in JS. `HashMap::entry` in Rust. `ArrayList(capacity)` in Java.
Watch out: —
Seen in PCSX2: Accelerator lookups in `AppMain.cpp`, `FrameForGS.cpp` and `GlobalCommands.cpp` replaced `find(k) != end()` followed by `at(k)` with one `find` and `iter->second`. `GSLocalMemory::GetPage2TileMap` calls `p2t[page].reserve(m.size())` before filling. ba7b0612d 2014-07 “Use the iterator to get the value from maps to avoid two look ups.” · f8e89b694 2017-05 “GSdx GSLocalMemory: Reserve space required in the std::vector before adding elements to it.”

### pcsx2-192 — Make generated code visible to the profiler
Lens: other · Source: PCSX2
Principle: Sampling profilers can't name code produced at run time, so JIT hot spots show up as unknown addresses. Write a symbol map from each generated block's address and size to a guest-level name, so tools can attribute samples correctly.
Why it is faster: —
Applies elsewhere: `/tmp/perf-PID.map` from Node (`--perf-basic-prof`) and the JVM (perf-map-agent). The JIT profiling APIs of VTune and Tracy.
Watch out: —
Seen in PCSX2: `common/src/Utilities/Perf.cpp` adds `Perf::Info{m_x86, m_size, m_symbol}` lists that write `/tmp/perf-<pid>.map`, mapping x86 ranges to EE, IOP or VU block PCs. The recompilers register their blocks with it. 92d763926 2015-12 “profiler: add functions to plug linux Perf Event on JIT”

### pcsx2-193 — Use a byte shuffle as a 16-entry lookup table
Lens: simd · Source: PCSX2
Principle: When the table is small (16 entries or fewer) and the indices are 4 bits wide, keep the table in vector registers and use a byte shuffle (`pshufb`/`vpshufb`) as the lookup. One instruction does 16 or 32 lookups at once, where a gather or scalar loop needs a memory access per element.
Why it is faster: —
Applies elsewhere: hex encoding and decoding, popcount by nibble, base64 and UTF-8 validation, mapping 4-bit codes to colours in image decoders.
Watch out: —
Seen in PCSX2: `GSBlock::ReadAndExpandBlock4_32` used to expand 4-bit texels by gathering from a 64-bit palette (`gather64_8`). The new `LoadPalVecs` splits the 16 palette entries across four vectors, one byte plane each. `ReadClut4AndWrite` then does `p0.shuffle8(idx)` through `p3.shuffle8(idx)` and interleaves the results with `sw8`/`sw16`. 679cc7af3 2021-04 “GS: pshufb-based ReadAndExpandBlock4_32”

### pcsx2-194 — Choose against hardware gather on CPUs where it is microcoded
Lens: simd · Source: PCSX2
Principle: The same vector instruction can cost very different amounts on different microarchitectures. On some CPUs, gather is split into dozens of micro-ops and is slower than plain scalar loads. Detect the CPU at startup and pick the implementation through a function pointer, so the fast path is not a pessimisation elsewhere.
Why it is faster: —
Applies elsewhere: runtime dispatch in codecs and compression libraries (choosing kernels by CPUID), choosing among AVX-512 and AVX2 paths in database engines, JS engines turning off an optimisation on CPUs known to be slow at it.
Watch out: —
Seen in PCSX2: `ReadAndExpandBlock8_32HSW` unswizzles to a stack block and then does `GSVector4i(pal[s[0]], pal[s[1]], ...)` in place of `VPGATHERDD`. The `GSLocalMemory` constructor uses `Xbyak::util::Cpu` to flag Haswell model IDs and all AMD Zen as `slowVPGATHERDD`, and installs the HSW variants in `m_psm[].rtxbP`. a6887715c 2021-04 “GS: Faster palette lookup for AVX2 platforms with slow VPGATHERDD”

### pcsx2-195 — Match vector code to each instruction's real cost
Lens: simd · Source: PCSX2
Principle: Some vector operations cost nothing extra and others are expensive on particular cores. Store a 128-bit constant once and broadcast-load it, since that is free on AVX2. Build a 256-bit register from two 128-bit loads rather than one load plus a cross-lane permute, which is very slow on early Zen. Fold mask-and-shift into the order that needs one constant, and drop the alignment special cases once unaligned loads cost the same as aligned ones.
Why it is faster: —
Applies elsewhere: image-processing and codec kernels tuned per core, JSON parsers avoiding cross-lane shuffles, and SIMD string search that drops its alignment prologue on modern CPUs.
Watch out: —
Seen in PCSX2: `GSBlock`: `m_r16mask` shrank from a `GSVector8i` to a `GSVector4i`, used through `GSVector8i::broadcast128`. A new `LoadSW128` replaces `load; load; sw128` (`vperm2i128`) with two half loads. `Expand16to32` became `(c << 3) & rmask` with dedicated L/H variants. `WriteColumn32` lost its 32/16/8-byte alignment branches behind `FAST_UNALIGNED`. 1f6b2e629 2021-04 “GS: Use broadcast loads on AVX2” · cbf4a83b3 2021-04 “GS: Minor Zen optimizations” · 244a4da28 2021-04 “GS: ReadBlock16 performance improvements” · 4139da82b 2021-04 “GS: Remove special casing alignment on AVX+”

### pcsx2-196 — Vectorize short loops that decode fields one at a time
Lens: simd · Source: PCSX2
Principle: A loop that pulls 4-bit or 8-bit fields out of a word one by one, tests each one and stores it, can be done with a few vector instructions: mask, unpack nibbles, compare, movemask. Saturating vector add and subtract also replace per-channel clamps.
Why it is faster: —
Applies elsewhere: unpacking bit-packed column data in analytics engines, RGBA8888-to-RGB565 conversion, protocol header parsing with movemask-based flag detection.
Watch out: —
Seen in PCSX2: `Gif_Tag::analyzeTag` now loads the 64-bit REGS field, masks it to `nRegs` nibbles, and splits the nibbles with `_mm_unpacklo_epi8`. It sets `hasAD` through `_mm_movemask_epi8(_mm_cmpeq_epi8(...))`. `ipu_dither_sse2` dithers with `_mm_adds_epu8`/`_mm_subs_epu8` and packs to 16 bpp. d0039c292 2022-03 “Gif_Unit: Vectorize analyzeTag()” · c533a6576 2019-10 “pcsx2: Add IPU dither block SSE2 implementation”

### pcsx2-197 — Don't vectorize scalar work
Lens: simd · Source: PCSX2
Principle: Putting independent scalars in vector lanes, or in an array that gets indexed, adds insert, extract and spill traffic and blocks register allocation. Use named scalar locals and plain field writes when the operation touches one lane.
Why it is faster: —
Applies elsewhere: geometry code that keeps a vec4 only to update one coordinate, and small fixed arrays in hot functions that compilers keep in memory.
Watch out: —
Seen in PCSX2: In `GSRasterizer::DrawTriangle`, `GSVertexSW2 dv[3]` became `dv0`/`dv1`/`dv2`. `edge.p.insert32<0,1>(...)` became `edge.p.y = vertex[...].p.x`, and `edge.p.xxzw()` became `edge.p.y = edge.p.x`. 56bba522a 2022-05 “GS:SW: Unvectorize unneccessarily-vectorized things”

### pcsx2-198 — Pack several divisions into one vector divide
Lens: algorithms · Source: PCSX2
Principle: Division has long latency and low throughput. When two or three values each need dividing by their own divisor, put the numerators in one vector and the divisors in another, and divide once. You get all the quotients for the cost of a single divide.
Why it is faster: —
Applies elsewhere: perspective divide for several vertices at once, normalising several vectors, computing ratios in a batch of statistics.
Watch out: —
Seen in PCSX2: `GSVertexTrace` with accurate STQ used to do `stq0.xyww() / q.xxxx()` and `stq1.xyww() / q.zzzz()` separately. It now computes `st = stq0.xyxy(stq1) / stq0.wwww(stq1)` once and shuffles the halves back out. A `noopt()` asm barrier stops clang from undoing the change. 2e1d14713 2020-11 “GS: Faster accurate_stq calculations”

### pcsx2-199 — Compute addresses incrementally, not through cached offset tables
Lens: algorithms · Source: PCSX2
Principle: A cache of per-configuration lookup tables costs a hash lookup, pointer chasing and cache pollution on every use. If the address formula is cheap, compute it inline. Make the format a compile-time constant wherever it is known, and use a helper that computes the row base once and only adds the column term per element.
Why it is faster: —
Applies elsewhere: tiled or Morton-order texture addressing, strided tensor indexing that hoists row offsets, and image libraries that specialise on pixel format at compile time.
Watch out: —
Seen in PCSX2: `GSLocalMemory::GetOffset` returned a heap-cached `GSOffset*` holding `pixel.row`/`pixel.col` arrays. `GSOffset` is now a small value type built by `GSOffset::fromKnownPSM(bp, bw, PSM_PSMT8)`. Loops use `paMulti()`/`bnMulti()` helpers that step per X, and `readWriteHelper` recomputes the row only at line ends. 8eb50c351 2021-02 “GS: Switch to new non-cached GSOffset” · 39c7f11b9 2021-03 “GS: Pixel loop optimizations” · 59fd815c3 2021-03 “GS: Faster GSOffset::PAHelper”

### pcsx2-200 — Skip the zero taps in polyphase filters
Lens: algorithms · Source: PCSX2
Principle: Half-band and interpolation filters have coefficients that are zero at every other tap, and upsampling buffers are half zeros. Loop only over the non-zero taps, store only the real samples, and select the phase by parity. That halves the multiply-adds.
Why it is faster: —
Applies elsewhere: audio resamplers, 2× image upscalers, and decimation stages in software-defined radio.
Watch out: —
Seen in PCSX2: In the SPU2 `Reverb.cpp` downsampler, the tap loop now steps `i += 2` and adds the centre tap (19) separately. `ReverbUpsample(bool phase)` stores samples at half rate in `RevbUpBuf[R][pos >> 1]`. It uses only the centre tap on one phase and only the even coefficients on the other. 458f22d21 2021-02 “SPU2: Optimize reverb downsampling.” · 64c526ede 2021-02 “SPU2: Optimize reverb upsampling”

### pcsx2-201 — Use a FIFO ring, not a scan for enabled slots
Lens: algorithms · Source: PCSX2
Principle: If entries complete in the order they were issued, keep them in a ring buffer with read and write positions and a count. Retiring work then means checking only the head and stopping at the first entry that is not ready. Scanning every slot for an `enable` flag and the oldest timestamp costs far more.
Why it is faster: —
Applies elsewhere: timer queues where every timer has the same duration, in-order retirement in CPU simulators, network send windows.
Watch out: —
Seen in PCSX2: The VU interpreter's `fmacPipe fmac[8]` and `ialu[8]`, which were scanned twice per flush to find a start position, became 4-entry rings with `fmacreadpos`/`fmacwritepos`/`fmaccount`. `_vuFMACflush` walks from the read position and returns at the first unfinished entry. The commit reports about 2× interpreter speed. 83143bd42 2021-09 “VU Int: Rewrote most of the FMAC/IALU handling, now with 2x performance”

### pcsx2-202 — Search a dense array of key-and-pointer pairs
Lens: memory-layout · Source: PCSX2
Principle: A linear search through a linked list of large nodes touches one cache line, and suffers a dependent miss, for each candidate. Keep a separate contiguous array of `{key, pointer}` pairs: eight fit in a 64-byte line, the hardware prefetcher streams them, and only the match is dereferenced. Compare the key as one 64-bit word instead of two 32-bit halves, and move hits on the slow list to the front.
Why it is faster: —
Applies elsewhere: small hash-bucket arrays in Swiss tables, inline caches in VMs, routing tables with packed prefixes.
Watch out: —
Seen in PCSX2: microVU `microBlockManager::search` walked `qBlockList` nodes and compared `pState.quick32[0..1]`. It now scans `std::vector<microBlockLinkRef> quickLookup` of `{microBlock*, u64 quick}`. The full-compare list moves the matched link to the head. ab295f0f1 2022-10 “x86/microVU: Add a reference list for quick block lookups” · bf95193d5 2022-10 “x86/microVU: Use 64-bit comparisons for quick lookup”

### pcsx2-203 — Compare large structs with wide vector compares and an early exit
Lens: simd · Source: PCSX2
Principle: To test two fixed-size records for equality, use the widest compares available (256-bit), AND the masks together, and check once with movemask. Test the first chunk on its own first, so mismatches exit early. Choose the width at runtime, so one binary takes the wide path on CPUs that support it.
Why it is faster: —
Applies elsewhere: memo caches keyed on structs, change detection in UI state diffing, deduplicating fixed-size records.
Watch out: —
Seen in PCSX2: `mVUsearchXMM` is a JIT-emitted comparator for the 160-byte `microRegInfo`. It went from ten SSE `PCMPEQD` blocks to `VMOVUPS`/`VPCMPEQD` on ymm, with an early `xForwardJB8` after the first 32 bytes. Block links are now 32-byte aligned. A later change emits AVX2 or SSE based on host CPU features instead of the build flag. 197d4d1c8 2022-10 “x86/microVU: Use AVX2 for full block comparisons” · e0a0e0b00 2022-10 “microVU: Choose AVX2/SSE at runtime”

### pcsx2-204 — Don't switch contexts for slices too small to do useful work
Lens: jit-interpreters · Source: PCSX2
Principle: Entering and leaving a component (a JIT, a coroutine, a worker) has a fixed cost. Record how much budget the next unit of work needs and skip the switch until that much has accumulated. When exact timing is not required, run in a minimum quantum, so a three-cycle slice does not pay the full entry cost.
Why it is faster: —
Applies elsewhere: batching small tasks in a work-stealing scheduler, Nagle-style coalescing, game loops that step physics only after a minimum timestep.
Watch out: —
Seen in PCSX2: microVU now writes `nextBlockCycles` at each block exit. `BaseVUmicroCPU::ExecuteBlock` returns early if `delta < nextBlockCycles`, and the EE JIT subtracts it before calling into VU0. `CalculateMinRunCycles` runs VU0 for at least 16 cycles, except inside interlocked COP2 blocks (`s_nBlockInterlocked`). d8dfe0a1e 2021-08 “VU: optimise entering VU JITs” · 6dc5087cb 2022-02 “VU: Run sync ahead on small blocks”

### pcsx2-205 — Key compiled-code caches by the entry the code was compiled from
Lens: jit-interpreters · Source: PCSX2
Principle: If the cache is indexed by where execution resumes rather than where the compiled unit starts, a resumed run misses and recompiles code it already has. Keep the original start address and use it as the lookup key, then pick the block to resume from inside that program.
Why it is faster: —
Applies elsewhere: shader caches keyed by pipeline rather than by draw, template caches keyed by source file rather than by render call site.
Watch out: —
Seen in PCSX2: VU registers gained `start_pc`, set through `SetStartPC` when a microprogram is launched. `mVUsearchProg` indexes `mVU.prog.quick[]`/`prog[]` by `start_pc/8` and then sets `quick.block = cur->block[startPC/8]`, so a partial run reuses the cached program. 4595aae0d 2020-12 “mVU: Keep start PC, modify prog search to avoid recompilation Also fix some M-Bit stuff”

### pcsx2-206 — A minimal setjmp/longjmp that saves only callee-saved registers
Lens: platform · Source: PCSX2
Principle: The C library's `setjmp`/`longjmp` may save signal masks, shadow stacks or SEH state, and on some platforms it unwinds frames. When you only need a non-local exit from generated code, a few lines of assembly that store and restore the callee-saved registers, the stack pointer and the return address are much cheaper.
Why it is faster: —
Applies elsewhere: coroutine and fiber libraries, interpreters that use longjmp for exceptions, parser error recovery.
Watch out: —
Seen in PCSX2: New `common/FastJmp.asm` and `FastJmp.cpp` provide `fastjmp_set`/`fastjmp_jmp` for x86-32 and x86-64. The x86-32 version saves eip, ebx, esp, ebp, esi and edi. `iR5900-32.cpp` switched `m_SetJmp_StateCheck` from `jmp_buf`/`longjmp` to `fastjmp_buf`. 4594e0281 2021-09 “Common: Add Fastjmp” · 8e1470f63 2021-09 “iR5900: Use fastjmp instead of longjmp”

### pcsx2-207 — Semaphores with a userspace fast path
Lens: concurrency · Source: PCSX2
Principle: Put an atomic counter in front of a kernel semaphore. Post increments it and makes a syscall only if a waiter is parked (the old value was negative). Wait decrements it and sleeps only if the count goes below zero. When nothing is contended, neither side enters the kernel, and acquire/release ordering is all that is needed.
Why it is faster: —
Applies elsewhere: futex-based mutexes, Go's runtime semaphores, lightweight semaphores in lock-free queue libraries.
Watch out: —
Seen in PCSX2: `DarwinSemaphore` called `semaphore_signal`/`semaphore_wait` on every Post and Wait. Now `if (__atomic_fetch_add(&m_counter, 1, __ATOMIC_RELEASE) < 0) semaphore_signal(...)`. Later, `Threading::UserspaceSemaphore` wraps `KernelSemaphore` the same way and replaces it for MTGS `m_sem_Vsync`, `m_sem_OnRingReset` and MTVU `semaXGkick`. cf02893d2 2021-02 “macOS: faster semaphores” · faf750a54 2022-11 “GS: Use semaphores with a fast userspace path”

### pcsx2-208 — Try the wait before releasing a lock to sleep
Lens: concurrency · Source: PCSX2
Principle: If a thread drops a lock, blocks, and takes the lock back, every wait wakes the lock's other waiters for nothing. Try a non-blocking acquire of the event first, and do the release, block and reacquire sequence only if it fails.
Why it is faster: —
Applies elsewhere: condition-variable loops that check the predicate before waiting, connection pools that try an idle connection before queueing.
Watch out: —
Seen in PCSX2: In `MTGS.cpp`, the XGKICK wait was `busy.PartialRelease(); semaXGkick.WaitWithoutYield(); busy.PartialAcquire();`. It is now guarded by `if (!vu1Thread.semaXGkick.TryWait())`. The commit says the GS thread was spending about 8% of its time waking the VU thread for nothing. 85f1acb7b 2021-03 “MTGS: Don't unlock mutex if you're just going to lock it again”

### pcsx2-209 — Spin briefly before sleeping on a work queue
Lens: concurrency · Source: PCSX2
Principle: When producer and consumer hand off work at high frequency, sleeping on a kernel object after every item makes threads ping-pong through the scheduler. Spin for a bounded time (microseconds) with pause instructions first. Encode running, spinning and sleeping in one atomic state word, so a producer signals the kernel only when the worker is really asleep.
Why it is faster: —
Applies elsewhere: adaptive mutexes in glibc and the JVM, game-engine job systems, low-latency message queues such as LMAX Disruptor wait strategies.
Watch out: —
Seen in PCSX2: The software renderer's `GSJobQueue` worker now keeps draining while `waited < SPIN_TIME_NS`, accumulating `ShortSpin()`, and `Wait()` spins on `IsEmpty()` before blocking. `Threading::WorkSema` has states `STATE_SPINNING`/`SLEEPING`/`RUNNING_N` and `WaitForWorkWithSpin`. It replaced `Semaphore` plus two ring-buffer mutexes in MTGS and `SysThreadBase`. c3e5b4225 2021-03 “GSdx:SW: Prevent thread ping-pong in software renderer sync” · d73373095 2022-03 “Common: Add WorkSema” · 1a0a0423e 2022-03 “Core: Switch MTGS and SysThreadBase to WorkSema”

### pcsx2-210 — Poll with a short sleep, not yield
Lens: concurrency · Source: PCSX2
Principle: `yield()` returns at once when no other thread is runnable, so a yield-based polling loop burns a whole core and competes with the threads doing real work. Sleeping for 1 ms frees the core at the cost of slightly more latency.
Why it is faster: —
Applies elsewhere: background poll loops for sockets or hardware status, file watchers, test harnesses that wait on a condition.
Watch out: —
Seen in PCSX2: The DEV9 network `RxThread` in `net.cpp` replaced `std::this_thread::yield()` with `std::this_thread::sleep_for(1ms)` when no packet is available. 3a877a51b 2021-05 “DEV9: Sleep instead of yield in RxThread”

### pcsx2-211 — Pin the main pipeline threads to separate cores
Lens: concurrency · Source: PCSX2
Principle: A few long-running threads that hand data to each other run better when each one stays on its own physical core. They keep warm caches, avoid migration, and don't land on SMT siblings of each other. Choose the cores from the processor topology.
Why it is faster: —
Applies elsewhere: audio callback threads, DPDK and low-latency trading cores, game engines that pin render and game threads.
Watch out: —
Seen in PCSX2: `VMManager` gained `AffinityControlMode`, which orders the EE, VU and GS threads onto distinct processors using `cpuinfo` topology. Software-renderer worker threads in `GSRasterizer.cpp` are placed according to the same setting. 543fb282f 2021-10 “VMManager: Set affinities for threads”

### pcsx2-212 — Make a synchronous decoder asynchronous with a read-ahead thread
Lens: io · Source: PCSX2
Principle: Put a thread in front of a blocking read-and-decompress API. It serves requests from a small set of decoded buffers and, while idle, decodes the chunks after the last request. Sequential reads then hit already-decoded data. Make each buffer at least as large as the largest read the caller treats as sequential, and cancel read-ahead as soon as a real request arrives.
Why it is faster: —
Applies elsewhere: streaming video and archive readers, database scans with prefetch, asset loaders for compressed game packs.
Watch out: —
Seen in PCSX2: `ThreadedFileReader` runs `Loop()` on `m_readThread` with two `Buffer`s and `TryCachedRead`. The CSO and CHD readers implement only `ChunkForOffset`/`ReadChunk`. A follow-up keeps filling a buffer with consecutive chunks up to `MINIMUM_SIZE = 128 KiB` and stops when `m_requestPtr` is set. bdcfcc65e 2021-03 “CDVD: Add ThreadedFileReader” · d4f87112e 2021-03 “CDVD: Add minimum buffer size to ThreadedFileReader”

### pcsx2-213 — Ring-buffer heap for producer/consumer data that dies in FIFO order
Lens: allocation · Source: PCSX2
Principle: When one thread allocates per-job data and worker threads free it in roughly the order it was made, a bump-pointer ring heap with atomic usage counters replaces general `malloc`/`free` and `shared_ptr` control blocks. Allocation is a pointer bump, and freeing is an atomic decrement.
Why it is faster: —
Applies elsewhere: per-frame transient allocators in renderers, log record buffers, message arenas in actor systems.
Watch out: —
Seen in PCSX2: New `GSRingHeap` with `alloc`, `free` and `make_shared`, tracking per-quadrant usage in `std::atomic<size_t> m_usage[]`. `GSRendererSW` now allocates `SharedData`, vertex/index `buff`, `clut` and `dimx` from `m_vertex_heap` rather than `new`/`_aligned_malloc`, and the job queue holds `GSRingHeap::SharedPtr`. 342170b07 2021-05 “GS: Add new heap for SW renderer allocations” · 6d4713e06 2021-05 “GS: Alloc sw renderer things on custom heap”

### pcsx2-214 — Over-align only hot allocations
Lens: allocation · Source: PCSX2
Principle: Aligned allocation gets slower as the alignment grows, because the allocator over-allocates and searches for a fitting address. Use cache-line alignment only for objects on hot paths, and minimal alignment for placeholder objects that are never accessed in loops.
Why it is faster: —
Applies elsewhere: hash-table sentinel buckets, empty-collection singletons, pool headers.
Watch out: —
Seen in PCSX2: In `newVif_HashBucket.h`, empty buckets were allocated with `_aligned_malloc(sizeof(nVifBlock), 64)` and now use 16. The commit says 64-byte memalign was noticeably slower. 314c84029 2022-11 “VIF: Reduce alignment of empty hash buckets”

### pcsx2-215 — Use small fixed buffers on the stack and stream large inputs
Lens: allocation · Source: PCSX2
Principle: A heap allocation for a small fixed-size scratch struct costs more than the work done with it. Use a stack array. For a whole-file conversion, process one fixed-size chunk at a time in a stack buffer, not one buffer the size of the file.
Why it is faster: —
Applies elsewhere: `stat`/`readdir` wrappers, checksum passes over large files, embedded firmware without a heap.
Watch out: —
Seen in PCSX2: The memory-card RAW conversion in `MemoryCardFile.cpp` did `malloc(size)` for the whole card. It now loops over 512-byte pages in `u8 buffer[512]`. In `IopBios.cpp`, HostFS `dread`/`getstat` replaced `new char[sizeof(fio_dirent_t)]` with stack arrays. 3a054b9f9 2020-11 “Avoid malloc when converting to RAW” · 1606a50bb 2021-04 “HostFS: Improving buffers allocation”

### pcsx2-216 — Allocate per-object side tables only when a feature uses them
Lens: laziness · Source: PCSX2
Principle: A large inline array in every instance costs memory, and time to zero it, even when the path that reads it is disabled. Store it behind a pointer that is allocated on first use, and test the pointer before using it.
Why it is faster: —
Applies elsewhere: per-node metadata maps in compilers, optional dirty bitmaps on image tiles, lazily created listener lists.
Watch out: —
Seen in PCSX2: `GSTextureCache::Source::m_valid` was `u32 m_valid[MAX_PAGES]`, memset on every construction. It is now `std::unique_ptr<u32[]>`, created in the non-preload update path. `InvalidateVideoMem` checks `if (valid && !s->CanPreload())`. b389c6e10 2022-01 “GS: Only allocate valid page array when needed”

### pcsx2-217 — Put JIT constants in a static array next to the code
Lens: memory-layout · Source: PCSX2
Principle: Generated code can reach data within ±2 GB through 32-bit RIP-relative displacements. A heap buffer may be placed out of that range, which forces 64-bit address loads. A static array in the binary's data section stays close to the code, and also removes the allocation and failure path.
Why it is faster: —
Applies elsewhere: constant pools in JIT compilers, PC-relative literal pools on ARM, small lookup tables in embedded code kept in internal SRAM.
Watch out: —
Seen in PCSX2: In `iR5900-32.cpp`, `recConstBuf` was `_aligned_malloc`ed in `recAlloc`. It became `alignas(16) static u32 recConstBuf[RECCONSTBUF_SIZE]`. f3b17cf02 2021-06 “iR5900: Move recConstBuf memory near recompiler memory”

### pcsx2-218 — Use a modern hash with a compile-time length
Lens: algorithms · Source: PCSX2
Principle: A hand-rolled hash that processes one 32-bit word per step is slow, and its distribution is unknown. A vectorised hash such as XXH3 is faster and better. Calling it with a constant length in each branch lets the inlined code drop its length dispatch.
Why it is faster: —
Applies elsewhere: hash-map keys over fixed-size structs, content-addressed caches, deduplicating fixed-size blocks.
Watch out: —
Seen in PCSX2: `GSTextureCache::PaletteKeyHash` was a 16-step unrolled add/xor loop per 16 entries. It became `XXH3_64bits(key.clut, 4 * key.pal)`, then `key.pal == 16 ? XXH3_64bits(clut, 64) : XXH3_64bits(clut, 1024)`. 42066ecab 2022-01 “GS: use xxHash to hash palettes.” · 8db0b5282 2022-01 “GS: optimize xxHash usage in palette hashing.”

### pcsx2-219 — Hash the compact source and cache the expensive result
Lens: laziness · Source: PCSX2
Principle: To find out whether an expensive conversion or upload can be reused, hash its compact input, not the expanded output. Keep converted results in a hash-keyed cache so identical content is never converted twice, and cap the cache by memory, falling back to the old behaviour when the cap is exceeded.
Why it is faster: —
Applies elsewhere: build caches keyed by source hash, image-thumbnail caches, GPU resource deduplication in game engines.
Watch out: —
Seen in PCSX2: `GSTextureCache` now hashes GS local-memory blocks (paletted textures hashed with their CLUT) rather than expanded RGBA. The new `TexturePreloadingLevel::Full` keeps a hash cache of uploaded textures, tracks `GetHashCacheMemoryUsage()`, and when usage passes 1 GB calls `RemoveAll()` and drops back to `Partial`. 364199916 2022-01 “GS: Hash local memory blocks instead of expanded textures” · 5d33af13c 2022-01 “GS: Add full (hash cache) texture preloading option” · ee82c7834 2022-01 “GS: Auto-disable hash cache when it's too large”

### pcsx2-220 — Read from the original instead of copying when nothing differs
Lens: gpu-rendering · Source: PCSX2
Principle: If the copy would be identical to the source (same size, same format, no reinterpretation), let the consumer read the source directly and mark it as shared. You save the allocation and the copy bandwidth.
Why it is faster: —
Applies elsewhere: copy-on-write strings and buffers, zero-copy views (`string_view`, slices), sampling a render target in place of blitting it.
Watch out: —
Seen in PCSX2: In `GSTextureCache::CreateSource`, when `use_texture` holds and the dimensions match `dst->m_texture`, it sets `src->m_texture = dst->m_texture; m_shared_texture = true` instead of `CreateTexture` plus `CopyRect`. The earlier OGL version created a dummy texture and set `m_from_target` so the shader samples the framebuffer. 1677ef318 2022-07 “GS/TextureCache: Elide copies when source matches target” · f406051ed 2019-04 “gsdx ogl: Bypass the texture cache when the frame buffer will be sampled.”

### pcsx2-221 — Do transfers where the data already is
Lens: gpu-rendering · Source: PCSX2
Principle: When both the source and destination of a copy already exist in device memory, do the copy on the device. Reading back to the CPU, moving the bytes and uploading again costs two bus transfers and a sync.
Why it is faster: —
Applies elsewhere: server-side copy in object stores (S3 CopyObject), `copy_file_range`/reflinks, GPU-to-GPU blits instead of readback.
Watch out: —
Seen in PCSX2: `GSRendererHW::Move` calls the new `GSTextureCache::Move`. When the source and destination formats match and both are cached targets, the copy is done with `CopyRect` on the GPU, and the local-memory move in `GSState::Move` is skipped. d552f8dc4 2022-04 “GS/HW: Implement moves in texture cache”

### pcsx2-222 — Simplify blend math using known value ranges
Lens: gpu-rendering · Source: PCSX2
Principle: If an operand's range is known per batch (alpha is exactly 0 or 1, or two terms are equal), rewrite `(A−B)·C + D` into a simpler equivalent and drop modes that turn out to be no-ops. The draw can then use the fixed-function path instead of the costly programmable fallback.
Why it is faster: —
Applies elsewhere: SQL planners folding predicates once column statistics are known, shader specialisation constants, removing no-op CSS compositing layers.
Watch out: —
Seen in PCSX2: In `GSRendererNew::EmulateBlending`: if `blend_a == blend_b`, or B==D with alpha exactly 1, the term becomes zero (A=B=C=0). If alpha max is 0, A and B are zeroed. PABE is skipped when `GetAlphaMinMax().max < 128`. On 16-bit targets, an FBMSK touching only unused bits is set to 0. On DX11, clamp-0 colclip uses a cheap `& 0xFF` wrap where it used the HDR path. 22dfe0528 2022-03 “GS-hw: Optimize blending when ALPHA.A is equal to ALPHA.B.” · 14c17916f 2022-03 “GS-hw: Optimize blending equations based on alpha value.” · 04ef9af70 2021-12 “GS-hw: Optimize PABE.” · f84cb1c9c 2022-02 “GS-hw: Optimize fbmask on 16bit format.” · d0cb0f59d 2019-06 “gsdx-d3d11: Extend BLEND_NO_BAR to also hit on clamp 0, it's faster than standard HDR algo.”

### pcsx2-223 — Issue barriers only when a shader reads what it writes
Lens: gpu-rendering · Source: PCSX2
Principle: GPU texture barriers serialise work and can split one draw into one draw per primitive. Work out exactly which shader configurations read the render target (a feedback loop). Emit a barrier only for those, and never combine framebuffer fetch with fixed-function blending when one of them is enough.
Why it is faster: —
Applies elsewhere: Vulkan render graphs that compute minimal pipeline barriers, avoiding `glFinish` or `memory_order_seq_cst` where weaker ordering is enough.
Watch out: —
Seen in PCSX2: `PSSelector::IsFeedbackLoop()` now needs a software-blend term that really references Cd (`(sw_blend_bits | blend_c) & 1`). `GSDeviceOGL::SendHWDraw(config, needs_barrier)` and the Vulkan equivalent skip `glTextureBarrier` and per-primitive splitting otherwise. `GSRendererHW` folds `blend_ad_alpha_masked` into `one_barrier`, so a draw that already needs a barrier does all blending in the shader. de724d957 2022-03 “GS/HW: Skip unnecessary barriers (usually two pass alpha)” · 77471bd9d 2022-05 “GS: Avoid using both blending and fbfetch”

### pcsx2-224 — Use hardware modes when the emulated one is equivalent
Lens: gpu-rendering · Source: PCSX2
Principle: If a general mode that needs shader emulation reduces, for the current parameters, to a mode the hardware supports natively, switch to the native mode. Region-clamp spanning the full texture is plain clamp, and region-repeat with a full mask is plain repeat.
Why it is faster: —
Applies elsewhere: query planners choosing an index scan when a predicate is a simple range, codecs using a hardware decoder when the stream fits its profile.
Watch out: —
Seen in PCSX2: In `GSRendererHW::Draw`, `CLAMP_REGION_CLAMP` with MINU=0 and MAXU=tw−1 becomes `CLAMP_CLAMP`. `CLAMP_REGION_REPEAT` with MINU=tw−1 and MAXU=0 becomes `CLAMP_REPEAT`. Region modes whose UVs never touch the boundary (per `tmm.uses_boundary`) become `CLAMP_CLAMP`. 8c16e3231 2022-01 “GS:HW: Add a few clamp mode optimizations”

### pcsx2-225 — Pack scalar uniforms into vec4 padding
Lens: gpu-rendering · Source: PCSX2
Principle: Uniform buffers are laid out in 16-byte slots, so a lone float in its own vec4 wastes 12 bytes and costs extra upload and comparison. Put scalars into the unused components of existing vectors.
Why it is faster: —
Applies elsewhere: std140 constant-buffer layout in game engines, packing flags into the spare bits of a struct, columnar packing of small fields.
Watch out: —
Seen in PCSX2: The OGL `PSConstantBuffer` merged `MaxDepth` (a separate `GSVector4`) into `TA_Af.z`, renamed `TA_MaxDepth_Af`. That drops one vec4 from the buffer and one lane from the dirty-compare in `Update()`. `common_header.glsl` was updated to match. 6c3e357d9 2021-12 “GS-ogl: Optimize fragment shader uniform buffer.”

### pcsx2-226 — Persist compiled shader binaries keyed by source hash
Lens: gpu-rendering · Source: PCSX2
Principle: Compiling and linking shaders at runtime causes stutter and slows startup. Hash the source of each stage, look the hash up in an on-disk index, and load the driver's program binary on a hit. Compile only on a miss, then append the result to the cache. Add a version constant so the cache is invalidated when the shaders change.
Why it is faster: —
Applies elsewhere: browser GPU process shader caches, Python `.pyc` files, ccache-style build caches.
Watch out: —
Seen in PCSX2: `GL::ShaderCache` stores `CacheIndexEntry` records (MD5 low/high plus length for each of the vertex, geometry and fragment stages, the blob offset and format). `GetProgram` calls `prog.CreateFromBinary(...)` on a hit. `GSDeviceOGL` replaced `Compile()`/`Link()` with `m_shader_cache.GetProgram(...)` and added `SHADER_VERSION`. 69ff12645 2021-12 “Common: Add GL/ShaderCache” · 55695bb5b 2021-10 “GS/OpenGL: Use ShaderCache”

### pcsx2-227 — Pick memory types by access direction
Lens: gpu-rendering · Source: PCSX2
Principle: CPU-to-GPU upload buffers should be write-combined and coherent, so no flushes are needed. GPU-to-CPU readback buffers should be host-cached, because uncached reads are very slow. Textures belong in device-local memory. A dedicated allocator also replaces one allocation per buffer with sub-allocation.
Why it is faster: —
Applies elsewhere: D3D12 upload and readback heaps, DMA buffer cacheability on embedded SoCs, mmap flags for shared memory.
Watch out: —
Seen in PCSX2: `Vulkan::StreamBuffer` moved to VulkanMemoryAllocator with `preferredFlags = VK_MEMORY_PROPERTY_HOST_COHERENT_BIT`. The GS download buffer prefers `HOST_CACHED`, and `Vulkan::Texture` requires `DEVICE_LOCAL`, failing cleanly on out-of-memory. 6187f36c3 2021-12 “Vulkan/StreamBuffer: Use VulkanMemoryAllocator” · 793ba944d 2022-01 “GS/Vulkan: Prefer coherent for upload and cached for download buffers” · 81a139b87 2022-01 “Vulkan: Force textures to be allocated in device local memory”

### pcsx2-228 — Treat a full overwrite as invalidation, not a draw
Lens: laziness · Source: PCSX2
Principle: If an operation provably overwrites a whole buffer with a constant (opaque, unmasked, untested, starting at the origin), don't run it through the expensive pipeline. Record the constant cheaply and mark any cached copies invalid.
Why it is faster: —
Applies elsewhere: `TRUNCATE` rather than `DELETE`, discard or invalidate on render-target load ops, `MADV_DONTNEED` in place of memset-zero.
Watch out: —
Seen in PCSX2: `GSRendererHW::Draw` checks for a sprite with no texture, opaque, `FBMSK == 0`, no ATE/ZTE and constant colour from (0,0). If `OI_GsMemClear()` handled it and the height is over 1024, it calls `InvalidateVideoMem`/`InvalidateVideoMemType` for the colour and depth targets and returns without a GPU draw. 05dd0831c 2022-11 “GS: Avoid large clears in HW”

### pcsx2-229 — Flush batched work only when state that matters has changed
Lens: laziness · Source: PCSX2
Principle: Batching renderers flush the pending batch when state changes. Compare the new state with the state the batch was built under, using XOR under a mask of relevant fields, and keep a bitmask of dirty registers. Flush only when that mask is non-zero and there is pending work. Writes that restore the old value, or that change fields the batch does not use, then cost nothing.
Why it is faster: —
Applies elsewhere: sprite batchers in 2D engines, React's shallow-compare bailout, coalescing database writes.
Watch out: —
Seen in PCSX2: `GSState` replaced `m_reg_draw_dirty`/`m_reg_texture_dirty` with `m_dirty_gs_regs`. Writes to PRIM, TEX0 and others set or clear their bit by comparing with `m_prev_env` (for TEX0, under mask `0x1f78001fffffffff`). `CheckFlushes` is skipped for vertices with the skip bit. `GSClut::WriteTest` now reports a pending CLUT write only if `(TEX0.PSM & 7) >= 3`, that is, a paletted format. 3922091a5 2022-06 “GS: New state opt optimisation pass” · 37ccd8198 2021-12 “GS: Avoid some unnecessary flushes”

### pcsx2-230 — Clip the input range to the visible output
Lens: laziness · Source: PCSX2
Principle: When output will be clipped (scissored), map the clip back to the input coordinates with the known linear gradient. Then you convert, upload or validate only the part of the source that can be seen.
Why it is faster: —
Applies elsewhere: decoding only the visible tiles of a large image, predicate pushdown into scans, virtualised list rendering.
Watch out: —
Seen in PCSX2: `GSState::GetTextureMinMax`, for non-perspective sprites (`FST == 1`), intersects the primitive rectangle with `scissor.in` and shrinks `st` by `(clip delta) * uv_range / pos_range`, unless repeat would wrap. The next commit limits this to single-sprite draws and fixes the repeat check. cf1d67d92 2022-04 “GS: Adjust texture coverage based on scissor” · 85c754b45 2022-05 “GS: Limit scissor optimisation, fix repeat condition”

### pcsx2-231 — Delete duplicate reads, moves and notifications
Lens: other · Source: PCSX2
Principle: Code collects repeats: a table re-read that a callee already performs, a value copied through a temporary register, a cross-thread update sent once per element when once per packet would do. Removing each one saves memory traffic, register pressure or queue traffic on a hot path.
Why it is faster: —
Applies elsewhere: repeated `getComputedStyle` calls, double JSON parses across layers, sending a change event per field instead of per commit.
Watch out: —
Seen in PCSX2: `GSTextureCache::LookupSource` dropped a `m_clut.Read32(TEX0, TEXA)` the caller had already done. The COP2 `QMFC2`/`QMTC2` recompiler emitted `MOVAPS` via a temporary XMM register and now moves directly. VIF STROW/STCOL now notify the MTVU thread only when `tag.size == 0`, that is, once the full row is written. c11ca2ff6 2022-07 “GS/TextureCache: Remove redundant CLUT read” · 7aa05c08f 2022-10 “EEJIT/COP2: Remove redundant instruction/allocation” · c9a544353 2020-12 “VIF: Slightly optimise when Row/Col are written to MTVU”

### pcsx2-232 — Force-inline tiny wrappers and mark leaf classes final
Lens: compiler-build · Source: PCSX2
Principle: Compilers do not reliably inline constructors of SIMD wrapper types or accessors defined in another file. That leaves real calls and register round trips, especially in debug builds. Define the accessors in the header with a force-inline attribute. Mark classes and overrides `final` so calls on the concrete type can be devirtualised.
Why it is faster: —
Applies elsewhere: math-vector libraries in game engines, `sealed` classes in C# and Kotlin for JIT devirtualisation, `#[inline(always)]` on Rust newtype wrappers.
Watch out: —
Seen in PCSX2: `GSVector4`/`4i`/`8`/`8i` `(__m128)` constructors got `__forceinline` because MSVC was not inlining them. `GSDevice::GetCurrent()` moved from the .cpp file to the header as `__fi ... const`. `GSRendererDX11` became `final`, with `DrawPrims(...) final` and private inline helpers. 9a435c1a7 2021-03 “GSdx: Re-add __forceinline to GSVector functions” · 212848e93 2021-12 “GSDevice: Force inline accessor methods” · 7b4133ac0 2019-01 “GSdx-d3d11: Add final to Renderer class and DrawPrims, also change some voids to inline.”

### pcsx2-233 — Allow the compiler to fuse multiply-add
Lens: compiler-build · Source: PCSX2
Principle: Some compilers keep `a*b+c` as two rounded operations by default. Enabling FP contraction lets them emit a single FMA instruction, which halves the instruction count and latency of multiply-add chains (with slightly different rounding).
Why it is faster: —
Applies elsewhere: DSP filters, matrix and physics code, ML inference kernels.
Watch out: —
Seen in PCSX2: `pcsx2/CMakeLists.txt` adds `-ffp-contract=fast` for GCC/Clang (Clang defaults to off) and `/fp:contract` for MSVC 1930+. ced58a047 2022-05 “CMake: Set fp-contract=fast”

### pcsx2-234 — Build constant tables at compile time
Lens: compiler-build · Source: PCSX2
Principle: Tables filled by an `InitVectors()` call at startup cost init time, depend on init order, and cannot live in read-only memory. Write them as `constexpr`/`constinit` initialisers, so they are emitted directly into `.rodata`.
Why it is faster: —
Applies elsewhere: CRC and trig tables, Unicode property tables, perfect-hash tables generated at build time.
Watch out: —
Seen in PCSX2: `GSBlock`, `GSClut`, `GSVector4i` and others went from mutable statics filled in `GS.cpp` (`GSBlock::InitVectors()` etc.) to `const` members built with `cxpr_set1_epi32`/`cxpr` constexpr constructors, such as `const GSVector4i GSBlock::m_r8mask(0, 4, 2, 6, ...)`. 862518e7d 2021-01 “gsdx: Use constexpr to initialize avx vectors without avx instructions”

### pcsx2-235 — Collapse per-field or per-row work into one block operation
Lens: memory-layout · Source: PCSX2
Principle: When elements are contiguous (equal source and destination stride, equal to the row size), copy or compare the whole span with a single `memcpy`/`memcmp`. A small struct compared with `memcmp` compiles to one wide compare and one branch, instead of one per field.
Why it is faster: —
Applies elsewhere: image blits that detect tightly packed rows, ORM dirty checks over POD structs, packet header comparison.
Watch out: —
Seen in PCSX2: `StringUtil::StrideMemCpy`/`StrideMemCmp` gained the fast path `if (src_stride == dst_stride && src_stride == copy_size) memcpy(dst, src, src_stride * count)`. `GSVector2T::operator==` changed from `x == v.x && y == v.y` to `memcmp(this, &v, sizeof(*this)) == 0`. 325542283 2022-03 “Common/StringUtil: Optimize equal stride copy case” · f2e6c61bf 2022-09 “GS: Use memcmp for GSVector2 comparisons”

### pcsx2-236 — Track the file position yourself
Lens: io · Source: PCSX2
Principle: Querying file length or position in a loop condition can hide syscalls (`Length()` is seek-to-end, tell, seek-back). Read the length once and advance your own counter on each read.
Why it is faster: —
Applies elsewhere: log and archive parsers, binary file readers in any language that call `tell()` or `stat()` per record.
Watch out: —
Seen in PCSX2: The `GSDumpDialog` loader did `while (m_dump_file->Tell() < m_dump_file->Length())` per packet. It now caches `length` and updates `pos` through a `READ_FROM_DUMP_FILE` macro, which the commit says saves several seconds on large dumps. c192ecf6b 2021-12 “GSDumpDialog: Cache length instead of querying every packet”

### pcsx2-237 — Keep connections open and batch commands
Lens: io · Source: PCSX2
Principle: Accepting a new socket for every request pays connection setup each time and can exhaust OS socket tables. Keep the connection open and frame messages on it. Let a client send many commands in one message, so round-trip latency is paid once per batch.
Why it is faster: —
Applies elsewhere: HTTP keep-alive, Redis pipelining, batched RPC and GraphQL query batching.
Watch out: —
Seen in PCSX2: `SocketIPC` added `MsgMultiCommand`: `ParseCommand` reads a `u16` batch count and loops over sub-commands into one return buffer. `StartSocket()` now accepts once with send and receive timeouts and keeps reading length-prefixed messages, where it used to `accept` per message. 89ce774d7 2020-08 “IPC: implement batch command processing” · b7bb3daa9 2021-02 “IPC: keep reusing the same socket to avoid windows overfill & better perf”

### pcsx2-238 — Skip expensive operations that would change nothing
Lens: laziness · Source: PCSX2
Principle: Before a costly idempotent operation, check whether the target is already in the requested state. Examples are recompressing a file that is already compressed, recreating a swap chain at the size it already has, and redrawing a widget that is already on screen.
Why it is faster: —
Applies elsewhere: `make`-style up-to-date checks, skipping `setState` when the value is unchanged, idempotent infrastructure apply.
Watch out: —
Seen in PCSX2: `FileSystem::SetPathCompression` returns early if `FILE_ATTRIBUTE_COMPRESSED` already matches. `VulkanHostDisplay::ResizeRenderWindow` skips `ResizeSwapChain` when the width and height are unchanged. `VirtualPadResources` redraws a button only on `widgetUpdateRequired` and clears the screen only when a rendered element goes away. 4543ddbcc 2022-04 “Common/FileSystem: Don't recompress already compressed files” · ca3833e71 2022-06 “VulkanHostDisplay: Avoid redundant resizes” · c89e9134f 2020-04 “recording: Done micro-optimizing GUI”

### pcsx2-239 — Specialise generated kernels on per-batch constants
Lens: jit-interpreters · Source: PCSX2
Principle: If a value is constant across a whole batch (every vertex has the same depth, say), compile or select a kernel variant that treats it as a constant. The variant drops interpolation, per-pixel stepping and the related overflow checks.
Why it is faster: —
Applies elsewhere: shader specialisation constants, query engines compiling predicates with literal values, regex engines specialising on literal prefixes.
Watch out: —
Seen in PCSX2: The software renderer's `sel.zequal` was restored. When it is set, `GSDrawScanline` and the JIT code generator broadcast `zbase` once (`f64toi32`) and skip the per-step `local.d[].z` adds and the `zoverflow` path. 9084ef35b 2022-05 “GS:SW: Restore zequal for optimization”

### pcsx2-240 — Pass and return wide values in registers
Lens: platform · Source: PCSX2
Principle: Returning a 64-bit or 128-bit value through a hidden pointer, or passing it by reference, forces a store to memory and a reload. Use calling conventions (`__vectorcall`, the SysV vector ABI) and types that keep such values in SIMD or 64-bit general registers.
Why it is faster: —
Applies elsewhere: math libraries returning `__m128` by value, Rust and C++ small-struct returns in registers, UUID and hash types passed by value.
Watch out: —
Seen in PCSX2: vtlb `read64`/`read128` handlers and `readCache64`/`128` now return `RETURNS_R64`/`RETURNS_R128` (`r64`/`r128` in SSE registers, `__vectorcall`), where they used to fill a `mem128_t*`. The memory revamp also passes 64-bit and 128-bit arguments by value in registers and uses 64-bit GPRs for LDL/LDR/SDL/SDR. e9518f78c 2021-09 “vtlb: Switch read64 and read128 handlers to return in sse regs” · 00bcb4cf0 2022-10 “System: Revamp memory allocation”

### pcsx2-241 — Reschedule a periodic event only while something consumes it
Lens: laziness · Source: PCSX2
Principle: A self-rescheduling event that fires whether or not its consumer is active turns into a busy loop that steals time from real work. Re-arm it only while the consumer channel is active.
Why it is faster: —
Applies elsewhere: timers that re-arm only while a socket has pending writes, UI animation loops that stop once idle, polling only while a request is outstanding.
Watch out: —
Seen in PCSX2: IPU0 rescheduled `IPU_INT_FROM(64)` unconditionally in `IPUdma.cpp`. Now `IPU_Fifo.cpp` raises it only `if (ipu0ch.chcr.STR)`, which fixed heavy slowdowns in Ratchet games with bad IPU streams. 378eccc6e 2020-12 “IPU: Stop IPU0 looping when there's no data for it to read Fixes bad slowdowns in Ratchet games when using EE Timing fix caused by bad IPU streams”

### pcsx2-242 — Patch the workload's own busy-wait loops
Lens: other · Source: PCSX2
Principle: Sometimes the cost lives in the program being run: spin loops that wait on timing, which are cheap on the original hardware and expensive to simulate. Binary patches that replace the spin with a direct path remove the work entirely.
Why it is faster: —
Applies elsewhere: replacing `while(!ready);` with proper waits in legacy code, hot-patching sleep loops in virtualised guests (paravirtual spinlocks).
Watch out: —
Seen in PCSX2: `GameIndex.dbf` gained `[patches]` for two CRCs (Primal and Ghosthunter). They overwrite microcode wait-loop words with `8000033c` and NOPs, which the commit says were “absolutely unnecessary and significantly kill performance”. a14e3c25b 2020-05 “Performance fix for Primal and Ghosthunter. (#3376)”

### pcsx2-243 — Measure at thread granularity, not per operation
Lens: other · Source: PCSX2
Principle: Timing every operation with `rdtsc` start/stop pairs adds overhead to the hot path. Ask the OS for each thread's CPU time once per reporting interval: it gives the same utilisation figure at almost no cost.
Why it is faster: —
Applies elsewhere: sampling profilers instead of instrumentation, `getrusage`/`QueryThreadCycleTime` in metrics agents.
Watch out: —
Seen in PCSX2: The `GSPerfMon::Start/Stop` calls around each worker draw (`WorkerDraw0..N` timers with `__rdtsc`) were removed. `PerformanceMetrics` now reads per-thread CPU time for each software-renderer thread (`GetGSSWThreadUsage`) once per update. 206f80c5f 2022-03 “GS: Compute SW CPU per-thread not per-draw”

### pcsx2-244 — Keep guest registers in host registers for the life of a block
Lens: jit-interpreters · Source: PCSX2
Principle: A JIT that loads each guest register from memory before every use and stores it after every write spends most of its time on memory traffic. Give the guest registers a slot in the host register allocator, use a liveness pass to decide which are worth holding, and write them back only at block exits or calls.
Why it is faster: —
Applies elsewhere: register promotion of hot locals in bytecode VMs, and scalar replacement of struct fields in an optimizing compiler.
Watch out: —
Seen in PCSX2: microVU kept the 16 VU integer registers (VI) in memory. It now caches them in x86 GPRs, shared with the EE register allocator. A new `X86TYPE_VIREG` class and `_allocIfUsedVItoX86()` allocate a VI only if `EEINST_VIUSEDTEST(vireg)` shows a later use in the block. A `MODE_COP2` flag keeps it off registers reserved for VU0. 00d768a6b 2022-12 “x86/microVU: Add VI caching”

### pcsx2-245 — Preload a block's operands into free registers at entry
Lens: jit-interpreters · Source: PCSX2
Principle: If registers are free when a block starts, scan ahead and load the values the block will read into them up front. Later instructions then find their operands already in registers and do not stall on loads scattered through the body.
Why it is faster: —
Applies elsewhere: software prefetching of loop inputs, and hoisting loads into a loop preheader.
Watch out: —
Seen in PCSX2: `mvuPreloadRegisters()` walks forward through the block until an XGKICK, calling `preloadVF`/`preloadVI` for every register read. It stops when fewer than `REQUIRED_FREE_XMMS = 3` or `REQUIRED_FREE_GPRS = 1` remain free, so temporaries still have room. 4f65ef4d3 2022-12 “x86/microVU: Preload registers at beginning of block”

### pcsx2-246 — Replace emitted store sequences with one shared wide-copy routine
Lens: jit-interpreters · Source: PCSX2
Principle: Emitting one immediate store per word to restore a state struct bloats every block that needs it. Put the data inline as a table, emit one call that passes the table's address, and have a single shared routine copy it with a few wide vector moves. Code shrinks and each copy needs a handful of instructions.
Why it is faster: —
Applies elsewhere: calling a tuned `memcpy` on a constant table instead of unrolled stores, and outlining repeated prologues in compilers.
Watch out: —
Seen in PCSX2: `mVUsavePipelineState` emitted one `xMOV(ptr32[...], imm)` per dword of `microRegInfo`. Blocks now `xLoadFarAddr(rax, ...)` to an aligned inline copy of the state and `xFastCall` `mVU.copyPLState`. That routine, generated once, moves 160 bytes with five `ymm` loads and stores on AVX2, or ten `xmm` moves otherwise. cdadad568 2022-12 “x86/microVU: Use unrolled copy loop for updating PL state”

### pcsx2-247 — Specialize instructions whose source is a hardwired zero register
Lens: jit-interpreters · Source: PCSX2
Principle: When an operand is architecturally always zero, the general instruction does wasted work. Add zero becomes a move, add immediate becomes a constant load, and a load at zero plus an offset becomes an absolute address that the translator computes at compile time.
Why it is faster: —
Applies elsewhere: peephole passes that fold `x + 0`, and RISC-V or MIPS translators that treat `x0`/`$zero` as a constant.
Watch out: —
Seen in PCSX2: In microVU, `IADD` with `_Is_ == 0 || _It_ == 0` now emits a single `xMOV`, and `IADDI`/`IADDIU` from vi00 emit `xMOV(regT, imm)` or `xXOR`. Loads and stores based on vi00 go through `mVUoptimizeConstantAddr()`, which returns a fixed `xAddressVoid`, so the address computation and the `mVUaddrFix` wrap are both skipped. 958467205 2022-12 “x86/microVU: Rewrite IADD from vi00 to mov” · 6018936dc 2022-12 “Optimize loadstores to offsets from vi00”

### pcsx2-248 — Forward a value straight into its consumer's register
Lens: jit-interpreters · Source: PCSX2
Principle: If a value is written to memory only so that a later instruction can read it back, the store and reload are wasted. Put the value directly into the register the consumer reads from, and leave memory stale until it actually has to be flushed.
Why it is faster: —
Applies elsewhere: store-to-load forwarding in compilers, and passing results in registers instead of through out-parameters.
Watch out: —
Seen in PCSX2: An EE `CTC2` to VU0's I register used to write memory and invalidate any cached XMM copy. It now allocates VF slot 33 (I) as an XMM register and broadcasts the GPR into it with `xPSHUF.D`, or `xMOVDZX` followed by `xSHUF.PS`. Later VNNNi instructions read I from that register. 1a7263cb9 2022-12 “x86/microVU: Forward CTC2 I direct to VNNNi instructions”

### pcsx2-249 — Skip a mode-register reload when the mode is unchanged
Lens: jit-interpreters · Source: PCSX2
Principle: Writing a control register such as the FP control word is a serializing, slow instruction. If the translator can tell at compile time that the old and new modes are the same, it should not emit the write.
Why it is faster: —
Applies elsewhere: graphics state trackers that drop redundant `glEnable` calls, and a skipped `fesetround` when the rounding mode already matches.
Watch out: —
Seen in PCSX2: microVU entry and exit always ran `xLDMXCSR`. `mvuNeedsFPCRUpdate()` now compares `g_sseMXCSR.bitmask` with the VU's mask, so the instruction is emitted only when the two differ, or on the VU1 thread, which owns its own MXCSR. 1dff9897c 2023-01 “x86/microVU: Only update MXCSR when it's different”

### pcsx2-250 — Pack small counters and flags into bitfields or spare bits
Lens: memory-layout · Source: PCSX2
Principle: A value that never exceeds 4 does not need a byte, and a pointer-aligned address leaves its low bits unused. Packing values into those bits shrinks structures that are copied, compared or hashed often, which cuts cache footprint and copy cost.
Why it is faster: —
Applies elsewhere: tagged pointers in language runtimes, and 4-bit fields in compact hash-table metadata.
Watch out: —
Seen in PCSX2: The microVU pipeline state stores each VF register's per-lane stall cycles (at most 4) in bitfields, which saves 64 bytes per block state. The block CRC was updated to match. Separately, the EE cache tag moved its `validPFN` bool into bit 11 of the tag's flag bits, which saves 4 bytes per tag. cd9b6c7ac 2023-04 “x86/microVU: Pack VF cycles into bitfields” · 16b1095a7 2025-01 “EE Cache: Shrink tag size by 4 bytes”

### pcsx2-251 — Fast-forward idle countdown loops to the next event
Lens: jit-interpreters · Source: PCSX2
Principle: A guest loop that only decrements a counter until it reaches zero burns host time while doing nothing observable. Detect it, compute how many iterations fit before the next scheduled event, advance the cycle counter and the loop register in one step, and exit.
Why it is faster: —
Applies elsewhere: idle-loop skipping in other emulators, and discrete-event simulators that jump straight to the next timestamp.
Watch out: —
Seen in PCSX2: `recSkipTimeoutLoop()` emits `new_cycles = min(v0*8 + cycle, nextEventCycle)` using `LEA` and `CMOVB`. It writes the cycle count back and subtracts `(new_cycles - cycle) >> 3` from v0. If v0 reaches zero it links straight to the loop exit; otherwise it jumps to `DispatcherEvent`. b3e6e2882 2023-04 “x86/iR5900: Fast forward timeout loops”

### pcsx2-252 — Inline a rare-case check as a branchless compare
Lens: branching-fast-paths · Source: PCSX2
Principle: Calling a helper only to test whether two inputs match a magic pair costs a call, argument setup and register saves on every execution. Emit `(a ^ K1) | (b ^ K2)` and one conditional jump instead, so the common path runs a few ALU instructions.
Why it is faster: —
Applies elsewhere: comparing several fields at once with XOR/OR, as in SWAR string compares and header-magic checks.
Watch out: —
Seen in PCSX2: `FPU_MUL_HACK`, a game-specific special case, was a C function called from every recompiled FPU `MUL`. The JIT now emits `xXOR(ecx, 0x3e800000); xXOR(edx, 0x40490fdb); xOR(edx, ecx); JNZ8` and loads the result from an aligned constant. As a side effect this fixed a clang crash in which the callee clobbered r8. 28b111b66 2023-03 “x86/iFPU: Inline FPU_MUL_HACK to dynarec”

### pcsx2-253 — Drop frame setup for code you never return through
Lens: jit-interpreters · Source: PCSX2
Principle: If JIT code is entered once and always left with a long jump, its entry stub does not need to save callee-saved registers or build a frame. It only has to meet the stack-alignment and shadow-space rules of the ABI. Keep the dispatchers inside the JIT arena as well, so generated code reaches them with direct relative jumps.
Why it is faster: —
Applies elsewhere: trampolines and coroutine switches that leave through `longjmp`, and VM interpreters with a tail-calling dispatch loop.
Watch out: —
Seen in PCSX2: The EE recompiler's entry point used to wrap execution in `xScopedStackFrame` and keep an `ExitRecompiledCode` label to return to. It now just does `xSUB(rsp, 32 + 8)` on Win32 (8 elsewhere), because it "never return[s] through this function, instead we fastjmp() out". It ends with `xJMP(DispatcherReg)`. The dispatchers themselves were moved into recompiler code space. 8e2af39c0 2023-10 “x86/R5900: EE rec doesn't need a stack frame” · 377746f15 2023-10 “x86: Move dispatchers to recompiler code space”

### pcsx2-254 — Clamp floats with integer min/max on their bit patterns
Lens: simd · Source: PCSX2
Principle: IEEE floats of the same sign order the same way as their bit patterns read as integers. A signed integer min clamps positive values, and an unsigned min clamps negative ones, whose sign bit makes them large. Two integer instructions replace a copy, a mask, an FP min/max pair and an OR to restore the sign.
Why it is faster: —
Applies elsewhere: radix sorts on float keys, and branchless float compares in shaders that use `floatBitsToInt`.
Watch out: —
Seen in PCSX2: `fpuFloat3`-style sign clamping used `MOVSS, AND.PS, MIN.SS, MAX.SS, OR.PS` plus a temporary register. It is now `xPMIN.SD(regd, g_maxvals); xPMIN.UD(regd, g_minvals);` using SSE4.1. 0d519a5f2 2023-12 “x86/FPU: Use SSE4 pminsd+pminud for sign clamping”

### pcsx2-255 — Let the hardware's saturation value handle half the edge cases
Lens: algorithms · Source: PCSX2
Principle: Before writing fix-up code, check what the instruction already returns for out-of-range input. If it already gives the correct answer for one side, only the other side needs patching, and that can often be done with a single compare and conditional move on the input bits.
Why it is faster: —
Applies elsewhere: relying on `cvttss2si`'s "integer indefinite" result, or on saturating SIMD packs, in place of explicit clamping.
Watch out: —
Seen in PCSX2: For FTOI/CVT.W, `cvttss2si` already returns `0x80000000` for negative overflow. The fix-up code went from `MOVMSKPS`, AND, ADD and CMP/CMOVE down to `xCMP(edx, 0x4f000000); xMOV(edx, 0x7fffffff); xCMOVGE(eax, edx)` on the raw float bits. The double-precision fallback paths were removed as well. c72e894fc 2025-05 “iR5900: Faster FTOI”

### pcsx2-256 — Mask irrelevant state out of a code-cache key
Lens: jit-interpreters · Source: PCSX2
Principle: When compiled blocks are looked up by entry state, any field in the key that the block ignores causes needless misses and recompiles. Mask those fields out of the comparison whenever a setting makes them irrelevant.
Why it is faster: —
Applies elsewhere: normalizing request headers before an HTTP cache lookup, and leaving unused uniforms out of shader-variant keys.
Watch out: —
Seen in PCSX2: In `microBlockManager::search`, when the MAC-flag speedhack (`mVUsFlagHack`) is on, the 64-bit quick key is compared with the MAC-flag bits masked off (`ref.quick & ~0x0C04`; the mask is `~0x3002` in the current tree). Blocks that differ only in incoming MAC flags now share code. 43e5ec25a 2025-10 “microVU: Ignore MAC flags in block match if not required”

### pcsx2-257 — Have the emitter pick non-destructive three-operand encodings
Lens: jit-interpreters · Source: PCSX2
Principle: Two-operand SIMD forms overwrite a source, so the translator has to copy it first whenever both inputs must survive. Have the assembler accept `dst, src1, src2` everywhere: it emits the VEX form when AVX is present and falls back to a move plus the legacy form otherwise. Call sites become simpler, and the extra moves disappear on modern CPUs.
Why it is faster: —
Applies elsewhere: LLVM's x86 backend preferring VEX encodings, and the three-address IRs in most compilers.
Watch out: —
Seen in PCSX2: A new `SIMDInstructionInfo` bitfield records each opcode's prefix, map, `is_commutative` and `is_mov`, and a series of commits moved nearly every SSE emitter to it. iMMI then dropped about 400 lines of `xMOVDQA(d, t); xPSRL.W(d, sa)` pairs in favor of `xPSRL.W(d, t, sa)`. The `ThreeArg()` helper uses a temporary only when `d == t` and AVX is off. de022ab68 2025-06 “Common: Add auto-switching AVX/SSE emitter functions” · 8779bf44d 2025-08 “iMMI: Use AVX instructions”

### pcsx2-258 — Skip work at compile time when masks make it dead
Lens: jit-interpreters · Source: PCSX2
Principle: A specialized code generator knows the write and fill masks when it compiles. If every lane is masked or filled from a constant row, it can omit the source read entirely, and if every lane is write-protected it can emit nothing.
Why it is faster: —
Applies elsewhere: query engines that drop scans for columns the projection never reads, and shader compilers that strip writes masked off by a constant.
Watch out: —
Seen in PCSX2: In the VIF unpack JIT's fill mode, `m0 = (m0 >> 1) | m0; if ((m0 & 0x55) != 0x55) xUnpack(upkNum);` skips loading the source when no vector actually uses it. Another check returns early when `doMask && m4 == 0xf`, meaning the whole vector is write-protected. fd2960c9c 2023-05 “VIF-JIT: Skip src reads on fill writes using regs” · bf6a0a035 2023-05 “VIF-JIT: Ignore completely write protected vectors”

### pcsx2-259 — Replace a memory mask constant with a register blend
Lens: simd · Source: PCSX2
Principle: ANDing with a mask from memory costs a load on every use. If the effect is to zero chosen lanes, blend with a zero register created once with `xor reg,reg`. The immediate blend mask encodes the lane selection, so nothing is read from memory.
Why it is faster: —
Applies elsewhere: `_mm_blend_ps` against `_mm_setzero_ps()` in hand-written SIMD, and immediate-operand lane selects in NEON.
Watch out: —
Seen in PCSX2: The VIF unpacker removed its `SSEXYZWMask` table. Unpacks that need to clear W now emit `xBLEND.PS(destReg, zeroReg, 0x8)`, where `zeroReg` (xmm2) is zeroed once per block for V2_32/V3 formats. dd2d4edff 2023-05 “VIF-JIT: Get rid of mem read for mask”

### pcsx2-260 — Share immutable per-job data across workers by reference
Lens: concurrency · Source: PCSX2
Principle: Have the producer build the job's read-only state, including generated function pointers, once. Workers then point at it instead of each memcpy-ing it and looking up their own copy of the JIT code. Each job gets cheaper, and so does each additional thread.
Why it is faster: —
Applies elsewhere: sharing compiled query plans across executor threads, and passing `Arc<Config>` to tasks instead of cloning it.
Watch out: —
Seen in PCSX2: `GSDrawScanline::BeginDraw` used to `memcpy(&m_global, ...)` per thread and resolve `m_ds`/`m_sp` per thread. Now `SetupDraw()` stores `sdata.ds/de/sp` in the shared `GSRasterizerData`, and every rasterizer thread reads `global` through `m_local.gd`. 3d84443bc 2023-01 “GS/SW: Share JIT code between all threads” · 90fc03783 2023-01 “GS/SW: Share global data between threads”

### pcsx2-261 — Address constants RIP-relative to free base registers
Lens: jit-interpreters · Source: PCSX2
Principle: On x86-64, generated code can reach constants near the code buffer with RIP-relative addressing. That releases the registers that were holding table base pointers, which cuts spills and the callee-saved push/pop pairs in the prologue.
Why it is faster: —
Applies elsewhere: position-independent code with RIP-relative globals, and literal pools in ARM JITs.
Watch out: —
Seen in PCSX2: The GS software scanline JIT dedicated `r10` to `&g_const` and also held r11, r13 and r14 as table bases. With `#define _rip_const(cptr) ptr[rip + ...]`, the prologue saves two fewer registers. Afterwards a comment notes "Free: r14, r15, rbp". 91c3e3b68 2023-01 “GS/SW: Use rip-relative addressing for constants”

### pcsx2-262 — Overlap constant tables and index them with unaligned loads
Lens: simd · Source: PCSX2
Principle: A table of shifted masks or step vectors can be one array with sliding windows into it, instead of N separately aligned vectors. Unaligned loads read any window. The table shrinks, more offsets fit in a one-byte displacement, and less cache is used.
Why it is faster: —
Applies elsewhere: a single "sliding" byte array for tail masks, such as loading from `mask + 16 - n` to mask the last `n` bytes.
Watch out: —
Seen in PCSX2: The AVX2 rasterizer constants `m_shift` and `m_test` are now one strip each. `load_shift(i)` is `GSVector8::load<false>(&g_const_256b.m_shift[8 - i])`, and the edge mask is `i8to32(&m_test[16 - skip]) | i8to32(&m_test[0 - (steps & (steps >> 31))])`. The commit message says this "allows more instructions to use 1-byte offsets". a291936bf 2024-10 “GS:SW: Use unaligned loads to reduce constant size on AVX2”

### pcsx2-263 — Make a thread return early from work it doesn't own
Lens: concurrency · Source: PCSX2
Principle: When work is partitioned by an ownership rule, test that rule before any setup. A thread that owns none of the item returns at once and does no per-item work at all.
Why it is faster: —
Applies elsewhere: sharded consumers discarding messages for other shards before parsing, and tile-binned rasterizers culling primitives per tile.
Watch out: —
Seen in PCSX2: In `GSRasterizer`, a horizontal line (`delta_y == 0.0f`) that falls on a scanline another thread owns, without AA, now returns before edge setup: `if (delta_y == 0.0f && !IsOneOfMyScanlines(ryi0) && !aa) return;`. 5f3dc1a76 2025-10 “GS/SW: Early exit optimization for multithreaded horizontal lines.”

### pcsx2-264 — Do per-vertex coordinate math in vector lanes
Lens: simd · Source: PCSX2
Principle: Fixed-point conversions on X and Y (offset, shift, widen) are the same operation on each component. Load both coordinates as one vector and apply widen, subtract and shift once, instead of running scalar code per component and per vertex.
Why it is faster: —
Applies elsewhere: SIMD bounding-box accumulation in physics engines, and vectorized pixel-coordinate transforms in image libraries.
Watch out: —
Seen in PCSX2: The CLUT draw-rect update in `VertexKick` used to compute `(XYZ.X - OFX) >> 4` as scalars. It now uses `GSVector4i::loadl(&XYZ).upl16().sub32(voffset).sra32(4)`. The vertex count per primitive also became a `constexpr NumIndicesForPrim(prim)`, so the loop is fully unrolled. 7c2a1f0f3 2023-02 “GS: Vectorize CLUT rect update”

### pcsx2-265 — Fill linear runs of a swizzled layout with a plain vector fill
Lens: memory-layout · Source: PCSX2
Principle: In a tiled or swizzled layout, writing a constant pixel by pixel through the address function is slow. When the region covers whole tiles that are contiguous in memory, the order no longer matters, so the region can be filled with a straight vector store loop. The general path is kept for the ragged edges.
Why it is faster: —
Applies elsewhere: clearing whole pages with `memset` in a paged allocator, and full-tile fast clears in GPU drivers.
Watch out: —
Seen in PCSX2: For a page-aligned memory clear (`left == 0 && top == 0 && (right & (pgs.x - 1)) == 0`), `GSRendererHW` takes `paMulti(...)` for each page row and writes `*(ptr++) = vcolor` across `pages_wide * pixels_per_page / 4` vectors. A separate variant preserves the alpha byte for 24-bit formats. 44460365c 2023-05 “GS/HW: Vectorize mem clears”

### pcsx2-266 — Compute an expensive predicate once per unit of work
Lens: laziness · Source: PCSX2
Principle: If a costly test is called from several places while handling one item, store its result in a `std::optional` member and reset it when the next item starts. Each item then pays for the test at most once.
Why it is faster: —
Applies elsewhere: per-request memoization in web frameworks, and the cached `hashCode` in Java strings.
Watch out: —
Seen in PCSX2: `GSRendererHW::PrimitiveCoversWithoutGaps()` walks the vertex list. It now writes into `std::optional<bool> m_primitive_covers_without_gaps`, which is reset on each draw. Similarly, `IsConstantDirectWriteMemClear()` is evaluated once into `is_possible_mem_clear` rather than in three places. 6babbc1ff 2023-06 “GS/HW: Cache PrimitiveCoversWithoutGaps() value” · 0822d3e3e 2023-04 “GS/HW: Cache possible mem clear value”

### pcsx2-267 — Memoize the result of a brute-force geometric search
Lens: lookup-tables · Source: PCSX2
Principle: If an offset is found by scanning a 2-D grid until an address matches, and the same pairs of surfaces come up again and again, cache the answer keyed on both surfaces' parameters. Repeat lookups become a single hash-map probe.
Why it is faster: —
Applies elsewhere: caching layout and position results in UI frameworks, and memoizing route lookups in a router.
Watch out: —
Seen in PCSX2: `TranslateAlignedRectByPage` searched block by block for `b2a_offset` such that `bn(x, y, tbp, sbw) == sbp`. It now builds a `SurfaceOffsetKey` from source and target `{bp, bw, psm, rect}` and checks `m_surface_offset_cache` first. `DirtyRectByPage` uses the same cache. 47897fc2b 2025-10 “GS/TC: Add caching to block offset calculation for TranslateAlignedRectByPage and DirtyRectByPage.”

### pcsx2-268 — Carry a computed result over to the next loop iteration
Lens: algorithms · Source: PCSX2
Principle: A scanning loop sometimes finds a candidate it can't use in the current iteration and drops it, only to compute the same thing again on the next pass. Save it in a small struct and consume it first on the next iteration.
Why it is faster: —
Applies elsewhere: a one-token lookahead buffer in a lexer, and "unget" in stream parsers.
Watch out: —
Seen in PCSX2: The draw-list overlap computation detects triangle-strip quads using `CheckTriangleQuads`. It now keeps a `SavedTristrip { saved, skip, axis_aligned, bbox }`, so a strip found but rejected at the end of one batch is reused when the next batch starts. e83a343ba 2026-02 “GS/HW: Save tristrips in draw list computation to avoid recomputing.”

### pcsx2-269 — Keep several batches open and resume the one that matches
Lens: gpu-rendering · Source: PCSX2
Principle: A guest that switches back and forth between a few render states forces a flush at every switch, which produces many tiny draws. Keep a few vertex and index buffers, each with its own state snapshot. When the state switches back to one that is still pending, append to that buffer, and flush all of them together when ordering or a hazard requires it.
Why it is faster: —
Applies elsewhere: sprite batchers that bucket by texture, and command-buffer sorting by material in game engines.
Watch out: —
Seen in PCSX2: `GSState` now has `MAX_DRAW_BUFFERS = 3` sets of `m_vertex_buffers`, `m_index_buffers` and `m_env_buffers`. A new draw looks for an earlier context whose TEX0 fields match under a mask (`0x1fffffffff`: TBP0, TBW, PSM, TW, TH, TCC, TFX) and joins it. Follow-up commits only compact the buffers when more than one is in use, and replace repeated `m_vertex->` dereferences with local references. 76c401b0f 2026-01 “GS: Implement draw buffering” · 7887919e7 2026-07 “GS: Reduce draw buffer copying” · 99cfbb49c 2026-07 “GS: Optimize draw buffering pointer logic”

### pcsx2-270 — Don't attach a depth buffer the draw can't observe
Lens: gpu-rendering · Source: PCSX2
Principle: If depth writes are masked and the depth test provably always passes, the depth attachment has no effect except bandwidth, allocation and target growth. Rewrite the test to ALWAYS and draw without a depth buffer.
Why it is faster: —
Applies elsewhere: render-graph culling of unused attachments, and skipping the depth prepass for UI layers.
Watch out: —
Seen in PCSX2: `GSRendererHW` now also sets `no_ds` when `ZTST == GEQUAL`, Z is constant (`m_vt.m_eq.z`) and Z equals the format's `max_z`, provided Z writes are masked. It then forces `TEST.ZTST = ZTST_ALWAYS`. This stopped some offscreen targets from growing. 1eaec773e 2023-02 “GS/HW: Don't allocate Z buffer if tests always pass”

### pcsx2-271 — Push descriptors inline instead of allocating sets per frame
Lens: gpu-rendering · Source: PCSX2
Principle: Allocating and updating descriptor sets from per-frame pools for every draw costs CPU time and pool management. Push descriptors write bindings straight into the command buffer, so no set objects are needed.
Why it is faster: —
Applies elsewhere: root descriptors and constants in D3D12, and argument buffers in Metal.
Watch out: —
Seen in PCSX2: The Vulkan backend now requires `VK_KHR_push_descriptor` and checks `maxPushDescriptors >= NUM_TFX_TEXTURES`. The texture, RT and PrimMin bindings moved into set 1, which is pushed, replacing per-frame allocations. `GSDeviceVK.cpp` shrank by about 300 lines. cf523d421 2023-08 “GS/Vulkan: Use push descriptors instead of per-frame allocations”

### pcsx2-272 — Pipeline GPU readback and encoding across frames and threads
Lens: io · Source: PCSX2
Principle: A synchronous readback stalls the CPU until the GPU catches up, and encoding inline stalls the render thread on top of that. Queue the readbacks into a ring of N staging textures, map each one a few frames later, and hand mapped frames to an encoder thread through a condition variable.
Why it is faster: —
Applies elsewhere: pixel-buffer-object ring buffers for screenshots, and the asynchronous capture pipelines in OBS-style recorders.
Watch out: —
Seen in PCSX2: `GSCapture` keeps `std::array<PendingFrame, NUM_FRAMES_IN_FLIGHT * 2>` entries, each a `GSDownloadTexture` moving through the states `Unused → NeedsMap → NeedsEncoding`. A dedicated `s_encoder_thread` waits on `s_frame_ready_cv`, and the OSD shows its CPU usage as "CAP". dedcf21a3 2023-01 “GSCapture: Make readback for capture asynchronous” · 6834367a3 2023-01 “GS: Move video capture encoding onto its own thread”

### pcsx2-273 — Merge dirty rectangles into fewer uploads
Lens: gpu-rendering · Source: PCSX2
Principle: Each texture upload has fixed driver and synchronization overhead. Coalesce overlapping or adjacent dirty regions before uploading, so several small transfers become a few large ones.
Why it is faster: —
Applies elsewhere: damage-region coalescing in compositors, and batched `glTexSubImage` calls in UI toolkits.
Watch out: —
Seen in PCSX2: `GSDirtyRectList` lost `GetDirtyRectAndClear`, which popped one rect at a time. It gained an indexed, `const` `GetDirtyRect(index, TEX0, clamp)`, and the texture cache now combines rects before updating the target. The commit says: "Fewer texture uploads makes Intel GPUs happy." f0798f651 2023-02 “GS/HW: Combine dirty rectangles for target updates”

### pcsx2-274 — Prove no overlap before paying for a hazard copy or barrier
Lens: gpu-rendering · Source: PCSX2
Principle: Sampling from the render target being drawn normally needs a copy or a barrier. If the sampled rectangle and the drawn rectangle don't intersect, or the primitives form non-overlapping quads, there is no hazard and both can be skipped.
Why it is faster: —
Applies elsewhere: alias analysis that allows `memcpy` instead of `memmove`, and fine-grained barrier tracking in render graphs.
Watch out: —
Seen in PCSX2: When the texture is the RT, `GSRendererHW` now computes `src_rect` from `m_vt.m_min/max.t` and returns early if `m_r.rintersect(src_rect).rempty()`. Separately, draw-list analysis recognizes triangle strips and fans whose triangles pair into non-overlapping quads by checking which side of the shared edge each unshared vertex falls on. Those draws need fewer barriers. 054e0509c 2026-05 “GS/HW: Avoid hazard copy if read and draw don't overlap” · a1173c53d 2026-01 “GS/HW: Reduce barriers in triangle strips/fans.”

### pcsx2-275 — Filter redundant API state changes with a shadow copy
Lens: gpu-rendering · Source: PCSX2
Principle: Driver calls to bind resources or upload constants cost CPU time even when nothing changes. Keep a shadow copy of what was last bound or uploaded, compare against it, and skip the call when they match.
Why it is faster: —
Applies elsewhere: the GL state caches in game engines, and diffing props before DOM writes in a virtual DOM.
Watch out: —
Seen in PCSX2: DX11 compares `ps_sr_views` and `ps_ss` against `ps_cached_sr_views` and `ps_cached_ss` before calling `PSSetShaderResources` or `PSSetSamplers`. A wrapper `UpdateSubresource(buffer, cb, cached, size)` uploads only when `memcmp` finds a difference. On Metal, binding a null texture no longer triggers a rebind. 722bc9427 2025-06 “GS/DX11: Cache shader resource and sampler.” · 936836e09 2025-07 “GS/DX11: Expand caching constant buffer updates.” · ff1b47942 2026-06 “GS:MTL: Reduce unneccessary texture rebinds”

### pcsx2-276 — Stage constant data into device-local memory
Lens: gpu-rendering · Source: PCSX2
Principle: Reading constants directly from CPU-visible upload heaps makes the GPU fetch them over the bus on every access. Write them to the upload buffer, then copy the written range once per batch into a device-local buffer that the shaders read.
Why it is faster: —
Applies elsewhere: staging buffers in Vulkan engines, and "dynamic but static per frame" buffer strategies.
Watch out: —
Seen in PCSX2: `D3D12StreamBuffer::Create(size, gpu_backed_buffer)` now also allocates a `D3D12_HEAP_TYPE_DEFAULT` buffer. `FlushMemory()` issues a `CopyBufferRegion` from `m_current_copy_offset` to `m_current_offset`, and the GPU address points at the default-heap buffer. 0e667a9b0 2026-03 “GS/DX12: Copy constant buffers to default heap”

### pcsx2-277 — Build tables at compile time and size buffers statically
Lens: allocation · Source: PCSX2
Principle: Tables computed at startup and buffers allocated on the heap add init code, an indirection and a failure path. When their size is fixed, make them `constexpr` arrays and static storage, so they sit in the binary's data section ready to use.
Why it is faster: —
Applies elsewhere: `constexpr` CRC tables, and static arenas in embedded firmware.
Watch out: —
Seen in PCSX2: The SPU2 `PsxRates[160]` are now produced by `constexpr ComputePSXRates()`, and `pcm_cache_data` became a fixed array of `pcm_BlockCount` instead of a `new`. The register table is built by `ComputeRegTable()` from a static original, and the alloc/free code in `spu2.cpp` was deleted. 6426f4432 2023-05 “SPU2: Get rid of dynamic allocations”

### pcsx2-278 — Lay out filter coefficients for SIMD multiply-add
Lens: simd · Source: PCSX2
Principle: An FIR filter is a dot product. Store the coefficients as aligned `s16` arrays in tap order, and arrange the interpolation table so the taps for one phase sit next to each other. A few multiply-add and horizontal-add instructions then replace a scalar loop.
Why it is faster: —
Applies elsewhere: polyphase resamplers in audio libraries, and convolution kernels in image codecs.
Watch out: —
Seen in PCSX2: Reverb downsampling and upsampling dropped their `s32` per-tap loops for `alignas(32)` `s16` coefficient arrays, using `GSVector8i` under AVX2. A new `hadds16` wraps `_mm256_hadds_epi16`. The Gaussian interpolation table was reshaped from `interpTable[0x200]` into `interpTable[i][0..3]` so the four terms are contiguous. f29346f0b 2023-10 “SPU2: Optimize reverb resampling” · 6bff6016f 2023-10 “SPU2: Rearrange interpolation table”

### pcsx2-279 — Sync coprocessors at their next event, not on a fixed time slice
Lens: scheduling · Source: PCSX2
Principle: Synchronizing two simulated processors on a fixed small interval forces many context switches that change nothing. Let the secondary processor run until its next scheduled event, or up to a larger cap, and schedule the main CPU's next sync from that point.
Why it is faster: —
Applies elsewhere: conservative time windows in parallel discrete-event simulation, and timer coalescing in OS schedulers.
Watch out: —
Seen in PCSX2: The IOP's next event was previously set to `cycle + 32`. It is now `cycle + iopWaitCycles` (384). The EE runs the IOP when `EEsCycle >= nextIopEventDelta`, replacing the fixed `EEsCycle > 192` threshold, and calls `cpuSetNextEventDelta(iopDelta - psxRegs.iopCycleEE)`. 7fbc63b8e 2023-06 “EE/IOP Sync: Optimize EE/IOP sync for more intelligent syncing”

### pcsx2-280 — Use small inline-buffer strings for per-frame text
Lens: allocation · Source: PCSX2
Principle: Building UI strings with `std::string` every frame causes repeated heap allocations. A string type with a fixed inline buffer keeps short strings on the stack, so steady-state frames do no allocation.
Why it is faster: —
Applies elsewhere: `SmallVector`/`SmallString` in LLVM, and `arrayvec`/`smol_str` in Rust.
Watch out: —
Seen in PCSX2: Fullscreen UI settings widgets switched from `std::string` and `StdStringFromFormat` to `SmallString::from_sprintf` and `TinyString`. `GetSmallStringValue`/`GetOptionalSmallStringValue` accessors were added, and `count_if` became `value.count('&')`. a1ec590bf 2024-04 “FullscreenUI: Avoid per-frame memory allocations with SmallString”

### pcsx2-281 — Re-check flush conditions only when an input actually changed
Lens: laziness · Source: PCSX2
Principle: If a check that decides whether to flush pending work runs on every incoming command, gate it on cheap change detection: a different primitive class, a dirty-register bit, or the first vertex that actually draws. Defer the full check until one of those triggers.
Why it is faster: —
Applies elsewhere: dirty-flag-driven layout invalidation in UI frameworks, and incremental build systems that check timestamps before hashing.
Watch out: —
Seen in PCSX2: In `GSState`'s GIF packed and REGLIST paths, `CheckFlushes()` now runs only if `!adc`, the primitive class changed, or `DIRTY_REG_XYOFFSET` is set. Otherwise it runs lazily before the first non-skipped `VertexKick`, tracked by a `flushes_checked` flag. ec9e5402c 2023-09 “GS: Reduce unnecessary flushes”

### pcsx2-282 — Bound content-addressed caches by entry count
Lens: memory-layout · Source: PCSX2
Principle: A cache bounded only by age can grow without limit when a workload creates thousands of short-lived unique entries. Add a hard cap on the number of entries and evict the least useful ones in a batch once the cap is passed.
Why it is faster: —
Applies elsewhere: `maxsize` on Python's `lru_cache`, and entry caps on CDN edge caches.
Watch out: —
Seen in PCSX2: The hardware texture hash cache now enforces `MAX_HASH_CACHE_SIZE = 800` in addition to a VRAM budget. Unreferenced entries are collected into `s_hash_cache_purge_list` and removed oldest first, after one game produced more than 16,000 textures. The palette map cap was later cut from 65535 to `MAX_CACHED_PALETES = 4096`. a4e99366f 2023-04 “GS/HW: Put a cap on the hash cache count” · 2fda2d512 2026-03 “GS/HW: Reduce number of cached palettes”

### pcsx2-283 — Use the pointer's sign bit to mark entries that need a handler
Lens: memory-layout · Source: PCSX2
Principle: A software page table can store host pointers and handler IDs in the same word. Tag the handlers by setting the top bit, then the hot path needs only one add and a sign test to choose between a direct access and a callback.
Why it is faster: —
Applies elsewhere: NaN-boxing and pointer tagging in JS engines, and the "present" bit in hardware page tables.
Watch out: —
Seen in PCSX2: `VTLBVirtual` holds `value = host_base - vaddr_page` for RAM, or a handler ID with `POINTER_SIGN_BIT` set. `isHandler(vaddr)` is `(sptr)(value + vaddr) < 0` and `assumePtr(vaddr)` is `value + vaddr`, so one add serves both the test and the address. Handlers come from a `RWFT[5][2][128]` table indexed by width and direction. current code: pcsx2/vtlb.h

### pcsx2-284 — Emit direct memory accesses and patch them when they fault
Lens: jit-interpreters · Source: PCSX2
Principle: Reserve a host address range that mirrors the guest address space, and emit guest loads and stores as plain host accesses with no lookup. If one faults on an unmapped or protected page, the signal handler rewrites that instruction into a jump to a thunk. The thunk saves live registers and calls the slow handler.
Why it is faster: —
Applies elsewhere: guard-page bounds checks in WebAssembly engines, and fastmem in other console emulators.
Watch out: —
Seen in PCSX2: `vtlb_DynBackpatchLoadStore()` receives the faulting code address and size together with bitmasks of live GPRs and XMMs. It emits a thunk through `recBeginThunk()` that saves only the caller-saved registers that are live, keeping the stack 16-byte aligned with Win32 shadow space. Then it overwrites the original instruction with a jump. `vtlb_UpdateFastmemProtection()` keeps the fastmem view's page protection in sync. current code: pcsx2/x86/ix86-32/recVTLB.cpp

### pcsx2-285 — Dispatch through a two-level PC table and link block exits directly
Lens: jit-interpreters · Source: PCSX2
Principle: To find compiled code for an arbitrary PC, index a page table by the upper half of the PC and a per-page array by the lower half, then jump indirectly. Exits with a known target are instead patched into a direct `jmp rel32` to the target block, or to the compiler stub until that block exists.
Why it is faster: —
Applies elsewhere: QEMU TB chaining, and inline caches in JavaScript JITs.
Watch out: —
Seen in PCSX2: `_DynGen_DispatcherReg` is four instructions: `mov eax,[pc]; mov edx,eax; shr eax,16; mov rcx,[recLUT+rax*8]; jmp [rcx+rdx*2]`. `BaseBlocks::Link()` writes `*jumpptr = target->fnptr - (jumpptr+1)` and records the site in a multimap so it can be unlinked later. Unmapped LUT pages point at a stub page instead of null. current code: pcsx2/x86/ix86-32/iR5900.cpp, pcsx2/x86/BaseblockEx.h, pcsx2/x86/BaseblockEx.cpp

### pcsx2-286 — Detect self-modifying code with page protection, falling back to inline compares
Lens: jit-interpreters · Source: PCSX2
Principle: Write-protect host pages that hold translated guest code, so a write to one faults and invalidates its blocks at no cost on normal runs. On pages that keep getting written because code and data share them, switch to blocks that compare their own source words inline. After several reprotects, mark the page permanently manual.
Why it is faster: —
Applies elsewhere: write-protection of JIT-ed pages in DBT systems such as Valgrind and DynamoRIO, and copy-on-write tracking.
Watch out: —
Seen in PCSX2: A block's page can be in mode `ProtMode_Write`, which calls `mmap_MarkCountedRamPage` to protect both the RAM and fastmem views, or `ProtMode_Manual`. Manual blocks emit `xCMP(ptr32[PSM(lpc)], imm); xJNE(DispatchBlockDiscard)` for every word. Counted blocks also add their size to a 16-bit `manual_page` counter, and a carry out of it jumps to `DispatchPageReset` to try reprotecting. When `manual_counter > 3` the page stays uncounted. current code: pcsx2/x86/ix86-32/iR5900.cpp, pcsx2/Memory.cpp

### pcsx2-287 — Run a backward liveness pass and fold known constants before emitting code
Lens: compiler-build · Source: PCSX2
Principle: Before generating code for a block, walk it backwards and record, per instruction, which registers are used later and where each is last used. The allocator can then drop dead values without writing them back, and a constant-propagation bitmask lets instructions with constant inputs be folded at compile time.
Why it is faster: —
Applies elsewhere: liveness analysis and SCCP in optimizing compilers, and the analysis passes in baseline JITs.
Watch out: —
Seen in PCSX2: Each `EEINST` carries per-register `regs[34]`, `fpuregs[33]`, `vfregs[34]` and `viregs[16]` bytes with `EEINST_LIVE`, `EEINST_USED` and `EEINST_LASTUSE` bits. `GPR_IS_CONST1`/`GPR_IS_CONST2` test `g_cpuHasConstReg`, and `g_cpuConstRegs[32]` holds the values. `GPR_IS_DIRTY_CONST` tracks constants that have not yet been flushed. current code: pcsx2/x86/iCore.h, pcsx2/x86/iR5900.h

### pcsx2-288 — Look up compiled variants by a packed key, falling back to a move-to-front list
Lens: algorithms · Source: PCSX2
Principle: When most lookups match on a few key fields, pack those fields into one 64-bit word and scan a vector comparing single integers. Keep the full-state comparison for the rare cases that need an exact match, and move each hit to the front of that list.
Why it is faster: —
Applies elsewhere: polymorphic inline caches, and self-organizing lists for small symbol tables.
Watch out: —
Seen in PCSX2: `microBlockManager::search` scans `quickLookup`, a vector of `{pBlock, quick}`, comparing against `pState->quick64[0]`, with constant-propagation checks on `vi15`. If `needExactMatch` is set, it walks `fBlockList` with `compareState()` and relinks the hit to the head of the list. current code: pcsx2/x86/microVU.h

### pcsx2-289 — Store hash chains as contiguous arrays ended by a sentinel
Lens: lookup-tables · Source: PCSX2
Principle: A linked hash chain costs a cache miss per hop. Store each bucket as a cache-line-aligned array of entries ending in an all-zero sentinel. A lookup then scans linearly through memory the prefetcher can follow, and needs no bounds check.
Why it is faster: —
Applies elsewhere: open-addressing and bucketized cuckoo tables, and SwissTable's group probing.
Watch out: —
Seen in PCSX2: In the VIF unpack JIT cache, `HashBucket::find` loops `chainpos++` comparing `key0`/`key1` until it reaches `startPtr == 0`. `add()` grows the array with `pcsx2_aligned_realloc(..., 64, ...)`, and the source comment says "64B align to reduce cache miss penalty in `find`". current code: pcsx2/Vif_HashBucket.h

### pcsx2-290 — Use a single-producer ring with indices on separate cache lines
Lens: concurrency · Source: PCSX2
Principle: To stream commands from one thread to another, use a power-of-two ring buffer. The producer owns the write index and the consumer owns the read index, each on its own cache line to avoid false sharing. Acquire/release ordering on those indices replaces locks.
Why it is faster: —
Applies elsewhere: the LMAX Disruptor, io_uring submission and completion queues, and audio callback FIFOs.
Watch out: —
Seen in PCSX2: MTGS declares `alignas(__cachelinesize) std::atomic<unsigned int> s_ReadPos` and `s_WritePos`, and its consumer loop runs `while (s_ReadPos.load(relaxed) != s_WritePos.load(acquire))`. MTVU uses the same pattern: a 16 MB `buffer[]` whose size must be a power of two, with `m_ato_read_pos`, `m_ato_write_pos` and a thread-local `m_read_pos`, each cache-line aligned. current code: pcsx2/MTGS.cpp, pcsx2/MTVU.h

### pcsx2-291 — Build a spin-then-sleep work semaphore on one atomic state word
Lens: concurrency · Source: PCSX2
Principle: Keep the worker's spinning, sleeping or running state and a pending-work count in a single atomic integer. Notifying the worker is then one `fetch_add`, and only the sleeping case needs a kernel post. Waiters spin briefly before falling back to a kernel semaphore.
Why it is faster: —
Applies elsewhere: futex-based mutexes, and Go's runtime parking with a spinning phase.
Watch out: —
Seen in PCSX2: In `WorkSema`, `NotifyOfWork()` is `if (m_state.fetch_add(2, release) == STATE_SLEEPING) m_sema.Post();`. The states are `SPINNING = -2`, `SLEEPING = -1` and `RUNNING_N >= 0`, with a `STATE_FLAG_WAITING_EMPTY` bit. A comment says to "optimize states for fast NotifyOfWork". `WaitForEmptyWithSpin()` calls `ShortSpin()` until `SPIN_TIME_NS` passes and then sleeps on `m_empty_sema`. current code: common/Threading.h, common/Semaphore.cpp

### pcsx2-292 — JIT one specialized function per pipeline-state key
Lens: jit-interpreters · Source: PCSX2
Principle: Rather than a generic pixel loop that branches on dozens of state bits per pixel, pack the state into an integer key and generate a branch-free routine for each key the first time it is seen. Cache the routines in a map from key to function.
Why it is faster: —
Applies elsewhere: SwiftShader and llvmpipe shader specialization, and regex engines compiling patterns to machine code.
Watch out: —
Seen in PCSX2: `GSCodeGeneratorFunctionMap` keeps `std::unordered_map<u64, VALUE> m_cgmap`, where the key is the scanline selector (`sel.key`), with code memory from `GSCodeReserve`. `GSFunctionMap` also keeps per-key frame, prim, tick and pixel statistics for profiling. There are x64 and arm64 generators (`GSDrawScanlineCodeGenerator.all.cpp`/`.arm64.cpp`). current code: pcsx2/GS/Renderers/Common/GSFunctionMap.h, pcsx2/GS/Renderers/SW/

### pcsx2-293 — Split rasterization by interleaved scanline bands, fed from a ring-buffer heap
Lens: concurrency · Source: PCSX2
Principle: Give each thread every Nth band of scanlines, so the load stays balanced without locking pixels and every thread can process the same primitive list. Allocate the shared per-draw data from a ring-shaped heap that one producer writes and many consumers free roughly in order, so per-draw allocation stays cheap.
Why it is faster: —
Applies elsewhere: sort-middle tiled rendering, and arena or region allocators for per-frame data.
Watch out: —
Seen in PCSX2: `GSRasterizerList` keeps a `m_scanline` ownership table (`m_thread_height` rows per band) and `GSJobQueue<GSRingHeap::SharedPtr<GSRasterizerData>, 65536>` workers. `IsOneOfMyScanlines()` gates the work for each primitive. `GSRingHeap` is documented as "A ring buffer pretending to be a heap… one producer… multiple consumer threads". current code: pcsx2/GS/Renderers/SW/GSRasterizer.h, pcsx2/GS/GSRingHeap.h

### pcsx2-294 — Compile the hot module once per ISA and dispatch at runtime
Lens: platform · Source: PCSX2
Principle: Build the SIMD-heavy sources several times, once each for SSE4, AVX and AVX2, each copy in its own namespace. At startup, detect the CPU and select the matching set of entry points, so one binary runs on old CPUs and uses wide vectors on new ones.
Why it is faster: —
Applies elsewhere: GCC `target_clones`, simdjson's runtime kernels, and per-ISA dispatch in OpenSSL.
Watch out: —
Seen in PCSX2: `MULTI_ISA_UNSHARED_START` opens `namespace CURRENT_ISA`, which is `isa_sse4`, `isa_avx` or `isa_avx2` depending on the compilation, and `MULTI_ISA_COMPILE_ONCE` marks code to compile only once. `ProcessorFeatures::VectorISA` chooses between SSE4, AVX, AVX2 and AVX512F. `GSVector4/8` wrappers, with NEON versions in `_arm64.h`, keep the source the same for every ISA. current code: pcsx2/GS/MultiISA.h, pcsx2/GS/GSVector*.h

### pcsx2-295 — Select vector operations from template-constant masks
Lens: simd · Source: PCSX2
Principle: When a write mask is a template parameter, the blend instruction can be chosen at compile time. An all-zero mask is a no-op, an all-ones mask is a plain store, a halfword mask is a cheap 16-bit blend, and only arbitrary masks fall back to byte blends.
Why it is faster: —
Applies elsewhere: `if constexpr` dispatch in Eigen and xsimd, and specialization on known strides in BLAS kernels.
Watch out: —
Seen in PCSX2: `GSVector8i::smartblend<mask>` returns `*this` for 0 and `a` for `0xffffffff`, uses `blend16<0x55>`/`<0xaa>` for `0x0000ffff`/`0xffff0000`, and handles other masks byte by byte. `GSBlock::WriteColumn32<i, alignment, mask>` swizzles two rows with `acbd()` and `sw64()`, then stores through `smartblend<mask>`. current code: pcsx2/GS/GSVector8i.h, pcsx2/GS/GSBlock.h

### pcsx2-296 — Index cached surfaces by memory page
Lens: lookup-tables · Source: PCSX2
Principle: To find which cached textures a memory write invalidates, keep one list of surfaces per page of guest memory. A write then visits only the pages it touches, instead of testing every cached surface for overlap. Index-based intrusive lists keep insertion and removal O(1) without pointer invalidation.
Why it is faster: —
Applies elsewhere: reverse maps from page to VMA in OS kernels, and spatial hash grids for collision broad-phase.
Watch out: —
Seen in PCSX2: `SourceMap` holds `std::array<FastList<Source*>, GS_MAX_PAGES> m_map` plus an `unordered_set` of all surfaces. `FastList` elements link through `u16 next_index` and `prev_index` into one growable buffer, and its iterators are "index based, not pointer based", so they survive `Grow()`. current code: pcsx2/GS/Renderers/HW/GSTextureCache.h, pcsx2/GS/Renderers/Common/GSFastList.h

### pcsx2-297 — Deduplicate texture uploads by content hash
Lens: algorithms · Source: PCSX2
Principle: The same guest texture often comes back at a different address or after an invalidation. Hash its bytes, along with its palette when it uses one, with a fast non-cryptographic hash, and reuse the GPU texture already uploaded under that hash instead of decoding and uploading it again.
Why it is faster: —
Applies elsewhere: content-addressed asset caches, and block deduplication in backup tools.
Watch out: —
Seen in PCSX2: `HashCacheKey::Create(TEX0, TEXA, clut, lod, region)` is looked up in `m_hash_cache`. If that misses, the lookup is retried with `WithRemovedCLUTHash()` so palette variants can share a decoded texture. Texture data is hashed block by block with `GSXXH3_64bits_update(&st, bp, GS_BLOCK_SIZE)`, and the CLUT hash covers 16 or 256 entries. current code: pcsx2/GS/Renderers/HW/GSTextureCache.cpp

### pcsx2-298 — Cache pipeline objects under a packed state selector
Lens: gpu-rendering · Source: PCSX2
Principle: Creating GPU pipeline objects is expensive. Pack every state that affects the pipeline into a fixed-size POD key, hash it, and keep the compiled pipelines in a map, backed by an on-disk pipeline cache that survives across runs.
Why it is faster: —
Applies elsewhere: PSO caches in Unreal and other game engines, and `VkPipelineCache` serialization.
Watch out: —
Seen in PCSX2: Vulkan uses `std::unordered_map<PipelineSelector, VkPipeline, PipelineSelectorHash> m_tfx_pipelines`. The selector is asserted to be 32 bytes, and `PipelineSelectorHash` runs `HashCombine` over `vs.key`, `ps.key_hi/lo`, `dss`, `cms`, `bs` and `key`. `VKShaderCache` persists compiled shaders and the pipeline cache. current code: pcsx2/GS/Renderers/Vulkan/GSDeviceVK.h, pcsx2/GS/Renderers/Vulkan/VKShaderCache.h
