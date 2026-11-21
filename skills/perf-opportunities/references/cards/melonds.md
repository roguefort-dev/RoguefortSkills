# melonDS — technique cards (174)

Each card is exactly 7 lines. Fetch one with: grep -A 6 "^### melonds-NNN " melonds.md

### melonds-001 — Truth-table bitmask for multi-bit predicates
Lens: lookup-tables · Source: melonDS
Principle: When a yes/no answer depends on a small number of state bits, precompute one integer per predicate in which bit n holds the answer for state n. Evaluating it is then a shift, an AND and a load, with no compound boolean expression and no branches to mispredict. Short-circuit the overwhelmingly common "always true" case before the lookup.
Why it is faster: —
Applies elsewhere: role × action permission matrices (`allowed[role] & (1<<action)`), character-class tests in lexers, state-machine guards in embedded firmware.
Watch out: —
Seen in melonDS: Every ARM instruction is conditional on the NZCV flags. Since the second commit with ALU code, `ARM::ConditionTable[16]` holds a 16-bit truth table per condition (EQ = `0xF0F0`, GE = `0xAA55`, …). `CheckCondition` returns early for `code == 0xE` (AL), and otherwise tests `ConditionTable[code] & (1 << (CPSR>>28))`. The same table and function are still in today's `ARM.h`. 97ec988d 2016-11 “more shit! some start of ALU emulation”

### melonds-002 — Decode once into a dense function table indexed by opcode bits
Lens: lookup-tables · Source: melonDS
Principle: Pick the few instruction or message bits that determine behaviour, pack them into an index, and use it to look up a handler in a table. That is one load and one indirect call, in place of a cascade of masks and compares or a sparse switch the compiler might turn into a binary search.
Why it is faster: —
Applies elsewhere: bytecode VMs (CPython's and Lua's dispatch), protocol parsers keyed on message-type bytes, keyboard scan-code handlers in firmware.
Watch out: —
Seen in melonDS: `ARM::Execute` builds `icode = ((CurInstr >> 4) & 0xF) | ((CurInstr >> 16) & 0xFF0)`, which is bits 27–20 and 7–4, and calls `ARMInstrTable[icode](this)`, a 4096-entry table in `ARM_InstrTable.h`. Thumb has a matching table. Later commits filled the table entry by entry, for example replacing `A_UNK` slots with `A_ADD_REG_LSL_IMM_S`. f74fb2dd 2016-11 “well, adding shit. laying out the base for the interpreter…” · 6d2092a1 2017-02 “blarg”

### melonds-003 — Generate one specialised handler per operand variant
Lens: compiler-build · Source: melonDS
Principle: If an operation has a few orthogonal variants (addressing mode, shift kind, whether flags are set), produce a separate small function for each combination with macros or templates. Each handler then contains only the code for its own variant. The decision is paid once, at table-build time, not again on every execution.
Why it is faster: —
Applies elsewhere: per-format pixel converters generated from one template, SQL engines with expression evaluators specialised by type, codecs with one inner loop per bit depth.
Watch out: —
Seen in melonDS: The interpreter's “macro soup”: `LSL_IMM`/`LSR_IMM`/`ASR_IMM`/`ROR_IMM` and their `_S` (flag-setting) and `_REG` forms are combined by `A_IMPLEMENT_ALU_OP(x)` and `A_CALC_OP2_REG_SHIFT_IMM(shiftop)` into functions such as `A_ADD_REG_LSL_IMM_S`. Load/store gets the same treatment through `A_WB_CALC_OFFSET_REG(shiftop)`. Each entry in the decode table is therefore straight-line code with no runtime switch on the shift type. 97ec988d 2016-11 “more shit! some start of ALU emulation” · 23d584ca 2016-12 “implement LDR/STR/LDRB/STRB. more macro soup.”

### melonds-004 — Fixed-slot event table with an occupancy bitmask
Lens: algorithms · Source: melonDS
Principle: When the set of event kinds is small and known in advance, give each kind a fixed slot and track live slots in a bitmask. You don't need a general sorted queue. Scheduling and cancelling become O(1) bit operations with no allocation or list walk, and finding the next deadline is a scan of a few cache-resident slots that stops once the mask is empty.
Why it is faster: —
Applies elsewhere: RTOS software timers keyed by ID, a game loop's fixed set of periodic jobs, network stacks with one retransmit or keepalive timer per connection state.
Watch out: —
Seen in melonDS: The first scheduler (`343797f4`) allocated events from `SchedBuffer` by linear search and inserted them into a delta-sorted linked list. The revamp replaced this with `SchedEvent SchedList[Event_MAX]` and `u32 SchedListMask`. `ScheduleEvent(id, …)` sets a bit, `CancelEvent` clears it, and `CalcIterationCycles` takes the minimum over the set bits. In today's `NDS::NextTarget`/`RunSystem` the slots hold absolute timestamps, and the loop does `mask >>= 1` and breaks as soon as `!mask`. 343797f4 2016-12 “* somewhat proper event scheduler * support for timers…” · 594286ee 2017-01 “* scheduler revamp, simpler design…”

### melonds-005 — Derive counters from elapsed time instead of ticking them
Lens: laziness · Source: melonDS
Principle: A counter that increases at a fixed rate doesn't need an event per tick. Store its base value and the time it was last synchronised. Compute `base + elapsed × rate` when someone reads it, and schedule a real event only for the moment it overflows.
Why it is faster: —
Applies elsewhere: token-bucket rate limiters that refill on access, TTL caches that compare timestamps instead of running sweeper threads, uptime and statistics counters derived from a monotonic clock.
Watch out: —
Seen in melonDS: Timers used to reschedule a `TimerIncrement` event on every prescaler tick. After the revamp, `TimerGetCounter(timer)` returns `Counter + (cycles << CycleShift)`. The prescaler is stored as a shift (`{0, 6, 8, 10}`) and the counter as 16.16 fixed point, so the division by the prescaler is a shift. Only `TimerOverflow` is a scheduled event. 594286ee 2017-01 “* scheduler revamp, simpler design…”

### melonds-006 — Keep an active-set bitmask to skip idle units
Lens: branching-fast-paths · Source: melonDS
Principle: If a hot loop updates N units and most are usually disabled, maintain a bitmask of the active ones whenever their configuration changes. The hot path then tests one cached byte per unit and never reads each unit's control register.
Why it is faster: —
Applies elsewhere: dirty-flag sets in UI frameworks, a bitmap of runnable queues in OS schedulers, enabled-channel masks in audio mixers and DMA engines.
Watch out: —
Seen in melonDS: `RunTimingCriticalDevices` called `RunTimer` for all four timers of each CPU, and each call checked `(timer->Cnt & 0x84) != 0x80`. Now `TimerStart` keeps `TimerCheckMask[cpu]` up to date, and the loop runs `if (timermask & 0x1) RunTimer(...)` and so on, from one register-held mask. `TimerCheckMask` is still part of `NDS` and its savestates. 9eb68c2e 2017-04 “misc optimizations”

### melonds-007 — Update each component only as often as its precision requires
Lens: other · Source: melonDS
Principle: Split periodic work into tiers by how fine-grained its observable effect is. Run only the fine-grained tier inside the innermost loop. Coarse units run once per outer iteration, and their per-update cost is spread over many cycles of work.
Why it is faster: —
Applies elsewhere: physics at a fixed step with AI at a lower tick rate, per-request versus per-second metrics aggregation, polling a sensor at different rates according to its bandwidth.
Watch out: —
Seen in melonDS: `TimerStart` now sets bit `0x01<<n` (a “tight” timer, prescaler 1) or `0x10<<n` (a “loose” timer, prescaled). `RunTightTimers` runs wherever cycle-exact timing matters. `RunLooseTimers(0, arm9cycles)` runs once per main-loop iteration, after the CPU or DMA has finished its slice. 78d384a7 2018-12 “temp commit”

### melonds-008 — Hoist per-step bookkeeping out of the inner loop into batches
Lens: other · Source: melonDS
Principle: Side work that only has to be correct at slice boundaries shouldn't run on every iteration. Accumulate the elapsed amount and apply it once per batch. That removes a call and its bookkeeping from the hottest loop.
Why it is faster: —
Applies elsewhere: flushing log or metric buffers per batch rather than per event, coalescing DOM writes into one animation frame, updating a progress bar every N items.
Watch out: —
Seen in melonDS: `ARMv5::Execute` used to call `NDS::RunTimingCriticalDevices(0, diff >> ClockShift)` after every instruction, and `DMA::Run` called it after every unit transferred. “make it fasterer” comments out both and calls `RunTimingCriticalDevices` once per iteration in `RunFrame`, after the ARM9 (or its DMAs) and the ARM7 have run their slices. Shortly before that, `7ba32ea0` had already merged the separate ARM9 and ARM7 branches into one `diff >> ClockShift` / `diff & ClockDiffMask` pair. b569c038 2018-12 “make it fasterer” · 7ba32ea0 2018-11 “make the ARM clock shift configurable…”

### melonds-009 — Flatten configuration into a page-granular lookup table
Lens: memory-layout · Source: melonDS
Principle: When attributes depend on address ranges that change rarely, such as permissions, latency or cacheability, expand the range list into one table entry per page each time the configuration changes. Every access then indexes `table[addr >> pageShift]` and no longer evaluates the ranges and their priorities. Precompute derived values too (for example 16- and 32-bit costs), so the hot path reads them directly.
Why it is faster: —
Applies elsewhere: CPU page tables and TLBs, IP routing tables expanded into direct-indexed arrays, per-tile permission grids in games and GIS.
Watch out: —
Seen in melonDS: The ARM9 protection unit's eight overlapping regions are expanded by `UpdatePURegions()` into `u8 PU_PrivMap[0x100000]`, one byte per 4 KB page holding read/write/execute/cache bits. Memory timings went from a `RegionTimings` struct per named region to `u8 ARM9MemTimings[0x100000][4]` (N16/S16/N32/S32 per page), filled by `SetARM9RegionTimings(start, end, buswidth, nonseq, seq)`. That function precomputes `N32 = N16 + S16` for 16-bit buses. In `29bca33b` the CPU's per-page `MemTimings[addr>>12]` replaced `ARM9MemTimings[CodeRegion]`, and `4aafdee1` caches a region's code timing in `RegionCodeCycles` when a jump happens. 86dae1a2 2018-12 “make this other branch where we're going to actually make it usable” · c6fb152d 2018-12 “start devolving things…” · 29bca33b 2018-12 “take it somewhere.” · 4aafdee1 2018-12 “more sensible cache timings”

### melonds-010 — Cache a direct pointer to the current region; refresh it only on region change
Lens: memory-layout · Source: melonDS · Also: ares
Principle: Sequential accesses almost always stay in the same backing buffer. Resolve the buffer (base pointer plus mask) once, and redo the lookup only when control flow can move to another region. The common access then becomes `mem[addr & mask]` and skips the general dispatcher.
Why it is faster: —
Applies elsewhere: iterators that hold a pointer into the current chunk of a rope or deque, a DB cursor that keeps the current page pinned, memory-mapped file readers that remap only at window boundaries.
Watch out: —
Seen in melonDS: `ARMv5::CodeRead32` now starts with `if (CodeMem.Mem) return *(u32*)&CodeMem.Mem[addr & CodeMem.Mask];`. `JumpTo` calls `SetupCodeMem(addr)` only `if (newregion != oldregion)`. The commit says the ARM7 isn't worth it because its code runs from WRAM, whose mapping prevents this shortcut. `MemRegion CodeMem` is still in `ARM.h`. 91b2f645 2018-12 “add back faster prefetch for ARM9” • Also seen in ares (ares-080): The N64 recompiler's per-instruction icache timing called `devirtualizeFast(ipu.pc)`, which used to walk the TLB every time. It gained a `DevirtualizeCache{vbase,pbase}`: a hit when `vaddr>>12 == vbase` returns `pbase + (vaddr & 0xfff)`. The cache was first cleared on every mapped store, and is now cleared only in `TLBWI`/`TLBWR`. A follow-up removed a redundant `pbase` store from the miss path and marked the function `alwaysinline`. 7f09b67df 2023-06 “n64: speed up recompiler icache lookup” · 345a7cb89 2023-06 “n64: clear devirtualize cache on TLB changes rather than memory writes” · 878419da0 2023-06 “n64: avoid unnecessary pbase writes in devirtualizeFast and hint as alwaysinline”

### melonds-011 — Replace mode-flag parameters with dedicated entry points
Lens: compiler-build · Source: melonDS
Principle: A function that takes a flags argument and branches on it inside makes every caller pay for that branch, even though the call site already knows which flag it is passing. Split it into one function per mode so each call site goes directly to the right body.
Why it is faster: —
Applies elsewhere: `read()` and `readSequential()` instead of `read(…, bool seq)`, `memcpy` versus `memmove`, separate `encodeKey` and `encodeValue` in serialisers.
Watch out: —
Seen in melonDS: `DataRead32(addr, val, flags)` checked `flags & RWFlags_Nonseq` on every access to pick N or S timing, and returned a bool. It became `void DataRead32(addr, val)` (non-sequential) plus `DataRead32S` (sequential, which only adds `MemTimings[...][3]`), and the same split was made for writes. LDM/STM loops call the `S` variants after their first access. 29bca33b 2018-12 “take it somewhere.”

### melonds-012 — Special-case the dominant pattern with a tight fast path
Lens: algorithms · Source: melonDS
Principle: Profile which shape of a general operation dominates. Give that shape a loop with no per-element generality: no address-space dispatch and no width or direction checks. Keep the general path for everything else.
Why it is faster: —
Applies elsewhere: `memcpy` paths for aligned sizes, JSON parsers with an ASCII-only string fast path, ORMs that bulk-insert homogeneous rows.
Watch out: —
Seen in melonDS: Games mostly feed the 3D engine through DMA from main RAM to the fixed port `0x04000400`. `DMA::Run` gained a dedicated loop for this: `GPU3D::WriteToGXFIFO(*(u32*)&NDS::MainRAM[CurSrcAddr&0x3FFFFF])`, with timing taken directly from `Waitstates`. It skips the generic `ARM9Read32`/`ARM9Write32` dispatch and the I/O switch. To make this possible, the FIFO command unpacker was moved out of `Write32` into its own `WriteToGXFIFO`. 1759672d 2017-04 “a few attempts at optimization”

### melonds-013 — Evaluate invariant predicates once at setup
Lens: laziness · Source: melonDS
Principle: If a condition depends only on parameters that are fixed for the lifetime of an operation, compute it once at start and store it as a bool. Each later iteration then tests a single flag and doesn't re-evaluate the whole compound expression.
Why it is faster: —
Applies elsewhere: deciding once per request whether to compress or log, computing `isPowerOfTwo` on buffer creation, prepared-statement flags.
Watch out: —
Seen in melonDS: The GXFIFO fast path first tested `CPU == 0 && (CurSrcAddr>>24) == 0x02 && CurDstAddr == 0x04000400 && DstAddrInc == 0` on every `Run`. The next commit computes `IsGXFIFODMA` in `DMA::Start()` and tests only that flag. The same line is in today's `DMA.cpp`. bdac2467 2017-04 “oh well”

### melonds-014 — Lift per-batch decisions into template parameters
Lens: compiler-build · Source: melonDS
Principle: When a decision is fixed for a whole batch (a polygon, a scanline, a layer), make it a compile-time template or constant parameter. Then select the right instantiation once per batch, through a function pointer or a switch. Inside the hot loop the compiler removes the dead side of each branch.
Why it is faster: —
Applies elsewhere: `std::sort` instantiated per comparator, image filters templated on channel count, shader permutations, Rust generics over a strategy trait.
Watch out: —
Seen in melonDS: `DepthTest(Polygon*, x, y, z)` checked `polygon->Attr & (1<<14)` for every pixel. It became `template<bool func_equal> DepthTest(oldz, z)`, with `fnDepthTest` chosen once per polygon. Later the rasterizer's `Slope<side>` (left/right edge) and `Interpolator<dir>` (X/Y) turned runtime `side` and `shift` arguments into template parameters. On the 2D side, `DrawScanlineBGMode<bgmode>` is instantiated per BG mode and `DrawSprite_Normal<window>` per window/colour pass. All of these templates are still in the current `GPU3D_Soft.h` and `GPU2D_Soft.cpp`. 796e6082 2017-04 “small optimizations to the 3D renderer” · ae78faf2 2017-05 “rework slope/edge code…” · d656e6e7 2017-08 “3D: faster and more accurate interpolation” · 3451e41b 2017-02 “tile hflip/vflip for text BGs…” · bc70f003 2017-04 “windows!”

### melonds-015 — Compute the expensive interpolation weight once, reuse it for every attribute
Lens: algorithms · Source: melonDS
Principle: Interpolating many attributes between the same two endpoints uses the same blend weight each time. Compute the weight once per sample, which may cost a division, and then derive each attribute with a multiply and a shift. One divide replaces one per attribute.
Why it is faster: —
Applies elsewhere: animation systems that compute one easing factor and lerp every property with it, barycentric weights shared across vertex attributes, colour gradients that interpolate several channels from one `t`.
Watch out: —
Seen in melonDS: The old inner loop divided by `denom` for Z, R, G, B, S and T separately: `((factor1 * rl) + (factor2 * rr)) / denom`. The new `Interpolator` class computes `yfactor` in `SetX(x)`, and then `Interpolate(y0, y1)` is `y0 + (((y1 - y0) * yfactor) >> shift)`. Polygon setup also moved Z/W into per-polygon `FinalZ[]`/`FinalW[]`, with W normalised by `WShift` so that the values fit in 16 bits. eb656857 2017-04 “optimize the 3D renderer somewhat”

### melonds-016 — Multiply by a precomputed reciprocal instead of dividing
Lens: algorithms · Source: melonDS
Principle: Integer division costs tens of cycles, multiplication only a few. If the divisor is fixed across many operations, such as a span length, compute `(1<<k) / d` once and replace each `x / d` with `(x * recip) >> k`, adding a rounding bias where needed.
Why it is faster: —
Applies elsewhere: compilers' magic-number division, fixed-point DSP scaling, hash-table modulo via multiply-shift (Lemire's fastrange), per-row normalisation in image resizers.
Watch out: —
Seen in melonDS: `Interpolator::Setup` computes `xrecip = (1<<30) / xdiff` once per span. The linear path changed from `((y1-y0) * x) / xdiff` to `(((s64)(y1-y0) * x * xrecip) + (3<<24)) >> 30`, and Z uses `xrecip_z`. The perspective weight went from three 64-bit divisions to one: `num / den` with a 32-bit `den`. The current `GPU3D_Soft.h` still has `xrecip_z = (1<<22) / xdiff`. d656e6e7 2017-08 “3D: faster and more accurate interpolation”

### melonds-017 — Split a loop by region so classification happens once per region
Lens: branching-fast-paths · Source: melonDS
Principle: If a loop body begins with "which zone is this element in?" and the zones are contiguous (head, middle, tail), write one loop per zone with the zone's logic built in. The per-element tests disappear, and the large middle loop gets the simplest body.
Why it is faster: —
Applies elsewhere: SIMD loops with scalar prologue and epilogue, convolution that handles image borders separately from the interior, text layout that treats the first and last line specially.
Watch out: —
Seen in melonDS: `RenderPolygonScanline` did `if (x < xstart+l_edgelen) edge |= 0x1; else if (x > xend-r_edgelen) edge |= 0x2;` for every pixel. It now has three loops (“part 1: left edge”, “part 2: polygon inside”, right edge), each with a fixed `edge`. For wireframe polygons the interior loop is skipped outright (`if (wireframe && !edge) x = xend-r_edgelen+1;`). 27c023a0 2017-06 “separate rasterizer inner loop into three loops for leftedge/middle/rightedge”

### melonds-018 — Clamp loop bounds up front instead of bounds-checking each iteration
Lens: branching-fast-paths · Source: melonDS
Principle: Clip the iteration range to the valid interval once, before the loop. The body can then assume every index is in range and drops its `continue`/`break` guards.
Why it is faster: —
Applies elsewhere: clipping a blit rectangle against the destination before copying, slicing an array to its valid window before mapping, intersecting a query range with the shard's key range.
Watch out: —
Seen in melonDS: Each of the three span loops began with `if (x < 0) continue; if (x > 255) break;`. These were replaced by a single `if (x < 0) x = 0;` before the left loop and `if (xend > 255) xend = 255;` before the right one. d1781c8b 2017-06 “remove more per-pixel checks”

### melonds-019 — Hoist loop-invariant work out of the inner loop
Lens: branching-fast-paths · Source: melonDS
Principle: Anything that depends only on outer-loop variables should be computed outside the inner loop. Compilers can't always do this for you, particularly when the value goes through memory they can't prove is unaliased.
Why it is faster: —
Applies elsewhere: per-row pointer arithmetic in image code, per-request config lookups pulled out of per-item loops, `len(arr)` cached outside a Python loop.
Watch out: —
Seen in melonDS: In `RenderPolygon`, the per-pixel `if (y == ytop) edge |= 0x4; else if (y == ybot-1) edge |= 0x8;` became `int yedge`, computed once per scanline, with each pixel starting from `int edge = yedge;`. 9eb68c2e 2017-04 “misc optimizations”

### melonds-020 — Fold an enable flag into a threshold so one compare covers both cases
Lens: branching-fast-paths · Source: melonDS
Principle: "If feature X is on, test `v <= ref`; otherwise test `v == 0`" turns into a single comparison once the threshold is precomputed as `ref` when the feature is on and as the neutral value when it is off. Recompute the threshold whenever its inputs are written, which is rare, and keep the per-element path branch-free.
Why it is faster: —
Applies elsewhere: a log-level filter stored as a single minimum level, with "disabled" mapped to a level above the maximum. A rate limit set to `INT_MAX` when it is off.
Watch out: —
Seen in melonDS: The per-pixel alpha test was `if (DispCnt & (1<<2)) { if (alpha <= AlphaRef) continue; } else { if (alpha == 0) continue; }`. Register writes now keep `AlphaRef = (DispCnt & (1<<2)) ? AlphaRefVal : 0;`, so the pixel loop runs just `if (alpha <= AlphaRef) continue;`. 9eb68c2e 2017-04 “misc optimizations”

### melonds-021 — Gate expensive secondary work on a cheap necessity check
Lens: laziness · Source: melonDS
Principle: If an expensive step only matters under a condition that is cheap to test, test the condition first and skip the step when it is false. The check might be a flag on the destination, or whether any consumer exists at all.
Why it is faster: —
Applies elsewhere: skipping layout when no style changed, not serialising debug fields unless a debugger is attached, not computing analytics events when there are no subscribers.
Watch out: —
Seen in melonDS: When the depth test against the top pixel failed, the rasterizer always retried against the hidden lower pixel, which exists for antialiasing. That retry now happens only `if (dstattr & 0x3)`, that is, when the top pixel is an edge (`77d0a0d7`). The display-FIFO emulation, which schedules an event every 8 pixels, runs only when `RunFIFO = GPU2D_A->UsesFIFO() || NDS::DMAsInMode(0, 0x04)` is true (`4afac282`). 77d0a0d7 2017-06 “only do lower-pixel rendering when rendering under an edge” · 4afac282 2017-06 “proper display FIFO emulation”

### melonds-022 — Give reduced-requirement variants their own stripped-down path
Lens: algorithms · Source: melonDS
Principle: When one kind of input needs only part of the full pipeline's outputs, route it through a dedicated function that computes only those. Any test that is constant for that kind, such as a cull or rejection check, can be moved to the start of the function.
Why it is faster: —
Applies elsewhere: depth-only prepasses and shadow-map passes that skip fragment shading, `HEAD` requests that skip body generation, count-only queries that skip row materialisation.
Watch out: —
Seen in melonDS: Shadow-mask polygons only write stencil bits. `RenderShadowMaskScanline` was split out of `RenderPolygonScanline`. It interpolates only Z, and the commit notes say colour and texcoords “aren't needed for shadow masks”. Because all pixels share the same alpha, it also does the alpha test once per span: `if (polyalpha <= RenderAlphaRef) return;`. 183b8b58 2017-08 “3D: move shadow mask rendering to separate function, optimize it…”

### melonds-023 — Pad buffers with a sentinel border to remove edge cases
Lens: memory-layout · Source: melonDS
Principle: Add a margin of dummy cells around a 2D buffer so that neighbourhood reads at the edges stay in bounds and return a neutral value. Every pixel then runs the same branch-free code. If several layers of the same buffer are needed, store them as whole separate planes at a fixed offset, not interleaved.
Why it is faster: —
Applies elsewhere: ghost cells in stencil and CFD codes, a null-terminator sentinel in string scanning, padded convolution inputs in CNN libraries.
Watch out: —
Seen in melonDS: The 3D colour, depth and attribute buffers became 258×194, with `FirstPixelOffset = ScanlineWidth + 1`, which adds “a offscreen 1px border which simplifies edge marking tests”. The second (lower-pixel) layer moved from a 258-entry interleave to a full plane at `+BufferSize`. 64a447bb 2017-06 “clean up some shit. make framebuffer access less weird.”

