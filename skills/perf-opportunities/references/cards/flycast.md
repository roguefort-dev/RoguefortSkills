# Flycast — technique cards (142)

Each card is exactly 7 lines. Fetch one with: grep -A 6 "^### flycast-NNN " flycast.md

### flycast-002 — Chain predicated tests instead of repeated test-and-set
Lens: branching-fast-paths · Source: Flycast
Principle: To answer "is any of these conditions true", run each later test only if the earlier ones failed, and write the result once at the end. That means predicated instructions or short-circuit logic. Redundant writes go away, the sequence gets shorter, and no branches are needed.
Why it is faster: —
Applies elsewhere: SIMD "any lane equal" checks, finding a zero byte in a word in a `strlen`, and validation code that combines checks with `||` and assigns once instead of in each `if`.
Watch out: —
Seen in Flycast: `shop_setpeq` (is any byte equal) on ARM ran four `TST`+`MOVW rd,1,EQ` pairs. It became `TST r1,0xFF000000` followed by `TST ..., CC_NE` for the other three bytes, and a single final `MOVW(rd,1,CC_EQ)`. A later commit also rewrote `shop_sbc` to set the carry with `EOR`+`LSR ...,true` and finish with a conditional `MOV(rd2,1,CC_CC)`, replacing a `MOV`/`SUB`/`ADC` sequence. 3ff594093 2013-12 “slight improvement to setpeq opcode” · 79e9c61ee 2014-01 “DYNAREC: More work on some opcodes”

### flycast-003 — Recognize multi-instruction idioms and replace them with one operation
Lens: jit-interpreters · Source: Flycast
Principle: Code often spells out a high-level operation as a long fixed sequence of primitive steps. A 32-step software division loop is one example. Pattern-match the whole sequence at translation time and emit one native instruction or call in its place. Dozens of operations, and their intermediate flag updates, become one.
Why it is faster: —
Applies elsewhere: compilers that recognize popcount or memcpy loops and replace them with an intrinsic, SQL optimizers that turn a correlated subquery into a join, and bytecode VMs with superinstructions for common opcode pairs.
Watch out: —
Seen in Flycast: When the SH4 decoder (`dyna/decoder.cpp`) sees the opening division instruction, `MatchDiv32` scans ahead for 32 repetitions of `ROTCL Rn` / `DIV1 Rm,Rn` that use the same registers. On a match it emits a single `shop_div32u`/`shop_div32s` and a fix-up `shop_div32p2`. The commit cited here removed a `printf` that fired every time a match failed. 3b1c38ad5 2015-03 “sh4/driver: don't log on div32 missmatch”

### flycast-004 — Run cheap peephole passes over the IR before code generation
Lens: jit-interpreters · Source: Flycast
Principle: Short passes over a block's intermediate representation can drop writes that are overwritten before anyone reads them, and can link constants across operations. That shrinks the code the backend has to emit. Any pass that breaks correctness on real workloads has to be switchable off without touching the others.
Why it is faster: —
Applies elsewhere: dead-store elimination in compilers, coalescing redundant DOM writes in UI frameworks, and collapsing repeated `SET`s to the same key in a write-ahead log before flushing.
Watch out: —
Seen in Flycast: `AnalyseBlock` in `dyna/shil.cpp` runs `srt_waw` (a "write after write without read" pass that removes dead writes to the T flag) and `constlink`. The `dejcond` pass broke Shenmue, so it was commented out at a small speed cost. A store-queue pass, `sq_pref`, runs only when `settings.dynarec.unstable_opt` is set. 8653e5e0f 2013-12 “dejcond breaks shenmue, disabled. Hrfm. This (might) reduce speed a tiny bit.” · fda7844f7 2013-12 “Remove a duplicated boolean expression from function srt_waw()...” · 90db7054a 2014-02 “idleskip should default to true”

### flycast-006 — Put speculative fast paths behind an explicit opt-in
Lens: jit-interpreters · Source: Flycast
Principle: Some shortcuts are safe for nearly every workload but break a few. Skipping integrity checks is one. Gate them behind a single "unstable optimizations" flag. Turn the flag on by default only when the speed it buys, such as avoiding audio dropouts, matters more than the rare breakage, and keep an allow-list of the known exceptions.
Why it is faster: —
Applies elsewhere: database `synchronous=OFF` and `fsync` toggles, the `-ffast-math` family of flags, and an HTTP cache that skips revalidation for assets marked immutable.
Watch out: —
Seen in Flycast: With unstable opts on, `DoCheck(pc)` in `dyna/driver.cpp` verifies blocks against self-modifying code only for an allow-list of addresses (Shenmue, Shenmue 2). With the flag off it checks every block. The Android UI's "Unstable Optimisations" toggle (`Dynarec.unstable-opt`) defaulted to true "to prevent crackling". e512b9544 2014-01 “Force block (header) checks by default.” · 05571cf7e 2014-01 “Frameskip, unstable opts” · 5a741af03 2014-02 “Enable unstable optimization by default to prevent crackling”

### flycast-007 — Use the ABI that passes floats in FP registers
Lens: compiler-build · Source: Flycast
Principle: On soft-float ABIs every float argument and return value moves between FPU and integer registers at each call. A hard-float convention keeps floats in FP registers across calls, which removes two transfers per argument on every call.
Why it is faster: —
Applies elsewhere: choosing `armhf` over `armel` for embedded Linux, `__vectorcall` on Windows for SIMD-heavy math libraries, and passing small structs by value in registers.
Watch out: —
Seen in Flycast: Under `ARM_HARDFP`, `ngen_CC_Param`/`ngen_CC_Call` in the ARM JIT now place `CPT_f32` arguments and return values in `f0`... with `VMOV` instead of in `r0`... `-mfloat-abi=softfp` was dropped from `core.mk`, and `Android.mk` notes that hardfp needs NDK r9b+. 35b38e3a7 2014-01 “hardfp calling support”

### flycast-008 — Ship with aggressive optimization flags and without debug info
Lens: compiler-build · Source: Flycast
Principle: For hot numeric code, compile at `-O3` with vectorization, the frame pointer freed for use as a general register, no exceptions and no RTTI. Hide symbols by default and give each function and data item its own section so the linker can drop unused code. Leaving `-g` in a release build bloats it, and on some toolchains, such as browser-hosted wasm/asm.js, it costs a great deal of memory at runtime.
Why it is faster: —
Applies elsewhere: release builds of game engines and codecs, `-fvisibility=hidden` with `--gc-sections` for shared libraries on mobile, and stripping source maps from production JS bundles.
Watch out: —
Seen in Flycast: `core.mk` builds with `-O3 -frename-registers -fsingle-precision-constant -ffast-math -ftree-vectorize -fomit-frame-pointer`. C and C++ flags were split so that `-fno-exceptions -fno-rtti` apply only to C++. Android adds `-fvisibility=hidden -ffunction-sections -fdata-sections`. `-g` was removed from the release flags. The emscripten build stopped running out of memory in Chrome and Firefox once `-g` was dropped and `-O3` was passed at link time. ee2a1b7d2 2014-05 “Split CFLAGS/CXXFLAGS on core.mk, warning fixes” · 718cace9a 2015-02 “emscripten: Fix chrome & firefox OOM by enabling opts” · 014d73949 2015-03 “Update for ndk-10d, improved build options” · 30709c948 2015-05 “Android makefile: C doesn't support -fvisibility-inlines-hidden”

### flycast-010 — Decouple producer and renderer with a single-slot queue that drops when busy
Lens: concurrency · Source: Flycast
Principle: Run rendering on its own thread and give it one pending-work slot. If a new frame arrives while the slot is still full, recycle the new one instead of queueing it. The producer never waits for the GPU, latency stays bounded at one frame, and memory stays bounded. The producer waits only for the short parse/prepare phase that must happen in order, not for the draw.
Why it is faster: —
Applies elsewhere: "latest value wins" mailboxes for sensor data, coalescing `requestAnimationFrame` updates, and video encoders that drop frames when the encode queue is full.
Watch out: —
Seen in Flycast: `QueueRender` (`pvr/ta_ctx.cpp`) now returns false and calls `tactx_Recycle(ctx); fskip++` if `rqueue` is occupied. `rend_thread` calls `rend->Process()` (TA parse and texture cache), then signals `re.Set()` before the async `Render()`. `rend_end_render` waits on `re` only if a render is pending, and `FinishRender` clears the slot. A later commit added `rend_framePending()` so frontends can check the slot, and builds without threads call `rend_single_frame()` inline. 411aade5e 2014-05 “Improve async/mt rendering & sync logic.” · 057b1075b 2015-07 “ta/mt: add rend_framePending? for better mt sync from clients” · 59c6012db 2015-02 “pvr/rend_if: TARGET_NO_THREADS support” · 29906d841 2014-12 “Texkill, palette_update on correct place.”

### flycast-011 — Evict cache entries by age, a few per frame
Lens: memory-layout · Source: Flycast
Principle: Store a "last dirtied" timestamp (a frame number) instead of a boolean in each cache entry. Then sweep periodically for entries older than a threshold. Cap the number freed per sweep so cleanup never spikes a frame. Memory stays bounded without a full LRU structure.
Why it is faster: —
Applies elsewhere: GPU resource caches in game engines, idle-connection reaping in connection pools, and incremental expiry of stale keys in key-value stores.
Watch out: —
Seen in Flycast: The `TextureCacheData::dirty` field in `rend/gles/gltex.cpp` went from `bool` to `u32` and now holds `FrameCount`. `CollectCleanup()` runs from `ProcessFrame` and deletes textures dirtied more than 120 frames ago, stopping after about 5 per call. f1ef4877b 2014-05 “Cleanup textures after they are dead for a while”

### flycast-012 — Track cache invalidation with page write-protection
Lens: memory-layout · Source: Flycast
Principle: A cache derived from memory that can change, such as decoded textures, can be invalidated without checking or hashing on every lookup. Write-protect the source pages, and when the fault handler fires, mark the dependent entries dirty. Lookups then cost nothing until a write actually happens. The protected range must cover all the data the entry depends on.
Why it is faster: —
Applies elsewhere: copy-on-write and dirty-page tracking in VMs and garbage collectors, `userfaultfd`-based snapshotting, and incremental backup tools that use write barriers.
Watch out: —
Seen in Flycast: `TextureCacheData` locks VRAM with `libCore_vramlock_Lock(start,end,this)`. Mip-mapped textures offset `sa`, so the lock missed their headers. A new `sa_tex` field now holds the true start, and the lock covers `sa_tex..sa+size-1`. 09caf4a55 2014-12 “Lock the entire texture, not just the pixel array.”

### flycast-013 — Keep expensive debug checks behind a runtime flag marked unlikely
Lens: branching-fast-paths · Source: Flycast
Principle: Calls like `glGetError` can force a driver to synchronize, so checking after every call is very slow. Wrap them in a macro that tests a configuration flag hinted as unlikely. The release cost drops to one predictable branch, and you can still turn the checks on without rebuilding.
Why it is faster: —
Applies elsewhere: Vulkan validation layers enabled only in debug, `assert`-style checks gated by an environment variable, and verbose logging behind `if (log.isDebugEnabled())`.
Watch out: —
Seen in Flycast: `glCheck()` in `gles.h` used to be an unconditional `verify(glGetError()==GL_NO_ERROR)`. It became `if (unlikely(settings.validate.OpenGlChecks))`, read from the `validate:OpenGlChecks` config key, which defaults to 0. A compile-time variant was tried and reverted because the runtime option already existed. 64b2fcbb1 2015-04 “Disable glCheck by default, move to validate:OpenGlChecks config” · e5965e356 2015-05 “gles: Disable glChecks by default, glGetError can be painfully slow” · 7d75ee383 2015-05 “gles: revert e5965e356...”

### flycast-016 — Run work in fixed time slices and do bookkeeping between slices
Lens: algorithms · Source: Flycast
Principle: Execute compiled units back-to-back while subtracting their cost from a budget. Handle interrupts, timers and other system updates only when the budget runs out. The inner loop does nothing except fetch and call, and the slow checks run once per slice.
Why it is faster: —
Applies elsewhere: cooperative schedulers that check for preemption every N operations, game loops with fixed-timestep physics, and interpreters that poll for signals every N bytecodes.
Watch out: —
Seen in Flycast: `ngen_mainloop` in the new x64 and C++ backends resets `cycle_counter = SH4_TIMESLICE` and loops `do { bm_GetCode(pc)(); } while (cycle_counter > 0);`. Each block starts by subtracting its `guest_cycles`. After the loop it calls `UpdateSystem()` and, if needed, `rdv_DoInterrupts_pc`. A follow-up fixed the reset so it happens every slice, not just once. 3852cf9ac 2015-07 “Minimal x64 jit: Desync'd, but runs” · 8cb6d6578 2015-07 “Count cycles correctly” · cd80850ac 2015-07 “First idea for a fully portable "jit"”

### flycast-017 — Pre-bind operands into closure arrays when you can't generate code
Lens: jit-interpreters · Source: Flycast
Principle: On platforms that forbid executable memory, you can still translate each block once into an array of small objects. Each object holds pointers resolved ahead of time to its operands and an `execute()` method. At run time there is no decoding and no operand lookup. Only the dispatch remains, making this a "threaded code" interpreter.
Why it is faster: —
Applies elsewhere: closure compilation in tree-walking interpreters (compiling an AST to nested lambdas), compiled query plans built from operator objects, and template engines that turn a template into an array of render functions.
Watch out: —
Seen in Flycast: `rec-cpp/rec_cpp.cpp` builds an `opcodeExec*` object for each SHIL op. Examples are `opcode_mov32` holding `u32* src, *dst`, and the `opcode_cc_aCaCbC` family, which stores a function pointer and register pointers taken from `reg_ptr()`/`imm_value()`. Each block becomes a `fnblock<cnt>` whose `execute()` subtracts cycles and runs the ops. Interpreter fallback was then turned off. This backend serves NaCl, emscripten and iOS. cd80850ac 2015-07 “First idea for a fully portable "jit"” · bfef96962 2015-07 “rec-cpp: Implement stuff up to the corejit layer” · 67a8ceabd 2015-07 “shil/param: add imm_value()”

### flycast-018 — Duplicate dispatch code so each indirect call site has one target
Lens: branching-fast-paths · Source: Flycast
Principle: The branch target buffer predicts an indirect call by its address. One shared `execute()` that calls many different handlers mispredicts constantly. Stamp out a separate copy of the dispatcher for each target function, for example by template id, so every call site always jumps to the same place. You trade code size for predictability.
Why it is faster: —
Applies elsewhere: "replicated switch" or computed-goto dispatch in interpreters such as CPython and Lua, and monomorphic inline caches in JS engines.
Watch out: —
Seen in Flycast: Each `opcode_cc_*` struct in `rec_cpp.cpp` became `template<int id> struct opex`. `createType` gives each distinct handler `fun` its own id through a `map<void*,int>` and picks `ctors[id]` from an `XREP_64`-generated table, so up to 64 handlers get private dispatcher copies. The commit message calls this wasteful and says the real goal is inlining. e4138f9a6 2015-07 “rec-cpp: Wastefully generate more dispatchers for better BTB cache locality”

### flycast-020 — Fully unroll loops whose trip count is a compile-time constant
Lens: compiler-build · Source: Flycast
Principle: When the iteration count is a template parameter, expand the loop body with macros into a chain of guarded calls. The compiler removes every guard above the count, which leaves straight-line code with no loop counter, no back-edge and a separate call site per position for the predictor. Keep a plain loop in debug builds.
Why it is faster: —
Applies elsewhere: fixed-size matrix and vector kernels, unrolled hash rounds in crypto code, and Duff's device-style copies.
Watch out: —
Seen in Flycast: `fnblock<cnt>::execute()` swapped `for (i<cnt) ops[i]->execute();` for `DREP_512(0)`, which expands to `if (x < cnt) ops[x]->execute();` for x=0..511. A later commit restores the loop under `_DEBUG` and adds a compile-time `MIPS_COUNTER` option. 1710f76ab 2015-07 “unroll the execution loop” · 51a6ed214 2015-07 “rec/all: optional mips counter on recs”

### flycast-021 — Cap unit size so fixed reservations and tables stay valid
Lens: jit-interpreters · Source: Flycast
Principle: If compiled blocks have a hard maximum size, the code emitter can reserve a fixed amount of space up front. Per-block tables and template instantiations can then use fixed arrays, with no overflow checks or reallocation in the middle of emitting. A soft limit ends blocks early, and a hard limit is asserted.
Why it is faster: —
Applies elsewhere: max-frame-size limits in network protocols that allow fixed buffers, page-size caps in B-trees, and bounded batch sizes in stream processors.
Watch out: —
Seen in Flycast: `dec_DecodeBlock` now ends a block once `oplist.size() >= BLOCK_MAX_SH_OPS_SOFT` (500) and asserts `<= 511`, which matches the 512-entry unrolled `fnblock`. The x64 JIT's free-space guarantee was corrected to 16 KB. The ARM7 recompiler raised its reserve from 1 KB to 64 KB because of large per-block compiles. a82f9704d 2015-07 “shrec: Limit max blocksize to 511 shops” · 82ef7d075 2015-07 “jitx64: 16KBs of emit space are guaranteed, not 64” · f2d8ed06f 2015-08 “arm7rec: Increase the emitter guarantee to 64 KB”

### flycast-022 — Bump-allocate the code cache and flush it all when full
Lens: allocation · Source: Flycast · Also: ares
Principle: Allocate translated code linearly from one big region. When it nears full, or when a known code-reload point is reached, throw everything away and start over. There is no fragmentation and no per-block freeing, and a reset is cheap compared with tracking which blocks are still live.
Why it is faster: —
Applies elsewhere: arena or region allocators for per-request memory, nursery collection in generational GCs, and frame allocators in game engines.
Watch out: —
Seen in Flycast: `rdv_CompilePC` flushes the cache when `emit_FreeSpace() < 16*1024` or at the BIOS boot entry points. Those addresses were fixed to the 0xAC... mirror the BIOS actually uses. In rec-cpp, a block that would exceed the fixed block-object pool forces a reset through `emit_Skip(emit_FreeSpace()-16)`. ae8ce5ad2 2015-02 “sh4/rdv_CompilePC: Fix boot time automatic cache flushes” · fc01428c3 2015-07 “rec-cpp: Force resets when running out of blocks” • Also seen in ares (ares-153): `bump_allocator::acquire()` returns `_memory + _offset` for a block of unknown size, `reserve(size)` commits it, and `release()` resets the offset to zero. The executable backing buffer is a static 64 MiB array aligned to 64 KiB and mprotected once, with no runtime mapping, on every platform except macOS. current code: nall/nall/bump-allocator.hpp, ares/ares/memory/fixed-allocator.cpp

### flycast-023 — Direct-mapped address-to-handler table
Lens: lookup-tables · Source: Flycast
Principle: To find the compiled code for an address, index a flat array by the address itself (shifted by the instruction alignment and masked), rather than searching a map. A lookup is one shift, one AND and one load. Unfilled entries point at a "compile me" stub, so the hot path has no branch for a miss. Size the table to the real address space it covers.
Why it is faster: —
Applies elsewhere: page tables, jump tables for dense switch statements, and flat arrays indexed by file descriptor in event loops.
Watch out: —
Seen in Flycast: `FPCA(x)` in `dyna/blockmanager.cpp` is `sh4rcb.fpcb[(x>>1)&FPCB_MASK]`, and `bm_GetCode` returns that entry directly. The table had a fixed 8M entries. It now follows `FPCB_SIZE = RAM_SIZE/2`, and its offset from the context block is checked with `verify(rcb_noffs(fpcb) == -(FPCB_SIZE*sizeof(void*)+0x40000))`. 9bc0a8ff0 2015-08 “shrec/bm: BM lookup table size follows RAM_SIZE” · 5b609a6be 2015-05 “linux: Sh4 dynarec works”

### flycast-025 — Bring up a new JIT with per-op interpreter fallback, then widen native coverage
Lens: jit-interpreters · Source: Flycast · Also: ares
Principle: Start a new backend by emitting calls to the interpreter for every instruction, plus a native dispatch loop. That alone removes decode overhead. Then emit native code for the most common ops (moves, memory, block exits) one at a time, turning off fallback once coverage is complete. Every step stays runnable.
Why it is faster: —
Applies elsewhere: porting a VM to a new CPU architecture, and incrementally rewriting hot paths of a Python service in a native extension.
Watch out: —
Seen in Flycast: `rec-x64/rec_x64.cpp` began as an Xbyak `BlockCompiler` that only emitted `shop_ifb` calls to `OpDesc[op]->oph`, plus `jdyn`/`mov32`. It then added `mov64`, `readm` and `writem`, with ABI-correct `call_regs` for Windows and SysV, and set `InterpreterFallback = false`. Finally it emitted every block end inline (static, conditional, dynamic and interrupt) and set `OnlyDynamicEnds = false`. 3852cf9ac 2015-07 “Minimal x64 jit: Desync'd, but runs” · b056cf6ce 2015-07 “Implement jitcore opset for x64” · 4aaff464b 2015-08 “shrec/jitx64: Implement all block ends” • Also seen in ares (ares-038): In v119r15, `n64/cpu/recompiler.cpp` grew from a decoder include into an explicit per-opcode switch that still mostly calls `CPU::instructionXXX` with pointers to registers. In v119r16, simple ops are emitted inline. For example, `ADDU` went from three `lea`s plus a `call(&CPU::instructionADDU)` to `mov(esi, Rs); add(esi, Rt); movsxd(rsi, esi); mov(Rd, rsi);`, while `DADDU` still calls `CPU::DADDU`. The PS1 CPU and the RSP followed the same path. 25771ba20 2021-05 “Update to ares v119r15 release.” · 1b574d0bb 2021-05 “Update to ares v119r16 release.”

### flycast-027 — Rasterize in blocks with trivial accept and reject
Lens: simd · Source: Flycast
Principle: Walk a triangle's bounding box in small tiles, such as 4×4 pixels, using the incrementally updated edge functions. If a tile is entirely outside any edge, skip it. If it is entirely inside all edges, shade it with no per-pixel coverage test. Only partially covered tiles need per-pixel masks. Most interior and exterior pixels skip the coverage work entirely.
Why it is faster: —
Applies elsewhere: software rasterizers (Larrabee-style, the WARP renderer), quadtree culling in maps, and coarse-then-fine hit testing in UI toolkits.
Watch out: —
Seen in Flycast: The imported SSE software renderer (`rend/soft/softrend.cpp`) steps 4×4 blocks, using `EvalHalfSpaceFAny` (OR of the three edge values) to skip and `EvalHalfSpaceFAll` (compared against per-edge `PlaneMinMax` bounds) to fill whole blocks with `PixelFlush<false>`. Partial blocks compute a 4-lane mask from `_mm_cmpgt_epi32` on the OR of the edge values. 8cce1a497 2015-08 “rend/soft: Import the experimental core from nulldc/drkpvr, better renderer modularization”

### flycast-028 — Shade four pixels per instruction with lane masks
Lens: simd · Source: Flycast
Principle: Evaluate interpolants as plane equations for four pixels at once (`x*ddx + y*ddy + c` in SSE registers). Depth-test all four with one compare, get a bitmask, return early if no lane passes, and blend with AND/ANDNOT/OR instead of branching per pixel. Work and branches drop by about 4×.
Why it is faster: —
Applies elsewhere: SIMD image filters, vectorized predicate evaluation in columnar databases, and branchless SIMD JSON parsing.
Watch out: —
Seen in Flycast: In `softrend.cpp`, `PlaneStepper::Ip` and `InStep` compute Z/U/V and colour for four pixels. `PixelFlush` calls `_mm_cmpgt_ps(invW,*zb)` and then `_mm_movemask_ps`, returns when the mask is 0, and merges colour and depth through the mask when the mask is not 0xF. Bilinear filtering was added later with `_mm_mullo_epi32` weights, and perspective correction with `_mm_div_ps`. 8cce1a497 2015-08 “rend/soft: Import the experimental core ...” · 3e9431b2b 2015-08 “softrend: billinear filtering, perspective uv, alpha test, ARGB1555”

### flycast-031 — Pre-expand data into the layout the inner loop consumes
Lens: memory-layout · Source: Flycast
Principle: If every sample needs a neighbourhood, such as the 2×2 texels for bilinear filtering, store each sample with its neighbours already decoded and laid out next to each other in aligned memory. You spend more memory, and in exchange each lookup is one aligned 16-byte load in place of four scattered loads and decodes.
Why it is faster: —
Applies elsewhere: padded or duplicated borders in image tiles, interleaving struct-of-arrays data for SIMD, and denormalized rows in OLAP tables.
Watch out: —
Seen in Flycast: The texture cache upload in `gltex.cpp` now allocates `_mm_malloc(w*h*16, 16)`. For each texel it stores four pre-decoded ARGB values (x+1,y+1 / x,y+1 / x+1,y / x,y, with wrap-around), so the soft renderer reads a whole bilinear quad at once. 3e9431b2b 2015-08 “softrend: billinear filtering, perspective uv, alpha test, ARGB1555”

### flycast-032 — Cull primitives that can't contribute before rasterizing them
Lens: algorithms · Source: Flycast
Principle: The signed area you already compute for winding tells you whether a triangle faces away or is degenerate (near-zero area). Return before any setup or per-pixel work when it does. Discarding early costs one multiply and compare and saves a whole rasterization.
Why it is faster: —
Applies elsewhere: frustum and back-face culling in engines, skipping zero-size elements in layout, and early-out filters before an expensive join.
Watch out: —
Seen in Flycast: `Rendtriangle` now saves `area`. If `CullMode != 0`, it returns when `fabsf(area) < FPU_CULL_VAL`. For modes 2 and 3 it also returns based on the sign of `area`, XORed with the strip parity `vertex_offset`. fcf273dd1 2015-08 “softrend: Add support for culling”

### flycast-033 — Split the output into bands, one per thread, with all mutable state per thread
Lens: concurrency · Source: Flycast
Principle: Divide the frame into horizontal bands and let each thread rasterize every primitive clipped to its own band. No pixel is shared, so the only lock is around the shared lookup cache. Per-call scratch state must move from globals onto each thread's stack. Leave a core for the rest of the program and cap the thread count, since gains flatten after 2–3 threads.
Why it is faster: —
Applies elsewhere: tiled parallel image processing, sharded counters, and parallel map-reduce over disjoint key ranges.
Watch out: —
Seen in Flycast: `softrend::Render` runs `#pragma omp parallel num_threads(tcount)` with `tcount = min(num_procs-1, pvr.MaxThreads)`, where MaxThreads defaults to 3. Each thread renders all lists clipped to `RECT{0,y_start,640,y_end}`. The interpolator state `IPs ip` and the `_mm_load_ps_r` temporary moved from statics to locals. `raw_GetTexture` is wrapped in `#pragma omp critical`. The commit before this one had made the scissor area a parameter. dfc341a5f 2015-08 “softrend: OpenMP support, fix present overflow during untiling” · 19fa2f167 2015-08 “softrend: Limits on #defines, scissor area as rect”

### flycast-034 — Compare and range-check floats as integers
Lens: branching-fast-paths · Source: Flycast
Principle: IEEE-754 floats that are non-negative sort the same way as their bit patterns read as signed integers. So `(x|y) < 0` tells you either value is negative, and `max(x,y) > bits(1024.0f)` is a range check, all on integer registers with no float compares. That is cheap on CPUs where FP compares and flag transfers are slow.
Why it is faster: —
Applies elsewhere: radix-sorting floats by bit pattern, fast clamping in DSP code, and branchless bounds tests in collision broad-phase.
Watch out: —
Seen in Flycast: `update_fz` in `pvr/ta_vtx.cpp` now takes the vertex. It returns early if `(s32)x|(s32)y` is negative or `max(x,y) > 0x44800000` (1024.0f), and compares `z` to `fZ_max` as an `s32`. Modifier-volume vertices no longer update the far Z at all. 332ab95b6 2014-01 “Don't count mv->z, only vtx->z within 0,0 & 1024,1024.”

### flycast-035 — Put the FPU in fast mode and reprogram it only when the mode changes
Lens: platform · Source: Flycast
Principle: On many CPUs, denormal operands send floating-point instructions down a slow microcode path. Enable flush-to-zero or "run-fast" mode when the semantics allow it. Writing the control register (`ldmxcsr`, `fmxr fpscr`) is itself costly, so cache the last mode you set and write the register only when the requested rounding or denormal mode differs.
Why it is faster: —
Applies elsewhere: audio DSP plugins that set FTZ/DAZ to avoid denormal stalls in reverb tails, and ML inference runtimes with flush-to-zero turned on.
Watch out: —
Seen in Flycast: `enable_runfast()` in `linux/common.cpp` sets the ARM VFP FPSCR to RunFast. `SetFloatStatusReg` in `sh4_core_regs.cpp` compares `fpscr.RM/DN` against `old_rm/old_dn` before mapping guest round-to-zero and denormals-as-zero onto MXCSR bits `3<<13` and `1<<15`. That mapping now also works under GCC (inline `ldmxcsr`) and through `_controlfp` under MSVC. 63485e7a7 2014-12 “SetFloatStatusReg: support gcc for x86” · c60fca497 2015-08 “sh4: Cleanup SetFloatStatusReg a bit” · c8f982b8f 2015-07 “linux-common: TARGET_NO_EXCEPTIONS, TARGET_NO_THREADS”

### flycast-036 — Swap through a scalar temporary, not a temporary array
Lens: memory-layout · Source: Flycast
Principle: To exchange two arrays element by element, use one register-held temporary inside the loop. Copying everything into a stack buffer first doubles the memory traffic, and the buffer stops the compiler from keeping values in registers.
Why it is faster: —
Applies elsewhere: in-place reversal or rotation of buffers, double-buffer swaps (better still, swap the pointers), and register-bank switches in VMs.
Watch out: —
Seen in Flycast: The register-bank and FPU-bank swap functions in `sh4_core_regs.cpp` changed `u32 temp[8]`/`temp[16]` to a single `u32 temp` per iteration. cbda836a9 2015-07 “she/core: Less stupid register swap functions”

### flycast-037 — Keep blocking work off the UI or event thread, and don't over-parallelize it
Lens: concurrency · Source: Flycast
Principle: Directory scans, network fetches, log generation and the main emulation loop all block whatever thread they run on. Move them to a background worker so the UI or browser event loop stays responsive. A single job that has nothing to run alongside gains nothing from a thread pool, so use the default serial executor for it.
Why it is faster: —
Applies elsewhere: Web Workers for parsing and compute, Android `WorkManager` or coroutines on `Dispatchers.IO`, and running a game loop off the browser main thread.
Watch out: —
Seen in Flycast: The Android `FileBrowser` moved its game scan into a `LocateGames extends AsyncTask` and builds the list in `onPostExecute`. Game details are now preloaded per list item by an `XMLParser` AsyncTask, so icons appear without a click. Later, the `executeOnExecutor(THREAD_POOL_EXECUTOR)` calls were replaced with plain `execute()`. On NaCl, `dc_init`/`dc_run` moved from `HandleMessage` to a `pthread_create`'d `emuthread`. db324d950 2014-01 “First step in converting to async to reduce loading times” · 8b76e8465 2014-04 “Preload game details to allow setting icons, faster loading” · 2d9767b3b 2014-12 “Avoid thread pool when executing non-parallel task” · d0674b3cd 2015-07 “nacl: Use nacl_io, emulation on background thread, print speed stats”

### flycast-038 — Decode images at the size you display them, and free them promptly
Lens: memory-layout · Source: Flycast
Principle: Read only the image header first to learn the dimensions, then decode with a subsample factor so the full-resolution bitmap is never allocated. Recycle temporary bitmaps as soon as the scaled copy exists. Peak memory, decode time and GC pressure all fall.
Why it is faster: —
Applies elsewhere: responsive `srcset` images on the web, thumbnail pipelines, and Glide/Coil downsampling on Android.
Watch out: —
Seen in Flycast: `InputModFragment` decodes the button sheet with `options.inSampleSize = 2` and halves its crop coordinates. A later commit calls `image.recycle()` before resizing. `XMLParser` uses `inJustDecodeBounds` to compute a sample ratio for a 72-pixel target before decoding the real image. 4525bfcca 2014-01 “Resample images when loading, Don't recycle during use” · 2a8b099ac 2014-06 “Clear the image from memory before resizing it” · 14d29d967 2015-07 “Add caching of image files for disk info”

### flycast-039 — Cache remote assets on local storage
Lens: io · Source: Flycast
Principle: Before fetching an asset over the network, look for a local copy keyed by its name. On a miss, download it, scale it and write the result to disk. Later loads cost one local file read with no network round trip and no decode-and-scale step.
Why it is faster: —
Applies elsewhere: HTTP caches and service workers, CDN edge caches, and avatar or thumbnail caches in chat apps.
Watch out: —
Seen in Flycast: `XMLParser` now looks for `getExternalFilesDir()/images/<index>.png` and decodes it with `BitmapFactory.decodeFile`. Only when the file is missing does it open the URL, sample-decode the image and write it out with `FileOutputStream`. 14d29d967 2015-07 “Add caching of image files for disk info”

### flycast-040 — Pass settings through a direct call, not a file round-trip
Lens: io · Source: Flycast
Principle: If two components in the same process exchange settings by writing a config file and having the other side parse it, every change costs file I/O and string parsing. Call a setter across the boundary instead. The value arrives at once and nothing touches the filesystem.
Why it is faster: —
Applies elsewhere: passing options over IPC or FFI instead of temp files, environment-free configuration of embedded scripting engines, and in-memory feature-flag stores.
Watch out: —
Seen in Flycast: Android's `ConfigureFragment` used to rebuild `emu.cfg` line by line. It now calls JNI setters (`JNIdc.dynarec`, `unstable`, `limitfps`, `mipmaps`, `frameskip`, `pvrrender`, ...) that write `settings.*` directly, and `LoadSettings` skips the file-based path on Android. The commit's own description is "stop wasting CPU on files". af3e06426 2014-02 “Convert Android config to GUI, stop wasting CPU on files”

### flycast-041 — Send input across the boundary when it changes, not every frame
Lens: io · Source: Flycast
Principle: If input state is copied across an expensive boundary (JNI, IPC) once per frame, you pay that cost even when nothing changed, and new input can wait up to a frame. Push the state from the event handler instead. Idle frames cost nothing and latency drops.
Why it is faster: —
Applies elsewhere: event-driven rather than polled UI state sync, change-data-capture instead of periodic table scans, and interrupt-driven rather than polled GPIO.
Watch out: —
Seen in Flycast: `GL2JNIView` used to call `JNIdc.kcode(kcode_raw,lt,rt,jx,jy)` inside the draw loop. It now has `pushInput()`, which runs from the touch handler and from the key and motion handlers in `GL2JNIActivity`. e46993f96 2014-02 “push the input data from java to c on each event”

### flycast-042 — Raise the priority of the thread that matters
Lens: concurrency · Source: Flycast
Principle: When an optional subsystem such as audio is turned off, give the main work thread maximum priority so the scheduler doesn't share time with threads that have nothing useful to do.
Why it is faster: —
Applies elsewhere: real-time priority for audio threads in DAWs, and pinning a game's main thread above background loaders.
Watch out: —
Seen in Flycast: When `sound_enabled` is false, `GL2JNIView` now calls `Thread.currentThread().setPriority(Thread.MAX_PRIORITY)` after creating `EmuThread`. 4782c144f 2014-02 “Add the thread priority to prevent unused sound thread use”