### melonds-024 — Pipeline a producer thread one row ahead with a counting semaphore
Lens: concurrency · Source: melonDS
Principle: Work that is consumed in order, row by row, can be produced on another thread. The producer posts a counting semaphore after each finished row and the consumer waits on it before using that row. The two threads overlap, and the consumer blocks only if it overtakes the producer. This requires the producer's work to be split into independent rows, so a whole-frame algorithm may need restructuring into a per-row one first.
Why it is faster: —
Applies elsewhere: video decoders that hand off finished macroblock rows, streaming parsers feeding a renderer, tile-based map rendering in a worker.
Watch out: —
Seen in melonDS: The software rasterizer was first changed from per-polygon to per-scanline: `RenderScanline(y)` iterates `PolygonList` with per-polygon edge state kept in `RendererPolygon`. It then moved to `RenderThreadFunc`, which does `Semaphore_Post(Sema_ScanlineCount)` after each line. `GPU2D::DrawScanline` calls `GPU3D::RequestLine(line)`, and that waits on the semaphore. `fa2db382` made the thread optional (`Config::Threaded3D`), keeping the synchronous path. The same `Sema_RenderStart`/`RenderThread` structure is in today's `GPU3D_Soft.cpp`. cb1b03b2 2017-05 “make the 3D renderer work per-scanline” · b66ac09e 2017-05 “make the renderer truly per-scanline” · 4b3caedb 2017-05 “first attempt at threading the 3D renderer” · fa2db382 2017-06 “(finally) make the threaded 3D renderer option actually work”

### melonds-025 — Hand off a snapshot, not shared live state
Lens: concurrency · Source: melonDS
Principle: At a natural sync point, copy the state a consumer thread needs into a private snapshot, then let the producer go on mutating its live copy. The consumer reads stable data with no locks, and neither side waits on the other's critical section.
Why it is faster: —
Applies elsewhere: double-buffered game state between simulation and render threads, RCU in the Linux kernel, immutable config objects swapped atomically.
Watch out: —
Seen in melonDS: At VBlank, `GPU3D::VBlank` latches `RenderDispCnt`, `RenderAlphaRef`, `RenderToonTable`, `RenderEdgeTable`, the fog tables and the clear attributes, and the renderer reads only those `Render*` copies (`9fe24cb1`). Hardware latches these registers too, and copying them also gave the later render thread data the CPU side wouldn't overwrite. In the frontend, the emulator thread `memcpy`s the framebuffer into `ScreenBuffer` and the UI draws from that copy, replacing a commented-out `ScreenMutex` (`0aa3434e`). 9fe24cb1 2017-05 “latch 3D registers upon VBlank” · 0aa3434e 2017-09 “double-buffer framebuffer, eliminates artifacts”

### melonds-026 — Keep a neighbour-dependent pass streaming by lagging it one row
Lens: memory-layout · Source: melonDS
Principle: If a post-pass on row y needs row y+1, don't fall back to a full second pass over the frame. Run the post-pass for y−1 immediately after producing y. The data is still in cache, and output can still stream out row by row.
Why it is faster: —
Applies elsewhere: line-buffered image filters (3×3 kernels keep three rows), streaming video deblocking, rolling-window aggregations.
Watch out: —
Seen in melonDS: Edge marking compares a pixel with the pixel below it. `RenderPolygons` now renders line 0, then in a loop renders line `y` and calls `ScanlineFinalPass(y-1)` (edge, fog, AA), and finishes with `ScanlineFinalPass(191)`. That keeps the per-scanline handoff to the render thread working. d3b3255c 2017-05 “* add 'push old pixel down' behavior for antialiasing * delay final pass (edge/fog/AA) one scanline…”

### melonds-027 — Partition at production time and pass pointer arrays
Lens: algorithms · Source: melonDS
Principle: If a consumer needs items grouped by category, count each category as items are produced. A single pass can then place pointers straight into their final slots. Passing pointers rather than copying records avoids moving large structs, and the consumer makes one pass instead of one per category.
Why it is faster: —
Applies elsewhere: render queues bucketed into opaque and transparent at submit time, counting sort, log routers that write into per-level buffers on ingest.
Watch out: —
Seen in melonDS: `SubmitPolygon` now increments `NumOpaquePolygons`. At VBlank one loop writes opaque polygons to `RenderPolygonRAM[io++]` and translucent ones to `RenderPolygonRAM[it++]`, a `std::array<Polygon*,2048>`. `RenderPolygons` lost its two filtering loops (`if (polygons[i].Translucent) continue;` and the reverse). 01404ac6 2017-07 “3D: move opaque/translucent sorting to GPU3D.cpp”

### melonds-028 — Pack a multi-field ordering into one integer key
Lens: algorithms · Source: melonDS
Principle: Ordering by (group, primary, secondary) can be encoded as a single integer by shifting each field into its own bit range, most significant field highest. The comparator becomes one integer compare with no chain of tie-breaks. A stable sort keeps the original order among equal keys.
Why it is faster: —
Applies elsewhere: render sort keys that pack layer, material and depth into a `u64`, database composite-key encodings, packed (priority, sequence) task-queue keys.
Watch out: —
Seen in melonDS: At submit time `poly->SortKey = (ybot << 8) | ytop`, with `|= 0x10000` added for translucent polygons. `YSort` is just `a->SortKey < b->SortKey`, used with `std::stable_sort` over the pointer array. d5376b41 2017-07 “3D: Y-sorting”

### melonds-029 — Do channel arithmetic on packed values in one register
Lens: simd · Source: melonDS
Principle: If colour channels sit in disjoint bit fields of one word with enough headroom between them, you can mask the word and multiply it as a whole. That scales the channel by a small factor without unpacking and repacking each one. It's a simple form of SWAR (SIMD within a register).
Why it is faster: —
Applies elsewhere: RGB565 or RGBA8888 alpha blending in embedded graphics, `(c & 0xFF00FF) * a` tricks in software rasterizers, popcount and byte-compare SWAR tricks.
Watch out: —
Seen in melonDS: 2D colour effects store pixels as 6-bit channels spaced 8 bits apart. Blending computes `g = (((val1 & 0x003F00) * eva) + ((val2 & 0x003F00) * evb)) >> 4` directly on the masked word, and does the same for R and B, followed by per-field saturation. Brightness up/down uses the same in-place approach. e0fa57fb 2017-03 “* color special effects (fade, blending) * sprite blending”

### melonds-030 — Precompute a per-row mask and test bits in the inner loops
Lens: memory-layout · Source: melonDS
Principle: If several inner loops all need to know whether position x is enabled for them, compute a per-position bitmask once per row. Each loop then tests one bit instead of re-evaluating rectangle or region logic. When the feature is off, fill the mask with all ones so the loops stay the same.
Why it is faster: —
Applies elsewhere: clip masks in 2D vector renderers, a per-line visibility bitmap in text editors, stencil buffers in GPUs.
Watch out: —
Seen in melonDS: `CalculateWindowMask(line, mask)` fills 256 bytes from the outside, OBJ, window 1 and window 0 settings in priority order. If windows are disabled it does `memset(windowmask, 0xFF, 256)`. BG loops test `windowmask[i] & (1<<bgnum)`, sprites test `& 0x10`, and blending tests `& 0x20`. Today's `GPU2D_Soft` keeps `alignas(8) u8 WindowMask[256]` and the `memset(…, 0xFF, 256)` fallback. bc70f003 2017-04 “windows!”

### melonds-031 — Check several coordinates against power-of-two bounds with one OR and mask
Lens: branching-fast-paths · Source: melonDS
Principle: With a power-of-two size, a coordinate is out of range exactly when it has bits set above the size bits. Negative values in two's complement have those bits set as well. So `(x | y) & ~(size-1)` checks both axes and both directions in one test. A mask of 0 turns the check off when the coordinates should wrap instead.
Why it is faster: —
Applies elsewhere: tilemap and texture addressing, ring buffers with power-of-two capacity, spatial hashing grids.
Watch out: —
Seen in melonDS: For rotscale/extended BGs, `overflowmask = (bgcnt & 0x2000) ? 0 : ~(coordmask | 0x7FF)`, and each pixel does `if (!((rotX|rotY) & overflowmask))` followed by `(rotX & coordmask)` for indexing. Four comparisons become one AND, and wrapping needs no separate code path. 4b3f635c 2017-02 “fixes to tiled extended BGs. support for 256color/direct bitmap BGs.”

### melonds-032 — Replace per-element modulo with a wrapping counter
Lens: algorithms · Source: melonDS
Principle: For "repeat every N" logic inside a sequential loop, keep a counter that resets when it reaches N rather than computing `i % N` or `i / N` for every element. If N comes from a small fixed set, a precomputed table is the next step.
Why it is faster: —
Applies elsewhere: pixelation and downsampling filters, round-robin assignment, emitting a separator every N items.
Watch out: —
Seen in melonDS: Mosaic tracks `BGMosaicY`/`BGMosaicYMax` and the OBJ equivalents with increment-and-reset per scanline, and an `xmos` counter per pixel. A new tile is fetched only `if (xmos == 0)`. Today's `GPU2D_Soft.h` goes further with a `constexpr` `MosaicTable[16][256]` and a `CurBGXMosaicTable` pointer. 82e42c5b 2017-07 “2D: mosaic”

### melonds-033 — Render once into a tagged line buffer, composite in a separate pass
Lens: algorithms · Source: melonDS
Principle: If many sources contribute to an output and ordering depends on attributes such as priority, render each source once into an intermediate buffer, with its attributes packed into each entry. A single cheap compositing pass then merges the buffers. This avoids re-rendering a source for every priority level or every neighbouring layer.
Why it is faster: —
Applies elsewhere: deferred shading's G-buffer, browser compositing layers, Z-buffered sprite batching.
Watch out: —
Seen in melonDS: `DrawSprites(line, spritebuf)` walks OAM once per line, writing colour with the priority in bits 15–19. `InterleaveSprites(spritebuf, prio, dst)` then copies matching pixels at each BG priority level. Colour effects keep the top two layers per pixel (`linebuf[i]`, `linebuf[256+i]`) and blend them in one pass at the end. Today this is `alignas(8) u32 BGOBJLine[256*2]`, cleared 64 bits at a time with `*(u64*)&BGOBJLine[i]`. 14f1cec9 2017-01 “some beginning of sprite support…” · e0fa57fb 2017-03 “* color special effects (fade, blending) * sprite blending”

### melonds-034 — Tabulate small periodic waveforms
Lens: lookup-tables · Source: melonDS
Principle: If a signal's value is a simple function of a small phase index and a small parameter, store every combination in a table. Generating a sample is then one indexed load, with no compares or arithmetic per sample.
Why it is faster: —
Applies elsewhere: wavetable synthesis, sine tables on microcontrollers, PWM duty patterns, CRC and gamma tables.
Watch out: —
Seen in melonDS: PSG (square-wave) channels use `const s16 PSGTable[8][8]`, indexed by duty cycle and phase, holding ±`0x7FFF`. The ADPCM decoder takes its step size from `ADPCMTable[89]` and the index delta from `ADPCMIndexTable[8]`. d8bbf254 2017-04 “* PCM8, PCM16, ADPCM, white noise * volume, volume div, panning, master volume”

### melonds-035 — Move small hot containers into header templates
Lens: compiler-build · Source: melonDS
Principle: A tiny container whose methods live in a `.cpp` file costs a real call for every push and pop unless link-time optimisation is on. Defining it in a header template makes every method inlinable at the call site. It also lets the element type be anything, instead of being fixed to `u32`.
Why it is faster: —
Applies elsewhere: header-only ring buffers in embedded code, `std::vector`, small-vector and arena types in game engines.
Watch out: —
Seen in melonDS: `FIFO.cpp` (non-template `u32` FIFO with out-of-line `Write`/`Read`/`Peek`) was deleted. `FIFO.h` now has `template<typename T> class FIFO` with inline methods, used both for the IPC FIFO and for `FIFO<CmdFIFOEntry>` in the GX pipeline. The current `FIFO.h` is still header-only. 2b7fac05 2017-02 “rework FIFO crap so it can also be used for the GXFIFO crapo.”

### melonds-036 — Default builds to an optimised configuration
Lens: compiler-build · Source: melonDS
Principle: If no build type is chosen, many build systems compile without optimisation, and users then run code that is several times slower than necessary. Make the optimised configuration the default.
Why it is faster: —
Applies elsewhere: `CMAKE_BUILD_TYPE` defaults, `cargo build --release` in install docs, `NODE_ENV=production` for bundlers.
Watch out: —
Seen in melonDS: `CMakeLists.txt` gained `set(CMAKE_BUILD_TYPE Release)`. af5fb2e8 2017-06 “add optimization to cmake build”

### melonds-037 — Compile out diagnostic instrumentation
Lens: compiler-build · Source: melonDS · Also: ares
Principle: Counters and checks added while debugging run on hot paths. Put them behind a preprocessor or build flag, so release builds contain no extra stores, branches or globals while the checks stay available on demand.
Why it is faster: —
Applies elsewhere: `assert`/`NDEBUG`, Rust's `debug_assert!`, tracing macros that compile to nothing.
Watch out: —
Seen in melonDS: Ad-hoc globals such as `arm9total`/`arm7total`, which were updated in `ARMv5::Execute`, `ARMv4::Execute` and `DMA::Run`, were replaced by `dbg_CyclesARM9` and related counters, all wrapped in `#ifdef DEBUG_CHECK_DESYNC`. 3dd8c7b1 2018-12 “finish the work on the main loop… also add #ifdef'd debug code to check for desyncs.” • Also seen in ares (ares-033): The CPU gained a `struct Debugger { hook<void (uint32)> op_exec, op_read; hook<void (uint32, uint8)> op_write; }` called from `op_step` and the bus accessors. The changelog says hooks don't perform the test or invocation when `DEBUGGER` isn't defined, so regular builds have no overhead. This replaced `debugvirtual` overrides. Earlier, the debugger had been changed "to not break out of core for simple things", going from sub-1 fps to near full speed with breakpoints set. 730e6ae4c 2012-02 “Update to v085r04 release.” · 92da6bde2 2011-01 “Update to v073r07 release.”

### melonds-038 — Move cold cases out of the hot dispatcher
Lens: compiler-build · Source: melonDS
Principle: A dispatcher that handles a few hot fast paths alongside a large switch of rare cases gets big, hurts instruction-cache density, and is less likely to be inlined. Moving the cold cases into a separate function leaves the hot dispatcher small.
Why it is faster: —
Applies elsewhere: `__attribute__((cold))` error handlers, slow-path fallbacks in allocators, rarely used HTTP methods routed to a separate handler.
Watch out: —
Seen in melonDS: `ARM9Read8/16/32` and the matching write functions in `NDS.cpp` had large inline `switch (addr)` blocks for I/O registers inside the per-region switch. These moved to `ARM9IORead8/16/32`, `ARM7IORead*` and the write equivalents. The main functions now hold only the per-region memory cases, such as `Palette[addr & 0x7FF]`. The commit message gives code organisation as the reason, so treat any speed gain as a side effect. 8ea2aaad 2017-01 “separate IO read/write handlers from the rest before it's too late…”

### melonds-039 — Schedule events by absolute deadline, not by countdown
Lens: algorithms · Source: melonDS
Principle: If every pending event stores "cycles remaining", each tick has to decrement all of them. Store an absolute timestamp instead. Advancing time is then one store, and checking an event is one compare against "now". Walk the active set as a bitmask and stop once no bits remain, so the scan never visits empty slots.
Why it is faster: —
Applies elsewhere: timer wheels and `setTimeout` queues that store expiry times, not TTLs. Game loops that compare against `nextSpawnAt`. RTOS tick-less schedulers that program the next hardware deadline.
Watch out: —
Seen in melonDS: Before, `NDS::RunSystem(cycles)` subtracted `cycles` from every `SchedList[i].WaitCycles`, and each CPU kept its own `CyclesToRun`/`ARM7Offset` bookkeeping. The redesign gives each CPU a 64-bit `ARM9Timestamp`/`ARM7Timestamp` and a target. `NextTarget()` takes the minimum event `Timestamp`, and `RunSystem(timestamp)` fires an event when `Timestamp <= SysTimestamp`. Both loops use `if (!mask) break; ... mask >>= 1;`, so they end at the last pending event. 669247e8 2019-01 “redesign main emu loop to use timestamps instead of being a trainwreck”

### melonds-040 — Synchronize coarse-grained when you can
Lens: concurrency · Source: melonDS · Also: ares
Principle: When two components run interleaved and resync at a fixed quantum, every resync costs dispatch overhead and cache warm-up. A larger quantum spreads that cost over more useful work. Precise events still cut a slice short, so the larger quantum only changes the default upper bound.
Why it is faster: —
Applies elsewhere: batch size in message consumers, the flush interval of a write-behind cache, the time slice of a cooperative scheduler, lockstep tick rate in multiplayer simulations.
Watch out: —
Seen in melonDS: `kMaxIterationCycles`, the longest the ARM9 and ARM7 can run before swapping when no event is pending, went from 16 to 64. `NextTarget()` still clamps the slice to the earliest scheduled event. 49b2f8d9 2019-06 “raise kMaxIterationCycles to 64” • Also seen in ares (ares-040): `md::CPU::wait()` called `Thread::synchronize()` on every bus wait. It now decrements `cyclesUntilSync` and syncs only when it reaches ≤0, adding `minCyclesBetweenSyncs` each time: 4 CPU cycles for the Mega Drive and Mega CD, 40 for the 32X. The commit notes that the dot renderer gains nothing from syncing more often than about a pixel. `M32X::SH7604::step` uses the same countdown before syncing the two SH-2s with the 68K. 8e94e664c 2021-08 “md: optimize how often we sync the cpu with other components.” · 2380e62c7 2021-08 “md: fix sync to be in terms of cpu cycles” · cbfd7ed27 2022-09 “m32x: unify interpreter/recompiler sync timing” • Also seen in ares (ares-132): `CPU::instruction()` now returns whether to synchronize. On the JIT path the budget is recomputed only when `Thread::clock >= jitClockTarget`, as `min(JitInterleaving, timerDelta, queueDelta)`. `Accuracy::CPU::JitInterleaving` is a constexpr tuning knob. 8b4f1fa16 2026-05 “n64: CPU now has configurable interleaving with the rest of the hardware”

### melonds-041 — Recompute a derived predicate when its inputs change, not when you read it
Lens: laziness · Source: melonDS
Principle: If a condition is read in a hot loop but its inputs change rarely, compute it at write time and cache the result in a single field. The hot path then reads one word instead of several loads and ANDs.
Why it is faster: —
Applies elsewhere: an `isDirty` flag maintained by setters, a cached `canSubmit` in UI state, a precomputed "any listener registered?" bit, an interrupt-pending flag in an RTOS.
Watch out: —
Seen in melonDS: After every instruction, both CPU loops checked `NDS::IF[n] & NDS::IE[n]` and then `IME`. Now `NDS::UpdateIRQ(cpu)` stores `IE & IF`, or 0 when IME is off, into `ARM::IRQ`. It runs on every write to IME, IE or IF, and the loop checks only `if (IRQ) TriggerIRQ();`. 00a55764 2019-06 “fasterer IRQ check. clean up code.”

### melonds-042 — Overlay several exit flags on one word and test them together
Lens: branching-fast-paths · Source: melonDS
Principle: If the hot loop has to stop for any of N rare reasons, store the reasons as bytes in a union with a single integer. One load and compare against zero covers all of them. You only work out which flag fired on the rare path.
Why it is faster: —
Applies elsewhere: Linux's `thread_info->flags`, checked once on syscall exit. A packed "needs attention" word in an event loop. Interpreters that fold pending signals, GC requests and breakpoints into one poll word.
Watch out: —
Seen in melonDS: `ARM::Halted`, `IRQ` and `IdleLoop` became `u8` fields of an anonymous struct, unioned with `u32 StopExecution`. JIT block epilogues emit a single `CMP [RCPU+StopExecution], 0` and use it to decide whether execution may chain into the next block. 81f38c14 2019-10 “integrate changes from ARM64 backend and more…unify Halted and IRQ in anticipation for branch inlining” · 3787bab1 2020-04 “implement block linking”

### melonds-043 — Skip a pass when its input is provably empty
Lens: algorithms · Source: melonDS
Principle: If an earlier stage can report "I produced nothing" cheaply, use that to skip the later stages that would only scan empty buffers. Also check that the gate condition is exact. A guard written with OR where AND was meant turns the optimization off without anyone noticing.
Why it is faster: —
Applies elsewhere: skip a render pass when its draw list is empty, return early from a merge when one side has zero rows, skip a JSON serializer when the change set is empty.
Watch out: —
Seen in melonDS: `GPU2D::DrawSprites()` now returns the number of sprites it drew, and `DrawScanlineBGMode<>` calls `InterleaveSprites`, a 256-pixel pass per priority level, only when `nsprites` is non-zero. Separately, `CalculateWindowMask` tested `DispCnt & ((1<<15)|(1<<12))`. That rendered the OBJ window whenever either bit was set, and the fix requires both. d2701a04 2019-01 “small optimization: avoid calling InterleaveSprites() if no sprites were drawn.” · 587d4326 2019-05 “don't do OBJ window if it isn't enabled (oops)”

### melonds-044 — SIMD within a register: process packed channels in one integer
Lens: simd · Source: melonDS
Principle: When the fields are narrow and you leave guard bits between them, one 32- or 64-bit multiply or shift updates several fields at once. Mask afterwards to throw away the bits that spilled across. This roughly halves the instruction count with no SIMD intrinsics, and it is portable everywhere. Pair it with an early exit for identity parameters, such as a blend factor of 100%.
Why it is faster: —
Applies elsewhere: the `(c & 0xFF00FF) * a` trick for alpha blending in software rasterizers, bit-sliced checksums, and doing popcount or byte-compare on a whole word in string search (`strlen` "has zero byte" tricks).
Watch out: —
Seen in melonDS: `ColorBrightnessUp/Down` handled red and blue separately. They now share one `rb = val & 0x3F003F` and compute `rb += (((0x3F003F - rb) * factor) >> 4) & 0x3F003F`. `ColorBlend5` returns at once when `eva == 32`. The final 6-bit to 8-bit color expansion in `DrawScanline` now converts two pixels per `u64`, using masks such as `0xFC000000FC0000`. 44e1593e 2019-05 “some optimization to the 2D pipeline” · 858c0eec 2019-05 “scale screen gap also moar optimization...”

### melonds-045 — Copy and fill in machine-word chunks
Lens: memory-layout · Source: melonDS
Principle: A loop that moves one 32-bit element per iteration does half the work it could on a 64-bit machine. Widen the element type, or replicate a fill value into both halves of a 64-bit word. Loop overhead and store count both halve, and compilers vectorize the wider loop more readily.
Why it is faster: —
Applies elsewhere: filling a framebuffer with a solid color, `memset` patterns of more than one byte, clearing tables in embedded code with `uint64_t` stores.
Watch out: —
Seen in melonDS: The backdrop fill now builds `u64 backdrop |= backdrop << 32` and stores `*(u64*)&BGOBJLine[i]` with stride 2. The scanline copy from `mode1gfx` to `dst` also moves 64 bits at a time. 44e1593e 2019-05 “some optimization to the 2D pipeline”

### melonds-046 — Cache a direct pointer for the common unambiguous mapping
Lens: memory-layout · Source: melonDS
Principle: When an address can map to zero, one or several backing stores, the general lookup has to OR together every possible source. Precompute, per page, a raw pointer that is valid only when exactly one store is mapped, and NULL otherwise. Reads take the pointer when it exists and fall back to the slow merge only for the rare overlapping case. Rebuild the pointer whenever the mapping changes.
Why it is faster: —
Applies elsewhere: TLBs, the `mmap`-backed fast path in a paged file reader, a resolved-handler cache in a router that falls back to pattern matching, inline caches in dynamic language VMs.
Watch out: —
Seen in melonDS: `GPU::ReadVRAM_ABG<T>` and its siblings looped over a bank bitmask and ORed every mapped bank. Now `VRAMPtr_ABG[0x20]` and friends hold `GetUniqueBankPtr(mask, offset)`, which returns NULL unless `mask` is a single bit. It finds the bit with a binary search on the mask. The read becomes `if (ptr) return *(T*)&ptr[addr & 0x3FFF];`. The `MAP_RANGE_PTR` macros refresh the pointers on every bank remap and after a savestate is loaded. 434a3007 2019-06 “fasterer BG/OBJ VRAM reads”

### melonds-047 — Store an index, resolve it at the last moment
Lens: laziness · Source: melonDS
Principle: If you resolve a lookup, such as a palette entry or a string intern, early, then you pay for it on every write, even for values that are later overwritten or hidden. Store the compact key with a tag that says how to resolve it, and resolve only the values that survive to the final pass. That pass can also be specialized for the common configuration.
Why it is faster: —
Applies elsewhere: deferred shading, which stores material IDs and shades only visible pixels. Late materialization in column stores. Interning IDs that are only turned back into strings at render time.
Watch out: —
Seen in melonDS: Sprite rasterization wrote `pal[color] | prio` into `OBJLine` for every covered pixel, overdraw included. Now it writes the raw color index plus attribute bits (`0x1000` = standard palette, `0x8000` = direct color). `InterleaveSprites` does the lookup only for pixels that pass the priority and window tests. It has a separate loop for the common no-extended-palette case, and the 16 extended palettes are cached once as a block, not slot by slot. 0010e296 2019-09 “GPU2D: delay palette lookup for sprites”

### melonds-048 — Hoist per-frame work out of per-layer loops
Lens: algorithms · Source: melonDS
Principle: If a transform depends only on the outer loop's state, compute it before the inner loop starts. Otherwise it runs once for every inner iteration, and the result stays the same.
Why it is faster: —
Applies elsewhere: computing a view matrix once per frame rather than per object. Normalizing a query once rather than per row compared. Compiling a regex outside the loop that uses it.
Watch out: —
Seen in melonDS: Horizontal sprite mosaic was applied inside `InterleaveSprites(prio)`, which runs up to four times per scanline, once per BG priority. It moved to `ApplySpriteMosaicX()`, which runs once in `DrawScanline_BGOBJ`. 58c2790e 2019-09 “uuuh we should only do it once per scanline”

### melonds-049 — Update derived tables incrementally
Lens: algorithms · Source: melonDS
Principle: When a big lookup table is derived from N small settings, changing one setting should rewrite only the entries that setting covers. Clearing and rebuilding the whole table costs time in proportion to the table's size, not to the change.
Why it is faster: —
Applies elsewhere: incremental view maintenance in databases, patching just one route in a routing table, re-rendering only the changed rows of a virtual list.
Watch out: —
Seen in melonDS: `ARMv5::UpdatePURegions()` did a `memset` on two 1 MB permission maps (`PU_UserMap`/`PU_PrivMap`) and repainted all 8 protection regions after any CP15 write. The new `UpdatePURegion(n)` repaints one region's range. `UpdatePURegions(bool update_all)` does a full rebuild only when the region layout or the enable bit changes. e117da23 2019-10 “smarter CP15 PU region updates. disable some useless logging.”

### melonds-050 — Swap buffer pointers instead of copying buffers
Lens: memory-layout · Source: melonDS
Principle: When a producer and a consumer share a frame, give each its own buffer and flip an index at the hand-off. A copy costs time in proportion to the frame size, and that grows with resolution. A flip costs O(1). Splitting one buffer into separately sized parts also lets each part scale on its own.
Why it is faster: —
Applies elsewhere: swap chains in graphics APIs, double-buffered audio callbacks, ping-pong buffers in DSP and compute shaders, `std::swap` of vectors in simulation steps.
Watch out: —
Seen in melonDS: The emulator thread used to `memcpy` the whole `GPU::Framebuffer` into the UI's `ScreenBuffer` every frame. Now there are `Framebuffer[2][2]` buffers: two copies, one per screen. `StartHBlank` on the last line toggles `FrontBuffer`, and `AssignFramebuffers()` hands the back buffers to the 2D engines. The UI reads `Framebuffer[FrontBuffer]` directly. The commit message notes "better performance at higher res". In current code, the threaded software 3D renderer also hands off work through semaphores (`Sema_RenderStart`, `Sema_ScanlineCount`), so the 2D compositor can consume 3D scanlines while they are still being rasterized. c88981b5 2019-05 “split framebuffer. setup doublebuffering inside GPU.cpp. avoid copying a whole framebuffer every frame. better performance at higher res.”

### melonds-051 — Read GPU results back asynchronously through a staging buffer
Lens: gpu-rendering · Source: melonDS
Principle: A synchronous readback stalls the CPU until the GPU has finished every queued command. Instead, issue the read into a pixel-pack buffer right after rendering and map it later, when the data is actually needed. By then the transfer has happened in the background. Measure before you split the transfer into chunks: one bulk transfer often beats several small ones.
Why it is faster: —
Applies elsewhere: WebGL2 `readPixels` into a PBO with a fence, staging buffers in Vulkan/D3D12, async GPU timer queries, screen-capture tools that keep N frames in flight.
Watch out: —
Seen in melonDS: `GLRenderer43::GetLine(0)` called `glReadPixels(..., Framebuffer)`, which blocks. `RenderFrame()` now binds `GL_PIXEL_PACK_BUFFER` and calls `glReadPixels(..., NULL)`, and `RequestLine(0)` later does `glMapBuffer` + `memcpy`. The first version used four 48-line strips. The next commit found reading all 192 lines at once faster. A later optional "chunked rendering" path interleaves `RenderSceneChunk(y, 48)` with readback of the previous strip. The commit flags it "might be faster, but not always". 8bc4ca3f 2019-04 “fasterer framebuffer read” · 2d0d501d 2019-04 “probs better this way” · 00341bba 2019-04 “actually, seems faster to read it all at once??” · b74587a4 2019-05 “optional path for chunked rendering (might be faster, but not always)”

### melonds-052 — Keep big data on the GPU; send it a small per-pixel recipe
Lens: gpu-rendering · Source: melonDS
Principle: If the CPU does the final composition, a high-resolution GPU image has to cross the bus every frame. Send the GPU a small instruction buffer instead: which layers apply, which blend mode and factors to use, which 3D texel to sample. A shader then does the composition next to the data. Bus traffic stays at the low CPU-side resolution no matter how far the GPU output is upscaled.
Why it is faster: —
Applies elsewhere: G-buffers and deferred lighting, UI compositors that upload layer descriptors rather than pixels, video players that upload subtitles as a separate plane, map renderers that send style IDs per tile.
Watch out: —
Seen in melonDS: With hardware acceleration on, `GPU2D::DrawScanline` emits a framebuffer row of `256*3 + 1` words. It holds the top 2D layer, the second layer with `ColorComposite()` already applied, and a control word with a compositing mode, EVA/EVB/EVY and the 3D x-coordinate. The final column carries the per-line display mode and master brightness. The screen fragment shader samples the upscaled 3D texture with `texelFetch(_3DTex, ...*u3DScale)` and does the blend itself, so the hi-res 3D image never comes back to the CPU. 70a32437 2019-05 “simpler GPU-compositing. will make it easier to grab neighbor 2D/3D pixels individually for filtering.” · fb4f972c 2019-05 “hires hax. somewhat functional”

### melonds-053 — Submit GPU work early and don't let presentation block the producer
Lens: gpu-rendering · Source: melonDS
Principle: GL drivers can hold commands in a client-side queue. An explicit flush after the frame's commands gets the GPU started while the CPU moves on. Vsync on a context that also paces emulation blocks the producer thread at the display's rate, so turn it off and pace the producer some other way.
Why it is faster: —
Applies elsewhere: calling `glFlush`/`queue.submit` right after recording, triple buffering or mailbox present mode in games, decoupling a video decoder from display refresh.
Watch out: —
Seen in melonDS: `uiGLSetVSync(0)` looks up `WGL_EXT_swap_control` and calls `wglSwapIntervalEXT(0)` on the main window's GL context. Later the Qt frontend's `GLRenderer::RenderFrame()` got a trailing `glFlush()`. f59094e0 2019-06 “OpenGL: disable vsync, atleast under Windows” · 24de8de5 2020-05 “flushing the john makes it a bit better”

### melonds-054 — Do nothing when nothing is happening
Lens: other · Source: melonDS
Principle: Idle work burns CPU and battery and steals time from the threads that matter. Pause device callbacks and redraws while the app is stopped or paused. When there is no frame-rate limit, sleep briefly so an unthrottled loop never spins at 100%.
Why it is faster: —
Applies elsewhere: `requestAnimationFrame` instead of tight loops, suspending `AudioContext` when hidden, stopping sensor polling in background mobile apps, capping menu-screen FPS in games.
Watch out: —
Seen in melonDS: The SDL audio and mic devices now open paused. `Run()` and unpause call `SDL_PauseAudioDevice(..., 0)`, and `Stop()` and pause call it with 1. `GLScreen_DrawScreen` skips the texture uploads and draw call unless `RunningSomething` is set. With the limiter off, the emu thread calls `SDL_Delay(1)` whenever a frame took under 1 ms, which caps it at about 1000 FPS. 64ab8302 2019-01 “pause audio/mic devices when they're not needed.” · 1a3314f0 2019-05 “GL: don't draw the screens when nothing is running” · 3984491c 2019-12 “…cap framerate at 1000FPS”

### melonds-055 — Store power-of-two scale factors as shift counts
Lens: branching-fast-paths · Source: melonDS
Principle: If a scale factor is always 1, 2, 4 and so on, store log2 of it. Indexing becomes `i << s`, buffer sizes become `n << (2*s)`, and the compiler never has to emit a multiply by a runtime value.
Why it is faster: —
Applies elsewhere: mipmap levels, page-size shifts in allocators, ring buffers sized as `1 << order`, texture atlases with power-of-two cells.
Watch out: —
Seen in melonDS: `GPU2D::LineScale` went from a multiplier (1 or 2) to a log2 value (0 or 1). `i*LineScale` became `i << LineScale` in the scanline and capture loops, and `GPU::SetFramebufferScale` sizes buffers as `(256*192) << (scale*2)`. The same commit replaced big per-call stack arrays (`u32 linebuf[1024*2+64]`, `spritebuf[256]`) with member buffers (`BGOBJLine`, `OBJLine`, `WindowMask`), so they are no longer threaded through every call as parameters. 2a0bc4e7 2019-05 “make GPU2D somewhat more flexible. change LineScale to be log2.”

### melonds-056 — Let the toolchain optimize across files, and debug with -Og
Lens: compiler-build · Source: melonDS
Principle: Link-time optimization lets the compiler inline and specialize across translation units. That matters when hot paths call small functions defined in other files. It also adds build-system pitfalls: you need LTO-aware `ar`, and some code miscompiles. Use `-Og` for debug builds, so they stay fast enough to reproduce real-time problems while keeping debuggable code.
Why it is faster: —
Applies elsewhere: `-flto`/ThinLTO in C++ release builds, Rust's `lto = "fat"`, Go's cross-package inlining, bundler tree-shaking and scope hoisting in JS.
Watch out: —
Seen in melonDS: LTO was added to the Codeblocks Release target, then to CMake (`add_compile_options(-O3 -flto)` with `gcc-ar`, plus `add_link_options(-flto)`). It was removed again during the Qt port. Debug builds got `add_compile_options(-Og)`. 3e760a2e 2019-05 “add LTO to Release build under Codeblocks” · 5c5d280d 2020-05 “Add -flto as link flag…” · b341514a 2020-05 “Use -Og for debug builds” · d5f16330 2020-05 “remove LTO”

### melonds-057 — Translate hot code into native blocks, keyed by address
Lens: jit-interpreters · Source: melonDS
Principle: An interpreter pays to fetch, decode and dispatch every instruction, every time it runs. A block JIT decodes a straight-line run once, emits host code for it, and caches the entry point by guest address. After that, each run of the block is a single indirect call. You can start by emitting calls to the existing interpreter handlers and replace them with native code one opcode at a time.
Why it is faster: —
Applies elsewhere: V8/LuaJIT/PyPy tracing and baseline JITs, eBPF JIT, QEMU TCG, regex engines that compile patterns to machine code, database query compilation (HyPer, Umbra).
Watch out: —
Seen in melonDS: `ARMJIT::CompileBlock` fetches up to N instructions until a branch and decodes each with `ARMInstrInfo::Decode`. The x64 `Compiler` is built on Dolphin's emitter. In the first version, every instruction was still interpreted through the emitted code. The block cache was a two-level table: `AddrMapping9[0x2000]` of 32 KB pages, sized to the smallest contiguous ARM9 region, pointing to arrays of entry points, one per halfword. `ARMv5::Execute` looks up a block and either calls it or compiles it. An AArch64 backend sharing the same front end (`ARMJIT_A64/`) arrived in 2020. Current code keeps that design in `ARMJIT.cpp` with `FastBlockLookupRegions`, and adds fastmem through a reserved virtual address space and SIGSEGV/vectored-exception handlers in `ARMJIT_Memory.cpp`. c5c342c0 2019-06 “JIT: base all instructions are interpreted” · 9d180c7b 2019-07 “jit: decrease blockcache AddrMapping size for ARM9” · d6cc7de6 2020-02 “move ARM64 JIT backend here”

### melonds-058 — Allocate registers with lookahead over the block
Lens: jit-interpreters · Source: melonDS
Principle: Map guest registers to host registers for the whole block. Keep a register's value in the host register and write it back only if it was modified. When you have to evict, choose the register used least in the rest of the block. Free registers that are never read again. Don't load a value that the next instruction will overwrite unconditionally.
Why it is faster: —
Applies elsewhere: linear-scan allocation in HotSpot C1 and V8, write-back caches with dirty bits, "dead store" and "dead load" elimination in any compiler backend.
Watch out: —
Seen in melonDS: `RegCache::Prepare(i)` counts future uses of each register (`ranking[reg]++` over `Instrs[j..]`). It unloads `LoadedRegs & ~futureNeeded`, evicts the register with the lowest rank, and writes back only `DirtyRegs`. Later, `LoadRegister(reg, loadValue)` skips the load for destination-only registers of unconditional instructions. It also preloads `NotStrictlyNeeded` registers when host registers are free, and that helps LDM/STM. ebce9f03 2019-06 “JIT: implemented most ALU instructions” · 5338c28f 2019-09 “load register only if needed…” · 81f38c14 2019-10 “…better handle LDM/STM in reg alloc”

### melonds-059 — Eliminate side effects nobody reads (dead flag elimination)
Lens: jit-interpreters · Source: melonDS
Principle: If an operation updates state that the next consumer overwrites before reading it, the update is dead. A backwards pass over the block can mark which updates are actually observed, and the code generator emits only those. Condition flags in instruction sets with flag-setting arithmetic are the classic case.
Why it is faster: —
Applies elsewhere: lazy flag evaluation in x86 emulators (Bochs, QEMU), liveness analysis for dead-store elimination, skipping recomputation of derived fields that are about to be reset.
Watch out: —
Seen in melonDS: `FloodFillSetFlags(instrs, start, flags)` walks backwards from each flag reader, and from the block end, where all flags are live. It marks only the writers whose N/Z/C/V results survive in `instrs[j].SetFlags`. The emitters then check `FlagsNZRequired()` and `SetFlags & 0x2` before emitting `TEST`/`SETcc` and CPSR merging. The commit notes this is "especially useful for thumb code", where almost every ALU op sets flags. 5ea91b8a 2019-08 “optimise away unneeded flag sets…” · 2ef77688 2019-09 “more fixes for flag optimisation + small cycle counting optimisation”

### melonds-060 — Fold compile-time-known costs into one constant
Lens: jit-interpreters · Source: melonDS
Principle: If an accounting value, such as cycles or a gas or budget counter, is the same every time a block runs, sum it at compile time and apply it in one instruction at exit. Emit runtime updates only for the parts that vary: conditional instructions and memory waits.
Why it is faster: —
Applies elsewhere: Ethereum clients metering gas per basic block, profilers that count per basic block rather than per instruction, rate limiters that charge per batch.
Watch out: —
Seen in melonDS: `Comp_AddCycles_C/CD/CDI` add to `ConstantCycles` when the instruction is unconditional and to the runtime counter otherwise. Conditional-skip paths leave out the fix-up jump unless the instruction had `IrregularCycles`. The block epilogue then does a single `MOV EAX, ConstantCycles`. After block linking this became `SUB [Cycles], ConstantCycles`. 2ef77688 2019-09 “…small cycle counting optimisation” · a687be98 2019-10 “new block cache and much more… load/store cycle counting”

### melonds-061 — Grow blocks across predictable branches (superblocks)
Lens: jit-interpreters · Source: melonDS · Also: ares
Principle: Ending a compiled block at every branch keeps blocks tiny, so dispatch overhead dominates and the optimizer sees little context. Follow unconditional and static branches, and calls and returns whose target is known, while compiling. Turn conditional branches into side exits. Larger regions give register allocation and flag elimination more room to work.
Why it is faster: —
Applies elsewhere: trace compilation in LuaJIT/TraceMonkey, superblock formation in Dynamo and HP compilers, function inlining across call sites in any optimizing compiler.
Watch out: —
Seen in melonDS: While building a block, `CompileBlock` interprets each instruction. When `DecodeBranch` finds a static target that was actually taken, it continues fetching from that target (`branch_FollowCondTaken`). A not-taken conditional branch becomes an early exit (`branch_FollowCondNotTaken`). Later, BL records `lr`, so a following `BX LR` resolves statically and the block continues past the return. a687be98 2019-10 “…blocks aren't stopped at any branch, but are being followed if possible to get larger blocks” · 81f38c14 2019-10 “…jit blocks follow simple returns” • Also seen in ares (ares-127): A first attempt added lazily backpatched links between blocks within a section, guarded by dirty, budget and queue checks. A month later blocks were extended through branches: a planning prepass builds the instruction window, and `emitAliasAddresses` publishes the internal targets. The linking code was then removed as "not useful anymore". 70f1cf07c 2026-04 “n64: add safe intra-section JIT block linking” · 49af8ab6a 2026-05 “n64: in CPU JIT, blocks now extend through branches”

### melonds-062 — Detect busy-wait loops and skip ahead in time
Lens: jit-interpreters · Source: melonDS
Principle: A loop that only reads memory and registers, with no stores and no loop-carried state, cannot make progress until something external changes. Once you prove that, stop executing it and jump the clock to the next scheduled event. The guest wastes no host time spinning.
Why it is faster: —
Applies elsewhere: Dolphin's idle skipping, `PAUSE`/`WFE` in spin locks, futexes replacing spin-waits, Node's event loop blocking in `epoll_wait` instead of polling.
Watch out: —
Seen in melonDS: `IsIdleLoop()`, adapted from Dolphin's PPCAnalyst, rejects loops that contain stores or inner branches, or that write a register read earlier in the same iteration. For a backward conditional branch to an idle loop, the compiler emits `OR [RCPU+IdleLoop], 1`. `ARMv5/ARMv4::Execute` then treats that like `Halted` and sets `ARM9Timestamp = ARM9Target`. A later fix changed the loop range to `&instrs[i - backwardsOffset]`. a687be98 2019-10 “…idle loop recognition” · 81f38c14 2019-10 “…fix idle loop detection”

### melonds-063 — Constant-fold loads from read-mostly data, with invalidation
Lens: jit-interpreters · Source: melonDS
Principle: If the code loads from an address known at compile time, such as a PC-relative literal pool, and that data is almost never written, bake the value in as an immediate. Track which constants were folded. When a store hits one of them, mark that region so future compiles do a real load, and invalidate the blocks that depended on the old value.
Why it is faster: —
Applies elsewhere: V8 constant-field tracking with deopt on write, Java `static final` folding, link-time constant propagation, CDNs that inline config values and purge them on change.
Watch out: —
Seen in melonDS: `Comp_MemAccess` with `rn == 15` and an immediate offset calls `Comp_MemLoadLiteral`. That function reads the value at compile time, emits `MOV rd, imm`, and records it via `RegCache.PutLiteral`, so later address computations become static too. `DecodeLiteral` detects PC-relative stores during block building. It sets bits in `CodeRanges[addr/512].InvalidLiterals`, one per 16-byte cell, and those cells fall back to real loads. DTCM is excluded, and a GUI toggle can turn the whole optimization off. a687be98 2019-10 “…optimised literal loads…and loads/stores from constant addresses” · 386100c0 2019-11 “make literal optimisation more reliable” · 60650fa8 2019-12 “disable literal optimations in DTCM”

### melonds-064 — Bind the handler at compile time when the operand is constant
Lens: jit-interpreters · Source: melonDS
Principle: A generic accessor has to decode an address into a device or region at run time. When the address is known at compile time, resolve the decode during compilation and emit a direct call to the specific handler, or a direct memory operand. The switch disappears from the hot path.
Why it is faster: —
Applies elsewhere: devirtualization and inline caching, resolving a route at build time in static site generators, binding SQL prepared statements to a specific index.
Watch out: —
Seen in melonDS: `GetFuncForAddr(cpu, addr, store, size)` returns `GPU3D::Write32` for 3D registers, `NDSCart::ReadROMData` for the cart data port and so on, instead of the generic `ARM9IORead32`. Static MainRAM addresses compile to `M(NDS::MainRAM + (addr & mask))`. a687be98 2019-10 “new block cache and much more…” · 81f38c14 2019-10 “integrate changes from ARM64 backend and more” · 0f53a345 2020-05 “rewrite JIT memory emulation”

### melonds-065 — Speculate on the region observed at first execution, with a cheap guard
Lens: jit-interpreters · Source: melonDS
Principle: Most memory instructions hit the same kind of memory every time. Record the region during the first, interpreted, run. Compile an inline fast path for that region behind a one-byte guard from a per-page classification table. Branch to the generic path only when the guard fails.
Why it is faster: —
Applies elsewhere: monomorphic inline caches in JS engines, branch-on-type guards in tracing JITs, a fast path for ASCII with a fallback for UTF-8, hot/cold path splitting with `[[likely]]`.
Watch out: —
Seen in melonDS: `CompileBlock` runs each instruction through the interpreter and saves `cpu->DataRegion`. `Comp_MemAccess` classifies it with `ClassifyAddress9/7`. For MainRAM, DTCM, WRAM and similar regions it emits `MOVZX tmp, byte [MemoryStatus9 + (addr>>9)]; CMP tmp, expectedTarget; JNE slowPath`, then a direct masked access. `MemoryStatus9/7` stores the region kind per 512-byte page, with bit 7 set when the page holds compiled code, so a store to a code page fails the guard and takes the invalidating slow path. 0f53a345 2020-05 “rewrite JIT memory emulation”

### melonds-066 — Track code pages at fine granularity for cheap self-modifying-code checks
Lens: jit-interpreters · Source: melonDS · Also: ares
Principle: To detect writes to compiled code without slowing every store, keep a compact per-page record of the blocks that cover each page. A store only has to test whether that page's block list is non-empty. Pick a page size small enough that writes to nearby data don't keep throwing away unrelated code. Translate mirrored addresses to one canonical "pseudo-physical" space so that each byte has a single entry.
Why it is faster: —
Applies elsewhere: write-protecting JIT pages (W^X) and catching the fault, card marking in generational GCs, dirty-page bitmaps in hypervisor live migration.
Watch out: —
Seen in melonDS: The block cache stores `CodeRanges[addr/256]`, each with a `TinyVector` of blocks and a `TimesInvalidated` counter. Generated store code checks `CMP word [CodeRanges + ...Blocks.Length], 0`. The field was narrowed to 16 bits so the compare works directly on memory. `AddrTranslate9/7` collapse mirrors into one pseudo-physical space. Granularity was tightened to fix Dragon Quest IX, where data shared code pages. a687be98 2019-10 “new block cache…more reliable code invalidation detection” · aa23f21b 2019-10 “decrease jit block cache address granularity” · 386100c0 2019-11 “make literal optimisation more reliable” • Also seen in ares (ares-126): Each 4 KiB section now carries `u8 lineBlocks[128]`, one entry per 32-byte line. `invalidateRange` walks only the lines a write touches, and marks the section dirty only if one of them holds code. Stores call `invalidateRange(address, Size)` once, where they used to make two separate calls. 68134837d 2026-05 “n64: avoid excessive JIT trashing by reducing the 4K section granularity”

### melonds-067 — Keep invalidated compiled code in a content-addressed side cache
Lens: jit-interpreters · Source: melonDS
Principle: Code that gets overwritten is often reloaded with identical bytes: overlays, DLL reloads, hot-reload. When you invalidate, keep the compiled artifact under a hash of its source. On recompile, hash the new source, and if the hash and address match, reuse the old artifact and skip compilation.
Why it is faster: —
Applies elsewhere: ccache and Bazel's content-addressed action cache, V8 code caching, webpack persistent caches keyed by content hash, shader caches keyed by SPIR-V hash.
Watch out: —
Seen in melonDS: `InvalidateByAddr` moves blocks into `RestoreCandidates` rather than deleting them. `CompileBlock` compares the instruction words and address ranges, and if they match it re-registers the old `EntryPoint`. Later commits keyed the table by `XXH3_64bits` of the instruction words, with a `LiteralHash` check, so blocks that come back are found regardless of which slot their address hashes to. The commit says this "makes Golden Sun burn a little slower through the JIT memory". Current `ARMJIT.cpp` still keys `RestoreCandidates` by `InstrHash`. a687be98 2019-10 “new block cache and much more…” · 3787bab1 2020-04 “implement block linking + some refactoring” · d91bbec0 2020-05 “use instr hash as key for restore candidates…”