### flycast-043 — Offer cheaper framebuffer formats and hardware-accelerated surfaces
Lens: gpu-rendering · Source: Flycast
Principle: Fill rate and bandwidth grow with bits per pixel. Letting users choose a 16-bit color or depth buffer on weak GPUs trades precision for frame rate. Requesting a hardware-accelerated window keeps composition on the GPU.
Why it is faster: —
Applies elsewhere: RGB565 framebuffers on embedded displays, lower-precision depth in mobile games, and `will-change`/GPU layers for CSS animation.
Watch out: —
Seen in Flycast: The Android manifest declares `android:hardwareAccelerated="true"` and the GLES 2 feature. A "Rendering Depth" spinner (16/24/32) was added "to provide fps boost" and feeds the EGL config chooser. d167caf10 2014-01 “Let's feed this thing a little speed and see if it gets excited” · 65c1b947a 2014-02 “Add render depth to available options to provide fps boost”

### flycast-044 — Parse to primitives instead of boxed wrappers
Lens: allocation · Source: Flycast
Principle: In managed languages, `Integer.valueOf` returns a boxed object that may be heap-allocated, only to be unboxed right away. `Integer.parseInt` returns a primitive `int` with no allocation.
Why it is faster: —
Applies elsewhere: `Long.parseLong` in Java hot paths, avoiding `Number` objects in JS, and value types versus boxing in C#.
Watch out: —
Seen in Flycast: Three `Integer.valueOf(...)` calls in `ConfigureFragment` became `Integer.parseInt(...)`. Their results were stored straight into `int` preferences. 6c123ad04 2014-03 “Replace some Integer.valueOf calls with Integer.parseInt in ConfigureFragment.java...”

### flycast-045 — Parallelize builds to the core count and keep per-target object directories
Lens: compiler-build · Source: Flycast
Principle: Set build parallelism from the machine's processor count instead of a hard-coded value. Give each target platform its own object directory so switching targets reuses earlier objects instead of rebuilding everything.
Why it is faster: —
Applies elsewhere: `make -j$(nproc)`, separate CMake build trees per configuration, and per-arch output folders in cross-compiling CI.
Watch out: —
Seen in Flycast: `custom_rules.xml` now passes `-j${env.NUMBER_OF_PROCESSORS}` (default `-j4`, overridable with `NDKBUILDOPT`) to ndk-build. The Linux Makefile writes to `obj-$(platform)/` instead of a shared `obj/`. bedb8564f 2014-10 “Update ant/custom-build.xml to use env processor count ...” · d295bad50 2015-07 “Makefile: Use separate obj directories for each platform”

### flycast-048 — Remember which register already holds a value
Lens: jit-interpreters · Source: Flycast
Principle: A code generator without a full allocator can still track which value each scratch register currently holds. When a later instruction needs that value again, it emits a register move or nothing at all instead of a fresh load. The cache must be cleared at calls and at control-flow joins.
Why it is faster: —
Applies elsewhere: peephole passes in simple template JITs, and query engines that reuse a column already decoded in the current tuple instead of decoding it again.
Watch out: —
Seen in Flycast: `shil_param_to_host_reg()` in the ARM64 dynarec checks `reg_cache[param._reg]` first. On a hit it emits `Mov` or nothing; on a miss it emits `Ldr` and records the result with `set_reg_cache`. Stores update the cache too. The emitter calls `reg_cache.clear()` or `flush_reg_cache()` around runtime calls and branches. 3d8b01c51 2019-01 “arm64: Use x28 as sh4 regs base pointer. Reg cache to avoid fetches.”

### flycast-050 — Map 4-wide math onto SIMD lanes
Lens: simd · Source: Flycast
Principle: A 4-element dot product or a 4×4 matrix-vector multiply written as scalar code takes 16 multiplies and a chain of dependent adds. As SIMD, it is one 4-lane multiply followed by two horizontal adds, or four broadcast-multiply-accumulate steps for the matrix. That cuts the instruction count several-fold and shortens the dependency chain.
Why it is faster: —
Applies elsewhere: 3D math libraries (vec4/mat4), audio mixing, image color-matrix transforms, and cosine similarity in vector search.
Watch out: —
Seen in Flycast: The ARM64 dynarec translates the SH4 `fipr` instruction as `Ld1` of both vectors, `Fmul(v0.V4S(), ...)`, then `Faddp` twice. If both operands are the same register, it loads once and squares. x64 uses `movaps`, `mulps` and `haddps` twice. `ftrv` is a matrix transform built from `pshufd` broadcasts and `vmulps`. 18a16f83a 2019-01 “arm64: more native opcodes implemented” · e241613b8 2019-01 “x64 dynarec: implemented swaplb, fipr, ftrv, frswap...”

### flycast-057 — Check the time budget inside each block, and branch out only when it runs out
Lens: jit-interpreters · Source: Flycast
Principle: Instead of returning to a scheduler after every unit of work, have each unit subtract its cost from a counter held in a register and branch out only when the counter goes negative. The common path is one subtract and one predicted-not-taken branch.
Why it is faster: —
Applies elsewhere: preemption checks in green-thread runtimes, gas metering in smart-contract VMs, and cooperative yield points in long loops.
Watch out: —
Seen in Flycast: Each ARM64 block now begins with `Subs(w27, w27, block->guest_cycles); B(&cycles_remaining, pl); GenCallRuntime(intc_sched);`. `intc_sched` runs `UpdateSystem` and either returns into the block or diverts to `rdv_DoInterrupts`. Blocks no longer return to the main loop. 75a04d6d2 2019-01 “arm64 dynarec: implement tail calling and block linking”

### flycast-059 — Don't discard derived caches on unrelated invalidation signals
Lens: jit-interpreters · Source: Flycast
Principle: A cache built from source data only needs invalidating when that data changes. If it is wired to a broader signal, for example a hardware cache-flush request that implies nothing about content changes, a program that sends the signal often forces endless rebuilds.
Why it is faster: —
Applies elsewhere: build systems that rebuild on mtime changes instead of content hashes, HTTP caches cleared on every deploy, and ORMs dropping query caches on unrelated writes.
Watch out: —
Seen in Flycast: `CCN_CCR_write` in `core/hw/sh4/modules/ccn.cpp` called `sh4_cpu.ResetCache()`, which threw away all translated code, whenever the guest set the ICI (instruction-cache invalidate) bit. That call is now commented out, because “Shikigami No Shiro II sets ICI frequently”. Self-modifying code is caught by block checks instead. 22b18d97a 2018-10 “Don't flush the dynarec cache when the sh4 instruction cache is flushed”

### flycast-061 — Use immediate operands and set up a stack frame only when needed
Lens: jit-interpreters · Source: Flycast
Principle: Moving a constant into a scratch register before using it adds an instruction and a dependency; most ISAs take small immediates directly. Likewise, saving the link register in every function wastes a store and a load in leaf paths. Save it lazily the first time the code actually calls out.
Why it is faster: —
Applies elsewhere: shrink-wrapping in optimizing compilers, leaf-function optimization, and immediate-operand selection in any code generator.
Watch out: —
Seen in Flycast: The ARM64 binary-op helper changed from `Mov(w10, imm); reg3 = &w10` to `Operand(op->rs2._imm)`. The unconditional `Str(x30, ...)` prologue became `if (!frame_reg_saved) { Str(x30, ...); frame_reg_saved = true; }` inside `ngen_CallRuntime`. Function addresses go in a deduplicated literal pool (`function_literals`). e5ee48efa 2019-01 “arm64: implement swaplb and pref. Various optimizations”

### flycast-062 — Materialize flags with setcc instead of mov/mov/cmov
Lens: jit-interpreters · Source: Flycast
Principle: To turn a condition into 0 or 1, `setcc al; movzx reg, al` is shorter and needs no constant register, while `mov reg,0; mov tmp,1; cmovcc reg,tmp` needs three instructions and a temporary. For float equality, a single `ucomiss` plus flag tests replaces a compare-to-mask, a move to a GPR and an AND.
Why it is faster: —
Applies elsewhere: branchless boolean code in compilers, and comparator functions in sort kernels.
Watch out: —
Seen in Flycast: In `rec_x64.cpp`, `shop_adc`, `shop_sbc` and the `shop_set*` family now use `setc`, `sete`, `setge` and so on, followed by `movzx`. `shop_fseteq` and `shop_fsetgt` use `ucomiss` with `seta` or `lahf; test ah,0x44; setnp`, replacing `cmpeqss`, `movd` and `and`. 31163b8d7 2019-01 “x64 dynarec: various fixes and optimizations”

### flycast-063 — Compile small fixed programs into straight-line native code
Lens: jit-interpreters · Source: Flycast
Principle: When a short program is run thousands of times per second and rarely changes, re-decoding its instruction fields on every pass wastes most of the time. Decode it once into native code, with every loop unrolled and every field turned into an immediate, and rerun that code until the program changes. An interpreter is then needed only as a fallback.
Why it is faster: —
Applies elsewhere: BPF packet-filter JITs, compiled regular expressions, and shader or audio-graph compilation in DAWs.
Watch out: —
Seen in Flycast: The AICA sound DSP runs a fixed 128-step microprogram. `DSPAssembler::Compile` (`core/hw/aica/dsp_arm64.cpp`) calls `DecodeInst` for each step at compile time and emits straight-line AArch64. Accumulators such as `ACC`, `B` and `MDEC_CT` live in fixed registers (w19–w29), and a stopped DSP compiles to four `Stp xzr` stores. The audio CPU, an ARM7, got an ARM64 dynarec in the same week. A follow-up stops compiling in the DSP interpreter whenever the DSP JIT is in use. 3a064cd06 2019-01 “DSP dynarec for arm64” · 9f3f13eef 2019-01 “aica arm dynarec for arm64” · 1501e2273 2019-01 “Don't enable the dsp interpreter if using the dynarec”

### flycast-064 — Fuse recognized instruction idioms, with a way to turn fusion off
Lens: jit-interpreters · Source: Flycast
Principle: Some long instruction sequences always compute one simple thing, such as a bit-serial division loop. Pattern-matching the whole sequence and emitting a single native operation removes dozens of instructions. Because the pattern can differ subtly at edge cases, keep a switch that disables the fusion where it breaks.
Why it is faster: —
Applies elsewhere: compilers recognizing popcount and rotate idioms, and database optimizers turning correlated subqueries into joins.
Watch out: —
Seen in Flycast: The SH4 decoder's `MatchDiv32` in `core/hw/sh4/dyna/decoder.cpp` fuses the 32-step `div1` sequence. A new `settings.dynarec.DisableDivMatching`, auto-set for certain titles, returns `false` early because the match “doesn't handle division by zero and edge cases”. Later per-game entries follow the same pattern. 315205caa 2018-07 “Add setting to disable div32 matching (Pro Pinball Trilogy)”

### flycast-066 — Copy fixed-size packets as aligned SIMD blocks, with an early exit for the common case
Lens: memory-layout · Source: Flycast
Principle: For a hot routine that copies a 32-byte record and then usually does nothing more, three things help. Allocate the destination 32-byte aligned and copy it with one or two vector moves, not four scalar doubles. Compute the state transition with shifts and masks. Return straight away, marked `likely`, when no handling is needed.
Why it is faster: —
Applies elsewhere: ring buffers of fixed-size messages, network packet descriptors, and command buffers in GPU drivers.
Watch out: —
Seen in Flycast: `ta_thd_data32_i` in `core/hw/pvr/ta.cpp` copied four `f64`s. It now copies one `simd256_t` (two `__m128` or `uint64x2_t`) with `*dst = *src`, replaces `% 32` with `& 31`, and does `if (likely(!must_handle)) return;`. The TA buffer is allocated with `OS_aligned_malloc(32, 2 MB)`. 8e3b9b434 2017-04 “Cleanup/optimize ta_thd_data32_i” · 6ae93b31d 2017-05 “ta: use 256-bit struct” · dec5c7bdd 2018-04 “Added helper functions for aligned memory (de)allocation” · fe30e5850 2018-04 “tactx: Actually use aligned alloc functions”

### flycast-071 — Use a minimal shader for depth-only passes
Lens: gpu-rendering · Source: Flycast
Principle: A pass that only writes depth doesn't need textures, lighting or blending. Bind the cheapest program available and disable blending, and the GPU spends no time on per-fragment work whose results are thrown away.
Why it is faster: —
Applies elsewhere: depth prepasses and shadow-map rendering in game engines, and occlusion-query passes.
Watch out: —
Seen in Flycast: The translucent-polygon depth pass in `DrawSorted(multipass)` called `SetGPState<ListType_Translucent,true>`, which set up the full pipeline shader. It now calls `glcache.Disable(GL_BLEND)`, `glcache.UseProgram(gl.modvol_shader.program)` and sets cull mode only, because “We use the modifier volumes shader because it's fast. We don't need textures”. bccfb5b70 2018-05 “Faster implementation for translucent polys depth pass.”

### flycast-072 — Replace a fitted approximation with a texture lookup
Lens: lookup-tables · Source: Flycast
Principle: If a curve is defined by a table, fitting an analytic function to it on the CPU and evaluating that function per pixel is both slow and inexact. Upload the table as a small texture and sample it in the shader. The fit disappears from the CPU, and per-pixel work becomes one fetch with hardware interpolation.
Why it is faster: —
Applies elsewhere: color-grading LUTs, tone-mapping curves, gradient ramps in UI rendering, and precomputed BRDF textures.
Watch out: —
Seen in Flycast: `tryfit()` in `gles.cpp` did a least-squares fit of the 128-entry fog table to `B*log2(x)+A` with up to 10000 repeated samples per point, and the shader evaluated `sp_LOG_FOG_COEFS`. `UpdateFogTexture()` now uploads the table to `GL_TEXTURE1` with `GL_LINEAR` filtering, and the fragment shader indexes it by exponent and mantissa of `w`. 828990b92 2018-06 “Texture based fog table”

### flycast-073 — Keep rendered results on the GPU instead of reading them back
Lens: gpu-rendering · Source: Flycast
Principle: Reading pixels back from the GPU stalls the pipeline and then costs CPU conversion and a re-upload. If the consumer is another GPU pass, bind the render target directly as a texture and skip the round trip.
Why it is faster: —
Applies elsewhere: post-processing chains, video players that hand decoded frames to the compositor as textures, and canvas libraries that avoid `getImageData`.
Watch out: —
Seen in Flycast: For RGB565 and ARGB4444 render-to-texture, `handleRGB565` and `handleARGB4444` did `glReadPixels` plus a per-pixel repack and upload. They were replaced by `fbt.tex = fbt.renderTex`, which samples the FBO texture directly. An earlier change began placing RTT textures in the texture cache “to avoid copy to/from vram”. 77d997401 2019-01 “RTT: Performance improved for DC RGB565 and ARGB4444 textures - glReadPixels() removed” · 43c125c20 2018-05 “Better texture cache: avoid duplicate. Attempt to put the RTT texture in the cache to avoid copy to/from vram.”

### flycast-074 — Avoid pixel formats the driver emulates slowly
Lens: gpu-rendering · Source: Flycast
Principle: Some drivers silently convert packed 16-bit formats on every access. On those GPUs, rendering to a native 8-bit-per-channel format and packing on the CPU, or not at all, can be far faster than the format that looks cheaper.
Why it is faster: —
Applies elsewhere: picking BGRA over RGBA for uploads on some platforms, choosing a camera API's native YUV layout, and avoiding unaligned DMA formats on embedded GPUs.
Watch out: —
Seen in Flycast: The RTT path in `RenderFrame` checks `glGetString(GL_RENDERER)`. On Adreno 506 it uses `GL_UNSIGNED_BYTE` instead of `GL_UNSIGNED_SHORT_5_5_5_1`, because the comment notes rendering is “extremely slow (probably by some internal format conversions)”. `handleARGB1555` then reads back RGBA8 and uploads it without `texconv`. 127072ec8 2018-12 “RTT: Fixed slow rendering of RGBA5551 framebuffer format on Adreno 506”

### flycast-076 — Drop a frame instead of blocking the producer
Lens: concurrency · Source: Flycast
Principle: When the consumer is still busy with the previous item, making the producer wait serializes the two threads and wastes the parallelism. Discarding the new item, or replacing the queued one, keeps the producer running at full speed and the system responsive.
Why it is faster: —
Applies elsewhere: video streaming that drops late frames, telemetry with lossy queues, and UI frameworks that coalesce redundant repaint requests.
Watch out: —
Seen in Flycast: When a frame was already queued, `QueueRender` in `core/hw/pvr/ta_ctx.cpp` used to `frame_finished.Wait()` for render-to-texture frames. It now always calls `tactx_Recycle(ctx); fskip++; return false;`, and the FIXME notes that waiting “kills performance”. b02eccd5c 2018-05 “Fix performance problem with RTT waiting”

### flycast-077 — Split data-parallel image work statically across cores
Lens: concurrency · Source: Flycast
Principle: Per-pixel filters on large images parallelize trivially by rows. Giving each thread one contiguous slice through a fork-join construct costs less than queuing many small tasks through a general thread pool with futures. Leave one core free for the main thread.
Why it is faster: —
Applies elsewhere: thumbnail generation, image resizing in web backends, and parallel loops in scientific code.
Watch out: —
Seen in Flycast: xBRZ texture upscaling and deposterization (`core/rend/TexCache.cpp`) first ran through `parallelize()` on a `ctpl::thread_pool` with 8-row chunks and one `std::future` per chunk. It was then switched to `#pragma omp parallel num_threads(min(procs-1, MaxThreads))`, with one equal slice per thread. 31d8c1f09 2018-08 “Backport texture upscaling with xBRZ” · 85781e96e 2018-08 “Use OpenMP instead of ctpl for multithreading”