### melonds-068 — Chain compiled blocks with patchable direct jumps
Lens: jit-interpreters · Source: melonDS
Principle: Returning to a dispatcher after every block costs a call, a lookup and an indirect branch. Leave a NOP slot at each block exit. The first time the exit runs, a stub looks up or compiles the successor and patches the slot into a direct `JMP`. Record each patch site so that invalidating the target can turn it back into NOPs.
Why it is faster: —
Applies elsewhere: QEMU TB chaining, PLT/GOT lazy binding in dynamic linkers, Dalvik/ART trace linking, inline cache patching in Smalltalk and JS VMs.
Watch out: —
Seen in melonDS: When the next target is static, block exits emit `J_CC(CC_S)` and `CMP [StopExecution],0`, then `NOP(5)` followed by a jump to `BranchStub[Num]`. The stub calls `ARMJIT::LinkBlock<Num>`, which compiles the target if needed, records `AddLink(codeOffset)`, and calls `Compiler::LinkBlock` to overwrite the NOPs with `JMP entry`. `UnlinkBlock` writes `NOP(5)` back. Return paths go through `ARM_Ret` in `ARMJIT_Linkage.s`. The feature is enabled with `JIT_BrancheOptimisations == 2`. 3787bab1 2020-04 “implement block linking + some refactoring currently only supported for x64”

### melonds-069 — Count budgets down so the sign flag is your exit test
Lens: branching-fast-paths · Source: melonDS
Principle: If a budget counts up toward a limit, you need a compare against the limit. Load the budget and subtract from it instead. The subtraction sets the sign flag when the budget runs out, so the exit check costs only a conditional jump on that flag.
Why it is faster: —
Applies elsewhere: `dec/jnz` loop counters, interpreter "ticks until yield" in Lua and Erlang reductions, token-bucket rate limiters with a decrementing balance.
Watch out: —
Seen in melonDS: Every `ADD [RCPU+Cycles], n` in the x64 JIT became `SUB [RCPU+Cycles], n`. `Execute` seeds `Cycles` with the time remaining before the target, so a linked block exit does `SUB ..., ConstantCycles; J_CC(CC_S) → return`, with no separate compare. 3787bab1 2020-04 “implement block linking + some refactoring”

### melonds-070 — Put a small lossy cache in front of an authoritative map
Lens: lookup-tables · Source: melonDS
Principle: A flat array indexed by every possible key is fast but huge. A general hash map is compact but slower. A fixed-size two-way set-associative table with a fast hash, which evicts silently, catches almost all lookups. A miss falls back to the authoritative map. Because the table is lossy, it never has to resize or chain.
Why it is faster: —
Applies elsewhere: CPU caches and TLBs, Go's per-P mcache, method caches in Objective-C `objc_msgSend`, DNS stub resolvers in front of a recursive resolver.
Watch out: —
Seen in melonDS: The old `FastBlockAccess[ExeMemSpaceSize/2]` array, one entry per halfword of every executable region, became `UnreliableHashTable<u32, JitBlockEntry, 0x800, nullptr>`. Its `Bucket` holds `{KeyA, KeyB, ValA, ValB}`, and slots are picked by `XXH3_64bits(&key,4) & (Size-1)`. The authoritative store is `std::unordered_map<u32, JitBlock*> JitBlocks`. The same structure backs `RestoreCandidates`. 3787bab1 2020-04 “implement block linking + some refactoring”

### melonds-071 — Use caller-saved registers too, spilling only around calls
Lens: jit-interpreters · Source: melonDS
Principle: Restricting allocation to callee-saved registers keeps calls cheap but leaves only a few registers for values. Allocate caller-saved registers as well, and save only the live ones around the occasional out-of-line call. Straight-line code keeps more values in registers, and the spill cost lands only on the slow paths.
Why it is faster: —
Applies elsewhere: standard register allocation in GCC/LLVM (call-clobbered vs. call-preserved), V8's register allocator spilling around runtime calls, hand-written assembly that saves only live scratch registers in hot loops.
Watch out: —
Seen in melonDS: `NativeRegAllocOrder` grew from `RBX, R12, R13, R14` (commented "this is sad") to include `R9, R10, R11`, so seven registers are available. New `PushRegs`/`PopRegs` save only the loaded guest registers whose host register is in `ABI_ALL_CALLER_SAVED`. They wrap memory slow paths and interpreter fallbacks. 5a0b5686 2020-05 “allow allocating caller saved registers currently system-v only”

### melonds-072 — Fuse instruction pairs into one operation
Lens: jit-interpreters · Source: melonDS
Principle: When an ISA splits one logical operation across two instructions and they almost always appear together, recognize the pair at decode time. Emit one combined operation. That removes an intermediate write-back and a dispatch, and lets the compiler see the full constant.
Why it is faster: —
Applies elsewhere: macro-op fusion in x86 and RISC-V cores, `LUI`+`ADDI` combining in RISC-V assemblers, peephole fusion of compare and branch in JITs, SQL engines fusing filter and project.
Watch out: —
Seen in melonDS: The two halves of Thumb's 32-bit BL (`tk_BL_LONG_1` + `tk_BL_LONG_2`) are merged in `CompileBlock` into one `tk_BL_LONG`. The second halfword goes into the upper 16 bits of `Instr`, and `T_Comp_BL_Merged()` computes the full target as a constant. 5338c28f 2019-09 “…do thumb bl long merge in the first step…”

### melonds-073 — Share generated thunks and tail-call into slow handlers
Lens: jit-interpreters · Source: melonDS
Principle: Inlining the complete slow path at every memory access bloats generated code and pollutes the instruction cache. Generate one thunk per (size, load/store) pair. Handle the fast cases inline inside it, and `JMP` straight into the C handler for the rest, so the handler returns directly to the original caller. Use `CMOV` for small selections such as max(a, b) instead of branches.
Why it is faster: —
Applies elsewhere: outlined runtime stubs in V8 and HotSpot, `__memcpy_chk` style trampolines, tail-call-optimized interpreters (`musttail`), branchless min/max in sorting networks.
Watch out: —
Seen in melonDS: `Gen_MemoryRoutine9(store, size)` emits a shared routine. It checks DTCM and ITCM with `SUB`/`CMP`/`J_CC(CC_B)` and handles them inline. Other addresses get a cycle cost computed with two `CMOVcc` and then `JMP NDS::ARM9Write32`, a tail call. A comment in the file says about 99% of accesses hit the same region every time, and the later speculative fast path builds on that observation. 2c44bf92 2019-07 “JIT: most mem instructions working + branching”

### melonds-074 — Close the gaps that force fallbacks to the slow path
Lens: jit-interpreters · Source: melonDS
Principle: In a hybrid system, every operation the fast engine can't handle costs a full context switch: flush state, call the slow path, reload. That cost far exceeds the operation itself. Covering the remaining common operations natively removes those transitions from hot loops.
Why it is faster: —
Applies elsewhere: adding missing intrinsics so a JIT doesn't deoptimize, handling a missing SQL function in a vectorized engine rather than falling back to row mode, WebAssembly SIMD covering ops that would otherwise call JS.
Watch out: —
Seen in melonDS: `A_Comp_MRS`/`A_Comp_MSR` now emit native code, operating on `RCPSR` and calling `ReadBanked`/`WriteBanked` only for SPSR. Before, each instruction meant a register flush plus an interpreter call. `A_Comp_Mul_Long` extended SMULL/SMLAL to UMULL/UMLAL, using zero-extension and a single `BSR` for the timing calculation. a9dd6e30 2020-04 “implement msr and mrs for the x64 JIT” · 68d55207 2020-04 “compile UMULLs and some fixes”

### melonds-075 — Don't maintain state nobody observes; reconstruct it on demand
Lens: laziness · Source: melonDS
Principle: A fast engine doesn't have to mirror every bookkeeping field of the reference model. Drop the fields that are only needed at rare boundaries, such as snapshots or mode switches, and rebuild them when one of those happens. The hot path stops paying for them.
Why it is faster: —
Applies elsewhere: JITs that rebuild interpreter frames only on deopt, databases that rebuild secondary state at checkpoint, and React reconstructing component stacks only when an error occurs.
Watch out: —
Seen in melonDS: The JIT stopped maintaining the two-entry prefetch pipeline (`NextInstr[0..1]`), so `Comp_JumpTo` no longer stores fetched opcodes. `ARMv5::FillPipeline()`/`ARMv4::FillPipeline()` rebuild it from memory when a JIT savestate is loaded, so the state stays usable by the interpreter. The commit also fixed Golden Sun: Dawn. 3001d949 2019-08 “abandon pipelining on jit fixes Golden Sun Dawn…the pipeline is manually filled when making a save state.”

### melonds-076 — Specialize hot functions on a small compile-time parameter
Lens: compiler-build · Source: melonDS
Principle: If a function branches on a parameter with only a few values, such as which CPU, which mode or which pixel format, make it a template or generic parameter. Each instantiation then folds its branches away and can use different constants, such as table sizes or shift amounts.
Why it is faster: —
Applies elsewhere: C++ templates on pixel formats in image codecs, Rust const generics, Go generics with type-specific instantiation, shader permutations.
Watch out: —
Seen in melonDS: `IsMapped(num, addr)`, `LookUpBlock(num, addr)` and `Invalidate16/32` became `template <u32 num>`. That let the ARM9 use 32 KB pages (`>> 15`, `AddrMapping9[0x2000]`) while the ARM7 keeps 16 KB pages (`>> 14`), with no runtime branch. The pattern continues in `TranslateAddr<Num>` and `LinkBlock<Num>`. 9d180c7b 2019-07 “jit: decrease blockcache AddrMapping size for ARM9”

### melonds-077 — Keep hot globals in the low 2 GB so generated code can address them directly
Lens: platform · Source: melonDS
Principle: On x86-64, a memory operand carries only a signed 32-bit displacement. If your tables live below 2 GB, which the default small code model makes likely, generated code can write `[reg + table]` as one instruction. Otherwise it first has to load a 64-bit address into a register. Assert the assumption so that a layout change fails loudly.
Why it is faster: —
Applies elsewhere: `-mcmodel=small` vs. `large`, `MAP_32BIT` for JIT code caches, LuaJIT's low-memory allocation for GC objects, compressed pointers in the JVM and V8.
Watch out: —
Seen in melonDS: `squeezePointer(T* ptr)` truncates a pointer to `int` and asserts that it round-trips. It is used for `MDisp(RSCRATCH, squeezePointer(CodeRanges))`, `MemoryStatus9`, `NDS::MainRAM` and `cpu9->CodeMem.Mem`, so these become single-instruction memory operands in emitted code. 3001d949 2019-08 “abandon pipelining on jit…” · 0f53a345 2020-05 “rewrite JIT memory emulation”

### melonds-078 — Translate hot interpreted code into native code at runtime
Lens: jit-interpreters · Source: melonDS
Principle: An interpreter pays for fetch, decode and dispatch on every instruction it runs. A just-in-time compiler pays that cost once per block of guest code, emits host machine code, caches it under the guest address, and later jumps straight to it. Hot loops then run at close to native speed, and the compiler can optimize across instructions because it sees the whole block at once.
Why it is faster: —
Applies elsewhere: JavaScript engines (V8, SpiderMonkey), the JVM's HotSpot, LuaJIT, eBPF in the Linux kernel, regex engines that compile patterns to machine code (PCRE-JIT), and query engines that compile SQL expressions (LLVM in Postgres).
Watch out: —
Seen in melonDS: The JIT branch began with every instruction still calling its interpreter handler from a block-shaped dispatcher, plus a `BlockCache` keyed by guest address. That gave a skeleton to which native emitters (borrowed from Dolphin's x64 emitter) could be added one instruction class at a time. The rewrite in `40b88ab0` introduced `JitBlockEntry LookUpBlock<cpu>(addr)`: `ARMv5::ExecuteJIT` now looks up the block, calls it if found and compiles it otherwise, then adds the returned cycle count to the timestamp. An ARM64 backend landed in early 2020. Today `ARMJIT.cpp` drives both `ARMJIT_x64` and `ARMJIT_A64`. c692287e 2019-06 “JIT: base all instructions are interpreted” · 40b88ab0 2019-10 “new block cache and much more...”

### melonds-079 — Eliminate dead side-effect computation with a backward liveness pass
Lens: jit-interpreters · Source: melonDS
Principle: Many operations produce secondary results, such as status flags or carries, that later code overwrites before anything reads them. A single backward pass over a block can track which results are still "live" (read before being overwritten) and mark everything else as dead. The code generator then skips the instructions that would have computed and stored the dead values.
Why it is faster: —
Applies elsewhere: dead-store elimination in compilers, skipping unused columns in a query plan (projection pushdown), tree-shaking unused exports in JS bundlers, not computing derived fields nobody subscribes to in a reactive UI.
Watch out: —
Seen in melonDS: ARM's `S`-suffixed instructions (and almost all Thumb ALU ops) set NZCV, and the next flag-setting instruction usually clobbers them. `floodFillSetFlags(instrs, start, flags)` walks backward from each flag reader, setting `FetchedInstr::SetFlags` only on the instruction that actually provides each flag. The emitters then check `FlagsNZRequired()` and `CurInstr.SetFlags & 0x2` before emitting `TEST`/`SETcc`/`LEA` flag packing. The follow-up added finer flag kinds (`A_StaticShiftSetC`, `A_SetC`) so the pass knows exactly which shifter forms write carry, and later the immediate-operand carry update was made conditional on the same mask. f378458c 2019-08 “optimise away unneeded flag sets...” · 85680d6f 2019-09 “more fixes for flag optimisation + small cycle counting optimisation” · 887ad27e 2020-07 “implement carry setting ALU op with imm”

### melonds-080 — Don't load a value that is about to be fully overwritten
Lens: jit-interpreters · Source: melonDS
Principle: When a cache or register allocator brings a variable into fast storage, it normally copies the old value in. If the next use is a pure write, that load is wasted memory traffic. Allocate the slot without filling it, and load only the values that are actually read.
Why it is faster: —
Applies elsewhere: write-allocate without fetch (`DC ZVA`, non-temporal stores), `O_TRUNC` instead of read-then-rewrite, `glInvalidateFramebuffer`/`LOAD_OP_DONT_CARE` on tiled GPUs, `std::vector::resize` without value-init.
Watch out: —
Seen in melonDS: `RegisterCache::LoadRegister(reg, loadValue)` now takes a flag. In `Prepare`, unconditional instructions (Thumb, or ARM `cond >= 0xE`) load only `Instr.Info.SrcRegs`; registers that are only destinations just get a host register mapped. Conditional ARM instructions still load everything, because a skipped instruction must leave the old value intact. 0e26aa4e 2019-09 “load register only if needed - do thumb bl long merge in the first step...”

### melonds-081 — Fuse adjacent operations into one at decode time
Lens: algorithms · Source: melonDS
Principle: Some logical operations are split across two encoded units only because the encoding is too narrow, for example a long branch built from two half-instructions. Recognizing the pair during decoding and replacing it with one synthetic operation removes the intermediate state between them, and the code generator handles a single simple case.
Why it is faster: —
Applies elsewhere: macro-op fusion in CPUs (cmp+jcc), combining `LUI`+`ADDI` into one constant load, merging adjacent small writes into one syscall (writev), and SQL engines fusing filter+project into a single operator.
Watch out: —
Seen in melonDS: Thumb `BL` is two 16-bit instructions (`tk_BL_LONG_1`/`_2`). `CompileBlock` now merges them while fetching. It rewrites the first as pseudo-op `tk_BL_LONG`, packs both halves into one 32-bit `Instr`, sets `DstRegs = 0xC000` (LR, PC) and steps back one slot. `T_Comp_BL_Merged` computes the full target from `CurInstr.Instr >> 16`, and an interpreter fallback `T_BL_LONG` was added in `9cf7780e`. 0e26aa4e 2019-09 “...do thumb bl long merge in the first step...” · 9cf7780e 2019-10 “decrease jit block cache address granularity...”

### melonds-082 — Fold statically known costs into one update per block
Lens: jit-interpreters · Source: melonDS
Principle: If a loop or block adds a constant to a counter at every step, add up the constants at compile time and emit a single add at the exit, or no add at all when the sum is zero. Only the parts that depend on runtime data need per-step code.
Why it is faster: —
Applies elsewhere: batching metrics increments into a thread-local and flushing once, Go's and Lua's instruction-count hooks amortized per basic block, gas metering in WASM/EVM runtimes charged per block, cost accounting in rate limiters.
Watch out: —
Seen in melonDS: The compiler keeps `ConstantCycles` and adds the fixed timing of each unconditional instruction to it at compile time. `Comp_AddCycles_CI` emits a runtime `ADD` only for the variable part, and a per-instruction `IrregularCycles` flag lets conditional instructions skip the extra "failed condition" jump when their timing is regular. The block epilogue later stopped emitting `ADD Cycles, 0`: `if (ConstantCycles) ADD(...)`. 85680d6f 2019-09 “...small cycle counting optimisation” · 9ee20174 2021-07 “don't add constant cycles if they're 0”

### melonds-083 — Grow compilation units across predictable control flow
Lens: jit-interpreters · Source: melonDS
Principle: Ending every compiled block at the first branch keeps blocks small, so each block exit costs a dispatcher round trip and cross-block optimization is lost. If a branch target is static, keep compiling at the target, as a superblock or trace. For a conditional branch, compile the path observed at compile time and emit a side exit for the other one.
Why it is faster: —
Applies elsewhere: trace-based JITs (LuaJIT, TraceMonkey), profile-guided basic-block layout in LLVM/BOLT, inlining small functions, and hot-path splitting in web servers' request routers.
Watch out: —
Seen in melonDS: In `40b88ab0`, `CompileBlock` decodes each branch with `DecodeBranch`. If the branch is static and was taken during the compile-time pre-execution, it continues fetching at the target and tags `branch_FollowCondTaken`. If it was not taken, it tags `branch_FollowCondNotTaken`. `Comp_SpecialBranchBehaviour` emits the side exit, and flag liveness treats those branches as reading all flags. `441869a1` also follows simple returns. SVC was excluded later because of its bank switch. 40b88ab0 2019-10 “...blocks aren't stopped at any branch, but are being followed if possible...” · 441869a1 2019-10 “...jit blocks follow simple returns...” · b28a9e4d 2021-08 “JIT: don't lengthen blocks on a skipped SVC”

### melonds-084 — Detect busy-wait loops and skip ahead to the next event
Lens: algorithms · Source: melonDS
Principle: A loop that only polls state it doesn't modify (it writes no memory, and no iteration depends on another) can't make progress until something external changes. Prove that with a dataflow check on the loop body, then replace the spinning with a jump straight to the next scheduled event. The time is skipped in one step instead of being simulated.
Why it is faster: —
Applies elsewhere: the OS turning spin-waits into futex sleeps, `PAUSE`/`WFE` hints, event loops that sleep until the next timer instead of polling, hypervisors detecting guest HLT/PAUSE loops (PLE), and test frameworks with virtual clocks that fast-forward idle time.
Watch out: —
Seen in melonDS: `IsIdleLoop(instrs, count)` (after Dolphin's PPCAnalyst) rejects any memory write or inner branch. It tracks `regsWrittenTo` and `regsDisallowedToWrite` and fails if an instruction writes a register that an earlier instruction read before it was written. A backward conditional branch over such a body gets `branch_IdleBranch`, and the emitted code sets `cpu->IdleLoop`. `ExecuteJIT` sees `StopExecution` and sets `ARM9Timestamp = ARM9Target`, jumping straight to the next scheduler event. `441869a1` unified Halted/IRQ/IdleLoop behind one `StopExecution` check. 40b88ab0 2019-10 “...idle loop recognition...” · 441869a1 2019-10 “...unify Halted and IRQ... fix idle loop detection”

### melonds-085 — Bake read-only data into generated code as immediates, with invalidation
Lens: jit-interpreters · Source: melonDS
Principle: When compiled code loads a value from memory that almost never changes, such as a constant pool next to the code, read it at compile time and emit it as an immediate. The load and its latency are gone, and the constant can feed further folding. To stay correct, record which addresses were baked in and recompile when any of them is written.
Why it is faster: —
Applies elsewhere: V8 constant-folding a `const` global with a dependency that deoptimizes on change, inline caches, C++ `constexpr`/link-time constant propagation, and CDN edge-side includes baked into static HTML with purge-on-update.
Watch out: —
Seen in melonDS: `Comp_MemLoadLiteral` handles `LDR rd, [pc, #imm]` by doing the read at compile time and emitting `MOV rd, Imm32(val)`. For unconditional instructions it also records the value in `RegCache.PutLiteral`, so later code knows `rd` is a constant. `3e748363` made this reliable: `DecodeLiteral` recognizes PC-relative stores, and their targets go into `InvalidLiterals` so such addresses are never folded. `ea6d0358` re-enabled it after the fastmem refactor and added sign-extended loads. 40b88ab0 2019-10 “...optimised literal loads...” · 3e748363 2019-11 “make literal optimisation more reliable...” · ea6d0358 2020-06 “make literal optimisation work again...”

### melonds-086 — Propagate constants into addresses to resolve dispatch at compile time
Lens: jit-interpreters · Source: melonDS
Principle: A generic memory access must classify the address at runtime (RAM, I/O, unmapped) before calling the right handler. If constant propagation shows the address is fixed, do the classification once at compile time and emit a direct access or a direct call to the specific handler. The runtime decode tree disappears.
Why it is faster: —
Applies elsewhere: devirtualization when the receiver type is known, compile-time route resolution in web frameworks, linkers resolving PLT calls to direct calls (`-fno-plt`, LTO), and memory-mapped register access through fixed pointers in embedded HALs.
Watch out: —
Seen in melonDS: In the rewritten `Comp_MemAccess`, `addrIsStatic` holds when `RegCache.IsLiteral(rn)` and the offset is an immediate without writeback. `staticAddress = LiteralValues[rn] + imm` is then passed through `ClassifyAddress9/7`. Main RAM/DTCM/WRAM get an inline fast path, and known I/O addresses call the specific I/O handler directly (the `PushRegs` placement fix in `fea9f95b`). 052ff736 2020-05 “rewrite JIT memory emulation” · fea9f95b 2020-05 “fix inlined IO register access”

### melonds-087 — Specialize code on the case observed at compile time
Lens: jit-interpreters · Source: melonDS
Principle: The first execution of a code path is a cheap profile. Record which branch of a polymorphic operation was taken (which memory region, which type), generate code specialized for that case, and keep a generic fallback for the others. Most code is monomorphic in practice, so the common path loses its runtime checks.
Why it is faster: —
Applies elsewhere: inline caches and type feedback in JS/Python JITs, PGO builds (`-fprofile-use`), adaptive query execution in Spark, and branch-likely hints from production profiles (AutoFDO).
Watch out: —
Seen in melonDS: The interpreter records `DataRegion` on every load and store. The compiler pre-executes each instruction while building the block, so `CurInstr.DataRegion` tells it which region the access really hit. It uses that both to pick constant wait-state cycle counts (`Comp_AddCycles_CDI`) and, in `052ff736`, as `expectedTarget` for an inline fast path. `1ad90cb3` widened the field from `addr >> 24` to `addr >> 20` so the compiler can tell sub-regions such as ITCM/DTCM apart. 40b88ab0 2019-10 “...load/store cycle counting...” · 1ad90cb3 2020-04 “include more information in DataRegion” · 052ff736 2020-05 “rewrite JIT memory emulation”

### melonds-088 — Let the MMU do address translation, and back-patch on fault
Lens: memory-layout · Source: melonDS · Also: Flycast
Principle: Reserve a large virtual address range and map the real backing memory into it with the same layout as the emulated or remote address space, mirrors included. Generated code then turns every access into one host load at `base + addr`. Accesses that hit unmapped holes, such as I/O, raise a fault. The handler rewrites that one call site into a slow-path call and resumes, so only the rare cases pay.
Why it is faster: —
Applies elsewhere: WebAssembly runtimes using 4/8 GB guard regions to drop bounds checks, Dolphin/Citra fastmem, LMDB and databases using `mmap` instead of explicit reads, sanitizer shadow memory, and userfaultfd-based lazy VM restore.
Watch out: —
Seen in melonDS: `ARMJIT_Memory.cpp` maps one shared memory object (memfd/ashmem on Linux and Android, `CreateFileMapping`+`MapViewOfFileEx` on Windows, `svcMapProcessMemory` on Switch) into two 4 GB windows, `FastMem9Start`/`FastMem7Start`. Only regions in use are mapped, and per-page state lives in `MappingStatus9/7`. Each fast access records a `LoadStorePatch` {offset, size, slow function}. On SIGSEGV or a vectored exception, `RewriteMemAccess(pc)` finds the patch, overwrites the site with `CALL patch.PatchFunc` plus NOP padding, and resumes. e335a8ca 2020-06 “first steps in bringing over the JIT refactor/fastmem” · c5381d29 2020-06 “reconcile DSi and JIT, fastmem for x64 and Windows” • Also seen in Flycast (flycast-207): `initMappings` maps one shared-memory file at several offsets, including VRAM at `0x04000000` and `0x06000000` and main RAM at `0x0C000000` with three mirrors. The x64 fast handler is `and_(call_regs[0], 0x1FFFFFFF); mov(eax, dword[rax + call_regs64[0]]);`. `rewriteMemAccess` recovers the call target from the return address, emits `call(MemHandlers[MemType::Slow][size][op])` over it (or the store-queue handler for `0xE0000000` addresses), and restores the address register from `r9`. current code: core/hw/mem/addrspace.cpp, core/rec-x64/rec_x64.cpp

### melonds-089 — Use page protection as a write barrier for cached derived data
Lens: platform · Source: melonDS
Principle: If you cache something derived from memory, such as compiled code or a parsed structure, you need to know when the source changes. Checking on every write is expensive. Instead, mark the source pages read-only. The first write faults, the handler invalidates the derived data and unprotects the page, and every later write runs at full speed.
Why it is faster: —
Applies elsewhere: garbage-collector card marking with `mprotect`, copy-on-write `fork()`, incremental snapshots and live migration (dirty-page logging), and Valgrind/DynamoRIO self-modifying-code detection.
Watch out: —
Seen in melonDS: When a block is compiled from a page, `SetCodeProtection(region, offset, true)` moves that page to `memstate_MappedProtected` and unmaps or protects it in every fastmem mirror. A guest store there faults into the slow write path, which calls `InvalidateByAddr` on the blocks overlapping the affected 512-byte `CodeRanges` slot. Invalidation tracking was refined to 512-byte granularity in `9cf7780e`. In the current code this is `SetCodeProtectionRange`, still driven by the same state table. e335a8ca 2020-06 “first steps in bringing over the JIT refactor/fastmem” · 9cf7780e 2019-10 “decrease jit block cache address granularity...”

### melonds-090 — Chain compiled units directly with patchable jumps
Lens: jit-interpreters · Source: melonDS
Principle: Returning to a central dispatcher after every block costs a hash lookup and an indirect branch. Block linking emits a patchable stub at each exit. The first time the exit runs, the stub is overwritten with a direct `JMP` to the target block. Invalidating a block puts the NOPs back in every exit linked to it. The trade-off is bookkeeping, and you lose regular safe points for checking interrupts.
Why it is faster: —
Applies elsewhere: QEMU TCG block chaining, PLT lazy binding (the first call resolves and patches the GOT), V8's on-stack replacement trampolines, and hot-patching via 5-byte NOP slots (Windows hotpatch, ftrace).
Watch out: —
Seen in melonDS: `Comp_SpecialBranchBehaviour` emitted `NOP(5)` at a static exit and a jump to `BranchStub[Num]`, which calls `ARMJIT::LinkBlock<cpu>`. That then calls `Compiler::LinkBlock(offset, entry)` to write `JMP entry` over the NOPs. `UnlinkBlock` restores them, and `JitBlock::Links()` remembers every incoming site. A new `ARMJIT_Linkage.s` held the dispatcher. Six months later the default was lowered so linking is no longer used (“branch linking is dead”). Branch following stayed. 3098c6a9 2020-04 “preparations for block linking” · 1c07932b 2020-04 “implement block linking + some refactoring...” · 65be1840 2020-10 “change JIT branch optimisations default to 1 branch linking is dead”

### melonds-091 — Recycle invalidated artifacts by content hash
Lens: jit-interpreters · Source: melonDS · Also: ares
Principle: Invalidation is often a false alarm: a write hits the page but leaves the bytes the artifact depends on unchanged, or code is unloaded and later reloaded identically. Park invalidated artifacts in a side table keyed by a hash of their inputs. Before rebuilding, check whether an identical one is parked and revive it, which skips the expensive recompile.
Why it is faster: —
Applies elsewhere: content-addressed build caches (ccache, Bazel, Nix), browser V8 code caching keyed by script hash, shader pipeline caches, and HTTP ETag revalidation (304 Not Modified).
Watch out: —
Seen in melonDS: `InvalidateByAddr` unlinks a block but stores it in `RestoreCandidates`. `CompileBlock` first checks for a match: originally on address plus an instruction-by-instruction compare, later on xxHash3 `InstrHash` with `PseudoPhysicalAddr` and `LiteralHash` verified (`efb79664`, “makes Golden Sun burn a little slower through the JIT memory”). The custom 2-way lossy hash table was replaced by `std::unordered_map` in `4299ef5f`, which is still the form today. 40b88ab0 2019-10 “new block cache and much more...” · efb79664 2020-05 “use instr hash as key for restore candidates...” · 4299ef5f 2020-08 “use unordered map for JIT RestoreCandidates...” • Also seen in ares (ares-078): `SH2::Recompiler::block()` first runs `measure()`, which copies the block's opcodes into `instructions[]` until it reaches a terminal opcode (`isTerminal` comes from the same decoder table). It then computes `XXH3_64bits(...) ^ address` and looks it up in a `hashset<BlockHashPair>` before calling `emit()`. Self-modifying code or code that is reloaded and then rewritten unchanged gets its old block back. TRAPA and SLEEP were reclassified as block terminators so the measured boundaries match. 600964d72 2023-03 “sh2: reuse previously recompiled blocks”

### melonds-092 — Direct-mapped lookup tables instead of hashing for dense keys
Lens: lookup-tables · Source: melonDS
Principle: When keys are addresses inside a few known, bounded regions, a hash map spends time hashing and probing. Give each region a flat array indexed by `(addr - regionBase) / alignment`, so a lookup is a shift, a load and a tag compare. Size the coarse first-level table to the real granularity of the address map so it stays small.
Why it is faster: —
Applies elsewhere: CPU page tables and TLBs, Linux's `struct page` array (vmemmap), file descriptor tables, interrupt vector tables, and sparse-set ECS component storage.
Watch out: —
Seen in melonDS: The first block cache used `AddrMapping[cpu][0x4000]` 16 KB slices pointing into per-region arrays. `4deecc7d` templated the lookup on CPU (`LookUpBlock<0>`) and gave ARM9 a 0x2000-entry table of 32 KB slices, matching its larger regions. In the current code, `FastBlockLookupITCM/MainRAM/SWRAM/...` are `u64` arrays with one entry per halfword. `LookUpBlock` checks `*entry >> 32 == (addr | num)` and returns the code offset in the low 32 bits. 4deecc7d 2019-07 “jit: decrease blockcache AddrMapping size for ARM9”

### melonds-093 — Widen the register pool by also using caller-saved registers
Lens: jit-interpreters · Source: melonDS
Principle: A code generator that uses only callee-saved registers never has to spill around calls, but on x86-64 that leaves only a handful of registers. Also allocating caller-saved (volatile) registers roughly doubles the pool. The price is a push/pop of just the live volatile registers around the rare out-of-line calls, which is cheaper than spilling guest state to memory all the time.
Why it is faster: —
Applies elsewhere: interprocedural register allocation in LLVM (`-fipa-ra`), leaf-function optimization, the `preserve_most`/`preserve_all` calling conventions for cold slow paths, and the Linux kernel's `__attribute__((no_caller_saved_registers))`.
Watch out: —
Seen in melonDS: The x64 `NativeRegAllocOrder` went from `RBX, R12, R13, R14 // this is sad` to also include `R9, R10, R11`, raising `NativeRegsAvailable` from 4 to 7. `PushRegs`/`PopRegs` save only loaded volatile registers around calls. The A64 backend grew from 8 to 15 (adding `W8–W15` as `CallerSavedPushRegs`) and passes `hasMemoryInstr` so blocks without memory calls skip the save sets. 80b88dbd 2020-05 “allow allocating caller saved registers currently system-v only” · aa430608 2021-06 “support allocating more registers for aarch64 JIT...”

### melonds-094 — Shrink the slow-path fallback set
Lens: jit-interpreters · Source: melonDS
Principle: In a mixed-mode system every operation that falls back to the generic path costs a full context switch: flush cached state, call out, reload. Converting even uncommon operations to the fast path removes those flushes from every block that contains them, and blocks no longer have to end early around them.
Why it is faster: —
Applies elsewhere: pushing more predicates down into an index scan instead of a residual filter, making a JS engine's baseline JIT handle more bytecodes inline, and adding SIMD paths for more cases in libraries such as simdjson.
Watch out: —
Seen in melonDS: The x64 JIT gained native `A_Comp_MRS`/`A_Comp_MSR` (a direct `MOV rd, RCPSR`; it calls `ReadBanked` only for SPSR). `A_Comp_SMULL_SMLAL` became `A_Comp_Mul_Long`, covering unsigned `UMULL/UMLAL` with a cheaper `BSR`-only cycle estimate. Data-processing instructions with a rotated immediate and the `S` flag now set carry inline instead of falling back. c2dd6a18 2020-04 “implement msr and mrs for the x64 JIT” · 1c98cefc 2020-04 “compile UMULLs and some fixes” · 887ad27e 2020-07 “implement carry setting ALU op with imm”

### melonds-095 — Peephole-trim generated code
Lens: jit-interpreters · Source: melonDS
Principle: Code generators built from per-operation templates leave small redundancies: moving a value into the register it is already in, or materializing a value that a following instruction could take directly. Checking for these at emit time costs nothing at runtime and shrinks every hot block.
Why it is faster: —
Applies elsewhere: LLVM's `MachineCopyPropagation`, peephole passes in GCC, eliminating no-op CSS declarations in minifiers, and collapsing redundant `SELECT` wrappers in ORMs.
Watch out: —
Seen in melonDS: The x64 fastmem store patch emitted `MOV ABI_PARAM3, rdMapped` even when the two were the same register, so it now checks `if (rdMapped != ABI_PARAM3)`. On A64, `Comp_Arithmetic` uses `ADCS` directly with a register operand instead of building `~op2` with `ORN` and two `ADDS`+`CSET` sequences, and it emits `CSET`/`CSINC` flag capture only when `S` is set. 17ce4d2a 2020-07 “x64 JIT: remove unecessary MOV” · 5a071c4c 2021-06 “some tiny A64 optimisations”

### melonds-096 — Hoist runtime-invariant choices out of hot loops with templates
Lens: compiler-build · Source: melonDS
Principle: A function pointer chosen once at setup time but called in an inner loop costs an indirect call per iteration, and the compiler can't inline through it. Make the choice a template parameter, branch once outside the loop to the right instantiation, and the call becomes direct and inlinable, often vanishing entirely.
Why it is faster: —
Applies elsewhere: `std::sort` with a comparator type versus `qsort` with a function pointer, Rust generics versus `dyn Trait`, pixel-format-specialized blitters in SDL and Skia, and per-dtype kernels in NumPy/PyTorch.
Watch out: —
Seen in melonDS: `GPU2D` used to set a `DrawPixel` member pointer (`DrawPixel_Accel`/`_Normal`) and call it per pixel. Now `DrawBG_Text/Affine/Extended/Large` and `InterleaveSprites` take `template<bool mosaic, DrawPixel drawPixel>`, chosen once per layer by the `DoDrawBG` macros. `DMA::Run9/Run7` similarly dropped the `BusRead16/BusWrite32` function pointers for `template <int ConsoleType>`, calling `DSi::ARM9Write16` or `NDS::ARM9Write16` directly per unit transferred. 690eed9e 2020-11 “GPU2D: don't an indirect call in tight loops” · 842379c4 2020-11 “harmless DMA micro optimisation”

### melonds-097 — Make container capacity a compile-time constant
Lens: allocation · Source: melonDS
Principle: A ring buffer whose size is a runtime field lives on the heap, behind a pointer, and its wraparound uses a variable compare or modulo. If the size is known in advance, make it a template parameter. The storage is then embedded inline (no allocation, one less indirection, better locality) and the bounds become immediates the compiler can optimize.
Why it is faster: —
Applies elsewhere: `std::array` versus `std::vector`, `boost::static_vector`/`SmallVector`, lock-free SPSC queues with power-of-two template sizes, and fixed-size message buffers in embedded RTOS queues.
Watch out: —
Seen in melonDS: `FIFO<T>* InputFIFO = new FIFO<u32>(16)` became `FIFO<u32, 16> InputFIFO`, with `T Entries[NumEntries]` inline, across AES, NWifi, SD, the GX command FIFO and IPC. The `new`/`delete` in `Init`/`DeInit` and the `->` indirections went away. The current `FIFO.h` keeps both the static `template<typename T, u32 NumEntries>` and a dynamic variant. fa4363ed 2020-12 “make FIFO size static whene possible”

### melonds-098 — Give the common simple case its own path
Lens: branching-fast-paths · Source: melonDS
Principle: General code handles the worst case, such as variable-length parameters, buffering and multi-step state. If most inputs are the trivial case, test for it first and handle it straight-line, skipping the buffering and bookkeeping. The general path is untouched and still covers everything else.
Why it is faster: —
Applies elsewhere: the small-string optimization, UTF-8 decoders with an ASCII fast loop, HTTP parsers special-casing `GET`, `memcpy` with small-size jump tables, and Redis encoding small hashes as ziplists.
Watch out: —
Seen in melonDS: `GPU3D::ExecuteCommand` collected every command's parameters into `ExecParams` and then ran a big switch. Now, when `CmdNumParams[entry.Command] <= 1`, it executes directly from `entry.Param`. That covers `MTX_MODE`, `COLOR`, `NORMAL`, `TEXCOORD`, `VTX_10` and most of the stream. Pipeline-stall rules were split into small inline helpers (`VertexPipelineCmdDelayed4/6/8`), and the table shrank to `u8`. d2c04c5c 2020-11 “GX: add fastpath for single parameter cmds”

### melonds-099 — Keep a flat shadow copy coherent with fine-grained dirty bits
Lens: laziness · Source: melonDS
Principle: When readers access data through an indirection (bank mapping, paging, a scatter list) at a far higher rate than writers change it, keep a flattened linear copy for the readers. Writers set a dirty bit per small chunk. Before each read phase, recopy only the dirty chunks plus any remapped ranges, so the hot loop reads a plain array.
Why it is faster: —
Applies elsewhere: GPU texture caches uploaded from dirty rectangles, React/virtual-DOM reconciliation, incremental materialized views, database buffer pools flushing dirty pages, and spreadsheet recalculation of only dirty cells.
Watch out: —
Seen in melonDS: Every VRAM write sets `VRAMDirty[bank][addr / 512]`. `VRAMTrackingSet<Size, Granularity>::DeriveState` combines bank dirtiness with mapping changes, and `MakeVRAMFlat_ABGCoherent` (and the OBJ, ExtPal, Texture and TexPal versions) copy only the dirty 512-byte chunks into `VRAMFlat_*`. Renderers now read `VRAMFlat_Texture[addr & 0x7FFFF]` instead of walking bank maps. `58dd1ec5` removed the intermediate `VRAMWritten_*`/`SyncDirtyFlags` pass per HBlank by marking banks directly on BG/OBJ writes. Display capture marks its output dirty too, and palette/OAM got the same flags. 6e8bac39 2020-11 “Merge vram dirty tracking...” · 0f353e0d 2020-12 “add palette and OAM dirty flag...” · 7081c2de 2021-01 “readd vram invalidation through display capture...” · 58dd1ec5 2021-02 “directly set VRAMDirty for VRAM BG/OAM writes”

### melonds-100 — Skip recomputation when every input is unchanged
Lens: laziness · Source: melonDS
Principle: If an expensive output is a pure function of a known set of inputs, compare the inputs (state registers, dirty flags of referenced memory) with last time. If nothing changed, reuse the previous output and skip the whole computation.
Why it is faster: —
Applies elsewhere: `React.memo`/`useMemo`, Make/Bazel incremental builds, browsers skipping repaint of unchanged layers, and video encoders emitting skip blocks for static regions.
Watch out: —
Seen in melonDS: Building on the VRAM tracker, the software 3D renderer computes `FrameIdentical = !(textureChanged || texPalChanged) && RenderFrameIdentical`. `RenderFrameIdentical` is computed in `GPU3D` by comparing the latched render state (`RenderDispCnt == DispCnt` and related registers) with the previous frame. When it holds, rasterization is skipped and last frame's color buffer is reused, which is common in menus and paused scenes. 6e8bac39 2020-11 “Merge vram dirty tracking...” (“also skip rendering if nothing changed”)

### melonds-101 — Machine-word bitsets with count-trailing-zeros iteration
Lens: memory-layout · Source: melonDS
Principle: Store bit flags in 64-bit words instead of bytes. Clearing, OR-ing and testing for any set bit then process 64 flags per instruction, and iterating set bits with `ctz` plus clear-lowest-bit jumps straight from one set bit to the next. Sparse dirty sets cost time proportional to the number of set bits, not their range.
Why it is faster: —
Applies elsewhere: Roaring bitmaps, `std::bitset`/Java `BitSet.nextSetBit`, chess bitboards, the Linux scheduler's runqueue bitmap, and free-list bitmaps in allocators (jemalloc, the Go GC).
Watch out: —
Seen in melonDS: `NonStupidBitField<Size>` was introduced with the VRAM tracker. Its `Iterator` uses `__builtin_ctzll(RemainingBits)` to visit set bits, used when copying dirty chunks. `6256a42e` moved storage from `u8 Data[Size/8]` to `u64 Data[(Size+63)>>6]`, made bit access `Data[Idx >> 6] & (1ULL << (Idx & 63))`, added `SetRange`, and replaced type-punned `((u32*)Data)[i]` writes. That removed undefined behaviour that broke optimized LTO Clang builds. 6e8bac39 2020-11 “Merge vram dirty tracking...” · 6256a42e 2021-02 “improve and fix NonStupidBitfield also get rid of some UB...”

### melonds-102 — Snapshot shared inputs so a worker thread gets more slack
Lens: concurrency · Source: melonDS
Principle: A producer/consumer pipeline is limited by how early the worker can start and how late the result is needed. If the worker reads from a private snapshot instead of live shared state, producers can keep mutating, and the hand-off point can move to the last moment the output is needed. Re-arm synchronization so stale work can't queue up.
Why it is faster: —
Applies elsewhere: triple buffering in game engines (simulation, render, present), RCU, Postgres MVCC snapshots letting readers run alongside writers, and audio engines rendering into a ring buffer ahead of the device callback.
Watch out: —
Seen in melonDS: The threaded software rasterizer had to finish by scanline 144, when real hardware stops reading texture VRAM. Textures are now read from the `VRAMFlat_*` snapshot, which is updated only at frame start, so `4a280682` moved `GPU3D::VCount144()` to the later sync point, as the comment explains (“we cache the entire texture memory anyway”). `b78bc4cb` fixed the thread's semaphores (`Semaphore_Reset(Sema_RenderStart)`, “otherwise more than one frame can be queued up”) and moved the 3D-line fetch after the capture latch. 4a280682 2021-01 “the rasteriser doesn't have to be done on line 144 thanks to the VRAM cache” · b78bc4cb 2021-01 “fixes to the threadedness of the sw rasteriser...”

### melonds-103 — Coalesce and debounce persistent writes off the hot thread
Lens: io · Source: melonDS
Principle: Writing a whole file every time a small piece of state changes turns a burst of N updates into N full rewrites, all blocking the caller. Mark the state dirty, flush at most once per natural boundary or after a quiet period, and do the write on a background thread from a copied buffer.
Why it is faster: —
Applies elsewhere: the Linux page cache's writeback, debounced autosave in editors, write-behind caches, Redis `appendfsync everysec`, and batching analytics events before sending.
Watch out: —
Seen in melonDS: The cart backup-memory emulator rewrote the save file at the end of every SPI write command. `62e3f41f` changed that to `SRAMFileDirty |= ...` and called `NDSCart::FlushSRAMFile()` once at the end of `RunFrame`. `bf97387f` then added `NDSCart_SRAMManager`, a flush thread that copies into `SecondaryBuffer` under a mutex and writes only once the request is about two seconds old (or at `DeInit`), using `FlushVersion` counters to avoid losing the final state. 62e3f41f 2020-11 “delay savefile flush to the end of the frame” · bf97387f 2021-01 “Improved SRAM performance (#925)”

### melonds-104 — Delete work whose result nobody reads
Lens: laziness · Source: melonDS
Principle: Code accumulates computations whose consumers have disappeared. Audit expensive steps on startup and other hot paths for live uses, and remove the ones with none. It's the cheapest optimization there is.
Why it is faster: —
Applies elsewhere: removing unused middleware from a request pipeline, dropping logging that formats strings at disabled levels, removing eager hashing of uploads that is never checked, and pruning unused indexes that slow writes.
Watch out: —
Seen in melonDS: `NDSCart::LoadROM` computed `CartCRC = CRC32(CartROM, CartROMSize)`, a full pass over the whole ROM image, only to print it. The call and the `CRC32.h` include were removed. 1649c0e0 2020-12 “don't calculate the CRC checksum of ROMs on load the value isn't used anywhere anyway”

### melonds-105 — Recompute derived state only when its inputs actually change
Lens: laziness · Source: melonDS
Principle: A setter that always recomputes dependent tables wastes work when callers rewrite the same value or change unrelated bits. XOR the old and new values, mask with the bits that matter, and recompute only when that is non-zero.
Why it is faster: —
Applies elsewhere: React `setState` bailing out on equal values, CSS engines restyling only when relevant properties change, observers firing on change rather than on write, and `glUseProgram` state caches.
Watch out: —
Seen in melonDS: Writes to `EXMEMCNT` from either CPU always called `SetGBASlotTimings()`, which rebuilds memory timing tables. Now `if ((oldVal ^ ExMemCnt[0]) & 0xFF) SetGBASlotTimings();`. Games poke this register often to hand slot ownership back and forth without changing timing. dd53b01f 2021-06 “only recalculate GBA slot timing if necessary”

### melonds-106 — Drop batch machinery when the batch size is always one
Lens: algorithms · Source: melonDS
Principle: Code written for N items per call (temporary arrays, inner loops, clearing buffers) costs real overhead when N is effectively always 1. Rewrite it to pass scalars. Values stay in registers, loops and memsets vanish, and the code gets shorter.
Why it is faster: —
Applies elsewhere: removing a generic batch API from a per-request handler, scalar fast paths in vectorized libraries for size 1, and avoiding single-element `Promise.all` or array allocations in hot JS.
Watch out: —
Seen in melonDS: `SPU::Mix` was always scheduled with `kSamplesPerRun = 1` but used `s32 leftbuf[32], rightbuf[32], ch0buf[32]...`, zeroing loops and per-channel `Run(buf, samples)` loops. Now `s32 Channel::Run()` returns one sample, `PanOutput(in, left, right)` accumulates into scalars, and the clamp and capture logic runs once. 9ac60a84 2020-10 “SPU: work with scalars instead of arrays”

### melonds-107 — Early-out for units that can't produce output
Lens: branching-fast-paths · Source: melonDS
Principle: In a loop over many components, check cheaply whether each one can affect the result this tick, because it is disabled, powered down or degenerate, and skip it before any per-tick work. Idle components then cost one branch instead of a full update.
Why it is faster: —
Applies elsewhere: game engines skipping sleeping physics bodies, ECS systems iterating only active archetypes, culling invisible DOM subtrees (`content-visibility: auto`), and NIC drivers' interrupt moderation in idle states.
Watch out: —
Seen in melonDS: `SPU::Channel::Run` now returns immediately for PCM/ADPCM channels with `Length + LoopPos < 16`, which hardware treats as silent. In `Wifi::USTimer` (which runs every microsecond), all TX/RX state-machine work is wrapped in `if (!(IOPORT(W_PowerState) & 0x300))`, and power-down is emulated with a `USUntilPowerOn` countdown. While a game has WiFi asleep, the hot timer does almost nothing. a32d997e 2020-08 “SPU: don't process channels with len<4” · 0a1fb5ae 2022-02 “Wifi power saving, hopefully enough!”

### melonds-108 — Precompute fixed-point weight tables for interpolation
Lens: lookup-tables · Source: melonDS · Also: ares
Principle: Interpolation kernels such as cosine and cubic evaluate transcendental or polynomial functions of the fractional position. Quantize the fraction to 8 bits and precompute the weights once as fixed-point integers. Per-sample work becomes a table load and a few integer multiply-adds.
Why it is faster: —
Applies elsewhere: audio resamplers (libsamplerate, SNES Gaussian tables), bicubic image scaling in libswscale, gamma/sRGB LUTs, and sine tables in DSP firmware on microcontrollers without an FPU.
Watch out: —
Seen in melonDS: The audio interpolation option builds `s16 InterpCos[0x100]` (`(1 - cos(i·π/255)) · 0x2000`) and `s16 InterpCubic[0x100][4]` (Catmull-Rom weights in 1:1:14 fixed point) in `SPU::Init`. Each channel indexes them with `(Timer >> 8) & 0xFF`. Today they are `constexpr std::array` built by lambdas at compile time, alongside a 512-entry SNES Gaussian table. 2df6b4fd 2021-08 “Audio interpolation (#1176)” • Also seen in ares (ares-072): v114r13 added `SPU::gaussianConstructTable()` for the PS1. It builds a 512-entry kernel from `sin` and `cos` terms, scales it so that the sum is `0x7f80` across each set of four phases, and rounds into `gaussianTable[]`. The SNES DSP's hard-coded constants were replaced by the same kind of generation. 46bba9b79 2020-07 “Update to ares v114r13 release.”

### melonds-109 — Batch draw calls by sorting on a state key
Lens: gpu-rendering · Source: melonDS
Principle: GPU drivers charge per draw call and per state change. Give every primitive a sort key that encodes all the state it needs, place primitives with equal keys next to each other, and issue one indexed draw per run. A single shared index buffer lets a run be one contiguous range.
Why it is faster: —
Applies elsewhere: sprite batching in 2D engines (Phaser, Unity), material-sorted render queues, Skia and browser compositors batching paint ops, and draw-call merging in UI toolkits (Dear ImGui's draw lists).
Watch out: —
Seen in melonDS: The GL renderer stores each `RendererPolygon`'s `IndicesOffset`/`EdgeIndicesOffset` into one `GL_ELEMENT_ARRAY_BUFFER` instead of raw client pointers. `RenderPolygonBatch` extends a run while `PrimType` and `RenderKey` match, then issues one `glDrawElements`. `de19ce62` added a direct three-index path for plain triangles via a shared `SetupVertex` helper. `8a3a8b7c` added opacity (bit `0x4000`) to `RenderKey` so batches never mix passes that must stay separate. de19ce62 2020-08 “3D/GL: ...add quicker codepath for triangles...” · 21dbca95 2020-11 “use proper index buffers” · 8a3a8b7c 2021-01 “include polygon attr opaqueness for translucent polygons in render key”

### melonds-110 — Replace full pipeline drains with fences and double buffers
Lens: gpu-rendering · Source: melonDS
Principle: `glFinish`-style calls stall the CPU until the GPU has drained all queued work, which serializes the two processors. Fence only the specific resource being handed between threads, wait on the GPU side where possible, double-buffer the shared target so producer and consumer use different copies, and flush the command stream once per frame instead of after every pass.
Why it is faster: —
Applies elsewhere: Vulkan semaphores and fences per frame-in-flight, WebGL `fenceSync`, Metal triple-buffered uniform rings, and video players handing decoded frames over with sync objects instead of `glFinish`.
Watch out: —
Seen in melonDS: The emulator thread renders into `CompScreenOutputTex[2]`, double-buffered in `40aae154`. The Qt paint thread displays the other texture. `64c6654d` replaced `glFinish()` on both sides with `glFenceSync` after producing (`FrontBufferSyncs`) and after consuming (`FrontBufferReverseSyncs`), plus `glWaitSync` on the opposite side. `a3f4aaf5` removed per-chunk `glFlush` calls from `RenderSceneChunk`/`RenderFrame` and issued one flush per frame. a3f4aaf5 2021-01 “call glFlush only once...” · 40aae154 2021-02 “prevent race condition around framebuffers” · 64c6654d 2021-02 “use syncs instead of glFinish”

### melonds-111 — Lay out GPU-shared structs to the API's alignment rules
Lens: gpu-rendering · Source: melonDS
Principle: Uniform and constant buffers have strict alignment and size rules (std140: 16-byte multiples). If the CPU struct doesn't match, every upload has to round sizes or repack, and tools or drivers may read past the end. Pad the struct explicitly and `static_assert` the size, so the struct can be `memcpy`-uploaded as-is.
Why it is faster: —
Applies elsewhere: std140/std430 layouts in GLSL, HLSL cbuffer packing, WebGPU uniform alignment, and network packet structs padded for DMA engines.
Watch out: —
Seen in melonDS: `ShaderConfig` gained `u32 _pad1[2]`, and the upload became `static_assert((sizeof(ShaderConfig) & 15) == 0); glBufferData(GL_UNIFORM_BUFFER, sizeof(ShaderConfig), ...)` instead of rounding up the size at each call. 53d5def9 2021-01 “pad ShaderConfig to a multiple of 16”

### melonds-112 — Present no more often than the display can show
Lens: other · Source: melonDS
Principle: When simulation outpaces the screen, as in fast-forward or unthrottled mode, drawing every produced frame wastes CPU and GPU on frames the monitor never displays. Measure the production rate and present only every k-th frame, so the extra throughput goes to simulation.
Why it is faster: —
Applies elsewhere: `requestAnimationFrame` instead of drawing per data update, throttling progress-bar redraws in CLIs, coalescing WebSocket ticks into one render per frame, and game servers decoupling tick rate from client snapshot rate.
Watch out: —
Seen in melonDS: In `EmuThread::run`, `winUpdateFreq = fps / round(fpstarget)` is recomputed each measurement interval, and `emit windowUpdate()` fires only when `++winUpdateCount >= winUpdateFreq`. 03b5c480 2022-02 “lower window update rate if rendering too fast”

### melonds-113 — Pace frames with a high-resolution clock and carried-over error
Lens: other · Source: melonDS
Principle: Frame limiters built on millisecond ticks and ad-hoc resyncs either oversleep (stutter) or busy-spin. Use the high-resolution performance counter, carry the sleep error from frame to frame (clamped to one frame), and measure how long you actually slept. Even when "uncapped", impose a high ceiling so the loop doesn't spin a core on trivial frames.
Why it is faster: —
Applies elsewhere: game loops with fixed timestep accumulators, media players' A/V sync, token-bucket rate limiters carrying fractional credit, and `clock_nanosleep(TIMER_ABSTIME)` loops in real-time control.
Watch out: —
Seen in melonDS: The SDL millisecond-tick limiter was replaced (after Citra's `perf_stats`) with `SDL_GetPerformanceCounter()` in seconds and `frameLimitError += frametimeStep - (curtime - lastTime)`, clamped to ±1 frame. It sleeps `round(frameLimitError*1000)` ms and subtracts the time actually slept. `42e08396` applies the same logic with `practicalFramelimit = 1/1000` when limiting is off, which caps the loop at 1000 FPS. ad7791f7 2020-11 “better framelimiter...” · 42e08396 2020-12 “always cap FPS to 1000”

### melonds-114 — Let a time slice stretch slightly to absorb a nearby event
Lens: algorithms · Source: melonDS
Principle: A scheduler that caps each run slice at a fixed maximum will, when an event falls just past the cap, run a full slice and then a tiny one. Every slice boundary has fixed overhead. Allow a small margin: if the next event is within it, extend the slice to reach the event directly.
Why it is faster: —
Applies elsewhere: timer slack in Linux (`prctl(PR_SET_TIMERSLACK)`) and Windows coalescable timers, batching nearby wakeups in mobile OSes, and Nagle-style coalescing of small writes.
Watch out: —
Seen in melonDS: `NDS::NextTarget()` computed `min(earliest event, SysTimestamp + kMaxIterationCycles)`, so an event 1 cycle after the cap produced a 16-cycle slice followed by a 1-cycle slice. With `kIterationCycleMargin = 8`, it returns `minEvent` whenever `minEvent < max + 8`. This resolved a TODO about exactly this. 737171c9 2021-09 “implement margins for scheduler hopefully this does not break anything”

### melonds-115 — Get release build flags right, and make builds faster to iterate
Lens: compiler-build · Source: melonDS
Principle: Overwriting release flags wholesale can silently drop important defines such as `-DNDEBUG`, leaving asserts in the shipped binary. Edit the default flag string in place. Use LTO with matching archiver tools so cross-module inlining actually happens, and speed up developer iteration with a faster linker and a compiler cache.
Why it is faster: —
Applies elsewhere: any CMake or autotools project. Rust's `lto = "thin"` and `codegen-units = 1`, Go's PGO, mold/lld in CI, and sccache for distributed caching.
Watch out: —
Seen in melonDS: `set(CMAKE_CXX_FLAGS_RELEASE "-O3")` wiped out CMake's `-O2 -DNDEBUG`, so release builds kept every `assert`. `c177fae5` uses `string(REPLACE "-O2" "-O3" ...)` instead. `14be591a` sets `CMAKE_AR`/`CMAKE_RANLIB` to `gcc-ar`/`gcc-ranlib` or `llvm-ar`/`llvm-ranlib` so LTO archives link correctly, adds `-fPIC`/`-pie`, and uses lld with Clang when it is found. `9b5d5f67` wires up `ccache` via `CMAKE_CXX_COMPILER_LAUNCHER`. 14be591a 2020-11 “Override CMAKE_AR/CMAKE_RANLIB... also use lld with clang if found (#828)” · 9b5d5f67 2021-07 “Use CCache if it exists” · c177fae5 2022-10 “Clean up optimization flags”

### melonds-116 — Declare constant tables constexpr
Lens: compiler-build · Source: melonDS
Principle: A lookup table that is never written but isn't declared `const` goes into writable `.data`. The compiler must assume it can change, so it can't fold loads from it. `constexpr` puts it in read-only memory, lets the compiler constant-fold indexed loads with known indices, and makes accidental writes a compile error.
Why it is faster: —
Applies elsewhere: CRC and Base64 tables in C libraries, `static const` lookup arrays in embedded firmware (kept in flash, not RAM), and frozen objects in JS engines enabling inline caching.
Watch out: —
Seen in melonDS: `DMATiming::MRAMRead16Bursts`, `MRAMRead32Bursts`, `MRAMWrite16Bursts`, `MRAMWrite32Bursts` and `MRAMDummy` became `constexpr u8[][256]`, and `DMA::MRAMBurstTable` became `const u8*`. 457dd56b 2022-07 “constexpr-s the DMA timing tables (#1489)”

### melonds-117 — Keep expensive handles open instead of reopening per use
Lens: io · Source: melonDS
Principle: Opening a file, device or connection involves path lookup, permission checks and buffer setup. Code that reopens the same resource for each operation repeats that cost and can hold several conflicting handles. Open once, share the handle, and close at teardown.
Why it is faster: —
Applies elsewhere: database connection pools, HTTP keep-alive, holding an `fd` for a log file instead of append-open-close, and SQLite in long-running apps.
Watch out: —
Seen in melonDS: DSi NAND access opened `Config::DSiNANDPath` separately in `DSi::LoadNAND`, in the eMMC device in `DSi_SD` and in the frontend. Now one `FILE* SDMMCFile` is opened once, used for every `fseek`/`fread`, and closed by `CloseDSiNAND()` in `DeInit`. 1d6cc3c6 2021-01 “keep only one handle of the NAND file around”

### melonds-118 — Acquire costly resources only while they are in use
Lens: laziness · Source: melonDS
Principle: Devices such as microphones, cameras and sensors cost CPU, power and OS attention while open, even when paused. Open them on demand when the feature is actually configured and running, and release them on pause or stop.
Why it is faster: —
Applies elsewhere: mobile apps releasing GPS and camera in `onPause`, the Web `getUserMedia` track `stop()`, lazily opening database connections, and Wake Lock APIs held only during active use.
Watch out: —
Seen in melonDS: The SDL capture device used to be opened at startup and only paused. New `micOpen()`/`micClose()` calls in `emuRun`/`emuPause`/`emuUnpause`/`emuStop` open it only when `Config::MicInputType == 1` (external mic), and close it otherwise. aaa97c92 2021-07 “only open microphone when actually needed. fixes #1165”

### melonds-119 — Make generated code visible to your profiler
Lens: other · Source: melonDS
Principle: Sampling profilers attribute time by mapping instruction addresses to symbols. Code generated at runtime has no symbols, so it shows up as one opaque blob. Register each generated function (name, start, size) with the profiler's JIT API so hot spots inside generated code can be found and optimized.
Why it is faster: —
Applies elsewhere: V8 `--perf-prof`, perf's `/tmp/perf-PID.map` convention, the JVM's `-XX:+UnlockDiagnosticVMOptions -XX:+DumpPerfMapAtExit`, LuaJIT's VTune and GDB JIT interface, and .NET EventPipe method-load events.
Watch out: —
Seen in melonDS: A new CMake option `ENABLE_JIT_PROFILING` (with `cmake/FindVTune.cmake`) defines `JIT_PROFILING_ENABLED`. The x64 compiler then calls `CreateMethod("FastMemStorePatch%d_%d_%d", ...)`, `CreateMethod("ReadBanked", ...)` and so on through VTune's JIT API for the generated thunks. 6944fdbe 2021-07 “add support for JIT profiling with VTune”

### melonds-120 — Store instead of read-modify-write when the result is known
Lens: memory-layout · Source: melonDS
Principle: `x |= 1` on a value that can only be 0 or 1 has the same result as `x = 1`, but it adds a load and a data dependency. In a per-pixel or per-element loop, a plain store pipelines better and lets the compiler vectorize or combine stores.
Why it is faster: —
Applies elsewhere: setting flags in byte arrays (visited sets in BFS), `memset` instead of OR-ing a mask byte by byte, and marking dirty bytes in card tables.
Watch out: —
Seen in melonDS: In `SoftRenderer::RenderShadowMaskScanline`, the left-edge fill changed from `StencilBuffer[256*(y&1) + x] |= 0x1;` to `= 1;`. 509107fb 2021-05 “set instead of or stencil buffer for left edges”

### melonds-121 — Fixed slot arrays of plain function pointers instead of maps of type-erased callables
Lens: memory-layout · Source: melonDS
Principle: A `std::map<id, std::function>` costs a tree walk, a pointer chase to a heap node, and an indirect call through the type-erasure layer, which may also have allocated. When the key space is small and known, use a fixed array of raw function pointers plus one `void*` context, with a captureless lambda as the thunk. Dispatch becomes one indexed load and one indirect call.
Why it is faster: —
Applies elsewhere: event loops and timer wheels in embedded C (`void (*cb)(void*, uint32_t)`), game-engine message handlers keyed by enum, replacing `Map<string, Function>` with arrays indexed by opcode in JS interpreters.
Watch out: —
Seen in melonDS: The 2023 OOP scheduler stored `std::map<u32, EventFunc> Funcs` per event, where `EventFunc = std::function<void(u32)>` built by `std::bind`. `NDS::RunSystem` called through that on every event. Now `SchedEvent` holds `std::array<EventFunc, 3> Funcs; void* That;` with `typedef void (*EventFunc)(void*, u32)`, and `MakeEventThunk(cls, f)` expands to `[](void* that, u32 p){ static_cast<cls*>(that)->f(p); }`. The dispatch is `evt.Funcs[evt.FuncID](evt.That, evt.Param)`. d0d010b0 2024-11 “don't use std::map and std::function in scheduler” · 70c67505 2023-11 “better, less hacky, more OOP-friendly scheduler design”

### melonds-122 — Hoist a mode switch out of the hot loop with a template parameter
Lens: compiler-build · Source: melonDS
Principle: If a flag such as "debugger attached" is checked inside an inner loop but only changes rarely, make it a template (or generic) parameter and branch once at the top level. The compiler emits one specialised copy of the loop per mode, and `if constexpr` deletes the checks from the fast copy entirely.
Why it is faster: —
Applies elsewhere: logging or tracing variants of a parser, `template<bool Checked>` bounds checks in a codec, Rust generics over a `const DEBUG: bool`, separate JIT tiers for instrumented and plain code.
Watch out: —
Seen in melonDS: `ARMv5::Execute()` called `GdbCheckB()` and `GdbCheckC()` on every instruction even when no GDB stub was enabled, and the JIT used a separate `ExecuteJIT()`. Both became `template <CPUExecuteMode mode> Execute()`, with the checks behind `if constexpr (mode == CPUExecuteMode::InterpreterGDB)`. `NDS::RunFrame()` picks `RunFrame<JIT>`, `RunFrame<InterpreterGDB>` or `RunFrame<Interpreter>` once per frame. dd386d12 2024-08 “use templates to only execute GDB stub related code if enabled”

### melonds-123 — Count set bits with a popcount intrinsic, not a loop
Lens: platform · Source: melonDS
Principle: Counting the bits of a mask with a loop of shift-and-test costs up to N iterations and N branches. `__builtin_popcount` compiles to one instruction on modern CPUs, and to a branch-free SWAR sequence where that instruction is missing.
Why it is faster: —
Applies elsewhere: bitmap indexes and Bloom filters, counting free slots in an allocator bitmap, Hamming distance in similarity search, `Integer.bitCount` in Java.
Watch out: —
Seen in melonDS: `A_LDM`/`A_STM` looped over 16 bits to work out the base for a descending block transfer, and `T_PUSH` looped over 9 bits. Each is now one expression, for example `base -= 4 * __builtin_popcount(cpu->CurInstr & 0xFFFF);`. 6c21f116 2026-01 “Use intrinsics to optimize bit counting operations (#2531)”

### melonds-124 — Pre-scale operands so a division fits a narrower type
Lens: algorithms · Source: melonDS
Principle: On many CPUs a 64-bit divide is several times slower than a 32-bit one, and on 32-bit targets it becomes a library call. If you can drop low bits from the operands up front without changing the quotient's required precision, the whole numerator and denominator fit in 32 bits and the division gets cheap.
Why it is faster: —
Applies elsewhere: fixed-point DSP on Cortex-M, where the 64-bit divide is a libgcc call; texture-coordinate math in software rasterisers; rate calculations in network schedulers.
Watch out: —
Seen in melonDS: The perspective `Interpolator` in `GPU3D_Soft.h` built an `s64 num = ((s64)x * w0n) << shift` and divided it by an `s32`. It now halves the W values when it is set up (`w0n = w0 >> 1; w0d = (w0 + ((w0 & ~w1) & 1)) >> 1; w1d = w1 >> 1;`), so `u32 num = (x * w0n) << shift; yfactor = num / den;` is a plain 32-bit divide. The rounding quirk the old branches handled is folded into the bit trick. 111fc6a3 2025-08 “(finally) change software renderer to use 32-bit division”

### melonds-125 — Convert a small palette once, then index it per pixel
Lens: lookup-tables · Source: melonDS
Principle: If every pixel looks up one of K colours and then converts it, do the conversion K times up front and keep the results in a table. The inner loop drops from several multiplies and divides per pixel to one load.
Why it is faster: —
Applies elsewhere: rendering GIF and indexed PNG images, terminal emulators mapping 256 ANSI colours to RGBA, sprite engines on microcontrollers, colour-mapping a heatmap.
Watch out: —
Seen in melonDS: `ROMIcon()` in the Qt frontend did the RGB555→RGBA8888 conversion (three `*255/31`s) for every one of the 1024 icon pixels. It now fills `u32 paletteRGBA[16]` first, and the loop does `iconRef[...] = paletteRGBA[pal_index]`. The commit measured about 7 ms down to 1 ms. 5a852cb0 2024-04 “ROMManager: optimise ROMIcon function”

### melonds-126 — Build lookup tables at compile time with constexpr
Lens: lookup-tables · Source: melonDS
Principle: If a table is computed at runtime, you pay an init call and an "initialised yet?" flag check, and you can't make the table `const`. Compute it in a `constexpr` function and it lands in read-only data, ready at load time, with no guard. Where the standard math functions aren't constexpr, a short Taylor series is enough.
Why it is faster: —
Applies elsewhere: CRC, sine and gamma tables in firmware stored in flash, Rust `const fn` tables, Zig `comptime`, perfect-hash tables generated at build time.
Watch out: —
Seen in melonDS: `CRC32()` checked `if (!tableinited) _inittable();` on every call. `GetCRC32Table()` is now `constexpr` and returns a `std::array<u32,256>`. The SPU's cosine-interpolation table `InterpCos` used `std::cos`, which isn't constexpr. It now uses a templated `constexpr double cosine<10>(theta)` Taylor series, so the whole `std::array<s16,0x100>` is evaluated by the compiler. 2a3a0712 2023-08 “Reduce code stink (#1818)” · ae91d89f 2023-12 “Use a `constexpr`-friendly cosine implementation (#1903)”

### melonds-127 — Align byte buffers to the widest type you read from them
Lens: memory-layout · Source: melonDS
Principle: A `u8[]` only guarantees 1-byte alignment. Reading it as `u32` or `u64` can then split cache lines, trap on strict-alignment CPUs, or be undefined behaviour. Declare it `alignas(u64)` and every wide access is a single aligned load, safe to vectorise.
Why it is faster: —
Applies elsewhere: network packet buffers parsed as structs, DMA buffers on ARM, ring buffers read 8 bytes at a time, `ArrayBuffer` views in JS (which require alignment for `Float64Array`).
Watch out: —
Seen in melonDS: `GPU.h` now declares `alignas(u64) u8 VRAM_A[128*1024]` through `VRAM_I`, as well as `Palette`, `OAM` and `VRAMFlat_Texture`, because renderers read them as `u16`/`u32`/`u64`. In `NDS.h`, the two-word divider and sqrt registers (`DivNumerator[2]` and others) are `alignas(u64)` because they are accessed as one 64-bit value. cbd65a13 2023-12 “Add `alignas` specifiers to some arrays based on how they're accessed (#1933)”

### melonds-128 — Pass strings by const reference
Lens: allocation · Source: melonDS · Also: Flycast
Principle: Passing `std::string` by value copies it, and if it is longer than the small-string buffer that means a heap allocation and a free on every call. A `const&` parameter costs one pointer. When you need a modified copy, build a new local instead of mutating the parameter.
Why it is faster: —
Applies elsewhere: path-handling helpers in C++ tools, `&str` instead of `String` in Rust APIs, `string_view` in parsers.
Watch out: —
Seen in melonDS: Constructors and methods of `FATStorage`, `DSi_MMCStorage`, `ARCodeFile`, `Savestate` and `Platform` took `std::string` by value and now take `const std::string&`. `InjectFile` used to reassign `path = "0:/" + path`. It now builds `std::string prefixedPath("0:/"); prefixedPath += path;`. 4b170b94 2023-04 “pointless micro optimisations” • Also seen in Flycast (flycast-117): The INI writer copied every `ConfigSection` and `ConfigEntry` (`ConfigSection section = section_it->second;`). It now uses `const auto&`, and `get_string()` returns `const string&`. The block manager loops over `blkmap` by `const auto&`. clang-tidy's `performance-unnecessary-value-param` changed `AddRef(RuntimeBlockInfoPtr)` and others to take const references. 834082519 2019-09 “ini: use reference to avoid copies. simplify” · bad10bae2 2019-09 “...Use const ref to avoid copies” · fe17d79e8 2021-03 “clang-tidy: run performance-unnecessary-value-param”

### melonds-129 — Serialise into a memory buffer, then write once
Lens: io · Source: melonDS
Principle: Thousands of small `fread`/`fwrite` calls each pay library locking and bookkeeping. On some platforms they also pay a syscall or a slow filesystem. Serialise into a growable in-memory buffer and do one bulk transfer. The same buffer also serves cases that never touch disk, such as rewind or netplay snapshots.
Why it is faster: —
Applies elsewhere: protobuf/FlatBuffers into a byte array before sending, `BufferedOutputStream`, building an HTTP response body before a single `write`, game save systems on consoles.
Watch out: —
Seen in melonDS: `Savestate` wrapped a `FILE*` and called `fwrite`/`fread` for every field. The old code had a `// TODO: buffering system! ... repeated fread/fwrite is slow on Switch`. It now works on `u8* buffer` with `buffer_offset`/`buffer_length`, and `Var8..64` are inline wrappers over `VarArray` in the header. The frontend writes the whole buffer at once and can also keep it in RAM. 391ad8c9 2023-06 “Implement in-memory savestates (#1693)”

### melonds-130 — Grant peers a run-ahead budget instead of syncing every step
Lens: concurrency · Source: melonDS
Principle: Lockstep simulation across processes is limited by how often each side blocks waiting for the other. When the leader knows no interaction can happen for the next T units, it sends T along with the acknowledgement. Followers then run freely until T and block only at real interaction points.
Why it is faster: —
Applies elsewhere: conservative parallel discrete-event simulation (lookahead in Chandy–Misra), distributed game servers with input delay, credit-based flow control in networks, lease-based caching.
Watch out: —
Seen in melonDS: In local multiplayer, clients synced at every received frame. `Wifi::SendMPAck` now computes `runahead = min(CmdCounter, nextbeacon)` minus the ack time and stores it in the ack header. In `CheckRX`, a client that receives an MP ack does `NextSync += runahead`, so it doesn't stop until the host's next command or beacon. An earlier, speculative early-reply attempt was reverted. 8fd46e5f 2023-07 “wifi: attempt two at optimizing the sync mechanism” · 8772258f 2023-07 “attempt at making local multiplayer faster” (later reverted)

### melonds-131 — Check a cheap precondition before an expensive call that will fail
Lens: other · Source: melonDS
Principle: Some API calls are slow even when they fail: device enumeration, driver round trips, timeouts. Others run once per item for nothing when the resource doesn't exist. Test the cheap fact first, such as a device count, a null handle, or a mode flag, and skip the work.
Why it is faster: —
Applies elsewhere: checking `navigator.mediaDevices` before opening a stream, polling a gamepad only if one is connected, discarding packets by header before full parsing, `if (log.isDebugEnabled())`.
Watch out: —
Seen in melonDS: `MicOpen()` tried to open an SDL capture device every time even with no microphones. On Windows 11 this hung the UI, and it now returns if `SDL_GetNumAudioDevices(1) == 0`. `Input::Process()` called `JoystickButtonDown` for 12 buttons and every hotkey each frame and now does so only `if (Joystick)`. `Wifi::CheckRX` drops MP command and reply frames before parsing when the console isn't in an MP session. ca5e8792 2023-07 “Don't try to open the mic device every time if SDL says it has none” · 8feeee61 2024-04 “Input: only check joystick input if a joystick actually exists” · 95adc87f 2024-04 “wifi: try ignoring MP frames if not engaging in MP comm”

### melonds-132 — Don't do filtering on the CPU path when a GPU path exists
Lens: gpu-rendering · Source: melonDS
Principle: Smooth image scaling in a CPU 2D painter is a per-pixel resample that can eat the whole frame budget. Turn it off for that path and only offer it where the GPU does it for free in texture sampling.
Why it is faster: —
Applies elsewhere: `image-rendering: pixelated` versus a scaled canvas, disabling anti-aliasing in software fallback renderers, Android `Paint.FILTER_BITMAP_FLAG` only on hardware layers.
Watch out: —
Seen in melonDS: `ScreenPanelNative::paintEvent` set `QPainter::SmoothPixmapTransform` whenever screen filtering was configured, which "can bring everything to a crawl". The hint was removed, and `actScreenFiltering` is now enabled only when the OpenGL display is active. 758db2b9 2023-05 “disable screen filtering when drawing with QT ...”

### melonds-133 — Read-only operations must not invalidate caches
Lens: laziness · Source: melonDS
Principle: If one routine handles both save and load and ends with "reset caches", saving throws away valid cached state, and the next frame pays to rebuild it (and may flicker). Invalidate only on the path that actually changes the underlying data.
Why it is faster: —
Applies elsewhere: ORM `flush()` that clears the identity map even for reads, serialising a scene graph that marks every node dirty, backup jobs that bust a CDN cache.
Watch out: —
Seen in melonDS: `GPU::DoSavestate` ended with an unconditional `ResetVRAMCache()`. It is now `if (!file->Saving) ResetVRAMCache();`. Frontends that save states every frame, for rewind, no longer rebuild the flattened VRAM each time. 740305cc 2024-01 “Don't reset the VRAM cache if saving a state (#1944)”

### melonds-134 — Two-level tile binning with bitmasks
Lens: gpu-rendering · Source: melonDS
Principle: A tiled GPU rasteriser first assigns primitives to screen tiles so each tile only shades what touches it. Binning in two stages, coarse tiles and then fine tiles only for primitives that hit the coarse tile, cuts the bounds tests sharply. Storing the results as bitmasks (one bit per primitive in a group of 32) keeps the bins compact and lets consumers walk them with `findLSB`.
Why it is faster: —
Applies elsewhere: tiled and clustered forward lighting, GPU particle culling, spatial hashing for collision broad-phase, vector-graphics renderers such as Vello and Pathfinder.
Watch out: —
Seen in melonDS: The `BinCombined` compute shader runs one workgroup per 32 polygons × coarse tile. Each lane tests one polygon against the coarse tile, and the hits are merged into `mergedMaskShared` with `atomicOr`. A ballot version is written but disabled. Each lane then takes one fine tile and tests only the surviving polygons, and writes `binnedMask` plus a coarse bit, `atomicOr(BinningMaskAndOffset[BinningCoarseMaskStart+...])`. The rasteriser's `ProcessCoarseMask` walks coarse bits, then fine bits, with `findLSB`. 043244a5 2024-05 “Compute shader renderer (#2041)” · 5df83c97 2024-05 “bump up compute shader renderer tile amount factor to 16 ...”

### melonds-135 — Bucket GPU work by pipeline state with a counting sort, then dispatch per bucket
Lens: gpu-rendering · Source: melonDS
Principle: Branching on material or blend mode inside one shader causes divergence and register pressure. Instead, count work items per state variant with atomics, turn the counts into offsets with a prefix pass, and scatter the items into contiguous ranges. Then bind each specialised shader once and dispatch exactly its range.
Why it is faster: —
Applies elsewhere: material sorting in deferred and visibility-buffer renderers, wavefront path tracing that sorts rays by material, sort-by-key before batched kernels in GPU databases.
Watch out: —
Seen in melonDS: `ComputeRenderer::RenderFrame` groups polygons into up to 256 `Variant`s by texture, sampler and blend mode. It reuses the previous or per-texture last variant to keep the CPU-side search short. While binning, each tile/polygon pair gets `inVariantOffset = atomicAdd(VariantWorkCount[variant].z, 1)`. `CalculateWorkListOffset` turns the counts into `SortedWorkOffset[]`, and `SortWork` scatters the work descriptors. The raster loop then binds one shader per variant. 043244a5 2024-05 “Compute shader renderer (#2041)”

### melonds-136 — Let the GPU size its own dispatches (indirect dispatch)
Lens: gpu-rendering · Source: melonDS
Principle: If the CPU needs a count the GPU just produced, it must stall on a readback. Have the shader write the dispatch dimensions into a buffer, and issue `glDispatchComputeIndirect`/`vkCmdDispatchIndirect` on it. The pipeline stays full and no work is spawned for empty buckets.
Why it is faster: —
Applies elsewhere: GPU-driven culling with `DrawIndirect`, stream compaction followed by a variable-size pass, WebGPU `dispatchWorkgroupsIndirect`.
Watch out: —
Seen in melonDS: `BinResultHeader` holds `VariantWorkCount[MaxVariants*4]` and `SortWorkWorkCount[4]`, laid out as dispatch arguments. The binning shader fills them, and the CPU calls `glDispatchComputeIndirect(offsetof(BinResultHeader, SortWorkWorkCount))` and `glDispatchComputeIndirect(offsetof(BinResultHeader, VariantWorkCount) + i*16)` per variant without ever reading the counts back. 043244a5 2024-05 “Compute shader renderer (#2041)”

### melonds-137 — Specialised shader permutations plus redundant-state elision
Lens: gpu-rendering · Source: melonDS
Principle: One "uber" shader that branches on mode flags pays for the worst-case register count everywhere. Compile one program per meaningful combination of flags. When drawing, remember the last bound program, texture and sampler, and skip API calls that wouldn't change anything. Driver state changes are often more expensive than the draws themselves.
Why it is faster: —
Applies elsewhere: Unity/Unreal shader keywords, WebGL wrappers that cache `gl.useProgram`, three.js's program cache, state trackers in Vulkan abstraction layers.
Watch out: —
Seen in melonDS: `GPU3D_Compute.h` keeps `ShaderRasteriseNoTexture[2]`, `...UseTextureDecal[2]`, `...Modulate`, `...Toon`, `...Highlight` and `...ShadowMask`, each indexed by W-buffer mode. The variant loop chooses from `shadersNoTexture[blendMode]`/`shadersUseTexture[blendMode]`, and calls `glUseProgram`, `glBindTexture` and `glBindSampler` only when `prevShader`/`prevTexture`/`prevSampler` differ. 043244a5 2024-05 “Compute shader renderer (#2041)”

### melonds-138 — Bake runtime configuration into shader source as constants
Lens: compiler-build · Source: melonDS
Principle: Values that are fixed for a whole session, such as resolution, tile size or buffer strides, can be injected as `#define`s before compiling. The shader compiler can then unroll loops, fold address math and size shared memory exactly. Picking those constants to suit the configuration, for example larger tiles at higher resolution, keeps per-tile overhead in proportion.
Why it is faster: —
Applies elsewhere: Vulkan specialization constants, CUDA templates over block size, WebGL shader strings with `#define` headers, JIT-compiled SQL with constants folded in.
Watch out: —
Seen in melonDS: `ComputeRenderer::CompileShader` prepends `#define ScreenWidth`, `MaxWorkTiles`, `TileSize`, `CoarseTileArea`, `ClearCoarseBinMaskLocalSize` and `const int CoarseTileCountY`. `SetRenderSettings` now scales `TileSize` from 8 up to 32 with the internal resolution (`TileScale = GetMSBit(2*ScaleFactor/9) << 1`) and adjusts the coarse grid and local sizes to match. Before, these were `static constexpr` 8×(8×4) at every scale. 77774538 2025-11 “Implement tile size scaling for the compute renderer (#2065)” · 043244a5 2024-05 “Compute shader renderer (#2041)”

### melonds-139 — Cache compiled program binaries keyed by source hash, and compile incrementally
Lens: io · Source: melonDS
Principle: Shader compilation can take seconds. Hash the final source string and store the driver's program binary under that hash on disk. On the next run, load the binary instead of compiling. Programs that are missing get compiled one per frame or step, so the UI never freezes for the whole set.
Why it is faster: —
Applies elsewhere: `glGetProgramBinary` and Vulkan pipeline caches, Python `.pyc`, ccache/sccache, V8 code caching, browser WASM module caches.
Watch out: —
Seen in melonDS: `OpenGLSupport.cpp` keeps `std::unordered_map<u64, ShaderCacheEntry> ShaderCache` keyed by `XXH3` of the source, persisted to a `shadercache` file with a magic and version. `ComputeRenderer::ShaderCompileStep(current, count)` compiles one of 33 programs per call, and `NeedsShaderCompile()` lets the frontend show progress. The commit says this made "shader compile times tolerable on Wintel". 043244a5 2024-05 “Compute shader renderer (#2041)”

### melonds-140 — Decode-once texture cache invalidated by dirty-block bitmaps
Lens: laziness · Source: melonDS
Principle: Converting textures from a source format on every use wastes work when the source memory rarely changes. Cache the decoded result keyed by its parameters. Track source writes in a coarse dirty bitmap, and per frame evict only the entries whose address range overlaps a dirty block. A content hash can confirm the bytes really changed.
Why it is faster: —
Applies elsewhere: glyph atlases in text renderers, decoded-image caches in browsers, compiled-template caches invalidated by file mtime, CPU page-table dirty bits used by incremental GCs.
Watch out: —
Seen in melonDS: `Texcache::Update()` gets dirty bitfields via `GPU.VRAMDirty_Texture.DeriveState(...)` and walks `Cache`, an `unordered_map<u64, TexCacheEntry>`. It erases an entry only if `GetRangedBitMask(...) & dirty[...]` hits its texture or palette range. `MaskedHash()` XXH64-hashes the source, handling address wrap-around. Decoders such as `ConvertNColorsTexture` read 16 bits at a time and consume indices with `val >>= colorBits`. 043244a5 2024-05 “Compute shader renderer (#2041)” · 58ab3321 2024-10 “handle address wrap around in texture cache ... also slightly optimise paletted texture conversion”

### melonds-141 — Pool GPU resources in size-class free lists
Lens: allocation · Source: melonDS
Principle: Creating and destroying GPU textures or buffers is slow and fragments driver memory. Keep freed resources on a free list per size class (here, per log2 width × log2 height), and pop from the matching list before allocating anything new.
Why it is faster: —
Applies elsewhere: slab allocators, render-target pools in game engines, `sync.Pool` in Go, buffer pools in database engines.
Watch out: —
Seen in melonDS: `Texcache` keeps `std::vector<TexArrayEntry> FreeTextures[8][8]`, indexed by `WidthLog2`/`HeightLog2`. Each entry is a layer of a shared texture array. An evicted entry does `FreeTextures[w][h].push_back(entry.Texture)`, and a new texture takes from that list first. `TexLoader.UploadTexture` then fills one layer instead of creating a texture. 043244a5 2024-05 “Compute shader renderer (#2041)”

### melonds-142 — Reserve address space up front, commit views on demand
Lens: platform · Source: melonDS
Principle: To map emulated or arena memory at fixed offsets, first reserve a large contiguous virtual range with no access (`PROT_NONE` mmap, or Windows placeholders), then map file views into slots of it as needed. This avoids races with other allocations, lets several instances coexist, and makes "unmap" just a remap to `PROT_NONE`, which keeps the reservation.
Why it is faster: —
Applies elsewhere: WebAssembly runtimes' 4–8 GB guard regions, V8 pointer-compression cages, Java heap reservation (`-Xmx`), the Go runtime arena, database buffer pools mapped with `MAP_FIXED`.
Watch out: —
Seen in melonDS: `ARMJIT_Memory` reserves `VirtmemAreaSize = 2*4GB + MemoryTotalSize` per instance. On Windows it uses `VirtualAlloc2` placeholders, splitting them on `MapIntoRange` (`MapViewOfFile3(..., MEM_REPLACE_PLACEHOLDER)`) and coalescing them on `UnmapFromRange` (`MEM_COALESCE_PLACEHOLDERS`). On POSIX, unmapping is `mmap(dst, size, PROT_NONE, MAP_ANON|MAP_PRIVATE|MAP_FIXED, -1, 0)`. Fastmem also became page-size-aware (4 KB/16 KB). 99ce9599 2024-11 “Multiinstance jit (#2201)” · f0503a6a 2024-11 “fix 4kb page check oops”

### melonds-143 — Bitmask slot allocator over a static region, with a heap fallback
Lens: allocation · Source: melonDS
Principle: For a few large, fixed-size blocks, carve a static buffer into N slots and track free slots in one integer. Allocation is `ctz(free)` and freeing is an OR, both under one lock. A static buffer also sits close to the program image, which keeps short relative jumps and calls in range. Fall back to `mmap` only once the slots run out.
Why it is faster: —
Applies elsewhere: connection-slot tables in embedded TCP stacks, fixed thread-pool stacks, Linux IDR/bitmap allocators, GPU descriptor heaps.
Watch out: —
Seen in melonDS: `ARMJIT_Global::AllocateCodeMem()` hands out one of 4 `CodeMemorySliceSize` slices from the static `u8 CodeMemory[]`, 16 KB-aligned and made executable once with `mprotect`/`VirtualProtect`. It uses `slice = __builtin_ctz(AvailableCodeMemSlices)`, and `FreeCodeMem` sets the bit back. Only a fifth concurrent JIT instance gets a `mmap`/`VirtualAlloc`'d slice. The fault handler is registered once, ref-counted. 99ce9599 2024-11 “Multiinstance jit (#2201)”

### melonds-144 — Diff a state snapshot to decide which stages to redo, and batch the unchanged spans
Lens: laziness · Source: melonDS
Principle: When state can change partway through a stream of work, XOR the current registers against the last snapshot. Each bit group in the diff maps to the pipeline stages it affects, so you redo only those stages. Accumulate items until a "critical" change occurs, then process the whole run in one call.
Why it is faster: —
Applies elsewhere: React-style reconciliation, immediate-mode UI renderers batching draws until a scissor or texture changes, sprite batchers flushing on material change, incremental build systems.
Watch out: —
Seen in melonDS: `GLRenderer2D::UpdateAndRender(line)` computes `dispcnt_diff = GPU2D.DispCnt ^ DispCnt`, `bgcnt_diff[]` and `layer_diff`, and masks them into `layer_pre_dirty` (per-layer prerender) and `comp_dirty`. Only on `comp_dirty` does it draw `RenderScreen(LastLine, line)`, covering every scanline since the last change in one pass. Non-critical per-line values go into a streamed `ScanlineConfigUBO`. ba317e2e 2026-01 “blackmagic3 - brand new OpenGL 2D renderer (#2553)”

### melonds-145 — Upload only the dirty sub-ranges of a GPU texture
Lens: gpu-rendering · Source: melonDS
Principle: Re-uploading a whole texture each frame wastes bus bandwidth when only a few blocks changed. Keep a per-block dirty bitmap of the CPU-side copy, scan it for runs of set bits, and issue one `glTexSubImage` per run.
Why it is faster: —
Applies elsewhere: partial vertex-buffer updates, tile-based remote desktop protocols (VNC dirty rects), WebGL canvas texture streaming, incremental snapshots of file blocks.
Watch out: —
Seen in melonDS: In `GLRenderer2D`, VRAM writes set bits in `VRAMDirty_ABG` and similar maps at `VRAMDirtyGranularity = 512`. Per update, `bgDirty.CheckRange(0, dirtybits)` gates any upload. The loop over `bgDirty.Data[i]` skips clean 64-bit words and uploads only the dirty rows of `VRAMTex_BG`. Extended palettes are handled the same way through `CheckRange`. ba317e2e 2026-01 “blackmagic3 - brand new OpenGL 2D renderer (#2553)”

### melonds-146 — Pack per-row parameters into spare channels of the data the shader already reads
Lens: gpu-rendering · Source: melonDS
Principle: If a parameter varies per row and is passed as a uniform, you need one draw per row or you get it wrong. Write the parameter into unused bits of a per-row texel that the shader already fetches. Mid-frame changes then come through for free, with no extra uploads or draws.
Why it is faster: —
Applies elsewhere: encoding material IDs in G-buffer alpha, per-instance data in texture rows, metadata in spare pixel bits of video frames.
Watch out: —
Seen in melonDS: The OpenGL compositor set the 3D layer's X scroll once per frame with `glUniform1i(Comp3DXPosLoc, ...)`. `GPU2D_Soft::DrawScanline` now writes `(xpos << 24) | ((xpos & 0x100) << 15)` into the per-line control word (`dst[256*3]`), and the shader rebuilds it from `mbright.a`/`mbright.b`. The uniform was removed. e234385c 2024-06 “OpenGL: add support for changing BG0HOFS midframe. fixes #2072”

### melonds-147 — Resample by recording deltas only when the signal changes
Lens: algorithms · Source: melonDS
Principle: A band-limited step synthesiser records each change of the input as a delta at a fractional timestamp. It then produces output at any rate by summing band-limited step kernels. When the input holds steady, it costs nothing, and you get high-quality resampling without running a filter per input sample.
Why it is faster: —
Applies elsewhere: chiptune and console audio (blargg's blip_buf), PWM-to-PCM conversion in firmware, event-driven signal simulation, sparse time-series downsampling.
Watch out: —
Seen in melonDS: `SPU::Mix` used to linearly interpolate into a 32 kHz/48 kHz output, and the frontend resampled again. It now does `if (output[0] != OutputLastSamples[0]) blip_add_delta(BlipLeft, BlipTimer, output[0] - OutputLastSamples[0]);`, and `blip_read_samples` produces audio at whatever rate the frontend requests. The frontend resampler was deleted. b7bfb539 2025-09 “Add high-quality resampling using blip-buf to the core (#2412)”

### melonds-148 — One power-of-two ring buffer instead of back buffer plus copy
Lens: memory-layout · Source: melonDS
Principle: A double-buffered scheme, where the producer fills a back buffer and copies it into the consumer FIFO at a boundary, adds a full copy and a boundary's worth of latency. Writing straight into a single ring buffer, with indices wrapped by `& (size-1)`, removes both. Draining it by fill level rather than at frame boundaries keeps latency steady and the buffer small.
Why it is faster: —
Applies elsewhere: audio callbacks in JUCE and PortAudio, kernel pipe and io_uring rings, lock-free SPSC queues, UART RX buffers.
Watch out: —
Seen in melonDS: `SPU::Mix` wrote into `OutputBackbuffer`, and `TransferOutput()` copied it into `OutputFrontBuffer` once per frame. Now `Mix` writes `OutputBuffer[OutputBufferWritePos++]` under the lock, with `&= 2*OutputBufferSize-1`. Later, blip output is pulled every `512*128` cycles (`BufferAudio()`) instead of at `EndFrame`, with the blip buffer shrunk from 8192 to 512. 3263ab11 2025-07 “Make SPU audio single-buffered (audio latency improvement) (#2286)” · 0e966353 2025-11 “buffer audio at a regular interval instead of at the end of frames”

### melonds-149 — Replace interpretation of known programs with native implementations, identified by hash
Lens: algorithms · Source: melonDS
Principle: If a guest only ever runs a handful of known programs on a co-processor, hash the program image when it loads. For known hashes, run a native implementation of its behaviour instead of interpreting it instruction by instruction, and fall back to the interpreter for anything unknown. That replaces millions of interpreted cycles with direct library calls.
Why it is faster: —
Applies elsewhere: HLE of audio microcode in console emulators, intrinsic recognition in compilers (memcpy loops become `memcpy`), browser fast paths for known polyfills, matching driver shaders by hash and substituting optimised ones.
Watch out: —
Seen in melonDS: `DSi_DSP::StartDSPHLE()` CRC32s the 32 KB program in NWRAM B and matches it: `0x7867C94B` is the DSi sound app's AAC ucode, followed by the SDK AAC v0–v4, G711 v0–v3 and graphics ucodes. It then creates `DSP_HLE::AACUcode`, `G711Ucode` or `GraphicsUcode`, using faad2 for AAC, instead of running the Teakra interpreter. 1a9a1351 2025-08 “DSP HLE (#2404)”

### melonds-150 — Share memory through the owner's mapping table instead of a mirrored flat copy
Lens: memory-layout · Source: melonDS
Principle: Keeping a second component's flat buffer in sync with banked memory means copying on every remap, and the two can drift apart. Give the component read and write callbacks that resolve addresses through the owner's existing bank-pointer table. Accesses are a shift, a mask and a load, and nothing is copied.
Why it is faster: —
Applies elsewhere: zero-copy views (`Span<T>`, `DataView`) over shared buffers, scatter-gather I/O, page tables instead of shadow copies in hypervisors.
Watch out: —
Seen in melonDS: Teakra kept its own flat DSP memory, which `DSi_DSP::OnMBKCfg` patched on every NWRAM bank write. Teakra now calls `DSPRead16`/`DSPWrite16`, which do `u8* ptr = DSi::NWRAMMap_B[2][(addr >> 15) & mask]; return ptr ? *(u16*)&ptr[addr & 0x7FFF] : 0;`. The MBK loop also caches the decoded `mVal` and uses shifts instead of `/4` and `%4`. cdd05c10 2022-10 “more DSP unfucking: make Teakra directly use melonDS's NWRAM banks ...”

### melonds-151 — Use a type tag and static_cast instead of dynamic_cast
Lens: branching-fast-paths · Source: melonDS
Principle: `dynamic_cast` walks RTTI structures at runtime and can do string compares across shared libraries. If the class hierarchy is closed and an enum already records the concrete type, compare the enum and `static_cast`, which is one integer compare.
Why it is faster: —
Applies elsewhere: LLVM's `isa<>`/`dyn_cast<>` using `classof` kinds, AST node kinds in compilers, tagged unions in game entity systems.
Watch out: —
Seen in melonDS: `ARMJIT_Memory::RemapNWRAM` did `auto* dsi = dynamic_cast<DSi*>(&NDS); assert(dsi);`. It now does `if (NDS.ConsoleType == 0) return; auto* dsi = static_cast<DSi*>(&NDS);`. 2a35af5b 2023-12 “Soften restrictions around `ARMJIT_Memory::RemapNWRAM`”

### melonds-152 — Set optimisation defaults in the build system without clobbering user flags
Lens: compiler-build · Source: melonDS
Principle: Release builds benefit from the highest optimisation level, and debug builds from `-Og`, which optimises but stays debuggable. Set these as the initial values of the toolchain flags rather than string-patching the final flags, so packagers and users can still override them. Turn LTO on by default for release, and back off only for compilers known to break.
Why it is faster: —
Applies elsewhere: Cargo `[profile.release] lto = true`, Meson `buildtype`, distribution-friendly CMake presets, Go's `-ldflags` defaults.
Watch out: —
Seen in melonDS: `CMakeLists.txt` used to edit `CMAKE_CXX_FLAGS_RELEASE` in place (`-O2`→`-O3`). `cmake/DefaultBuildFlags.cmake`, loaded through `CMAKE_USER_MAKE_RULES_OVERRIDE`, now sets `CMAKE_CXX_FLAGS_RELEASE_INIT` to `-O3` and `CMAKE_CXX_FLAGS_DEBUG_INIT "-g -Og"` for GCC. `ENABLE_LTO_RELEASE` defaults ON and is forced OFF on GCC 15 to work around an internal compiler error. Windows builds add `WIN32_LEAN_AND_MEAN NOMINMAX` to trim `windows.h` parsing. 474bf6e7 2024-05 “Set default optimization flags less intrusively” · ee2c6cc7 2024-05 “actually add the cmake script too” · 7c1d2a64 2024-11 “Set WIN32_LEAN_AND_MEAN ...”

### melonds-153 — Direct-mapped, tagged lookup table for translated code
Lens: jit-interpreters · Source: melonDS
Principle: To find compiled code for a guest address, a hash map adds hashing and probing on every block exit. A flat array per memory region, indexed by address offset, with each 64-bit entry holding a tag (the address) and the code offset, turns the lookup into one load and one compare. Code memory is far smaller than address space, so the arrays stay affordable.
Why it is faster: —
Applies elsewhere: inline caches in JS engines, direct-mapped TLBs, dispatch tables in bytecode VMs keyed by PC, CPU branch target buffers.
Watch out: —
Seen in melonDS: `ARMJIT.h` declares `u64 FastBlockLookupMainRAM[MainRAMMaxSize/2]` and the same for ITCM, SWRAM, VRAM, BIOS and NWRAM, one slot per halfword. `LookUpBlock` is `u64* e = &entries[offset/2]; if (*e >> 32 == (addr | num)) return AddEntryOffset((u32)*e);`. `ARMv5::Execute<JIT>` caches the current region's base, start and size in `FastBlockLookup*` and calls `SetupExecutableRegion` only when the PC leaves that range. current code: src/ARMJIT.h, src/ARMJIT.cpp, src/ARM.cpp

### melonds-154 — Resurrect invalidated compiled code when its source bytes hash the same
Lens: jit-interpreters · Source: melonDS
Principle: Self-modifying or reloaded code invalidates compiled blocks, but often the same bytes come back, for example overlays reloaded or code copied to the same spot. Retire invalidated blocks into a map keyed by a hash of their instructions. On recompile, if the hash, start address, literal hash and address ranges all match, reuse the old machine code instead of compiling again.
Why it is faster: —
Applies elsewhere: content-addressed build caches (Bazel, ccache), V8 code caching across reloads, shader caches keyed by bytecode hash, memoised template compilation.
Watch out: —
Seen in melonDS: `ARMJIT::RetireJitBlock` puts blocks into `std::unordered_map<u32, JitBlock*> RestoreCandidates`, keyed by `InstrHash`. `CompileBlock` computes `XXH3_64bits` of the instruction words and of the literal values. It sets `mayRestore = prevBlock->StartAddr == blockAddr && prevBlock->LiteralHash == literalHash` plus a per-range comparison, and on a match re-registers the old entry without emitting code. current code: src/ARMJIT.cpp

### melonds-155 — Fine-grained code index plus page protection for invalidation
Lens: jit-interpreters · Source: melonDS
Principle: To catch writes to code that has been translated, mark each small chunk (512 bytes here) that contains translated code in a bitmap, and write-protect the host pages it lives on. Ordinary data writes run at full speed. Only a write to a protected page faults or takes a slow path, and that path consults the bitmap to invalidate just the blocks that are affected.
Why it is faster: —
Applies elsewhere: QEMU's TB invalidation with page flags, generational GCs' card marking and write barriers, copy-on-write snapshots, file watchers with coarse then fine granularity.
Watch out: —
Seen in melonDS: `ARMJIT` keeps `AddressRange CodeIndexMainRAM[MainRAMMaxSize/512]` per region, each holding the blocks that touch it plus a bitmask of code words. `ARMJIT_Memory::SetCodeProtection` switches a page between `memstate_MappedRW` and `memstate_MappedProtected`, which `SetCodeProtectionRange` applies with `mprotect`/`VirtualProtect`. Mirrors are protected as they are mapped (`hasCode ? memstate_MappedProtected : memstate_MappedRW`). current code: src/ARMJIT.h, src/ARMJIT.cpp, src/ARMJIT_Memory.cpp

### melonds-156 — Optimistic fast-path memory access with fault-driven backpatching
Lens: jit-interpreters · Source: melonDS
Principle: Emit guest loads and stores as raw host accesses into a reserved window. Unmapped or protected pages fault. The signal handler then either maps the page lazily and retries, or patches that one instruction into a call to the slow handler and resumes. Hot accesses keep a single instruction, and rare MMIO accesses pay once to be rewritten.
Why it is faster: —
Applies elsewhere: WebAssembly bounds checks done with guard pages, Dolphin and other JIT fastmem, JVM implicit null checks, userfaultfd-based lazy loading.
Watch out: —
Seen in melonDS: `ARMJIT_Memory::FaultHandler` checks `IsJITFault(FaultPC)`. If `memStatus[addr >> PageShift] == memstate_Unmapped`, it tries `MapAtAddress`. Otherwise it calls `RewriteMemAccess(pc)`, which looks up `LoadStorePatches[pc]`, emits `CALL(patch.PatchFunc)` and pads with NOPs, toggling W^X around the write. Store slow paths are prebuilt per console/CPU/size/register as `PatchedStoreFuncs[...]`. current code: src/ARMJIT_Memory.cpp, src/ARMJIT_x64/ARMJIT_LoadStore.cpp, src/ARMJIT_x64/ARMJIT_Compiler.cpp

### melonds-157 — Map one backing file at many addresses to get mirrors for free
Lens: jit-interpreters · Source: melonDS
Principle: When the same physical memory has to appear at several addresses (mirrors, banked windows), map the same shared-memory object at every address. The MMU then provides the aliasing, so an access through any address is a plain load with no address masking or table lookup.
Why it is faster: —
Applies elsewhere: "magic" ring buffers mapped twice back to back, a shared-memory IPC segment mapped by several processes, Linux `memfd` plus double mapping for W^X JIT code.
Watch out: —
Seen in melonDS: `ARMJIT_Memory::MapIntoRange` does `mmap(dst, size, PROT_READ|PROT_WRITE, MAP_SHARED|MAP_FIXED, MemoryFile, offset)` on POSIX, `MapViewOfFile3` on Windows and `svcMapProcessMemory` on Switch. It maps the right slice of the single `MemoryFile` into the ARM9 and ARM7 fastmem windows for every mirror of main RAM, WRAM, VRAM and the rest. `Mapping::Unmap` removes the mappings when the bank configuration changes. current code: src/ARMJIT_Memory.cpp

### melonds-158 — Detect side-effect-free spin loops and fast-forward time
Lens: jit-interpreters · Source: melonDS
Principle: A loop that writes no memory and whose registers don't depend on the previous iteration can't exit until something outside changes, such as an interrupt or a timer. Prove that statically when compiling and mark the back-branch. At runtime, a hit on it skips straight to the next scheduled event instead of spinning.
Why it is faster: —
Applies elsewhere: the OS idle loop executing `HLT`/`WFI`, Dolphin's idle skipping, replacing busy-wait polling with futex or eventfd waits, game loops sleeping until the next tick.
Watch out: —
Seen in melonDS: `IsIdleLoop()` in `ARMJIT.cpp` rejects memory writes, MSR/MRC and inner branches, and tracks `regsWrittenTo`/`regsDisallowedToWrite` so that no register read before being written is written later. A static back-branch inside the block that passes gets `branch_IdleBranch`. `Execute<JIT>` then sets `NDS.ARM9Timestamp = NDS.ARM9Target` when `IdleLoop` is set. The back-jump scan had compared `instrs[i]` instead of `instrs[j]`, so detection was broken until 2023. current code: src/ARMJIT.cpp, src/ARM.cpp · fix e8967a93 2023-04 “fix JIT backjump detection”

### melonds-159 — Register allocation by remaining-use ranking within a block
Lens: jit-interpreters · Source: melonDS
Principle: In a linear block, a simple allocator does well if, before each instruction, it scans the rest of the block. Registers never used again get freed immediately (and written back only if dirty). When it has to evict, it picks the register with the fewest remaining uses. This is a cheap approximation of Belady's optimal replacement.
Why it is faster: —
Applies elsewhere: linear-scan allocators in baseline JITs (V8 Sparkplug/Maglev, LuaJIT), the lookahead eviction policy of tile caches, CDN eviction with future-access hints.
Watch out: —
Seen in melonDS: `RegisterCache::Prepare(thumb, i)` loops `for (j = i; j < InstrsCount; j++)` to build `futureNeeded` and `ranking[reg]++`. It unloads `LoadedRegs & ~futureNeeded`, and while space is short it evicts the loaded, not-currently-needed register with the lowest rank. `UnloadRegister` calls `SaveReg` only if the bit is set in `DirtyRegs`. Guest-literal values can also be held as known constants (`LiteralsLoaded`). current code: src/ARMJIT_RegisterCache.h

### melonds-160 — Backward liveness for condition flags: compute only flags someone reads
Lens: jit-interpreters · Source: melonDS
Principle: Guest arithmetic sets condition flags that are usually overwritten before anything reads them. Walk the block backwards from each flag reader and from the block exit, marking only the producers whose flags are actually consumed. Every other flag computation is dropped from the generated code.
Why it is faster: —
Applies elsewhere: lazy flag evaluation in QEMU, Rosetta and Bochs, dead-store elimination in compilers, demand-driven attribute computation in spreadsheets.
Watch out: —
Seen in melonDS: `FloodFillSetFlags(instrs, start, flags)` walks backwards and ORs `instrs[j].SetFlags |= instrs[j].Info.WriteFlags & flags`, clearing satisfied bits and stopping when none remain. It runs from each instruction with `ReadFlags`, using all of NZCV when a following conditional branch is involved, and from the block end with `0xF`. Backends emit flag computation only for bits in `SetFlags`. current code: src/ARMJIT.cpp

### melonds-161 — Treat constants loaded from code as immediates, guarded by invalidation
Lens: jit-interpreters · Source: melonDS
Principle: PC-relative loads from literal pools inside code almost always read constants. The translator can read the value at compile time and emit an immediate, removing the memory access. Because the literal lives in a code-indexed region, a later write to it invalidates the block, so correctness holds. Addresses that turn out to be written are blacklisted.
Why it is faster: —
Applies elsewhere: constant folding of `static final` fields in JVM JITs (with deoptimisation), V8 constant-field tracking, link-time constant propagation.
Watch out: —
Seen in melonDS: `DecodeLiteral()` recognises `LDR Rd, [PC, #imm]` in ARM and Thumb. When `LiteralOptimizations` is on, the compiler loads the value as a known constant and includes the literal's range in the block's address ranges and `literalHash`. `TinyVector<u32> InvalidLiterals` records literal addresses that were later written, so they are never treated as constant again. current code: src/ARMJIT.cpp, src/ARMJIT.h

### melonds-162 — Follow statically known branches inside a translation block
Lens: jit-interpreters · Source: melonDS
Principle: Ending a block at every branch costs a dispatcher round trip. If the target is a constant, keep translating at the target within the same block. For a conditional branch, follow one side and emit an exit for the other. Blocks get longer, and more code is optimised together for register allocation and flag liveness.
Why it is faster: —
Applies elsewhere: superblocks and trace formation in Dynamo and LuaJIT, basic-block merging in compilers, the trace cache of Pentium 4 and Netburst.
Watch out: —
Seen in melonDS: With `BranchOptimizations`, `CompileBlock` calls `DecodeBranch()`. When it is static it sets `branch_StaticTarget`, and when the target is within the block budget it continues fetching there and tags the instruction `branch_FollowCondTaken`/`branch_FollowCondNotTaken`. The flag-liveness pass treats followed conditional branches as needing all flags. current code: src/ARMJIT.cpp

### melonds-163 — Condition evaluation through a 16×16 truth-table bitmask
Lens: lookup-tables · Source: melonDS
Principle: Testing a 4-bit condition code against 4 flag bits with a switch is a branch that mispredicts often. Precompute `table[cond]` as a 16-bit mask over all flag combinations, and test it with one shift and an AND.
Why it is faster: —
Applies elsewhere: permission matrices (`allowed[role] & (1 << action)`), character-class tables in lexers, state-machine transition masks.
Watch out: —
Seen in melonDS: `ARM::CheckCondition(code)` is `if (code == 0xE) return true; if (ConditionTable[code] & (1 << (CPSR>>28))) return true;`, with `static const u32 ConditionTable[16]` in `ARM.cpp`. The interpreter loops call it for every ARM instruction. current code: src/ARM.h, src/ARM.cpp

### melonds-164 — Instruction dispatch through a table indexed by extracted opcode bits
Lens: branching-fast-paths · Source: melonDS
Principle: Decoding an instruction with nested ifs and switches costs several dependent branches. Take the bits that tell instructions apart, build a dense index from them, and call through a table of specialised handlers, one per addressing mode and flag variant. The result is one indirect call per instruction.
Why it is faster: —
Applies elsewhere: bytecode VMs (CPython's opcode table, Lua), packet classifiers, protocol parsers keyed by message type.
Watch out: —
Seen in melonDS: `ARMv5::Execute<Interpreter>` does `u32 icode = ((CurInstr >> 4) & 0xF) | ((CurInstr >> 16) & 0xFF0); ARMInterpreter::ARMInstrTable[icode](this);`, which is 4096 entries. Thumb uses `THUMBInstrTable[(CurInstr >> 6) & 0x3FF]`. The JIT's fallback likewise indexes `InterpretARM[ARMInstrInfo::ak_Count]` by the pre-decoded kind. current code: src/ARM.cpp, src/ARMInterpreter.cpp, src/ARMJIT.cpp

### melonds-165 — Per-page cost tables and a cached region pointer for instruction fetch
Lens: memory-layout · Source: melonDS
Principle: If every fetch had to classify its address to find the backing memory and its access cost, the hot loop would carry a decoder. Precompute a table of costs per 4 KB page, and cache a pointer to the current code region. Re-resolve only when a branch crosses into a different region. Where it's free, fetch two 16-bit instructions with one 32-bit read.
Why it is faster: —
Applies elsewhere: software TLBs in emulators and interpreters, segment-base caching in VMs, read-ahead in stream decoders.
Watch out: —
Seen in melonDS: `ARMv5` has `u8 MemTimings[0x100000][4]`, indexed by `addr >> 12`, and `MemRegion CodeMem`. `JumpTo` does `if (newregion != oldregion) SetupCodeMem(addr);` and sets `RegionCodeCycles = MemTimings[addr>>12][0]`. In Thumb mode it keeps the second halfword: `if (R[15] & 0x2) { NextInstr[1] >>= 16; CodeCycles = 0; }`, the "two-opcodes-at-once fetch". The ARM7 uses `ARM7MemTimings[addr >> 15]`. current code: src/ARM.cpp, src/ARM.h

### melonds-166 — Bitmask-guarded event scheduler with a coalescing margin
Lens: algorithms · Source: melonDS
Principle: For a small, fixed set of event types, an array of timestamps plus a bitmask of active entries is faster than a priority queue. Scanning skips inactive slots and stops as soon as the remaining mask is zero. When computing the next run target, if an event falls just past the maximum slice, extend the slice to include it rather than running a tiny extra slice.
Why it is faster: —
Applies elsewhere: timer wheels in kernels, RTOS ready-bitmaps (`ctz` over priorities), timer slack in Linux (`timerfd` coalescing), batching in game tick schedulers.
Watch out: —
Seen in melonDS: `NDS::RunSystem` and `NextTarget` loop `for (i...) { if (!mask) break; if (mask & 1) ...; mask >>= 1; }` over `SchedList[Event_MAX]` with `SchedListMask`. `NextTarget` returns `minEvent` if `minEvent < SysTimestamp + kMaxIterationCycles + kIterationCycleMargin`, otherwise the max. `NextTargetSleep` considers only SPU and RTC while the system is asleep. current code: src/NDS.cpp, src/NDS.h

### melonds-167 — Pipeline producer and consumer row by row with a counting semaphore
Lens: concurrency · Source: melonDS
Principle: If a renderer produces rows in order and a compositor consumes them in the same order, run the renderer on its own thread and have it post a counting semaphore after each finished row. The consumer waits on the semaphore once per row it needs. Both run concurrently with only one row of lag, and no per-row index exchange is needed.
Why it is faster: —
Applies elsewhere: progressive JPEG and video decode feeding display, streaming parsers feeding a renderer, GPU readback ring buffers, the Unix pipe model.
Watch out: —
Seen in melonDS: `SoftRenderer3D::RenderPolygons` runs `RenderScanline(y)` and then `ScanlineFinalPass(y-1)`, a deferred one-line post-process, and posts `Sema_ScanlineCount` after each one. `GetLine(line)` on the emulation thread does `Semaphore_Wait(Sema_ScanlineCount)` once per line. The comment explains that both go top to bottom, so no specific row needs waiting for. Frame start and end use `Sema_RenderStart`/`Sema_RenderDone`. current code: src/GPU3D_Soft.cpp, src/GPU3D_Soft.h

### melonds-168 — Skip regenerating output when every input is unchanged
Lens: laziness · Source: melonDS
Principle: If a frame's inputs (geometry, textures, palettes) are identical to the previous frame's, the output is too. Track a cheap "identical" flag as inputs are submitted, combine it with dirty flags for referenced memory, and reuse the last result instead of rendering.
Why it is faster: —
Applies elsewhere: React `memo`/`shouldComponentUpdate`, HTTP ETag/304, incremental compilation, damage-region tracking in compositors.
Watch out: —
Seen in melonDS: `SoftRenderer3D::RenderFrame` computes `FrameIdentical = !(textureChanged || texPalChanged) && GPU3D.RenderFrameIdentical`. When it is set, the threaded path just posts `Sema_ScanlineCount` 192 times, and the unthreaded path skips `ClearBuffers()` and `RenderPolygons()`, leaving the previous colour buffer in place. current code: src/GPU3D_Soft.cpp

### melonds-169 — Incremental fixed-point edge stepping and a linear fast path for perspective interpolation
Lens: algorithms · Source: melonDS
Principle: Rasterisers walk polygon edges by adding a precomputed fixed-point slope per scanline instead of dividing per row. For attribute interpolation, compute one perspective-correct factor per pixel and use it to interpolate every attribute linearly. When both endpoints have equal depth weight, skip the division entirely.
Why it is faster: —
Applies elsewhere: Bresenham and DDA line drawing, scanline polygon fill in embedded UIs, audio envelope generators, font rasterisers.
Watch out: —
Seen in melonDS: `SoftRenderer3D::Slope` computes `Increment = (x1-x0) * yrecip` once, and `Step()` does `dx += Increment`. X-major handling and anti-aliasing coverage come from `dx` and `Increment` bits. `Interpolator<dir>` sets `linear = true` when the W values are equal (and low bits allow it), then interpolates with no division. Otherwise it computes one `yfactor` per x and blends all attributes with it. current code: src/GPU3D_Soft.h

### melonds-170 — SWAR colour math: blend packed channels in one integer
Lens: simd · Source: melonDS
Principle: If channels sit in separate bytes of a 32-bit word with spare headroom bits, one integer multiply-add can process several channels at once (SIMD within a register). Mask the result to separate the fields and clamp per field. The blend costs a few ALU ops instead of unpacking and repacking each channel.
Why it is faster: —
Applies elsewhere: alpha blending in software compositors (`(rb*a) & 0xFF00FF`), pixman and Skia's scalar fallbacks, colour fades in LED firmware.
Watch out: —
Seen in melonDS: `GPU_ColorOp.h`: `ColorBrightnessUp` works on `rb = val & 0x3F003F` (red and blue together) and computes `rb += (((0x3F003F - rb) * factor + bias*0x010001) >> 4) & 0x3F003F`. Green is handled separately. `ColorBlend4`/`ColorBlend5` multiply masked fields in place without shifting them down. All of these are `static constexpr` and shared by the soft 2D renderer and `GPU_Soft.cpp`'s master brightness. current code: src/GPU_ColorOp.h, src/GPU2D_Soft.cpp, src/GPU_Soft.cpp

### melonds-171 — Template the per-pixel loops on rarely changing mode flags
Lens: compiler-build · Source: melonDS
Principle: Layer and sprite renderers check flags such as mosaic, window or BG mode per pixel or per tile even though these change per scanline at most. Making them template parameters produces separate tight loops without those branches, and the caller dispatches once per scanline.
Why it is faster: —
Applies elsewhere: image filters specialised by pixel format, `template<bool HasAlpha>` blitters, codec inner loops specialised by bit depth.
Watch out: —
Seen in melonDS: `GPU2D_Soft.h`: `template<u32 bgmode> DrawScanlineBGMode`, `template<bool mosaic> DrawBG_Text/Affine/Extended/Large`, and `template<bool window> DrawSprite_Normal/Rotscale/DrawSpritePixel`. Line buffers such as `BGOBJLine`, `OBJLine` and `WindowMask` are `alignas(8)`. current code: src/GPU2D_Soft.h, src/GPU2D_Soft.cpp

### melonds-172 — Keep a flattened copy of banked memory, rebuilt only for dirty blocks
Lens: laziness · Source: melonDS
Principle: When reads go through a configurable bank mapping, resolving every access is slow. Keep a flat linear copy for readers. Record writes in a dirty bitmap at a fixed granularity, and before reading, re-copy only the blocks that are dirty or whose mapping changed. Iterate the dirty bits with count-trailing-zeros so clean regions cost nothing.
Why it is faster: —
Applies elsewhere: shadow copies of device registers, write-combining caches, dirty-page tracking in live VM migration, incremental backup by changed blocks.
Watch out: —
Seen in melonDS: `GPU::WriteVRAM_*` sets `VRAMDirty[bank][addr / VRAMDirtyGranularity]`, with granularity 512. `VRAMDirty_Texture.DeriveState(VRAMMap_Texture, gpu)` merges bank dirtiness and mapping changes into a `NonStupidBitField`, whose iterator uses `__builtin_ctzll`. `MakeVRAMFlat_TextureCoherent(dirty)` re-copies only those blocks into `VRAMFlat_Texture`. The software 3D renderer, the texture cache and the GL 2D renderer all read from the flat copies. current code: src/GPU.h, src/GPU.cpp, src/NonStupidBitfield.h

### melonds-173 — Snapshot decoder state at the loop point to make looping free
Lens: laziness · Source: melonDS
Principle: Adaptive codecs such as ADPCM depend on all previous samples, so jumping back to a loop start would normally mean decoding again from the beginning. Save the decoder state (predictor and step index) when playback first passes the loop point, and restore it on every loop.
Why it is faster: —
Applies elsewhere: keyframes in video decoders, checkpoints in streaming parsers, saving PRNG state for replays, resumable compression contexts.
Watch out: —
Seen in melonDS: `SPUChannel::NextSample_ADPCM` records `ADPCMValLoop`/`ADPCMIndexLoop` when `Pos == (LoopPos<<1)`, and from the header when `Pos == 0`. When `(Pos>>1) >= LoopPos + Length`, it sets `Pos = LoopPos<<1; ADPCMVal = ADPCMValLoop;` and restores the index too. Decoding uses `ADPCMTable[89]`/`ADPCMIndexTable[8]` lookups. current code: src/SPU.cpp

### melonds-174 — Burst-prefetch source data into a small per-stream FIFO
Lens: memory-layout · Source: melonDS
Principle: Fetching each sample through the full bus or memory-dispatch path costs a call every time. Read 16-byte bursts of 32-bit words into a small ring per stream (8 words, wrapped by `& 7`), and consume samples from there. This cuts dispatch calls several-fold and matches how real DMA engines fetch.
Why it is faster: —
Applies elsewhere: `BufferedReader`, audio mixers pre-reading sample chunks, DMA descriptor rings, read-ahead in storage stacks.
Watch out: —
Seen in melonDS: `SPUChannel::FIFO_BufferData()` reads `burstlen = 16` bytes (clipped at loop end) with `NDS.ARM7Read32` into `FIFO[FIFOWritePos]`, `FIFOWritePos &= 0x7`. `FIFO_ReadData<T>` serves 8-bit, 16-bit or ADPCM samples from it. Output interpolation uses the compile-time `InterpCos`/`InterpCubic` tables and a 512-entry Gaussian table. current code: src/SPU.cpp, src/SPU.h