### flycast-078 — Tile vertical passes by column blocks
Lens: memory-layout · Source: Flycast
Principle: A filter that reads a pixel's upper and lower neighbors while walking down full image columns strides through memory and misses the cache constantly. Process narrow column blocks top to bottom instead. The rows above and below stay in cache across consecutive iterations.
Why it is faster: —
Applies elsewhere: separable blur and convolution, matrix transposition, and stencil computations in simulations.
Watch out: —
Seen in Flycast: `deposterizeV` in `TexCache.cpp` loops `for xb in blocks of BLOCK_SIZE (32)`, then `for y`, then `for x` within the block, so the three rows it touches stay cache-resident. `deposterizeH` walks rows naturally. 31d8c1f09 2018-08 “Backport texture upscaling with xBRZ”

### flycast-079 — Load assets on a background thread and remember misses
Lens: concurrency · Source: Flycast
Principle: Disk reads and PNG decoding on the render thread cause stalls. Push requests to a worker thread through a mutex-protected queue and let the renderer use the result when it is ready. Also record every key that had no file, so later requests skip the filesystem probe.
Why it is faster: —
Applies elsewhere: image lazy-loading in apps, a negative DNS cache, and 404 caching in CDNs and module resolvers.
Watch out: —
Seen in Flycast: `CustomTexture::LoadCustomTextureAsync` (`core/rend/gles/CustomTexture.cpp`) enqueues a `TextureCacheData*` and signals `wakeup_thread`. `LoaderThread` computes the hash and decodes the PNG. `LoadCustomTexture` returns immediately for hashes in `unknown_hashes`. A follow-up keeps a texture from being deleted while `custom_load_in_progress` is set. dd280ee24 2018-12 “Custom textures loading and dumping” · 3f98c2ba3 2018-12 “Don't delete a texture when async load is pending”

### flycast-081 — Replace spin-waits and shared counters with events and a ring buffer
Lens: concurrency · Source: Flycast
Principle: A `while (flag) ;` loop burns a whole core while it waits. A producer that may write only when the buffer is empty causes either stalls or dropped data. Block on an event instead. Then turn the buffer into a single-producer, single-consumer ring with an atomic write index, so producer and consumer run independently and an underrun can be detected and filled with silence.
Why it is faster: —
Applies elsewhere: audio callbacks in any OS, lock-free logging queues, and network driver RX/TX rings.
Watch out: —
Seen in Flycast: `coreaudio_push` in `core/oslib/audiobackend_coreaudio.cpp` did `while (samples_ptr != 0 && wait) ;`, now replaced with `bufferEmpty.Wait()`. A follow-up changed `samples_temp` into an 8 KB ring with `std::atomic<int> samples_wptr` and a reader-side `samples_rptr`. On underrun, the callback writes zeros instead of stale data. 39e0f378c 2018-04 “OSX: ... Use semaphore in core audio driver” · a953a0baa 2018-04 “Use ring buffer for samples and detect buffer underruns”

### flycast-082 — Pass absolute deadlines to timed waits
Lens: concurrency · Source: Flycast
Principle: POSIX `pthread_cond_timedwait` takes an absolute time. Passing a relative duration puts the deadline in the past, so the call returns at once and the caller spins or misses its pacing. Compute `now + timeout`, normalizing nanoseconds into seconds.
Why it is faster: —
Applies elsewhere: any POSIX condition-variable or semaphore code (`sem_timedwait`), and thread pools with idle timeouts.
Watch out: —
Seen in Flycast: `cResetEvent::Wait(msec)` in `core/linux/common.cpp` set `ts.tv_sec = msec/1000` directly. It now calls `clock_gettime(CLOCK_REALTIME, &ts)` (Mach `clock_get_time` on Darwin), adds the timeout and carries `tv_nsec` overflow. The commit subject says this “Fixes Android FPS drop”. 2b25244c9 2018-10 “cResetEvent: needs absolute timeout value. Fixes Android FPS drop.”

### flycast-083 — Detect changes by content hash, not write counters
Lens: algorithms · Source: Flycast · Also: Envoy
Principle: Bumping a revision counter on every write marks data dirty even when it is written back unchanged, or toggled away and back. Hashing the current contents when they are used shows whether anything actually differs, so a dependent cache is rebuilt only for real changes. A fast non-cryptographic hash keeps the check cheap.
Why it is faster: —
Applies elsewhere: ETags, React memoization keyed by value, build caches keyed by content (Bazel, ccache), and asset bundlers' content-hashed filenames.
Watch out: —
Seen in Flycast: Palette writes used to increment `_pal_rev_256` and `_pal_rev_16`, so 2D games that rewrote palettes every frame forced a texture re-upload on each render. `palette_update()` now computes `pal_hash_16[64]` and `pal_hash_256[4]`, and `TextureCacheData` compares its stored `palette_hash`. The FNV-1a byte loop was then replaced by `XXH32` over each bank, which the commit message measures as “15 us -> 6 us”. e54ac3654 2018-09 “Texture cache optimization for frequent palette updates” · 7dc5fef35 2018-11 “Use xxhash for palette hashing” • Also seen in Envoy (envoy-008): d8124438d5 2021-03-02 `addOrUpdateCluster` already computed the config hash for change detection, so it now passes that hash into `ClusterData` instead of hashing the proto again in the constructor. • 2cc469fd79 2022-06-27 Scoped RDS stores `config_hash_` per scope and skips scopes whose incoming hash is equal. • 20676d7180 2023-09-06 `GrpcServiceConfigWithHashKey` precomputes the hash at config load and compares hashes before the deep comparison. 9e106e591a 2023-09-11 applied it to ext_proc, which no longer hashes the whole GrpcService proto per stream. • c0322159f6 2023-12-05 added endian-independent scalar hashing. 1fbc9e51ca 2023-12-13 replaced serialize-then-hash with a reflection walk that feeds typed fields to xxHash. Hashing was 2-10x faster, which the changelog expects to cut config update or startup time by 10-25%. • 107460365a 2026-02-18 A cached metadata hash on each host replaced `MessageDifferencer::Equivalent` in EDS updates. For 5,000 hosts the comparison went from 4.4 ms to 0.001 ms. • f3bc124a4f 2026-10-07 FCDS stores a hash per filter-chain config instead of the full message and skips unchanged chains. 6cff6c2768 2026-11-13 keeps the deterministic serialization as a cache key and compares bytes instead of running a reflection diff.

### flycast-084 — Watch address ranges cheaply and act only when dirty
Lens: laziness · Source: Flycast
Principle: Rather than re-presenting a buffer every frame, set a dirty flag when a write lands inside the watched range, using a test of `!dirty` first so most writes pay one load and a branch. Act only when the flag is set. Likewise, arm write traps only on regions that actually exist, because each trap costs a fault.
Why it is faster: —
Applies elsewhere: dirty-rectangle repainting in GUIs, incremental snapshotting with soft-dirty page bits, and change observers in reactive UI frameworks.
Watch out: —
Seen in Flycast: `pvr_write_area1_8/16/32` in `core/hw/pvr/pvr_mem.cpp` set `fb_dirty` when `!fb_dirty` and the address lies within `fb1` or `fb2` `_watch_addr`. `rend_vblank` renders the framebuffer only when nothing else rendered that frame and the flag is set. Earlier, `libCore_vramlock_Lock` stopped write-protecting the VRAM mirror when VRAM isn't mirrored (`VRAM_SIZE == 0x800000` check). e59d7eaf3 2018-08 “Direct framebuffer writes support” · 13b2c4d8a 2015-08 “naomi: Don't lock vram mirrors if vram isn't mirrored”

### flycast-085 — Fast-path no-op instructions and decode fields only where used
Lens: algorithms · Source: Flycast
Principle: In an interpreter, unpacking every field of every instruction up front wastes work on fields that most paths never read. Decode each field inside the branch that needs it. Also recognize the all-zero instruction, which is common in sparse programs, and run a short special case for it.
Why it is faster: —
Applies elsewhere: protocol parsers that read only the fields a handler touches, lazy JSON or protobuf decoding, and bytecode VMs with NOP fast paths.
Watch out: —
Seen in Flycast: `AICADSP_Step` (`core/hw/aica/dsp_interp.cpp`) adds `if (IPtr[0]==0 && IPtr[1]==0 && IPtr[2]==0 && IPtr[3]==0) { ...; continue; }`. It moves `TWA`, `IWA`, `TABLE`, `MWT`, `MRD`, `NOFL`, `MASA`, `ADREB` and `NXADR` extraction into the branches that use them, and defines `verify()` away in `RELEASE` builds. 11738c9b1 2018-08 “DSP: interpreter optimization”

### flycast-087 — Avoid I/O that produces nothing new
Lens: io · Source: Flycast
Principle: Two common wastes: writing a file back right after reading it, and running a full dry-run serialization only to learn the output size. Write only when something changed, and get sizes from metadata such as `fseek`/`ftell` or `stat`.
Why it is faster: —
Applies elsewhere: settings stores that persist only on change, and upload handlers that size buffers from `Content-Length`.
Watch out: —
Seen in Flycast: `cfgOpen` in `core/cfg/cfg.cpp` used to reopen the config with `"wt"` and call `cfgdb.save` right after parsing. That rewrite was removed. `dc_loadstate` used to call `dc_serialize(&data, &total_size)` just to size its buffer, and now takes the size from the state file with `fseek(f, 0, SEEK_END); ftell(f)`. 99d75220e 2015-08 “cfg/cfg: Don't save the config immediately after opening.” · 4b78e2475 2019-02 “allocate mem based on state file size instead of current state size”

### flycast-089 — Ship optimized builds that target the host ISA, with unused checks compiled out
Lens: compiler-build · Source: Flycast
Principle: A debug flag like `-O0 -g` left in the shared build file can make release binaries several times slower. Enabling the target's SIMD unit (`-mfpu=neon`) lets the compiler and assembler use it. Compiling out unused features and assertions with preprocessor guards removes their checks from hot loops.
Why it is faster: —
Applies elsewhere: CI checks that release flags are set, `-march` tuning for known deploy targets, and `NDEBUG` stripping of `assert`.
Watch out: —
Seen in Flycast: `core/core.mk` was reverted from `-c -O0 -g` to `-c -O3`. The generic ARMhf rule in `shell/linux/Makefile` adds `-mfpu=neon` to `MFLAGS` and `ASFLAGS` when the platform string contains `neon`. `#define NO_MMU` wraps the MMU exception paths in `sh4_interpreter.cpp` in `#if !defined(NO_MMU)`. ae2b5e33b 2018-07 “core.mk: Revert `-O0 -g` to `-O3`” · 225b38dd1 2015-08 “Add optional NEON support in the generic ARMhf rule.” · d85e4d8df 2015-09 “sh4/mmu: Don't use exceptions when disabled”

### flycast-092 — Don't maintain data for a feature that is switched off
Lens: laziness · Source: Flycast
Principle: If a derived resource only matters when a feature is on, add the feature flag to its dirty check. Updates then cost nothing while the feature is disabled. The same applies to bookkeeping that only a strict or debug mode reads: compile it out of the normal build.
Why it is faster: —
Applies elsewhere: skip building analytics payloads when telemetry is opted out; don't regenerate a search index while search is disabled; compile out trace hooks in release firmware.
Watch out: —
Seen in Flycast: The GL renderers rebuilt the fog lookup texture whenever `fog_needs_update` was set. Now the check is `if (fog_needs_update && settings.rend.Fog)`. Separately, the interpreter's `OCBI`/`OCBP`/`OCBWB`/`PREF` handlers only touch the emulated operand cache under `#ifdef STRICT_MODE`. d524e3381 2019-04 “only update the fog texture if fog is enabled” · 287836803 2021-01 “interp: don't touch the sh4 cache in normal mode”

### flycast-095 — Give volatile entries their own cache area so they don't flush the stable ones
Lens: jit-interpreters · Source: Flycast
Principle: If a few entries are invalidated over and over, mixing them with long-lived entries forces frequent full flushes. Count failures per key. Once a key goes over a threshold, put it in a small separate area that can be cleared on its own, and leave the main area alone.
Why it is faster: —
Applies elsewhere: generational GC nurseries; an LRU cache with a separate small queue for one-hit wonders (2Q/ARC); CDNs giving short TTLs to frequently changing objects.
Watch out: —
Seen in Flycast: `rdv_BlockCheckFail` now counts `blockcheck_failures` per block. After more than 5 failures, the address goes into `smc_hotspots`. `rdv_CompilePC` emits those blocks into a separate `TempCodeCache` of `TEMP_CODE_SIZE`, and `bm_ResetTempCache(false)` clears only that area when it fills, so the main code cache survives. 992088098 2019-04 “dynarec: reserve code cache space for frequently changing blocks”

### flycast-096 — Use page protection to detect writes instead of checking on every use
Lens: memory-layout · Source: Flycast
Principle: Rather than checking on every execution that cached derived data still matches its source, write-protect the source pages with the MMU. The common case then costs nothing, and only a real write traps and invalidates. Protect each page once, when the first dependent registers, not on every registration. Because protected data is known to be immutable, it can also be folded into constants at compile time.
Why it is faster: —
Applies elsewhere: copy-on-write `fork()`; userfaultfd-based snapshotting; GC write barriers that use `mprotect`; detecting mutation of mapped config files.
Watch out: —
Seen in Flycast: Every compiled block used to emit code that compared its SH4 source against RAM before running. Now RAM pages that hold code are `mprotect`ed. Blocks on protected pages skip the check, and the SSA pass reads `readm` from an immediate address on a `read_only` block at compile time (`constprop_values[...] = ReadMem32(imm)`). A write fault throws away the blocks on that page and recompiles them with checks. Later, both RAM and the VRAM texture cache protect a page only when its lock list is empty (`if (list.empty()) _vmem_protect_vram(...)`). 318852e26 2019-06 “dynarec: use mprotect to protect code areas” · 6ab8b185c 2019-09 “Protect vram pages only when needed...” · bad10bae2 2019-09 “dyna: only protect ram when needed...”

### flycast-097 — Offer a cheaper validation level
Lens: jit-interpreters · Source: Flycast
Principle: Fully verifying a cached item can cost nearly as much as rebuilding it. A sampled check that compares only the first word catches most real changes for a fraction of the cost, and can be offered as a user-selectable tradeoff.
Why it is faster: —
Applies elsewhere: HTTP caches validating with ETag/size instead of comparing content; `rsync` quick-check on mtime+size; file watchers comparing headers before hashing.
Watch out: —
Seen in Flycast: `CheckBlock` in each backend gained `SmcCheckLevel`. `FullCheck` emits 8-byte `cmp`s over the whole block, `FastCheck` compares only the first dword at `block->addr`, and `NoCheck` emits nothing. 6438a402d 2019-03 “dynarec: add option to control smc code checks: none, fast, full”

### flycast-099 — Run SSA optimization passes on the IR before code generation
Lens: compiler-build · Source: Flycast
Principle: Version every register write (SSA). That makes constant propagation, dead-code removal and algebraic identities (a+0, a*1, a&0) simple and safe. Evaluate fully constant ops at compile time, using the same reference implementation the interpreter uses so results can't drift. Allocate host registers on the SSA form.
Why it is faster: —
Applies elsewhere: query-plan constant folding; a template engine pre-evaluating static expressions; shader compilers; Babel/Terser constant folding.
Watch out: —
Seen in Flycast: `core/hw/sh4/dyna/ssa.h` added `AddVersionPass`, `ConstPropPass`, `DeadCodeRemovalPass` and `ConstantExpressionsPass`, and `ssa_regalloc.h` added an SSA register allocator. Fully constant ops are run through the canonical SHIL implementations (`ExecuteConstOp`). Later a `WriteAfterWritePass` drops stores that are overwritten before being read. 692556ca5 2019-06 “dynarec: ssa initial commit” · 623d70d71 2019-06 “ssa register allocator and more ssa stuff” · 70ea7c4d7 2019-06 “ssa: use canonical implementations to compute const ops” · 318852e26 2019-06

### flycast-100 — Jump straight past trampoline blocks
Lens: jit-interpreters · Source: Flycast
Principle: If a branch target does nothing but jump elsewhere (an unconditional branch with a no-op delay slot), point the link at the final destination. This is jump threading, and it saves a dispatch per hop.
Why it is faster: —
Applies elsewhere: collapsing redirect chains in a URL shortener or router; path compression in union-find; linkers relaxing chained branches.
Watch out: —
Seen in Flycast: `SingleBranchTargetPass()` in `ssa.h` calls `skipSingleBranchTarget(block->BranchBlock)`. It looks at the target's opcode, and if it is `bra disp12` with a `nop` delay slot, it follows it (and detects infinite loops), adjusting cycle counts. 2d8bc6d6e 2021-02 “dynarec: skip single branch targets”

### flycast-105 — Put the hot special case first in a generated helper
Lens: jit-interpreters · Source: Flycast
Principle: When one address range needs very different handling and is very hot, test for it at the top of the generated stub and handle it inline. Everything else tail-calls the general handler. The hot case avoids the full dispatch, and the rest pays only a compare and a branch.
Why it is faster: —
Applies elsewhere: a `memcpy` fast path for small sizes; HTTP servers answering health checks before routing; allocators with a size-class fast path.
Watch out: —
Seen in Flycast: A `MemType::StoreQueue` write stub checks `(addr >> 26) == 0x38`. On a match it masks `addr & 0x3F` and stores directly into `p_sh4rcb->sq_buffer`, otherwise it `jmp`s to `WriteMem32`/`64`. The rewrite path picks this stub when the faulting address is in that range. It was done for x64/x86, ARM64 and ARM32. 65290a70f 2021-03 “x64/x86: store queue dynarec optimization” · e2f309b5c 2021-03 “arm64: store queue write dynarec optimization”

### flycast-107 — Flat page table for translation, with partial flushes
Lens: lookup-tables · Source: Flycast
Principle: A directly indexed array (vpn → physical page) turns translation into a shift and a load that generated code can inline. Fill it lazily on misses, and clear only the region affected by a mapping change instead of the whole table.
Why it is faster: —
Applies elsewhere: a flat array of sparse IDs to slots in an ECS; a radix page table in a hypervisor; Unicode property tables indexed by code point block.
Watch out: —
Seen in Flycast: The 4 GB `vmem32` mirroring scheme for WinCE games was removed. In its place is `u32 mmuAddressLUT[0x100000]`, filled with `mmuAddressLUT[vaddr >> 12] = paddr & ~0xfff`. The ARM64 rec keeps its base in `x27`. `mmuAddressLUTFlush(full)` can clear just slot 0 instead of the whole table. 373dffc65 2021-05 “wince: ditch vmem32. use address LUT in dynarec. finer grained flush”

### flycast-109 — Replace if-chains with a switch on dense keys
Lens: branching-fast-paths · Source: Flycast
Principle: A long chain of equality tests runs in linear time. A `switch` over register offsets, or over a few high address bits, lets the compiler emit a jump table or a binary search. Branch hints on the common case help further.
Why it is faster: —
Applies elsewhere: opcode dispatch in bytecode VMs; HTTP method or header-name parsers; MMIO register decoders in device drivers.
Watch out: —
Seen in Flycast: `pvr_WriteReg` went from a sequence of `if (addr == X_addr)` to a single `switch (addr)`. The area-0 read handler now switches on `addr >> 21` with `expected(base, 2)` instead of cascading range checks on `addr >> 16`. 0dc4b0753 2019-10 “pvr: use a switch instead of 100 ifs” · 6af509159 2021-03 “optimize area0 mem handlers”

### flycast-112 — Bypass translation for accesses known to be physical
Lens: memory-layout · Source: Flycast
Principle: When a caller always holds physical addresses, as DMA engines do, call the untranslated accessor. That skips the translation layer and its exception paths.
Why it is faster: —
Applies elsewhere: kernel code using direct-map addresses instead of walking page tables; ORMs offering raw queries for bulk jobs.
Watch out: —
Seen in Flycast: Channel-2 DMA in `dmac.cpp` used `ReadMem32(src)`, which goes through the MMU function pointers. It now uses `ReadMem32_nommu(src)`. d82e5e201 2019-04 “don't use mmu for dma”

### flycast-115 — Collapse a family of lookup tables into one
Lens: lookup-tables · Source: Flycast
Principle: If several tables differ only by a shift or a size parameter, keep one table and derive the variants with cheap arithmetic. The footprint drops by the number of variants, and the shared table stays hot in cache.
Why it is faster: —
Applies elsewhere: Morton/Z-order encoding with a single 256-entry spread table; one CRC table shared across message widths.
Watch out: —
Seen in Flycast: Texture detwiddling used `u32 detwiddle[2][8][1024]`, built with `twiddle_slow` for each axis and size. It became one `u32 detwiddle[1024]` that interleaves bits (`detwiddled |= (j & shift) << i`), about 64 KB down to 4 KB. 9826afa06 2020-02 “upload all texture mipmap levels to gpu” (simpler/smaller detwiddle table)

### flycast-119 — Drop empty work items before they reach the GPU
Lens: gpu-rendering · Source: Flycast
Principle: Zero-length draws and empty passes still cost state changes and command overhead. Remove them where they are produced, for example by popping the last list entry if it ended up empty.
Why it is faster: —
Applies elsewhere: React skipping empty fragments; culling zero-area UI elements before batching; skipping empty DMA descriptors.
Watch out: —
Seen in Flycast: The TA vertex parser gained `List::PopLast()`. When a new polygon starts and `CurrentPP->count == 0`, the empty `PolyParam` is removed. Empty render passes are eliminated too. 219a00163 2019-09 “pvr: eliminate empty polygons and render passes”

### flycast-120 — Build indices in a separate pass and merge compatible strips
Lens: gpu-rendering · Source: Flycast
Principle: Don't emit indices while parsing. Do it in a later pass that can see neighboring primitives, so consecutive strips with identical state can be joined with repeated (degenerate) vertices into one draw. Invalid vertices can be filtered in the same pass.
Why it is faster: —
Applies elsewhere: sprite batchers in 2D engines; map tile renderers merging same-material strips; text renderers batching glyph quads.
Watch out: —
Seen in Flycast: `ta_vtx.cpp` used to append indices inline as each strip was parsed. A new `make_index(polys, first, end, merge, ctx)` runs after parsing for the opaque, punch-through and translucent lists. It repeats `last_good_vtx` to link strips whose parameters match, and skips vertices with huge or negative coordinates. 8ada1d532 2020-03 “pvr: defer index building and strip merging, filter out infinite verts”

### flycast-121 — Use the expensive algorithm only when the input needs it
Lens: algorithms · Source: Flycast
Principle: Exact per-element ordering is costly. When metadata says the order is already defined, fall back to the cheap path. A coarser, cheaper option can also be offered as a tradeoff against accuracy.
Why it is faster: —
Applies elsewhere: sorting only when data isn't already sorted (Timsort runs); skipping the alpha-sort pass for opaque-only scenes; a planner choosing a hash join only above a size threshold.
Watch out: —
Seen in Flycast: The gl4 per-pixel (A-buffer) renderer now does per-pixel sorting only for passes with `autosort`. Non-autosort passes are drawn in submission order without the sorted A-buffer shader. Earlier, the GL renderer gained a "Per Strip" sort option next to "Per Triangle" and "Per Pixel" (`settings.rend.PerStripSorting` → `SortPParams` instead of `GenSorted`). f7655091b 2020-01 “gl4: only use per-pixel sorting when needed” · 433590b61 2019-05 “rend: Add per-strip TR sort option”

### flycast-123 — Push constants for small per-draw parameters
Lens: gpu-rendering · Source: Flycast
Principle: Small per-draw values written straight into the command buffer as push constants need no buffer allocation, mapping or descriptor update for each draw.
Why it is faster: —
Applies elsewhere: WebGPU immediate data; Metal `setVertexBytes`; root constants in D3D12.
Watch out: —
Seen in Flycast: The Vulkan renderer moved per-polygon parameters (clip test mode, trilinear alpha and so on) into push constants. Uniform buffers keep only per-frame data, laid out to std140 rules. 5b35b52fd 2019-10 “vulkan: fix uniforms alignment. Use push constants for per-poly params”

### flycast-125 — Linear tiling for small textures that change often
Lens: gpu-rendering · Source: Flycast
Principle: An optimally tiled image needs a staging copy and a layout transition on every update. For small images rewritten nearly every frame, a linear host-writable image avoids that copy and is faster overall.
Why it is faster: —
Applies elsewhere: streaming video frames into linear textures; dynamic UI atlases; CPU-written lookup textures in compute pipelines.
Watch out: —
Seen in Flycast: Small Vulkan textures use linear tiling and are written directly, and only the others set `needsStaging`. In-flight tracking was turned off for normal textures, because destroying and recreating textures after a skipped frame caused more skipped frames. The swap chain was capped at 2 images. 171312471 2020-07 “vulkan: linear tiling for small texs. don't track in-flight texs”

### flycast-126 — Record uploads into one command buffer per frame
Lens: gpu-rendering · Source: Flycast
Principle: Every command buffer has its own allocation, begin/end and submission cost. Record all of a frame's texture uploads into a single shared command buffer that is submitted with the frame, instead of one buffer per texture or overlay.
Why it is faster: —
Applies elsewhere: batching WebGPU `writeTexture` calls into one encoder; coalescing many small DMA requests into one descriptor chain.
Watch out: —
Seen in Flycast: Texture uploads first moved to a command buffer separate from rendering. Later, `texCommandPool.Allocate()` provides one buffer that all texture updates share for the frame, and `PrepareOverlay` returns one `vk::CommandBuffer` instead of a vector of unique buffers. 8f8f270cf 2019-10 “vulkan: use separate command buffer to upload tex...” · 4a8ce39bf 2021-04 “vk: use a single command buffer for all texture updates”

### flycast-128 — Pack per-element records into fewer bits
Lens: memory-layout · Source: Flycast
Principle: When millions of records (per-pixel fragments, per-polygon parameters) are written and read each frame, bandwidth is the bottleneck. Pack the fields the consumer needs into a couple of words, and colors into one u32, then unpack where they are used.
Why it is faster: —
Applies elsewhere: G-buffer normal packing; columnar formats with bit-packing; telemetry structs packed for radio links.
Watch out: —
Seen in Flycast: The Vulkan OIT renderer replaced its copy of the full `global_param_tr` with `trPolyParams`, two u32 per polygon (`(tsp & 0xffff00c0) | ((isp >> 16) & 0xe400) | ...`), which cut per-pixel and per-poly memory. gl4 then adopted packed pixel colors (`unpackColors(pixel.color)`) and packed poly params, sharing the GLSL with Vulkan. 4730da31d 2019-12 “vulkan: fix modvols... optimizations” · 408c65fcb 2021-05 “gl4: pack/unpack pixel colors. pack poly params. share GLSL with vk”

### flycast-129 — Cache descriptor bindings by resource identity
Lens: gpu-rendering · Source: Flycast
Principle: Allocating and writing a descriptor set for every draw is expensive. Key sets by (texture, sampler state) and reuse them. Keep samplers independent of textures so a handful of sampler objects serves everything.
Why it is faster: —
Applies elsewhere: bind-group caches in WebGPU engines; sampler deduplication in glTF loaders.
Watch out: —
Seen in Flycast: A `SamplerManager` hands out samplers keyed by `tsp.full & TSP_Mask`, so samplers are no longer created per texture. Per-poly descriptor sets live in `std::map<std::pair<Texture*, u32>, vk::UniqueDescriptorSet> perPolyDescSetsInFlight`, later keyed directly by texture pointer. ebec2966d 2019-10 “vulkan: create SamplerManager...” · f9603dc11 2021-04 “vk: use texture pointers instead of u64”

### flycast-130 — Read contiguous data in one pass
Lens: io · Source: Flycast
Principle: If two logical streams turn out to be adjacent in memory, such as interlaced fields stored line after line, treat them as one buffer and read it in a single pass instead of two strided passes.
Why it is faster: —
Applies elsewhere: coalescing adjacent byte-range requests; merging adjacent free blocks before a scan.
Watch out: —
Seen in Flycast: When reading the framebuffer, if the display is interlaced and `width == modulus && FB_R_SOF2 == FB_R_SOF1 + width * bpp`, the code sets `modulus = 0; height *= 2` and reads the whole frame at once. 3f1326a3a 2020-12 “rend: read framebuffer as non-interlaced when possible”

### flycast-131 — Measure throughput and shed optional work when behind
Lens: scheduling · Source: Flycast
Principle: Track real speed over a short sliding window. When the core loop falls behind real time, skip optional output such as frames, and don't block on consumers such as audio that would slow it further.
Why it is faster: —
Applies elsewhere: adaptive bitrate in video players; game loops with fixed updates and variable rendering; backpressure that drops low-priority telemetry.
Watch out: —
Seen in Flycast: `spg.cpp` keeps 4-entry ring buffers of `sh4_sched_now64()` and wall time per vblank and sets `SH4FastEnough = cpu_speed >= 85.0`. Auto frame skip uses this flag. The audio stream only waits on push when emulation is at full speed (`do_wait = full_rps < 50 ? false : wait`). a00aad5fa 2020-12 “pvr: auto frame skip to replace current and previous synchronous rendering” · d6f682a32 2019-05 “audiostream only wait if at speed” · 0fd51fb52 2020-06 “...skip less frames in synchronous mode”

### flycast-134 — Tune build flags for the hot targets
Lens: compiler-build · Source: Flycast
Principle: Code generation flags can matter as much as code changes. Build at a high optimization level even for debug builds that are used to play, enable relaxed FP math consistently on every platform, and link a non-PIE executable when absolute addressing helps generated code.
Why it is faster: —
Applies elsewhere: `-O2` for game debug builds; `-ffast-math` in audio DSP code; `-fno-pie` for firmware.
Watch out: —
Seen in Flycast: Android debug builds now use the release flags plus `-O3` and `-fno-stack-protector`. The macOS Xcode project turned on `GCC_FAST_MATH` to match the other platforms. The Linux x64 Makefile added `LDFLAGS := -no-pie`. 493a833f5 2020-08 “android: compile with -O3 in debug...” · 2bb5a5de9 2020-11 “Enable -ffast-math flag in macOS build also...” · fd7cf9a0f 2019-06 “linux-x64: make a non position independent exec”

### flycast-136 — JIT-compile small fixed programs that run continuously
Lens: jit-interpreters · Source: Flycast
Principle: A short program that stays the same for long stretches but runs at a very high rate (a DSP microprogram per audio sample, a co-processor loop) is worth compiling to native code. That removes the per-instruction decode and dispatch overhead. Inside the generated code, address host-side state relative to the instruction pointer instead of loading base pointers.
Why it is faster: —
Applies elsewhere: eBPF JIT; compiled regexes (PCRE JIT); shader-like expression evaluators in data pipelines.
Watch out: —
Seen in Flycast: `core/hw/aica/dsp_x64.cpp` added an x64 JIT for the AICA DSP alongside the existing ARM64 one. The AICA ARM7 sound CPU got an x64 recompiler (`arm7_rec_x64.cpp`) that reads ARM registers with `dword[rip + &arm_Reg[...]]`, with simple LDM/STM handling added to it and to the ARM32 rec. 091c14c78 2020-02 “X64 dsp jit compiler” · 882e14f13 2020-02 “aica: x64 jit for arm”

### flycast-137 — Pool large per-frame objects and recycle idle ones
Lens: allocation · Source: Flycast
Principle: If a big object is created and destroyed every frame, keep the released ones in a small free list and hand them back on the next request. When a lookup misses, reuse an entry nobody has touched for a while before allocating a new one. Allocation, page faults and zeroing then happen once instead of every cycle.
Why it is faster: —
Applies elsewhere: request or response buffer pools in HTTP servers, connection pools, recycling particle or entity objects in a game loop, reusing `ArrayBuffer`s for WebSocket frames.
Watch out: —
Seen in Flycast: `tactx_Alloc` in `core/hw/pvr/ta_ctx.cpp` used to `new` a `TA_context` (with its big vertex and parameter arrays) for every frame, and `tactx_Recycle` deleted it. The pool came back: up to 4 contexts are `Reset()` and kept in `ctx_pool` behind a mutex. Later, `tactx_Find` began stamping `lastFrameUsed`. On a miss it takes over any context unused for more than 60 frames before it allocates a new one: `if (FrameCount - ctx->lastFrameUsed > 60) oldCtx = ctx;` f9e38b11d 2022-05 “pvr: restore TA_context pool” · 37c17e430 2023-02 “pvr: re-use TA contexts if not used for 60 frames”

### flycast-139 — Don't zero buffers you'll overwrite, and swap instead of copy
Lens: memory-layout · Source: Flycast
Principle: Value-initializing a large array (for example `std::array<u8, 4096>` inside a map) writes every byte, even when the next line `memcpy`s over it. A constructor that leaves the storage uninitialized removes that write. Handing a whole container to its consumer with `std::swap` costs O(1), where copying it is O(n).
Why it is faster: —
Applies elsewhere: `Vec::with_capacity` plus `set_len` in Rust, `new Uint8Array(buf)` views instead of copies in JS, and double-buffer handoff between producer and consumer threads.
Watch out: —
Seen in Flycast: `PageMap` was `unordered_map<u32, std::array<u8, PAGE_SIZE>>`. It became a map of a `struct Page { Page() { /* don't initialize data */ } u8 data[PAGE_SIZE]; }`. `getPages()` used to return a reference that the caller copied. It is now `getPages(PageMap& other) { std::swap(pages, other); pages = PageMap(); }`, which hands each snapshot over without copying. cdca559d9 2022-12 “ggpo: optimize memwatch, save/load state. endOfFrame on vblank in”

### flycast-140 — Track dirty pages with memory protection; switch it off when unused
Lens: memory-layout · Source: Flycast
Principle: Write-protect memory and use the first fault on each page to record that it changed. Then unprotect the page so later writes run at full speed. Each snapshot only has to save the pages that changed. The trap costs something on every protected page, so arm it only while a feature actually needs the delta, and only for regions that exist.
Why it is faster: —
Applies elsewhere: incremental checkpointing (CRIU, VM live migration), generational GC write barriers, and copy-on-write snapshots in databases.
Watch out: —
Seen in Flycast: Rollback netplay saves only the RAM, VRAM and ARAM pages touched since the last frame. Each watcher does `memcpy` of the page and then `unprotectMem`. `memwatch::protect()` gained `if (!config::GGPOEnable) return;`, so offline play never takes the faults. The RAM watcher was extended from half of RAM to all of it, and it is now checked first in `writeAccess`. Later, Elan RAM is only mapped, and only watched, in Naomi 2 mode. 458868a63 2021-09 “ggpo: always save delta ram. Disable ff,cheats,savestates when online” · 99bf7f77c 2021-09 “single-threaded mode. refactoring.” · 32e3e7d9c 2022-12 “vmem: no more 4GB vmem mode. only map elan RAM in naomi2 mode”

### flycast-141 — Don't guard memory you already know will be rewritten
Lens: memory-layout · Source: Flycast
Principle: Page-protection invalidation works well when writes are rare. If a region is about to be rewritten by a producer you control, protecting it only produces a storm of faults. Leave such regions unprotected, and drop protection the moment the cached copy dies rather than whenever the cleanup actually runs.
Why it is faster: —
Applies elsewhere: skip cache invalidation hooks for buffers your own code regenerates, and unregister file watchers before deleting the directory they watch.
Watch out: —
Seen in Flycast: Render-to-texture targets are written by the emulated GPU every frame. The GL, Vulkan, DX9 and DX11 back ends called `texture->protectVRam()` after reading one back, so the next frame's writes faulted. All five sites now call `unprotectVRam()`. In `BaseTextureCacheData::Delete()`, `unprotectVRam()` moved ahead of the early return for a custom load still in progress, so a texture marked for deletion stops trapping writes immediately. 6a5db32d5 2022-06 “don't protect RTT textures vram” · cf9ccea70 2023-02 “texcache: unprotect vram immediately even if actual Delete is delayed”

### flycast-143 — Memoize per-object GPU bindings within a frame
Lens: gpu-rendering · Source: Flycast
Principle: If the same object is drawn several times in one frame (by multiple passes, for example), build its binding state the first time and reuse it after that. Allocating and writing descriptors is CPU and driver work that scales with the number of draws.
Why it is faster: —
Applies elsewhere: bind-group caches in WebGPU, memoized React element props, and caching prepared statements per query shape.
Watch out: —
Seen in Flycast: The Vulkan OIT renderer called `perPolyAlloc.alloc()` and `updateDescriptorSets` on every `bindPerPolyDescriptorSets`, and the OIT path draws each polygon in several passes. An `unordered_map<const void *, vk::DescriptorSet> perPolyDescSets` keyed by `&poly` now returns the set on later binds. `nextFrame()` clears the map. ad0859019 2022-03 “vulkan: cache per-poly desc sets in map”

### flycast-145 — Primitive restart instead of degenerate triangles
Lens: gpu-rendering · Source: Flycast
Principle: You can join triangle strips into one draw by repeating vertices to create zero-area triangles. The GPU still has to set up and reject each of those triangles. A primitive-restart index (`0xFFFFFFFF`) ends the strip outright, and the index buffer gets shorter.
Why it is faster: —
Applies elsewhere: terrain and mesh renderers in WebGL2 and Vulkan, line-strip batching for charts, and any tool that converts geometry into strips.
Watch out: —
Seen in Flycast: `makePrimRestartIndex()` in the new `core/hw/pvr/ta_util.cpp` writes `*ctx->idx.Append() = ~0;` between merged polygons and around skipped infinite vertices. `makeIndex()` remains as the degenerate-triangle fallback for GLES, which lacks `GL_PRIMITIVE_RESTART`. The commit says it uses primitive restart “to avoid too many degenerate triangles.” f9feaa313 2022-12 “pvr: sort triangles during parse. Use primitive restart”

### flycast-146 — Do shared work once in the producer, not in every consumer
Lens: algorithms · Source: Flycast
Principle: If several back ends each sort or reorganize the same input, move that step into the common stage that builds the data and store the result there. Every consumer then draws from the prepared data, and the work drops out of the time-critical draw loop.
Why it is faster: —
Applies elsewhere: sort or index data once at ingestion rather than in each report query, and precompute layout in a build step rather than in each client.
Watch out: —
Seen in Flycast: Each renderer (GL, GL4, DX9, DX11, Vulkan) used to call `SortPParams` or its own `SortTriangles` inside its draw function. Per-strip sorting moved into `ta_parse_vdrc`/`ta_parse_naomi2`. Then per-triangle sorting moved too: `sortTriangles(rend_context&, RenderPass&, ...)` fills `ctx.sortedTriangles` using the same index buffer as everything else, and the renderers only consume the result. 4c7df6c7f 2022-12 “pvr: don't merge polys in per-triangle. merge polys in per-strip” · f9feaa313 2022-12 “pvr: sort triangles during parse. Use primitive restart”

### flycast-148 — Move per-vertex transforms from CPU to GPU
Lens: gpu-rendering · Source: Flycast
Principle: Multiplying every vertex by model and projection matrices is exactly the work a vertex shader does in parallel. Send untransformed vertices plus one matrix reference per object and let the GPU apply the transform. CPU cost drops from O(vertices) to O(objects).
Why it is faster: —
Applies elsewhere: instanced rendering with per-instance matrices, CSS `transform` instead of recomputing element positions in JS, and GPU skinning instead of CPU skinning.
Watch out: —
Seen in Flycast: The Naomi 2 Elan T&L emulation in `core/hw/pvr/elan.cpp` computed `v = projectionMatrix * curMatrix * v;` for each vertex on the CPU. The rework stores the matrices once with `taMVMatrix = ta_add_matrix(glm::value_ptr(curMatrix))`, has each `PolyParam` reference them, and adds new `naomi2.cpp` shader paths for GL and GL4 that transform and light on the GPU. cb99e7d92 2022-01 “naomi2: do T&L on the gpu. bypass TA format. env mapping.”

### flycast-149 — Hoist per-state decoding out of per-item loops
Lens: algorithms · Source: Flycast
Principle: If a value derived from state (an unpacked color, a decoded flag) changes only when the state changes, compute it at that moment and keep the result. Inner loops then do a cheap add instead of repeating the decoding and branching for every item.
Why it is faster: —
Applies elsewhere: parse a CSS color once, not on every paint. Resolve a locale or format object once per request. Compute a shader uniform per material, not per vertex.
Watch out: —
Seen in Flycast: `addModelColors()` runs for every Naomi 2 vertex. It tested `curGmp->paramSelect.d0/s0/d1/s1` and called `unpackColor()` each time. `State::updateGMP()` now fills static `gmpDiffuseColor0`…`gmpSpecularColor1` vectors when the model parameters change, and the per-vertex code reduces to `baseCol0 += gmpDiffuseColor0;` and so on. The commit describes it as “Cache model colors to avoid unpacking for each vtx”. 0362fd107 2022-02 “naomi2: disable elan when net rollbacking, optimizations. ggpo: RTT fix”

### flycast-153 — Ask the OS for the fast GPU explicitly
Lens: platform · Source: Flycast
Principle: On hybrid-graphics laptops, the default adapter is often the integrated GPU. Enumerate adapters by performance preference and create the device on the discrete one, falling back to the default if that isn't available.
Why it is faster: —
Applies elsewhere: `powerPreference: "high-performance"` in WebGL and WebGPU, the `NvOptimusEnablement` export, and Metal device selection.
Watch out: —
Seen in Flycast: `DX11Context::init` passed `nullptr` (the default adapter) to `D3D11CreateDevice`. It now queries `IDXGIFactory6::EnumAdapterByGpuPreference(0, DXGI_GPU_PREFERENCE_HIGH_PERFORMANCE, ...)` and passes that adapter with `D3D_DRIVER_TYPE_UNKNOWN`. 1377e61d0 2023-09 “DX11: Use high performance GPU on Windows 10 (1803 or later)”

### flycast-154 — Don't allocate surface buffers you never use
Lens: gpu-rendering · Source: Flycast
Principle: If all real rendering goes to offscreen framebuffers, the window's swap-chain surface needs no depth or stencil planes. Requesting them anyway costs memory, and on some drivers the bandwidth to clear and resolve them.
Why it is faster: —
Applies elsewhere: `{depth: false, stencil: false, antialias: false}` when creating a WebGL context for 2D or offscreen work.
Watch out: —
Seen in Flycast: The EGL, WGL and GLX context setups (`core/wsi/egl.cpp`, `wgl.cpp`, `xgl.cpp`) requested 24-bit depth and 8-bit stencil for the window surface. The commit removes them (“No need for depth/stencil surface”). 1ab4eb00c 2022-12 “tex cache: minor key mask fix. egl: no need for depth/stencil surface”

### flycast-156 — Retire GPU resources at the frame fence
Lens: gpu-rendering · Source: Flycast
Principle: A resource the GPU may still be reading can't be freed right away, and waiting for the device to go idle stalls the pipeline. Move ownership into a per-frame list that is cleared once that frame's fence signals, and keep the CPU running.
Why it is faster: —
Applies elsewhere: epoch-based reclamation and RCU in lock-free structures, and deferred `destroy()` of WebGPU buffers after `onSubmittedWorkDone`.
Watch out: —
Seen in Flycast: Vulkan textures used to be moved into four parallel trash vectors in `TextureCache`. Now there is a `Deletable`/`FlightManager` interface: `Texture::deferDeleteResource()` swaps the image, view, staging buffer and allocation into a `ResourceDeleter` and calls `manager->addToFlight(...)`. `CommandPool::BeginFrame` waits on that slot's fence and runs `inFlightObjects[index].clear()`. 939e2bc68 2023-12 “vk: better deferred deletion. fix validation layer issues”

### flycast-157 — Use fused multiply-add when the CPU has it
Lens: simd · Source: Flycast
Principle: One FMA instruction replaces a multiply and an add: half the instructions, one fewer register move, and one rounding step instead of two. Check the CPU at runtime (or JIT time) and keep the separate multiply and add as a fallback. Also keep the fallback when results must be bit-identical across machines.
Why it is faster: —
Applies elsewhere: `std::fma` or `-mfma` in DSP and ML kernels, `Math.fround`-sensitive code that must stay deterministic, and BLAS dispatch by CPU feature.
Watch out: —
Seen in Flycast: The x64 JIT (`core/rec-x64/rec_x64.cpp`) gained its own `shop_fmac` case. It emits `vfmadd231ss(rd, rs2, rs3)` when `cpu.has(Cpu::tFMA) && !config::GGPOEnable`, and `movss/mulss/addss` otherwise. Netplay keeps the non-fused path so that every peer computes identical results. 879372cb7 2023-09 “rec-x64: use FMA when available”

### flycast-158 — Compile a fixed microprogram into straight-line native code
Lens: jit-interpreters · Source: Flycast
Principle: When a small program runs unchanged millions of times (for example 128 steps for every audio sample), decode it once and emit host code with every field already resolved and every unused operation left out. Runtime decoding and switch dispatch disappear completely.
Why it is faster: —
Applies elsewhere: compiling BPF filters and regexes to machine code, shader compilation, and generating specialized SQL expression evaluators.
Watch out: —
Seen in Flycast: The new `core/hw/aica/dsp_arm32.cpp` adds a `DSPAssembler` built on vixl. For each of the 128 steps it calls `DecodeInst(mpro, &op)` at compile time and emits only the loads, shifts and adds that this step's `op.XSEL`, `op.IRA`, `op.BSEL` and so on select. Accumulators stay in fixed registers (`ACC`=r5, `B`=r10). This brought the DSP JIT to 32-bit ARM, alongside the existing back ends. e5ca63db2 2021-08 “aica: arm32 dsp dynarec. Correct dsp level when dsp disabled”

### flycast-159 — Outline rare checks from generated code and test before updating
Lens: jit-interpreters · Source: Flycast
Principle: Checks that every compiled block repeats inline make the code cache bigger, and a bigger cache means more i-cache misses. Move them into one shared helper and emit a call. For a countdown, load and test the counter first, take the slow path only when it has run out, and then subtract. The fast path becomes a load, a compare, a subtract and a store.
Why it is faster: —
Applies elsewhere: shared slow-path stubs in V8 and LuaJIT, and moving cold error handling into `[[unlikely]]` non-inlined functions.
Watch out: —
Seen in Flycast: In the ARM and ARM64 JITs, each block's MMU prologue (vaddr compare plus the FPU-disabled test) was inlined. It is now `call(checkBlockFpu)` or `call(checkBlockNoFpu)`. The cycle counter no longer has a dedicated `r9`, which frees r9 for register allocation (`alloc_regs = {5,6,7,9,10,11}`). Each block runs `Ldr r1,[cycle_counter]; Cmp r1,0; B pl, cyclesRemaining; ... call(intc_sched)` and then subtracts its own cycles. 0b1f69bfd 2023-03 “dynarec: more accurate cycle counting. limit code duplication”

### flycast-160 — Reserve patch space at link sites
Lens: jit-interpreters · Source: Flycast
Principle: Generated code that may later be rewritten in place, for example a direct branch that might become a longer call sequence, should be padded with NOPs to the size of the longest form. Relinking then overwrites bytes in place, with no need to recompile the block or jump through a trampoline.
Why it is faster: —
Applies elsewhere: hot-patchable function prologues (`-fpatchable-function-entry`), and inline caches in JS engines that reserve room for the patched sequence.
Watch out: —
Seen in Flycast: At block exits in `core/rec-ARM64/rec_arm64.cpp`, `GenBranch(...)` and `GenCall(linkBlock*Stub)` are now followed by `Nop(); Nop();`, and three NOPs follow the `arm64_no_update` branch. With the full MMU on, linked blocks can then be re-patched in place. c04c696ba 2023-01 “arm64 dynarec: pad link section with nop to allow full mmu relink”

### flycast-164 — Make fixed values visible as compile-time constants
Lens: compiler-build · Source: Flycast
Principle: An option that can no longer change, or a pointer to a fixed address, still costs a load and a test on every use if it is stored as a mutable global. Declaring it `constexpr`, or `T * const` with an initializer, lets the compiler fold the value and delete dead branches.
Why it is faster: —
Applies elsewhere: `if constexpr` feature flags, `process.env.NODE_ENV` replaced at bundle time, and Rust `const` config with dead-code elimination.
Watch out: —
Seen in Flycast: After the integer-division safe mode was removed, `extern Option<bool> DynarecSafeMode` became `constexpr bool DynarecSafeMode = false;`, so the `if (config::DynarecSafeMode) return false;` guards in the decoder's `MatchDiv32u/s` compile away. AICA register views that `libAICA_Init` used to assign at runtime (`CommonData`, `DSPData`, `MCIEB`…) became statically initialized `* const` pointers into `aica_reg`. 0d2e6f5e9 2021-09 “remove safe mode/div matching option” · 41723a831 2023-11 “use static init when possible. use unnamed bit-fields. refactor vmuBeep”

### flycast-165 — Specialize on access width with templates, not a runtime size argument
Lens: compiler-build · Source: Flycast
Principle: A handler that takes `u32 sz` and branches on 1/2/4 pays that switch on every call, and it can't return a correctly typed value. Templating on the value type makes the size a compile-time fact. Each instantiation is branch-free, and calls with a size known at the call site resolve directly.
Why it is faster: —
Applies elsewhere: templated serializers and binary readers (`read<T>()`), DataView-style typed accessors, and generic numeric kernels in Rust.
Watch out: —
Seen in Flycast: `ReadMem_aica_reg(u32 addr, u32 sz)`, `WriteAicaReg<u32 sz>(reg, u32 data)`, the RTC accessors and others became `template<typename T> T ReadMem_aica_reg(u32 addr)` with explicit instantiations. Holly `sb_ReadMem`/`sb_WriteMem` dropped their `sz` parameter and the `data8/data16/data32` switch, since all accesses are 32-bit. 039c66624 2022-04 “more types.h clean up. better aica handler templates” · f5fa1f621 2022-12 “holly, sh4: reg access handlers don't need size. SB_FFST constant 0”

### flycast-166 — Dispatch through per-entry function pointers instead of flag tests
Lens: lookup-tables · Source: Flycast
Principle: Give every entry in a table (here, hardware registers) its own read and write function pointers, defaulting to a plain storage accessor or an “invalid” stub. Each access is then one indirect call with no flag checks. Template parameters bake the address, mask and type into each handler.
Why it is faster: —
Applies elsewhere: syscall tables, opcode dispatch tables in interpreters, and route tables mapping paths to handler closures.
Watch out: —
Seen in Flycast: Holly's `sb_ReadMem` tested `sb_regs[offset].flags & REG_RF`, then `REG_WO`, before choosing between stored data and a callback. The new `HwRegister` in `core/hw/hwreg.h` stores `read8/16/32` and `write8/16/32` pointers. `MMRegister<Module, Mask, Base>::setReadWrite<Addr, T, Mask, OrMask>()` installs `readModule<Addr,T>`/`writeModule<...>` instantiations, and plain data lives separately in `u32 sb_regs[0x540]`. 099a758c9 2023-02 “hw: new hardware register implementation”

### flycast-168 — Move loops to the far side of a chatty protocol
Lens: io · Source: Flycast
Principle: When a client drives a server one tiny command at a time (step, check, step), the round trips dominate. Support a command that carries the loop condition, such as “step while PC is in [from, to)”, and run the loop on the server.
Why it is faster: —
Applies elsewhere: Redis pipelines and Lua scripts, stored procedures instead of row-at-a-time queries, and server-side cursors.
Watch out: —
Seen in Flycast: The GDB stub (`core/debug/gdb_server.cpp`) used to answer “vCont not supported”, so GDB fell back to single steps. It now advertises `vCont;c;C;s;S;t;r` and parses `r%x,%x` into `stepRange(from, to)`, which calls `emu.stepRange` to step locally until the PC leaves the range. The commit calls this “ranged stepping (faster)”. 13f3c8281 2022-12 “Fixed single stepping, added support for vCont packets and ranged stepping (faster), initial work to support other breakpoint types”

### flycast-169 — Preallocate a bounded worst-case buffer instead of growing it
Lens: allocation · Source: Flycast
Principle: Size a streaming buffer once, from the real worst case plus some slack, and drop input when it is full. This replaces `realloc` growth on the hot path and caps the memory and latency a stall can pile up.
Why it is faster: —
Applies elsewhere: fixed-size ring buffers for audio and telemetry, bounded channels in Go and Rust, and backpressure in Node streams.
Watch out: —
Seen in Flycast: `retro_audio_upload` in `shell/libretro/audiostream.cpp` grew `audio_out_buffer` with `realloc` whenever `audio_buffer_idx` exceeded it, up to 1 MiB. Both buffers are now allocated once at `(44100 / 25) * 2 * 10` samples (ten worst-case frames). A `drop_samples` flag discards input until the next upload, so the realloc path is gone. 6c6b447ea 2022-05 “(libretro) Limit maximum number of audio samples that can be uploaded per call of retro_run()”

### flycast-170 — Let fixed-function hardware interpolate, via shader variants
Lens: gpu-rendering · Source: Flycast
Principle: Interpolating attributes by hand in the shader, with flat w and `noperspective` plus manual multiplies, or clipping on the CPU, competes with hardware built for exactly that. Some GPUs run this badly. Add a compile-time shader variant that puts the real depth in `pos.w` and leaves interpolation and clipping to the hardware.
Why it is faster: —
Applies elsewhere: `#define`-driven uber-shader permutations, and keeping varyings perspective-correct instead of hand-correcting them in fragment code.
Watch out: —
Seen in Flycast: The new `rend.NativeDepthInterpolation` option adds a `DIV_POS_Z` shader constant to the GL, GL4, DX9, DX11 and Vulkan shader generators. When it is set, the vertex shader does `vpos /= vpos.z; vpos.z = vpos.w;` and drops `NOPERSPECTIVE`. The commit removes the CPU-side Sutherland–Hodgman modifier-volume clipping from `dx11_renderer.cpp` (“to help AMD gpus”). 77a11d2a2 2022-05 “native depth interpolation option to help AMD gpus”

### flycast-171 — Gate data-parallel code on the compiler's own feature macro
Lens: concurrency · Source: Flycast
Principle: Enable OpenMP whenever the toolchain supports it, and guard the parallel code with the standard `_OPENMP` macro instead of a hand-maintained “no OpenMP” define. Platforms that support it, macOS included here, then get the parallel path without per-platform exclusions.
Why it is faster: —
Applies elsewhere: `#ifdef __ARM_NEON` or `__AVX2__` SIMD paths, and Rust `cfg(target_feature)`.
Watch out: —
Seen in Flycast: CMake used to skip OpenMP on Apple and define `TARGET_NO_OPENMP`. Now `find_package(OpenMP)` runs whenever `USE_OPENMP` is set, and `TexCache.cpp` and the libretro core test `#ifdef _OPENMP`. The xBRZ texture upscaler (`UpscalexBRZ`) then splits its work across `omp_get_num_procs() - 1` threads wherever OpenMP is present. 12929c457 2022-05 “openmp: use _OPENMP preprocessor, search openmp pkg unconditionally if enabled”

### flycast-173 — Compile translation units in parallel
Lens: compiler-build · Source: Flycast
Principle: MSVC compiles a project's source files one at a time unless you pass `/MP`. That flag fans the compiles out across cores, which cuts full rebuild times on large C++ code bases.
Why it is faster: —
Applies elsewhere: `make -j` and Ninja, Gradle parallel builds, and `tsc --build` with project references.
Watch out: —
Seen in Flycast: In the `WIN32` branch of `CMakeLists.txt`: `if(MSVC) set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /MP") endif()`. 3f30dd01a 2022-12 “Support for multi-processor compilation with MSVC (/MP option) (#830)”

### flycast-175 — Do the indirection lookup in the shader, filter by hand
Lens: gpu-rendering · Source: Flycast
Principle: Indexed-colour images have to be expanded to full colour before the hardware can filter them. Expanding on the CPU costs a full conversion and re-upload each time the palette changes. Upload the raw indices once, look up the palette in the fragment shader, and do the four-tap bilinear blend in shader code. A palette change then only updates a small palette texture.
Why it is faster: —
Applies elsewhere: colour-mapped scientific visualisation (upload scalar data plus a 1-D LUT), sprite palette swaps in 2D engines, GPU decoding of indexed PNG/GIF.
Watch out: —
Seen in Flycast: `IsGpuHandledPaletted` used to accept only point-sampled palette textures. It now accepts `tsp.FilterMode <= 1`. Every backend (GL, GL4, DX9, DX11, Vulkan, OIT) gained a `pp_Palette == 2` shader variant that fetches four neighbouring indices with texel offsets, maps each through `getPaletteEntry`, and blends them by the fractional coordinate: `float4 c01 = getPaletteEntry(tex.Sample(texSampler, sampleUV, int2(0, 1)).a);`. Hardware filtering is switched off for those draws. e714e9451 2024-01 “rend: do bilinear filtering of palette textures on the GPU”

### flycast-176 — Present each frame once, and draw overlays only when they are visible
Lens: gpu-rendering · Source: Flycast
Principle: Full-screen copies and overlay passes cost fill rate and bandwidth even when they add nothing to the image. Check that a frame is blitted to the back buffer exactly once, and guard each optional overlay with a cheap visibility test before issuing its draws.
Why it is faster: —
Applies elsewhere: compositors that skip unchanged layers, UI toolkits that drop invisible widgets from the draw list, game HUDs gated on game state.
Watch out: —
Seen in Flycast: The ImGui OpenGL backend had its own `ImGui_ImplOpenGL3_DrawBackground()` that cleared the screen and called `RenderLastFrame()` again, so each rendered frame was blitted twice. That function was removed and VMU and crosshair drawing were unified. Crosshair loops in the GL, DX9 and DX11 overlays are now wrapped in `if (crosshairsNeeded())`. 4f834610b 2024-05 “gl: refactor vmu and xhair drawing. Blit rendered frame only once” · ec82c7b9e 2024-05 “gl: only display crosshairs when needed” · 552e5b7d7 2024-01 “dx9,dx11: only draw lightgun crosshair when necessary”

### flycast-177 — Port the virtual-memory fast path to a constrained OS
Lens: memory-layout · Source: Flycast
Principle: Mapping guest memory into one reserved host range lets generated code access it with a single base+offset load. When a platform can't map every alias, map only the writable mirror. Then route the protect/unprotect and pointer-to-offset helpers around the missing aliases, rather than giving up the fast path entirely.
Why it is faster: —
Applies elsewhere: database buffer pools on `mmap`, ring buffers that map the same pages twice, sandboxes that reserve a guard-paged heap.
Watch out: —
Seen in Flycast: The Nintendo Switch build got a real fastmem implementation in `libnx_vmem.cpp` (reservations plus per-page `svcSetMemoryPermission`). Because the Switch can only map the first writable mirror, `protectVram`, `getVramOffset`, `bm_Reset` and `bm_getRamOffset` take their non-vmem branch under `#ifndef __SWITCH__`. A comment in `initMappings` now pins the buffers to “the first r/w mirror”. 70a69cf48 2024-01 “switch: fastmem/vmem implementation”

### flycast-178 — Stop the per-tick pipeline as soon as the unit goes idle
Lens: branching-fast-paths · Source: Flycast
Principle: If an early stage of a per-sample pipeline can switch the unit off, skip the remaining stages for that tick. The work saved grows with the number of idle units, and in a sound chip most voices are idle most of the time.
Why it is faster: —
Applies elsewhere: particle systems that skip dead particles mid-update, physics engines that put bodies to sleep, short-circuiting middleware chains.
Watch out: —
Seen in Flycast: In the AICA channel step, `StepAEG` can disable the channel. The filter envelope, stream and LFO steps now run only if the channel is still enabled: `StepAEG(this); if (enabled) { StepFEG(this); StepStream(this); lfo.Step(this); }`. This also keeps the reset sample address from being changed. 5393235c4 2024-09 “aica: skip remaining step functions when channel is disabled”

### flycast-179 — Hash the payload and skip uploads that would change nothing
Lens: gpu-rendering · Source: Flycast
Principle: Re-uploading identical data every frame costs bus bandwidth and driver synchronisation. Keep a cheap hash of what was last sent and return early when the new data hashes the same.
Why it is faster: —
Applies elsewhere: ETag/If-None-Match in HTTP, React memoisation, build systems that skip a step when the content hash of its inputs is unchanged.
Watch out: —
Seen in Flycast: The Vulkan `QuadBuffer::Update` hashes its four vertices with a boost-style combine and skips `buffer->upload` if the hash matches: `if (quadVerticesHash == lastUploadHash) return;`. The same idea was applied to texture updates. If a texture's VRAM hash is unchanged after a write fault, the conversion is skipped, VRAM is re-protected, and the update returns. a8bc1c12f 2024-10 “vk: Optimize redundant `QuadBuffer` uploads” · 5efdf58e8 2025-01 “custom texture: skip updating a texture if its hash hasn't changed”

### flycast-181 — Let the vertex fetch unit normalise integers
Lens: gpu-rendering · Source: Flycast
Principle: Declaring byte attributes as normalised (UNORM) makes the input assembler convert them to the 0–1 float range for free. That removes a convert and a divide from every vertex shader invocation.
Why it is faster: —
Applies elsewhere: packed normals as `SNORM`, 16-bit UV coordinates as `UNORM16`, and vertex compression in mobile games in general.
Watch out: —
Seen in Flycast: Vertex colours changed from `eR8G8B8A8Uint` to `eR8G8B8A8Unorm` in every Vulkan pipeline. In the shaders, `vtx_base = vec4(in_base) / 255.0;` became `vtx_base = in_base;`. 8f74f7883 2024-10 “vk: Use `VK_FORMAT_R8G8B8A8_UNORM` for vertex colors”

### flycast-182 — Don't fetch attributes the shader variant never reads
Lens: gpu-rendering · Source: Flycast
Principle: Every declared vertex attribute costs input-assembler bandwidth even if the shader ignores it. Build the attribute list per pipeline variant and cut off the tail the variant doesn't use.
Why it is faster: —
Applies elsewhere: splitting position-only streams for depth pre-passes, columnar storage that reads only the columns a query needs.
Watch out: —
Seen in Flycast: `GetMainVertexInputStateCreateInfo(full, naomi2)` now passes `std::size(vertexInputAttributeDescriptions) - 1` attributes for non-Naomi 2 pipelines, which drops the trailing normal attribute. The full set is kept for Naomi 2. 595842f60 2024-10 “vk: Remove naomi2 vertex attributes when not needed”

### flycast-183 — Follow vendor best-practice hints for sampler state
Lens: gpu-rendering · Source: Flycast
Principle: Some GPUs run a slower sampler path when the three wrap modes differ or when LOD clamping is enabled, even for 2-D textures that never use them. Make the unused W mode match V, and limit mip levels through the image view instead of the sampler.
Why it is faster: —
Applies elsewhere: keeping bind groups uniform for WebGPU on tilers, keeping descriptor layouts identical so state can be shared.
Watch out: —
Seen in Flycast: Vulkan samplers now set `const vk::SamplerAddressMode wRepeat = vRepeat;` instead of a fixed `eClampToEdge`, and pass `vk::LodClampNone` as max LOD. Both changes answer ARM's `BestPractices-Arm-vkCreateSampler-*` validation messages. b94acb70a 2024-10 “vk: Use matching u/v/w sampler address-mode when possible” · b20db6a8b 2024-10 “vk: Disable LOD clamping in samplers (#1674)”

### flycast-184 — Don't throw away compiled pipelines on a resize
Lens: gpu-rendering · Source: Flycast
Principle: Compiled GPU pipelines are expensive to create and don't depend on framebuffer size. Create the pipeline manager once and keep it when attachments are recreated, so a window resize doesn't cause a burst of shader compiles.
Why it is faster: —
Applies elsewhere: keeping prepared SQL statements across connection resets, keeping JIT caches across configuration reloads.
Watch out: —
Seen in Flycast: In the Vulkan OIT drawer, `screenPipelineManager->Init(...)` moved inside the `if (!screenPipelineManager)` block. Resizing the framebuffer no longer resets the pipelines. 757732d0f 2024-08 “vk oit: load correct attachment. Only reset pipelines when needed”

### flycast-185 — Stream in small chunks and decrypt only right before use
Lens: io · Source: Flycast
Principle: Filling a large buffer in one go creates a long stall on the emulated timeline and wastes work if the consumer only reads part of it. Use small buffers (1–2 KB, or 16 sectors), refill them on demand, and run the expensive transform (decryption) only just before the data is transferred.
Why it is faster: —
Applies elsewhere: chunked HTTP responses, video players that decode just ahead of the playhead, TLS record-sized writes.
Watch out: —
Seen in Flycast: The M1 and M4 cartridge buffers dropped from `u8 buffer[32768]` to 1 KB and 2 KB, and `enc_fill()` now runs only when a DMA consumes the buffer. This fixed frame drops in vf4evoct. GD-ROM DMA buffers shrank to 16 sectors, and HLE BIOS reads are scheduled in chunks. Cartridge DMA now copies 1 KB per scheduler event: `std::min<int>(SB_GDLEN - SB_GDLEND, 1024) * dmaXferDelay`. 96aac6613 2024-11 “naomi: improve performance of M1 and M4 carts dma” · c860807fe 2024-11 “gdrom: reduce dma buffer size. reios: schedule dma reads by chunks” · d7c168f40 2024-10 “naomi: copy cart dma data in chunks”

### flycast-186 — Fault in content by segment, tracked with a bitmap
Lens: laziness · Source: Flycast
Principle: Loading and decrypting a whole image up front makes startup proportional to image size. Split it into fixed segments, keep one "loaded" bit per segment, and load and decrypt a segment the first time a read touches it. Startup becomes nearly instant, and data that is never read is never processed.
Why it is faster: —
Applies elsewhere: demand paging, lazily loaded container image layers, virtual file systems that fetch blocks on read.
Watch out: —
Seen in Flycast: `GDCartridge` no longer reads and DES-decrypts the whole DIMM image at load. It keeps `std::vector<bool> loadedSegments` with `SEGMENT_SIZE = 16_KB`, and every DMA or pointer request calls `loadSegments(addr, size)`, which reads and decrypts only the missing segments. This shortened load times and avoided Android ANRs. fbd52a72d 2024-11 “naomi: load and decrypt gd-rom on demand”

### flycast-189 — Keep block-entry checks out of the common mode
Lens: jit-interpreters · Source: Flycast
Principle: Every check in a compiled block's prologue runs on every block entry. Emit the expensive validations only in the mode that needs them, and order them so the cheap common check returns early.
Why it is faster: —
Applies elsewhere: JIT guards that are emitted only when a deoptimisation dependency exists, debug-only bounds checks.
Watch out: —
Seen in Flycast: On the ARM and ARM64 recompilers, the FPU-enabled check now runs after the PC and block checks, and it is emitted only when `mmu_enabled() && block->has_fpu_op`. The shared stub was split into `checkBlockAddr` and `checkBlockFpu`, and both return early: `Tst(r2, 1 << 15); Bx(eq, lr);`. `rdv_LinkBlock` also returns immediately when the MMU is on. 6f58d204b 2025-10 “arm, arm64 dynarecs: check FPU enabled after PC and block checks”

### flycast-190 — Turn on the expensive mode only when the guest actually needs it
Lens: laziness · Source: Flycast
Principle: Full address translation slows every memory access. Start in the cheap mode and watch for the first event that requires the full mode, here a TLB mapping outside the one pattern the cheap mode handles. Switch only then, and filter out known-bogus triggers so they don't cause a needless switch.
Why it is faster: —
Applies elsewhere: JITs that start untyped and specialise on observed types, databases that create an index after repeated scans, copy-on-write promotion.
Watch out: —
Seen in Flycast: In `fastmmu.cpp`, a store-queue mapping still updates `sq_remap`. Any other mapping created while `CCN_MMUCR.AT == 1` sets `mmuOn = true; mmu_set_state();` (“Enabling on-demand Full MMU support”). A follow-up ignores VPNs `0x30040` and `0x30000` and invalid entries, which many arcade and Visual Concepts games create at boot. fd72eebbe 2026-01 “mmu: enable full MMU when non-SQ mapping is created” · 7ce50e522 2026-01 “mmu: stricter test before enabling on-demand MMU”

### flycast-191 — Test feature macros by value, not by existence
Lens: platform · Source: Flycast
Principle: SDK headers often define platform macros on every target, set to 0 or 1. Testing `defined(X)` then picks the wrong branch everywhere. Here that branch silently disabled the JIT on real devices. Test the value with `#if X`.
Why it is faster: —
Applies elsewhere: `NDEBUG`, `_POSIX_C_SOURCE`, `__has_include` checks, and feature flags in build systems generally.
Watch out: —
Seen in Flycast: In `core/build.h`, `#if defined(TARGET_OS_SIMULATOR)` was true on every Apple target, because Apple defines the macro as 0 or 1. That defined `TARGET_NO_REC` and caused a performance regression on macOS and iOS. The fix is `#if TARGET_OS_SIMULATOR`. 5add29cbe 2025-01 “fix perf regression on macOS and iOS. Add Silent Scope (JP) identifier.”

### flycast-193 — Allocation-free ring buffer for diagnostics
Lens: allocation · Source: Flycast
Principle: A logger that builds `std::string`s and pushes them onto a deque allocates on every message. That is slow, and it can deadlock if the crash being logged happened inside malloc. Format into stack buffers and keep history in a fixed array of char slots with a wrapping index.
Why it is faster: —
Applies elsewhere: flight recorders in kernels (ftrace ring buffers), crash reporters that preallocate, real-time audio loggers.
Watch out: —
Seen in Flycast: `InMemoryListener` replaced `std::deque<std::string>` with `std::array<char[MAX_MSGLEN], MaxLines>` and `linesIndex = (linesIndex + 1) % MaxLines`. `LogManager::Log` now `snprintf`s the timestamp header and `vsnprintf`s the body into one stack buffer. The network listener formats into a `char temp[MAX_MSGLEN]`. 97a0396af 2026-03 “don't allocate memory when logging”

### flycast-194 — Point existing file code at a memory stream
Lens: io · Source: Flycast
Principle: When a serialiser already writes to a `FILE*`, the fastest way to keep a snapshot in RAM is to give it a memory-backed `FILE*`. You get no disk I/O and no second code path.
Why it is faster: —
Applies elsewhere: `open_memstream` for building logs, `io.BytesIO` in Python tests, tmpfs-backed scratch files.
Watch out: —
Seen in Flycast: A special slot index `-2` makes `dc_savestate`/`dc_loadstate` call `fmemopen(quicksave_buf, QUICKSAVE_DEFAULT_SIZE, "wb")` on a static 32 MB buffer instead of opening a file. New hotkeys `EMU_BTN_SAVESTATE_RAM` and `EMU_BTN_LOADSTATE_RAM` trigger it. 3eef0196c 2026-02 “added in-ram savestates (#2252)”

### flycast-196 — Load previews asynchronously and cap their resolution
Lens: io · Source: Flycast
Principle: Thumbnails only need to be thumbnail-sized. Capture them at a bounded width so encoding and decoding stay cheap, and decode them off the UI thread so menus never wait on disk.
Why it is faster: —
Applies elsewhere: image galleries with progressive thumbnails, file managers generating previews in the background.
Watch out: —
Seen in Flycast: `GetLastFrame` in every renderer now takes a maximum width or height, and savestate screenshots are limited to 640 pixels wide. The pause menu loads the savestate thumbnail in an async task. 35acb7e62 2024-05 “ui: load savestate thumbnails asynchronously. limit thumbnail size”

### flycast-197 — Unprotect only the bytes you patch
Lens: jit-interpreters · Source: Flycast
Principle: Toggling write permission on the whole free code region for one patch costs a large page-table change and TLB flushes. Flip permissions on just the few bytes around the patch site.
Why it is faster: —
Applies elsewhere: hot-patching frameworks (Detours, livepatch) that touch only the target pages, W^X JITs that toggle per page.
Watch out: —
Seen in Flycast: When the x64 dynarec rewrites a faulting fast-memory call, it now computes `u8 *retAddr = *(u8**)context.rsp - 5;`, checks that the address is inside the code buffer, and calls `jit_set_exec(retAddr, 16, false/true)`. Previously it toggled the entire free space of `codeBuffer`. The same change enabled vmem on UWP. c7d030c2e 2025-01 “uwp: enable vmem. fix dynarec crash. windows: dll loading utility”

### flycast-198 — Bound the producer–consumer queue and drop stale frames
Lens: concurrency · Source: Flycast
Principle: An unbounded queue between the emulation and render threads lets the producer run arbitrarily far ahead, which builds latency and drift. Merge duplicate commands and cap the queue depth by discarding the oldest frame-present request, which is safe to drop.
Why it is faster: —
Applies elsewhere: video pipelines that drop late frames, input queues that coalesce mouse moves, mailbox-style channels.
Watch out: —
Seen in Flycast: `PvrMessageQueue::enqueue` already deduplicated Render, RenderFramebuffer and Stop messages. It now also caps the queue at `MAX_QUEUE_DEPTH = 4` by erasing the oldest `Present`. `rend_term_renderer` drains the queue before teardown. f6f5927e1 2026-05 “pvr: bound render-message queue and drain it on renderer term to fix libretro audio drift (#2322)”

### flycast-200 — Schedule polling only when the device exists, and poll less often
Lens: laziness · Source: Flycast
Principle: A periodic poll event on the emulated timeline costs time even when nothing listens. Register it only when the device is configured, and use the slowest interval the protocol tolerates.
Why it is faster: —
Applies elsewhere: lazily started watchers, adaptive polling intervals in mobile sync, interrupt coalescing in NIC drivers.
Watch out: —
Seen in Flycast: The broadband adapter's receive poll went from `SH4_MAIN_CLOCK / 10000` (100 µs) to `1_sh4ms`. `bba_Init()` now runs only if `config::EmulateBBA` is set on a console or Atomiswave, so other games schedule no poll event. 5543a95c4 2026-10 “bba: start polling only when needed. Increase poll interval to 1 ms”

### flycast-201 — Send as soon as data arrives, and replace timer polling with posted work
Lens: io · Source: Flycast
Principle: Holding output until a frame delimiter appears, or draining a queue on a timer, adds latency equal to the wait. Write each byte to the socket's send buffer as it arrives, and post the read handler to the I/O loop instead of polling on a timer.
Why it is faster: —
Applies elsewhere: setting `TCP_NODELAY`, event-driven servers that replace polling loops, Nagle trade-offs in games.
Watch out: —
Seen in Flycast: DCNet's `PPPSocket::send(u8)` appends to `sendBuffer` and calls `doSend()` right away. It no longer waits for a `0x7e` PPP flag or a full buffer. The modem read path uses `io_context::post()` instead of a timer and the `TsQueue<u8> fromModem`. 305cae496 2025-03 “dcnet: don't try to detect ppp frames on output. send data immediately” · 0414a9050 2025-02 “dcnet: don't use timer for modem. better error reporting. race condition”

### flycast-203 — Move heavy implementations out of widely included headers
Lens: compiler-build · Source: Flycast
Principle: Code defined in a header is re-parsed and re-instantiated by every file that includes it. Moving a large class body into a single `.cpp` cuts compile time and rebuild fan-out without changing behaviour.
Why it is faster: —
Applies elsewhere: pimpl idiom, precompiled headers, forward declarations in public APIs.
Watch out: —
Seen in Flycast: About 620 lines of the SH4 SSA optimizer (`SSAOptimizer::Optimize`, its passes and helpers) moved from `ssa.h` to a new `ssa.cpp`. Driver helpers became `static` and `ngen.h` lost four declarations. 30b42a8a7 2026-02 “sh4 dyna: lighter headers”

### flycast-204 — Let the build tool run compiler processes across projects in parallel
Lens: compiler-build · Source: Flycast
Principle: The compiler's per-file parallelism (`/MP`) stops at project boundaries. MSBuild's multi-tool task scheduler spreads compiles over all cores across projects, so builds finish sooner.
Why it is faster: —
Applies elsewhere: `make -j`, Ninja's global job pool, distributed compilation (distcc, sccache).
Watch out: —
Seen in Flycast: For Visual Studio generators, CMake now appends `UseMultiToolTask=true` and `EnforceProcessCountAcrossBuilds=true` to `CMAKE_VS_GLOBALS` instead of adding `/MP`. The UWP CI build passes `-- /m`. b71541f8d 2024-02 “cmake: improved parallelism in msbuild”

### flycast-208 — Tag the page table's pointer bits with a handler id or mirror shift
Lens: memory-layout · Source: Flycast
Principle: Base pointers aligned to 256 bytes leave their low bits free. Store either a handler index (when the pointer part is null) or a shift amount in those bits. One load then yields both whether the page is RAM and how to wrap the address. Mirroring becomes `addr <<= s; addr >>= s;`, which needs no separate mask table.
Why it is faster: —
Applies elsewhere: tagged pointers in Lisp and JS engines, pointer-plus-flag tricks in lock-free lists, NaN-boxing.
Watch out: —
Seen in Flycast: `memInfo_ptr[0x100]` covers the address space in 16 MB pages. `readt` does `uintptr_t iirf = (uintptr_t)memInfo_ptr[page]; void *ptr = (void *)(iirf & ~HANDLER_MAX);`. If `ptr` is non-null (marked `likely`), it shifts by `iirf & HANDLER_MAX` and dereferences. Otherwise it calls `RF32[id]`. `mapBlock` pre-biases each entry with `FindMask(mask) - (j & mask)` so mirrors share one pointer. current code: core/hw/mem/addrspace.cpp

### flycast-210 — Classic SSA passes on a block's intermediate code
Lens: compiler-build · Source: Flycast
Principle: Versioning each register write lets cheap linear passes prove facts locally: propagate constants, drop dead writes, fold back-to-back shifts, and follow trivial branch chains. Each removed operation is host code that never has to be generated or run.
Why it is faster: —
Applies elsewhere: LLVM's mem2reg/SCCP/DCE, eBPF verifier simplifications, query-plan rewriting.
Watch out: —
Seen in Flycast: `SSAOptimizer::Optimize` runs `AddVersionPass`, `ConstPropPass`, `DeadCodeRemovalPass`, `SimplifyExpressionPass`, `CombineShiftsPass`, `DeadRegisterPass`, `IdentityMovePass` and `SingleBranchTargetPass`. `CombineShiftsPass` merges adjacent same-direction shifts while the total stays ≤ 31. `skipSingleBranchTarget` follows up to five `bra`+`nop` hops within the block's pages. current code: core/hw/sh4/dyna/ssa.cpp

### flycast-211 — Spill the register whose next use is furthest away
Lens: compiler-build · Source: Flycast
Principle: When registers run out, evicting the value needed latest minimises reloads. This is Belady's rule, and it is optimal for a single straight-line block. A value with no further use is evicted immediately.
Why it is faster: —
Applies elsewhere: cache replacement simulations (OPT), register allocators in linear-scan JITs, page replacement in teaching OSes.
Watch out: —
Seen in Flycast: `RegAlloc::SpillReg` scans forward through `block->oplist` for each allocated register's first use. It picks the candidate with the largest `first_use`, or breaks out at once if a register has none. Dirty victims are marked `write_back`, and evictions are delayed through `pending_flushes`. current code: core/hw/sh4/dyna/ssa_regalloc.h

### flycast-212 — Recognise a long idiom and replace it with one operation
Lens: compiler-build · Source: Flycast
Principle: Guest compilers emit fixed instruction sequences for operations the guest CPU lacks. Pattern-match the whole sequence at decode time and emit a single host operation instead of dozens of emulated steps.
Why it is faster: —
Applies elsewhere: compilers spotting popcount or memcpy loops, peephole optimisers fusing multiply-add, CPU macro-op fusion.
Watch out: —
Seen in Flycast: `MatchDiv32` scans 32 `rotcl`/`div1` pairs that use consistent registers. If `match == 65`, the decoder emits `shop_div32u` or `shop_div32s` followed by `shop_div32p2`, which replaces the 64-instruction software division loop. current code: core/hw/sh4/dyna/decoder.cpp

### flycast-213 — Table-driven state machine with a "slow path" bit
Lens: lookup-tables · Source: Flycast
Principle: Pack the current state and the relevant input bits into one index, and store the next state plus a "needs handling" flag in a byte table. The common case is one load and a predictable not-taken branch. Only the rare transitions call the handler.
Why it is faster: —
Applies elsewhere: lexer DFA tables, protocol parsers (HTTP and UTF-8 validators), regex engines.
Watch out: —
Seen in Flycast: Each 32-byte TA write computes `u32 state_in = (ta_cur_state << 8) | (pcw.ParaType << 5) | ((pcw.obj_ctrl >> 2) & 31); u32 trans = ta_fsm[state_in];`. It returns early when `!(trans & 0xF0)` (marked `likely`). `fill_fsm()` builds the 2048-entry table at load. The design dates back to the 2013 import (1c16baeb0). current code: core/hw/pvr/ta.cpp

### flycast-215 — Specialise the parser per record type and continue through a function pointer
Lens: compiler-build · Source: Flycast
Principle: Use template parameters for the record layout so each vertex format compiles to straight-line code. Remember "what to parse next" as a function pointer, so parsing can resume mid-record across buffer boundaries without re-decoding the header.
Why it is faster: —
Applies elsewhere: resumable protocol decoders (coroutine-style), simdjson's stage-2 state, specialised deserialisers generated per schema.
Watch out: —
Seen in Flycast: `template <u32 poly_type, u32 poly_size> ta_poly_data` loops `ta_handle_poly<poly_type,0>`. For a 64-byte vertex split across the buffer end, it parses the first half and sets `TaCmd=ta_handle_poly<poly_type,2>`. `TaTypeLut::table[256]` maps `obj_ctrl` to a type id, which indexes `ta_poly_data_lut[15]`. current code: core/hw/pvr/ta_vtx.cpp, core/hw/pvr/ta.h, core/hw/pvr/ta_const_df.h

### flycast-217 — Invalidate cached conversions by page write faults, and evict by age
Lens: memory-layout · Source: Flycast
Principle: Re-hashing every texture each frame is too slow. Write-protect the VRAM pages a cached texture came from, and mark the texture dirty only when the guest writes there. Palettes are checked with a cheap hash comparison, and textures untouched for N frames are evicted.
Why it is faster: —
Applies elsewhere: incremental backup through dirty-page tracking, GPU drivers tracking CPU writes to mapped buffers, browsers invalidating cached layout.
Watch out: —
Seen in Flycast: `VramLocks` is an array of per-page `std::vector<vram_block*>`. `VramLockedWriteOffset` invalidates every texture on the faulting page and unprotects it. `NeedsUpdate()` returns true if `dirty` is set or `palette_hash != pal_hash_16[tcw.PalSelect]`. `CollectCleanup` evicts textures that have been dirty for more than 120 frames. current code: core/rend/TexCache.cpp, core/rend/TexCache.h

### flycast-218 — Shadow API state to skip redundant driver calls, and pool object names
Lens: gpu-rendering · Source: Flycast
Principle: Every GL call costs driver validation even when it changes nothing. Keep a CPU-side copy of the state and issue the call only on a real change. Generate texture names in batches rather than one at a time.
Why it is faster: —
Applies elsewhere: state trackers in bgfx and sokol, connection pools, ID preallocation in databases (sequence caching).
Watch out: —
Seen in Flycast: `GLCache::BindTexture` checks `if (texture != _texture)` before calling `glBindTexture`. The blend, depth, cull, program and capability setters do the same. `GenTexture()` refills with `glGenTextures(TEXTURE_ID_CACHE_SIZE /*32*/, _texture_ids)` and returns `_texture_ids[--_texture_cache_size]`. current code: core/rend/gles/glcache.h

### flycast-221 — Recompile a small program only when it changes, and skip it when empty
Lens: jit-interpreters · Source: Flycast
Principle: Mark a dirty flag when program memory is written. On the next step, recompile only if the flag is set, and if the program is all zeros, stop running it entirely.
Why it is faster: —
Applies elsewhere: shader hot-reload, regex compile caches, GPU command buffers re-recorded only on scene change.
Watch out: —
Seen in Flycast: `aica::dsp::writeProg` sets `state.dirty` for writes in `0x3400–0x3C00`. `step()` scans `DSPData->MPRO` once. If every word is 0 it sets `state.stopped` and returns from then on. Otherwise it calls the host DSP recompiler (x86, x64, arm32 or arm64) before `runStep()`. current code: core/hw/aica/dsp.cpp, core/hw/aica/dsp_x64.cpp

### flycast-222 — Size the entry table to the maximum guest RAM and mask once
Lens: jit-interpreters · Source: Flycast
Principle: Size the code-entry table for the largest RAM configuration and mask the PC to that size, so the dispatch loop never needs the actual RAM size. Skip fill words the block builder knows are padding, and cap block length to keep timing granular.
Why it is faster: —
Applies elsewhere: power-of-two ring buffers indexed with a mask, hash tables with mask instead of modulo.
Watch out: —
Seen in Flycast: The ARM7 recompiler stores `EntryPoints[(pc & (ARAM_SIZE_MAX - 1)) / 4]`, and `flush()` resets every entry to `arm_compilecode`. Blocks stop after 32 ops, and opcodes `0` and `0x41474553` ('SEGA') emit nothing. Register versions are tracked SSA-style for the backend. current code: core/hw/arm7/arm7_rec.cpp

### flycast-224 — Full-width opcode dispatch table and timeslice batching
Lens: lookup-tables · Source: Flycast
Principle: With 16-bit opcodes, a 65,536-entry table of handler pointers decodes any instruction with one load. Running a whole timeslice before checking interrupts, with exceptions thrown out of line, keeps the inner loop to fetch, index and call.
Why it is faster: —
Applies elsewhere: bytecode VMs with dispatch tables, Lua's opcode switch, packet classifiers keyed by header bytes.
Watch out: —
Seen in Flycast: `OpCallFP* OpPtr[0x10000]` is filled by expanding each opcode mask. `Sh4Interpreter::Run` loops `do { u32 op = ReadNexOp(); ExecuteOpcode(op); } while (ctx->cycle_counter > 0);`, then adds `SH4_TIMESLICE` and calls `UpdateSystem_INTC()`. Guest exceptions use C++ `throw` (`SH4ThrownException`), so the loop body has no error checks. current code: core/hw/sh4/interpr/sh4_interpreter.cpp, core/hw/sh4/sh4_opcode_list.cpp

### flycast-226 — Recycle per-frame contexts through a small pool, keeping their capacity
Lens: allocation · Source: Flycast
Principle: Per-frame command contexts hold large vectors. Return them to a bounded pool and `clear()` their vectors instead of freeing them, so later frames reuse both the object and its grown capacity with no allocation.
Why it is faster: —
Applies elsewhere: frame allocators in game engines, request-object pools in web servers, `sync.Pool` in Go.
Watch out: —
Seen in Flycast: `tactx_Alloc()` pops from `ctx_pool` before allocating. `tactx_Recycle` pushes contexts back, keeping at most four. `rend_context::Clear()` calls `idx.clear()`, `global_param_op.clear()`, `modtrig.clear()`, `sortedTriangles.clear()` and so on, all of which keep their storage. current code: core/hw/pvr/ta_ctx.cpp, core/hw/pvr/ta_ctx.h

### flycast-227 — Short-circuit hot guest system calls at a magic address
Lens: other · Source: Flycast
Principle: When a guest OS calls a known service through a fixed entry point, the emulator can intercept it at dispatch and compute the answer natively, skipping thousands of emulated instructions.
Why it is faster: —
Applies elsewhere: vDSO `gettimeofday`, paravirtualised hypercalls, HLE BIOS in console emulators.
Watch out: —
Seen in Flycast: With `USE_WINCE_HACK`, `bm_GetCodeByVAddr` catches odd-address calls. `0xfffffde7` (GetTickCount) sets `Sh4cntx.r[0] = sh4_sched_now64() * 1000 / SH4_MAIN_CLOCK; Sh4cntx.pc = Sh4cntx.pr;` (“This should make this syscall faster”). QueryPerformanceCounter is handled the same way. current code: core/hw/sh4/dyna/blockmanager.cpp
