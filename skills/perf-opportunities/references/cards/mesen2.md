# Mesen2 — technique cards (168)

Each card is exactly 7 lines. Fetch one with: grep -A 6 "^### mesen2-NNN " mesen2.md

### mesen2-001 — Replace range-check cascades and switches with a precomputed table
Lens: lookup-tables · Source: Mesen2
Principle: When a function maps a small key to a result through a chain of comparisons or a big switch, compute every answer once at startup and store it in an array. The hot path then does one indexed load instead of a series of compares and mispredictable branches.
Why it is faster: —
Applies elsewhere: HTTP status-code to reason-phrase arrays, character-class tables in a lexer, per-opcode size tables in a bytecode VM, per-page wait-state tables in embedded memory controllers.
Watch out: —
Seen in Mesen2: `MemoryManager::IncrementMasterClock` classified every bus access by bank and page with nested `if`s. It now runs `GenerateMasterClockTable()` once and does `_masterClock += _masterClockTable[addr >> 8]`. Later the disassembler's `CpuDisUtils::GetOpSize` swapped a 30-case switch for `OpSize[(int)addrMode]`, and `Disassembler::GetSource` swapped its per-memory-type switch for a `_sources[type]` array, as part of a ~30% disassembler speedup. b2af2264 2019-02 “Code optimization” · 5c6197f5 2019-08 “Debugger: Optimized disassembler (~30% faster)”

### mesen2-002 — Shrink tables and buffers so the working set stays in cache
Lens: memory-layout · Source: Mesen2
Principle: A lookup table only helps while it stays cache-resident. Index it by just the bits that actually change the answer, and size buffers to what is really needed. Fewer cache lines get touched, and the hot data stops evicting itself.
Why it is faster: —
Applies elsewhere: collapse a 64K-entry Unicode table to a two-level table, use a 256-entry table indexed by a hashed key, right-size per-connection ring buffers in a server.
Watch out: —
Seen in Mesen2: The master-clock table was `_masterClockTable[2][0x10000]` (128 KB, indexed by the full bank+page plus a FastROM flag). It became a single 2 KB `_masterClockTable[0x800]` keyed only by the two bank bits that matter, the page, and the speed flag. The SPC sample buffer was also cut from `0x100000` to `0x20000` entries. 884aa1ab 2020-06 “Minor performance improvements (5-15%)” · b82b087b 2020-02 “SPC: Reduced buffer to a more reasonable size”

### mesen2-003 — Schedule the next event by timestamp instead of polling every tick
Lens: algorithms · Source: Mesen2
Principle: If a simulation loop checks "is something due now?" on every tick, the check runs millions of times. Keep only the time of the next pending event and compare against that one value. The event handler then works out the event after it. The common path becomes a single integer compare.
Why it is faster: —
Applies elsewhere: timer wheels and min-heaps in event loops (libuv, game engines), RTOS tickless idle that programs the hardware timer for the next deadline only.
Watch out: —
Seen in Mesen2: `MemoryManager::Exec` first compared `_hClock` against several hard-coded positions on every step. It then moved to a `bool _hasEvent[]` table indexed by `_hClock`. Later it moved again to `if(_hClock == _nextEventClock) ProcessEvent();`, where `ProcessEvent` switches on `_nextEvent` (HdmaInit, DramRefresh, HdmaStart, …) and sets the next one. 7bc96a86 2019-04 “Performance improvements and refactoring for timing changes” · 884aa1ab 2020-06 “Minor performance improvements (5-15%)”

### mesen2-004 — Stop processing once the output is fully covered
Lens: branching-fast-paths · Source: Mesen2
Principle: In layered compositing, where front layers hide back layers, count how many output slots have been filled. Once every slot is final, skip the remaining layers. On busy frames this can remove most of the back-layer work.
Why it is faster: —
Applies elsewhere: front-to-back rendering with early-out, painter's-algorithm UIs that stop at the first opaque full-screen layer, the first-match-wins rule in a cascading config lookup.
Watch out: —
Seen in Mesen2: `Ppu::RenderTilemap`, `DrawSprites` and `RenderBgColor` now return immediately when `_pixelsDrawn == 256` (main screen) or `_subPixelsDrawn == 256` (sub screen). Each pixel write increments these counters. This was part of a commit that reported about 20% faster PPU emulation. 596d6b9c 2019-02 “PPU: Optimizations (runs ~20% faster)”

### mesen2-005 — Record summary bits while loading data, then skip empty passes
Lens: laziness · Source: Mesen2
Principle: While data is being fetched anyway, keep a cheap aggregate flag: "does any item have property X?". Later passes that only handle X-items can test the flag and skip the whole loop.
Why it is faster: —
Applies elsewhere: a "has-nulls" bit per column chunk in a columnar database, dirty flags per tile in a UI compositor, a per-batch "contains transparent sprites" flag in a 2D renderer.
Watch out: —
Seen in Mesen2: `GetTilemapData` ORs `(_vram[addr] & 0x2000) >> 13` into `_layerData[i].HasPriorityTiles`, and the high-priority `RenderTilemap` pass returns early when the flag is clear. At end of scanline the sprite pipeline also fills `_hasSpritePriority[4]`, so `RenderSprites<priority>` skips priorities that no sprite on the line uses. f68aa2a8 2019-07 “PPU: Skip high priority tile processing if scanline contains no high priority tiles (optimization)” · 667aed6f 2019-07 “PPU: Refactoring and optimizations”

### mesen2-006 — Hoist loop-invariant computations out of per-element loops
Lens: compiler-build · Source: Mesen2
Principle: Any expression inside a loop that doesn't depend on the loop variable gets recomputed on every iteration, and the compiler often can't hoist it when it involves member fields or calls. Compute it once before the loop.
Why it is faster: —
Applies elsewhere: compute the row pointer once per row in image kernels, read `array.length` or a config lookup once before a JS loop, precompute a stride in matrix code.
Watch out: —
Seen in Mesen2: In `Ppu::RenderTilemap`, the tilemap row, Y offset and base address were computed for every pixel. They moved above the `for(x)` loop. A later commit introduced `outBaseAddress = (_scanline << 8)` in every per-pixel loop, and then `pixelFlags` (derived from `_colorMathEnabled`) was hoisted out of the inner pixel loop. 596d6b9c 2019-02 “PPU: Optimizations (runs ~20% faster)” · 86326215 2019-02 “PPU: Precalculate some flags through templates for performance” · e09b7a05 2019-07 “PPU: Minor color math optimization”

### mesen2-007 — Turn runtime mode flags into template parameters
Lens: compiler-build · Source: Mesen2
Principle: If a hot loop tests the same few configuration flags on every iteration, dispatch once to a template instantiation for each flag combination. Each specialized copy has the dead branches compiled out and constants folded in.
Why it is faster: —
Applies elsewhere: templated image filters per pixel format, specialized SQL operator kernels per data type, shader permutations in game engines.
Watch out: —
Seen in Mesen2: `Ppu::RenderTilemap` now picks among template instantiations on `largeTiles`, `activeWindowCount` (0/1/2) and `applyMosaic` in a small dispatcher, so these are compile-time constants inside the per-pixel loop. Later commits added `hiResMode` and `directColorMode` as template parameters in the same way. 86326215 2019-02 “PPU: Precalculate some flags through templates for performance” · f68aa2a8 2019-07 “PPU: Skip high priority tile processing…”

### mesen2-008 — Unroll small fixed-count loops
Lens: compiler-build · Source: Mesen2
Principle: When a loop's trip count is a small constant, or one of a few known values, writing out the body removes loop overhead and index arithmetic. It also lets the compiler schedule the independent operations together.
Why it is faster: —
Applies elsewhere: hand-unrolled CRC or hash rounds, an explicit 4-wide accumulate in a dot product, Duff's-device style copies in embedded code.
Watch out: —
Seen in Mesen2: The tile-decoding loop `for(plane = 0; plane < bpp; plane++)` was replaced with explicit 2/4/8-plane branches that read `_vram[pixelStart + 0/1/16/17/…]` directly. The master clock's generic `while(cyclesToRun >= 2)` template became dedicated `IncMasterClock4/6/8/40()` functions that call `Exec()` a fixed number of times. 596d6b9c 2019-02 “PPU: Optimizations (runs ~20% faster)” · d46c8c3f 2019-07 “Refactored master clock code to improve performance”

### mesen2-009 — Fuse parallel passes over the same data into one loop
Lens: algorithms · Source: Mesen2
Principle: If two passes walk the same input and write to two separate outputs, merge them into one loop. Shared work like address computation, tile decode and palette lookup then happens once, and the input is read while it is still in cache.
Why it is faster: —
Applies elsewhere: compute min, max and sum in one scan instead of three, render a shadow map and a depth pre-pass in one sweep, fuse map/filter stages in stream libraries.
Watch out: —
Seen in Mesen2: Each SNES BG mode was rendered twice, as `RenderMode1<true>()` and then `RenderMode1<false>()`, once for the main screen and once for the sub screen. Now `RenderTilemap`/`RenderSprites` decode each pixel once and write to both `_mainScreenBuffer` and `_subScreenBuffer`, each with its own window mask (`mainWindowCount`/`subWindowCount`). 667aed6f 2019-07 “PPU: Refactoring and optimizations”

### mesen2-010 — Unswitch loops: move invariant branches outside
Lens: branching-fast-paths · Source: Mesen2
Principle: If a condition is fixed for the whole loop, write two loops and test the condition once. Each loop body gets smaller and branch-free, and the simple case no longer pays for the complex one's bookkeeping.
Why it is faster: —
Applies elsewhere: separate fast paths for contiguous vs strided arrays, distinct loops for grayscale vs RGB images, a separate loop when no filter is active in a query engine.
Watch out: —
Seen in Mesen2: `Ppu::ApplyColorMath` tested `if(hiResMode)` per pixel and carried `prevMainPixel`/`prevX` through every iteration. It is now split into a hi-res loop and a plain loop. The plain loop only calls `ApplyColorMathToPixel(_mainScreenBuffer[x], _subScreenBuffer[x], …)`. a15b22a0 2020-07 “PPU: Fixed color math logic for high res mode … + Minor refactoring for performance”

### mesen2-011 — Strength-reduce affine math to running accumulators
Lens: algorithms · Source: Mesen2
Principle: When a per-element value is `base + step*i`, keep a running sum and add `step` each iteration. This replaces multiplies, or a precomputed temporary array, with one add per element.
Why it is faster: —
Applies elsewhere: DDA line drawing, texture-coordinate interpolation in software rasterizers, address generation in DSP loops.
Watch out: —
Seen in Mesen2: `RenderTilemapMode7` filled two 256-entry stack arrays `lutX[]`/`lutY[]` on every scanline. It now starts `xValue`/`yValue` at `_drawStartX` and adds `Matrix[0]`/`Matrix[2]` per pixel. That removes 2 KB of stack writes and only touches the range actually drawn. ae20428b 2019-07 “PPU: Minor mode 7 optimization”

### mesen2-012 — Replace modulo with a decrementing counter
Lens: algorithms · Source: Mesen2
Principle: A repeating period checked with `(i - start) % n` costs an integer division on every check. Keep a countdown instead: decrement it, and reload `n` when it hits zero. That is one decrement and one compare.
Why it is faster: —
Applies elsewhere: "every Nth item" logging, downsampling loops, PWM generation and periodic tasks in firmware.
Watch out: —
Seen in Mesen2: Mosaic handling in `GetTilemapData`/`GetChrData` computed `(scanline - _mosaicStartScanline) % _mosaicSize`. It now maintains `_mosaicScanlineCounter`, which is decremented once per scanline in `ProcessEndOfScanline`, and a per-pixel `mosaicCounter` in `RenderTilemap`. 6ba622df 2019-07 “PPU: Minor refactoring/optimizations”

### mesen2-013 — Store data at the width it is accessed
Lens: memory-layout · Source: Mesen2
Principle: If every access reassembles a value from two bytes (`a[i] | a[i+1] << 8`), store the array as native 16-bit words. Each read becomes one load with simpler indexing.
Why it is faster: —
Applies elsewhere: keep palettes as `uint32_t` RGBA rather than byte triples, decode a wire format into typed structs once, use typed arrays instead of DataView byte access in JS.
Watch out: —
Seen in Mesen2: CGRAM changed from `uint8_t[512]` to `uint16_t[256]`. Palette fetches in `RenderTilemap`, `RenderSprites`, `RenderBgColor` and Mode 7 went from `_cgram[off] | (_cgram[off+1] << 8)` to `_cgram[idx]`. Byte-level register access is handled only in the rare read/write paths. aa9f7b3f 2019-07 “PPU: Convert CGRAM to uint16_t array (performance + code simplicity)”

### mesen2-014 — Lay out hot fields by access frequency to fit cache lines
Lens: memory-layout · Source: Mesen2
Principle: Put the fields the inner loop touches together at the start of the object, packed with small types, and move rarely used fields to the end. The hot path then touches one or two cache lines instead of many.
Why it is faster: —
Applies elsewhere: hot/cold splitting of structs in game ECS components, ordering fields in a kernel `task_struct` or a network connection object, packing hot fields in a JVM object.
Watch out: —
Seen in Mesen2: `BaseNesPpu` was reorganized with explicit `//64 : end of cache line` markers. Rendering state (`_cycle`, `_scanline` as `int16_t`, shift registers, sprite counters, `_mapper`, `_currentOutputBuffer`) fills the first line, `_paletteRAM` and `_secondarySpriteRAM` fill the second, and fields marked "not used in rendering" come after. 5147f2db 2021-05 “Optimize field layout for PPU (cache line optimizations)”

### mesen2-015 — Interleave tables that are read together
Lens: memory-layout · Source: Mesen2
Principle: If a kernel always reads `A[i]` and `B[i]` together, store them interleaved as one array of pairs. Both values then arrive in the same cache line, and the access pattern becomes sequential.
Why it is faster: —
Applies elsewhere: array-of-structs for coefficient pairs in FIR filters, interleaved vertex attributes for GPUs, key/value adjacency in open-addressing hash tables.
Watch out: —
Seen in Mesen2: The SPC DSP core was switched to the "fast" variant. Its Gaussian interpolation table is stored as `interleved_gauss[512]`, where `interleved_gauss[i] = gauss[(i & 1) * 256 + 255 - (i >> 1 & 0xFF)]`, with the comment "to improve cache coherency". d12a582d 2019-02 “SPC: Switched to fast DSP core (better for development for now)”

### mesen2-016 — Replace virtual dispatch with static polymorphism (CRTP)
Lens: platform · Source: Mesen2
Principle: A virtual call in the hottest function blocks inlining and costs an indirect branch per call. If the concrete type is fixed at construction, make the base class a template on the derived type. Calls then resolve at compile time and inline completely.
Why it is faster: —
Applies elsewhere: CRTP iterators and expression templates in C++ numeric libraries, Rust generics instead of `dyn Trait` in tight loops, sealed classes so JITs can devirtualize.
Watch out: —
Seen in Mesen2: `NesPpu` had a `virtual DrawPixel()` that is "called 3.7 million times per second". It became `template<class T> class NesPpu` with `DefaultNesPpu`, `HdNesPpu` and `NsfPpu` as `final` subclasses. These provide `__forceinline` `DrawPixel`, `StoreTileAbsoluteAddress`, `IsEmulatorBgEnabled` and similar hooks, and the no-op versions compile away. 6251adb8 2021-05 “Templated NES PPU - WIP”

### mesen2-017 — Force-inline small functions on the hottest path
Lens: compiler-build · Source: Mesen2
Principle: Compilers decline to inline across translation units, or when their heuristics guess wrong. For tiny functions called every cycle, move the definition into the header and mark it force-inline. This removes call overhead and exposes the body to further optimization.
Why it is faster: —
Applies elsewhere: `static inline` accessors in embedded HALs, `[MethodImpl(AggressiveInlining)]` in .NET hot paths, `#[inline(always)]` on Rust bit-twiddling helpers.
Watch out: —
Seen in Mesen2: `InternalRegisters::ProcessIrqCounters` moved from the .cpp into the header as `__forceinline`, and `Ppu::FetchSpritePosition` was marked force-inline, for a reported ~5% gain. On NES, `IsRenderingEnabled`, `LoadTileInfo`, `GetPixelColor`, the scroll-increment helpers and the sound mixer's `GetOutputVolume` got the same treatment. `Disassembler::GetDisassemblyInfo` was also moved inline for the trace logger. d2409314 2019-08 “Performance: Force inlined 2 functions for a ~5% speed boost” · f92f0699 2021-04 “NES - More minor speed improvements” · 32bf942a 2021-04 “Trace logging work / optimization”

### mesen2-018 — Shape hot-loop control flow around the common case
Lens: branching-fast-paths · Source: Mesen2
Principle: Put the frequent case in the first branch, move rare code into a separate non-inlined function, and hoist calls that every branch makes. A do-while also saves the entry test when the body always runs at least once. The hot loop shrinks and the predicted path becomes the fall-through.
Why it is faster: —
Applies elsewhere: `likely()`/`unlikely()` annotations in the Linux kernel, outlining error handling into cold functions, ordering `if` chains by measured frequency in parsers.
Watch out: —
Seen in Mesen2: `NesPpu::Exec` tested `if(_cycle > 339)` first. Now `if(_cycle < 340)` handles the common cycles 1–340 first, and the once-per-scanline work moved into `ProcessScanlineFirstCycle()`. In `ProcessScanline`, a `LoadTileInfo()` call duplicated in three branches was hoisted above them. `NesPpu::Run` became a do-while ("Always need to run at least once … slightly faster"). aa7e1b89 2021-04 “NES - Switch condition order for Exec() to improve performance” · f92f0699 2021-04 “NES - More minor speed improvements” · d5554dba 2021-04 “NES - Minor performance improvement”

### mesen2-019 — Cache configuration lookups in member fields
Lens: memory-layout · Source: Mesen2
Principle: Going through an accessor chain such as `console->GetConfig().Flag` on every pixel adds pointer chasing and may block optimization. Copy the value into a member field when the configuration changes, and read the field in the hot path.
Why it is faster: —
Applies elsewhere: snapshot feature flags into locals at request start, cache `getComputedStyle` results outside animation loops, latch register settings into RAM variables in firmware.
Watch out: —
Seen in Mesen2: `NesPpu::GetPixelColor` read `_console->GetNesConfig().BackgroundEnabled` and `.SpritesEnabled` for every pixel. They are now copied into `_emulatorBgEnabled`/`_emulatorSpritesEnabled` in `UpdateMinimumDrawCycles()`, which runs once per frame. 3ae858aa 2021-04 “NES - Minor performance fix”

### mesen2-020 — Profile-guided optimization from representative workloads
Lens: compiler-build · Source: Mesen2
Principle: Build an instrumented binary, run it on realistic inputs, and recompile with the recorded profile. The compiler then lays out hot code contiguously, inlines where it counts, and orders branches by real frequencies.
Why it is faster: —
Applies elsewhere: PGO builds of browsers, CPython and databases, using recorded production traffic as the training set. Android baseline profiles are another example.
Watch out: —
Seen in Mesen2: A `PGOHelper` project was added. It loads every ROM in `PGOGames/`, runs each in console mode for 5 seconds through an exported `PgoRunTest`, and "make pgo" uses the result to produce a PGO-optimized binary. cc8ddabf 2019-02 “PGO build configuration”

### mesen2-021 — Compile validation checks out of release builds
Lens: compiler-build · Source: Mesen2
Principle: Defensive checks on internal invariants cost a compare, a branch and exception-handling code on every call. Wrap them in a debug-only guard, so release builds of a hot accessor shrink to the bare load.
Why it is faster: —
Applies elsewhere: `assert()`/`debug_assert!`, bounds checks enabled only in debug STL builds, `#ifdef DEBUG` parameter validation in embedded drivers.
Watch out: —
Seen in Mesen2: `Disassembler::GetSource` threw `runtime_error` when `_sources[type].Data == nullptr`. The check is now inside `#if _DEBUG`, so the per-instruction trace and disassembly path just returns `_sources[(int)type]`. 32bf942a 2021-04 “Trace logging work / optimization”

### mesen2-022 — Guard expensive hooks with an inline flag check
Lens: branching-fast-paths · Source: Mesen2
Principle: Split an instrumentation hook into a tiny inline wrapper that tests one boolean, and an out-of-line body that does the real work. When the feature is unused, as it is most of the time, each call costs one predictable branch.
Why it is faster: —
Applies elsewhere: `if (log.isDebugEnabled())` guards, static-key tracepoints in Linux, feature-flag checks before building expensive telemetry payloads.
Watch out: —
Seen in Mesen2: `BreakpointManager` became per-CPU, with `__forceinline CheckBreakpoint()` returning `-1` when `!_hasBreakpoint` and only otherwise calling `InternalCheckBreakpoint()`. Cheat lookups use a similar guard, `_hasCheats && _bankHasCheats[addr >> 16]`, before anything else. 7ae6605e 2019-10 “Debugger: Performance improvements + refactoring …” · 6e70ab88 2019-12 “Cheats: Fixed performance issue with WRAM cheats”

### mesen2-023 — Compute derived data only when someone consumes it
Lens: laziness · Source: Mesen2
Principle: Instrumentation that builds artifacts on every event (disassembly, state snapshots, trace rows) should check first whether any view or logger is active. Work for an absent consumer is pure overhead.
Why it is faster: —
Applies elsewhere: only serialize spans when a tracer is attached, skip DOM diffing for hidden tabs, emit metrics only when a scraper is subscribed.
Watch out: —
Seen in Mesen2: In `CpuDebugger::ProcessRead` and `SpcDebugger::ProcessRead`, `BuildCache` and the full `GetState` + trace-log calls now run only if `_traceLogger->IsCpuLogged(type)` or that CPU's debugger window flag is set. Earlier, the UI began sending breakpoints to the core only for CPUs whose debugger window is open (`BreakpointManager.AddCpuType`). 7ae6605e 2019-10 “Debugger: Performance improvements + refactoring -Reduce breakpoint check overhead -Only run SPC/DSP when needed -Only build disassembly cache when needed” · 02cab979 2019-04 “Debugger: Breakpoints - Only enable CPU/SPC breakpoints if the matching debugger window is opened …”

### mesen2-024 — Catch up secondary components lazily on access
Lens: laziness · Source: Mesen2
Principle: Rather than stepping every subsystem in lockstep, let a secondary component fall behind. Run it forward to the current time only when someone reads or writes it, or at a frame boundary. It then runs in long, cache-friendly bursts instead of being interleaved cycle by cycle.
Why it is faster: —
Applies elsewhere: lazy evaluation of physics bodies that are off-screen, catch-up simulation in multiplayer servers, deferred timer processing in cooperative schedulers.
Watch out: —
Seen in Mesen2: `Spc::Run()` advances to `GetMasterClock() * _clockRatio` and is called from the SPC port read/write paths and at end of frame. The debugger's per-PPU-cycle hook used to call `_spc->Run()` and `_cart->RunCoprocessors()` on every cycle. It now does so only when the SPC/DSP is being traced or debugged. 7ae6605e 2019-10 “Debugger: Performance improvements + refactoring …”

### mesen2-025 — Hash map lookup behind a coarse bitmap filter
Lens: algorithms · Source: Mesen2
Principle: A linear scan of a rule list on every memory access grows with the number of rules. First reject most addresses with a cheap per-region bitmap. For the rest, look up the exact key in a hash map.
Why it is faster: —
Applies elsewhere: a Bloom filter before a disk lookup, per-page "has watchpoint" bits in debuggers, route-prefix bitmaps before full route matching.
Watch out: —
Seen in Mesen2: `CheatManager::ApplyCheat` looped over `vector<CheatCode>` comparing addresses. It now queries `unordered_map<uint32_t, CheatCode> _cheatsByAddress`, still behind the `_bankHasCheats[addr >> 16]` filter. The commit message describes the old version as a WRAM-cheat performance problem. 6e70ab88 2019-12 “Cheats: Fixed performance issue with WRAM cheats”

### mesen2-026 — Mark consumed items instead of sorting
Lens: algorithms · Source: Mesen2
Principle: If items are later matched by key rather than taken in order, sorting them up front is wasted work. Scan the small set for a match and overwrite the matched entry with a sentinel so it can't match again.
Why it is faster: —
Applies elsewhere: small fixed-size schedulers that scan instead of keeping a heap, tombstoning matched entries in a short pending-request list.
Watch out: —
Seen in Mesen2: `GbPpu` ran two `std::sort` calls on the (at most 10) sprite indexes and X positions at the end of OAM evaluation on each scanline. `FindNextSprite` now scans all entries for `_spriteX[i] - 8 == _drawnPixels` and sets `_spriteX[i] = 0xFF` once fetched. eceb8d79 2020-05 “GB: PPU - Removed unnecessary sort operation”

### mesen2-027 — Pass references to live state instead of building snapshots
Lens: memory-layout · Source: Mesen2
Principle: Copying a large aggregate on every call, whether a local copy of a struct or a full multi-component state snapshot, costs memory bandwidth for data the callee mostly ignores. Pass a reference to just the part that is needed.
Why it is faster: —
Applies elsewhere: `const T&` in range-for loops, passing slices or spans instead of copying arrays, avoiding `structuredClone` of a whole store when a selector needs one field.
Watch out: —
Seen in Mesen2: `BreakpointManager::CheckBreakpoint` copied each `Breakpoint bp = breakpoints[i]`. It now indexes in place, which fixed slowdowns with many breakpoints. Later, condition evaluation stopped filling a whole `DebugState` (CPU, PPU, SPC, DMA, coprocessors) and instead uses `BaseState& state = _debugger->GetStateRef(_cpuType)`. 7226e7db 2019-03 “Debugger: Fixed performance issue when adding multiple breakpoints” · 27aec67f 2021-04 “CPU state rework”

### mesen2-028 — Store values in arrays, not shared pointers to heap objects
Lens: allocation · Source: Mesen2
Principle: A `vector<shared_ptr<T>>` costs a heap allocation and a reference-count block per element, plus a pointer chase on each access. Storing `T` by value, with an "initialized" flag, makes the data contiguous and allocation-free.
Why it is faster: —
Applies elsewhere: struct-of-arrays in ECS game engines, value types instead of boxed objects in C#/Java collections, preallocated ring buffers of plain structs for log events.
Watch out: —
Seen in Mesen2: The disassembler's per-byte caches went from `vector<shared_ptr<DisassemblyInfo>>` to `vector<DisassemblyInfo>` with `IsInitialized()`/`Reset()`, which "greatly reduced memory usage". The trace logger likewise replaced `shared_ptr<BaseState>[]` ring buffers with typed arrays (`CpuState[]`, `SpcState[]`, `NesCpuState[]`, …) filled by copy in `AddRow`. b1d1a3f4 2019-04 “Debugger: Greatly reduced memory usage when debugger tools are opened” · 32bf942a 2021-04 “Trace logging work / optimization”

### mesen2-029 — Move buffers instead of copying them
Lens: allocation · Source: Mesen2
Principle: If nested buffers are pushed and popped by value, every level copies its whole contents. Hold them through owning pointers and move the pointer, which turns an O(size) copy into a pointer swap.
Why it is faster: —
Applies elsewhere: `std::move` of vectors into containers, Rust ownership transfer, swapping ByteBuffers in Netty pipelines instead of copying.
Watch out: —
Seen in Mesen2: `Serializer` kept `BlockData _block` and a `vector<BlockData>` stack, so `StreamStartBlock`/`StreamEndBlock` copied vectors. They became `unique_ptr<BlockData>` moved with `std::move`. This was part of the "improved save/load state performance" that run-ahead needed. 5f2c502b 2019-12 “Run ahead support (+ improved save/load state performance) …”

### mesen2-030 — Match compression effort to how long the data lives
Lens: io · Source: Mesen2
Principle: Snapshots that are thrown away within a frame shouldn't be compressed at all, and frequent short-lived ones should use the fastest level. Taking snapshots less often cuts the cost further. Use strong compression only for data written to disk.
Why it is faster: —
Applies elsewhere: LZ4 or no compression for in-memory caches but zstd for archives, uncompressed scratch files in build systems, low gzip levels for dynamic HTTP responses.
Watch out: —
Seen in Mesen2: Rewind compression switched from default `compress()` to `compress2(…, MZ_BEST_SPEED)`, and the interval between rewind snapshots went from 30 to 60 frames. For run-ahead, `Serializer::Save` gained a `compressionLevel == 0` path that writes raw bytes, plus a matching uncompressed load. 4aea5123 2019-07 “Rewind: Tweak rewind to make it less cpu/memory intensive” · 5f2c502b 2019-12 “Run ahead support (+ improved save/load state performance) …”

### mesen2-031 — Skip presentation work for frames nobody will see
Lens: laziness · Source: Mesen2
Principle: When simulation runs faster than the display can show it, keep simulating every step but skip the rendering and output stages for frames that will never be displayed. Pace the skipping by elapsed time, not a fixed ratio, so you present exactly as often as the display needs.
Why it is faster: —
Applies elsewhere: fixed-timestep game loops that render at most once per vsync, skipping chart redraws when data arrives faster than the refresh rate, dropping intermediate video frames under load.
Watch out: —
Seen in Mesen2: When fast-forwarding, `Ppu::RenderScanline` first skipped 3 of every 4 frames (`_frameCount & 0x03`). This changed to time-based `_skipRender`, set when `_frameSkipTimer.GetElapsedMS() < 10`, which also skips BG fetches and `SendFrame`. Run-ahead frames force `_skipRender = true`, and `SoundMixer` drops their audio. 3715e844 2019-04 “PPU: Skip frames while fast forwarding to improve maximum speed (no impact on emulation accuracy)” · f5bb073b 2019-07 “PPU: Improved frame skipping logic” · 5f2c502b 2019-12 “Run ahead support …”

### mesen2-032 — Produce output at native resolution and upscale only when needed
Lens: memory-layout · Source: Mesen2
Principle: If a pipeline always produces data at the worst-case size, every frame pays for 4× the pixels of the common case. Emit at the actual resolution and let downstream stages scale by a multiplier.
Why it is faster: —
Applies elsewhere: render canvases at device resolution only on hi-DPI screens, keep thumbnails at display size, transcode video at its native frame size.
Watch out: —
Seen in Mesen2: The SNES PPU always wrote a 512×478 buffer. It now emits 256×239 unless a hi-res mode is in use. `DefaultVideoFilter::ApplyFilter`, `NtscFilter`, `DrawCommand` and Lua screen access use `width`/`overscanMultiplier` instead of hard-coded `512` and `*2`. f16970a2 2019-07 “Video: Output standard resolution frames in 256x239 instead of always doubling resolution”

### mesen2-033 — Bulk-copy blocks instead of per-element virtual calls
Lens: memory-layout · Source: Mesen2
Principle: Dumping memory one byte at a time through a virtual accessor makes one indirect call per byte. Add a block-level interface that each backend implements with `memcpy`, or with a masked loop when it must. The cost then becomes memory bandwidth instead of call overhead.
Why it is faster: —
Applies elsewhere: `readv`/`pread` of whole pages instead of byte reads, `ReadOnlySpan` copy APIs, DMA transfers instead of CPU loops in embedded systems.
Watch out: —
Seen in Mesen2: `MemoryDumper::GetMemoryState` called `_memoryManager->Peek(i)` for all 16 MB of CPU address space. It now calls `PeekBlock(i, buffer+i)` once per 4 KB page. `RamHandler::PeekBlock` does `memcpy` when the page isn't mirrored, and register handlers return zeros so the dump has no side effects. 41ee29d6 2019-03 “Debugger: Improved memory tools performance”

### mesen2-034 — Snapshot source data once, then compute from the copy
Lens: memory-layout · Source: Mesen2
Principle: Fetch shared source buffers once per refresh and pass raw pointers to the computation, rather than calling getters inside per-pixel helpers. This removes repeated indirection and lets the computation run on the snapshot without holding the producer.
Why it is faster: —
Applies elsewhere: copy a shared array out of a lock and process it outside, React selectors reading the store once per render, a GPU readback once per frame instead of per query.
Watch out: —
Seen in Mesen2: `PpuTools::GetRgbPixelColor` called `_ppu->GetCgRam()` for every pixel, and `GetTilemap`/`GetTileView` pulled VRAM from the PPU internally. The UI now fetches VRAM/CGRAM (or the selected source memory) once with `DebugApi.GetMemoryState` and passes `uint8_t* vram, cgram` to both functions. 2100ddb7 2019-03 “Debugger: Improved tilemap viewer performance” · d7b2d3af 2019-03 “Debugger: Improved tile viewer performance”

### mesen2-035 — Make incremental caches stop at the first valid entry
Lens: laziness · Source: Mesen2
Principle: When extending a cache by walking forward from a point, stop as soon as you reach an entry that is already valid. On invalidation, flag only the consumer that depends on the changed data. Work then scales with what actually changed.
Why it is faster: —
Applies elsewhere: incremental parsing that stops when the reparsed region rejoins the old tree, build systems that invalidate only dependent targets, per-shard dirty flags.
Watch out: —
Seen in Mesen2: `Disassembler::BuildCache` kept decoding forward past already-cached instructions. It now `break`s when it reaches an entry that is `IsInitialized()` and `IsValid(cpuFlags)`. `InvalidateCache` takes a `CpuType` and sets `_needDisassemble` only for that CPU, and only if an entry was actually reset. Previously every write marked both the CPU and SPC views dirty. cf6b21bf 2019-06 “Debugger: Fixed performance issue with disassembly”

### mesen2-036 — Pause the producer for rare reads instead of locking every write
Lens: concurrency · Source: Mesen2
Principle: If a high-frequency producer takes a mutex on every append only so that an occasional reader can see a consistent buffer, the producer pays all the time. Remove the per-append lock. When the reader or configuration code needs consistency, it briefly halts the producer thread instead.
Why it is faster: —
Applies elsewhere: stop-the-world safepoints for rare GC operations, RCU read-mostly patterns, single-writer ring buffers where the consumer pauses the writer for a snapshot.
Watch out: —
Seen in Mesen2: `TraceLogger::Log` acquired `_lock` for every logged instruction. The lock is gone, and `SetOptions` and similar calls instead create a `DebugBreakHelper`. From a non-emulation thread, it issues `BreakRequest` and waits for `IsExecutionStopped()`, then resumes when it goes out of scope. 32bf942a 2021-04 “Trace logging work / optimization”

### mesen2-037 — Throttle and filter UI refresh notifications
Lens: gpu-rendering · Source: Mesen2
Principle: A producer that can emit thousands of updates per second will swamp a UI thread that redraws on each one. Drop updates that arrive within a minimum interval, skip refreshes when a change counter hasn't moved, and don't broadcast events the user didn't trigger.
Why it is faster: —
Applies elsewhere: `requestAnimationFrame` coalescing, debounce/throttle on scroll handlers, ETag/version checks before re-rendering a dashboard widget.
Watch out: —
Seen in Mesen2: Event, palette, tile and tilemap viewers ignore refresh notifications less than 10 ms after the last one (`_lastUpdate`), capping them at ~100 FPS. `Debugger::SleepUntilResume` sends `CodeBreak` only when `_cpuStepCount == 0` (user-caused). The trace logger only re-fetches when `state.MasterClock` differs from `_previousMasterClock`, which a fix made consistent. 08b1388c 2019-03 “Debugger: Limit refresh to ~100 FPS on all debugger windows …” · 3059d686 2019-03 “Debugger: Refresh UI only when the code break was caused by user action …” · 909e75a5 2020-07 “Debugger: Fixed trace logger refreshing when no changes …”

### mesen2-038 — Use the renderer's native pixel format
Lens: gpu-rendering · Source: Mesen2
Principle: If bitmaps are stored in a format other than the one the drawing backend composites in, every draw converts each pixel. Allocate surfaces in the native format, for example premultiplied alpha, and blits become straight copies.
Why it is faster: —
Applies elsewhere: premultiplied-alpha textures in game engines, BGRA surfaces on Windows/Direct2D, `ImageBitmap` with premultiply options in browsers.
Watch out: —
Seen in Mesen2: Debugger viewers (event viewer, hex box selection cache, palette, sprite, tile and tilemap viewers) and two config forms switched their GDI+ `Bitmap`s from `Format32bppArgb` to `Format32bppPArgb`, the format GDI+ composites in. 237718c6 2019-06 “Debugger: Use Format32bppPArgb pixel format for better draw performance”

### mesen2-039 — Draw only the visible range, with a fast path for simple content
Lens: gpu-rendering · Source: Mesen2
Principle: Virtualize a large view: ask the data source for just the visible rows, and bound the loop by row count rather than pixel height. Use a cheap text primitive when the content needs no complex layout. Cost then scales with the viewport, not the dataset.
Why it is faster: —
Applies elsewhere: virtualized list and table components, viewport-only log viewers, skipping HarfBuzz shaping for pure-ASCII runs.
Watch out: —
Seen in Mesen2: The Avalonia `HexEditor` was bound to a full `byte[]`. It now uses `IHexEditorDataProvider` with `Prepare(first, last)` backed by a new `MemoryDumper::GetMemoryValues(start, end)`, and the draw loop runs for `VisibleRows + 2`. The string column builds a `TextLayout` only when a character is above 127. Otherwise it draws with a plain formatted text object. 43445203 2021-03 “Hex editor perf #1”

### mesen2-040 — Cache a failed prefix in incremental search
Lens: algorithms · Source: Mesen2
Principle: In search-as-you-type, if the query "abc" had no match, then no extension of "abc" can match either. Remember the failed prefix and skip the search until the user edits back before it.
Why it is faster: —
Applies elsewhere: autocomplete boxes, IDE "find in file", and command palettes over large symbol sets.
Watch out: —
Seen in Mesen2: `ctrlScrollableTextbox.cboSearch_TextUpdate` stores `_noMatchString` when `Search` fails, and returns early while `cboSearch.Text.StartsWith(_noMatchString)`. The cache is cleared when focus leaves. The same commit added `HexUtilities::ToHexChar`, which returns `const char*` from a `constexpr` table so `FastString` can write hex without building `std::string`s. 60067db2 2019-07 “Debugger: Fixed crash in disassembly, slightly improved search performance”

### mesen2-041 — Move blocking I/O and slow device scans off the UI thread
Lens: concurrency · Source: Mesen2
Principle: Disk reads, archive decompression and hardware enumeration can stall for tens of milliseconds or more. Run them on a background task and stop polling slow APIs periodically. Re-scan only at moments when a short delay is acceptable.
Why it is faster: —
Applies elsewhere: loading thumbnails asynchronously in galleries, enumerating USB/Bluetooth devices on hot-plug events instead of timers, `async` file reads in Electron renderers.
Watch out: —
Seen in Mesen2: `ctrlRecentGame` now reads the save-state zip and decodes `Screenshot.png` inside `Task.Run`. `WindowsKeyManager` stopped calling `_directInput->UpdateDeviceList()` every 5 s (it "takes more time and sometimes causes issues/freezes"). DirectInput devices are now refreshed when a game loads and, in a `Task.Run`, when the input config opens. 63b23a7b 2019-12 “Recent games: Improve UI responsiveness when disk is slow” · 857caa3e 2019-12 “Input: Turn off hot plugging for directinput devices …”

### mesen2-042 — Sort and truncate by cheap metadata before reading contents
Lens: io · Source: Mesen2
Principle: To show the top N items of a large set, rank them by cheap filesystem metadata, keep N, and open only those. Defer per-item parsing until the item is actually displayed.
Why it is faster: —
Applies elsewhere: "recent files" menus, photo galleries ordered by mtime, log rotation tools selecting the newest files.
Watch out: —
Seen in Mesen2: `ctrlRecentGames.Initialize` opened every `*.rgd` zip to read `RomInfo.txt`, sorted all of them, and then cut the list to 36. Now it orders file names by `FileInfo.LastWriteTime`, keeps the first 36, and the tile shows the file name and mtime without parsing the zip. e4ab66a4 2020-01 “UI: Fixed load performance issues with game selection screen”

### mesen2-043 — Sample inputs as late as possible, and sleep in short slices
Lens: io · Source: Mesen2
Principle: For latency, read user input after the frame-pacing sleep, right before the frame that uses it, not before the sleep. For long waits, sleep in small slices and re-check for pending commands, so pause and stop requests aren't delayed by a whole frame period.
Why it is faster: —
Applies elsewhere: late input sampling in VR/game loops, chunked sleeps in worker threads that must respond to cancellation, short poll intervals in embedded super-loops.
Watch out: —
Seen in Mesen2: `UpdateInputState()` and `ProcessAutoJoypadRead()` moved from the PPU's vblank start in `ProcessEndOfScanline` into `Console::Run` after the frame-limiter wait, cutting one frame of input lag. `FrameLimiter::WaitForNextFrame` now sleeps 40 ms at a time when more than 50 ms remain. `ProcessEndOfFrame` breaks out early on stop, pause or lock requests. 173350b8 2019-07 “Input: Reduced input lag by 1 frame by moving the polling to be after the emulation sleeps, rather than before.” · 96e7edb1 2019-11 “Core: Make pause/fast forward/save state/etc more responsive when running emulation at low speeds”

### mesen2-044 — Lift loop-invariant mode flags into template parameters
Lens: compiler-build · Source: Mesen2
Principle: If an inner loop tests flags that cannot change for the whole call, branch once outside the loop and call a version compiled for that exact combination. With the flags as compile-time constants (`if constexpr`), the compiler deletes the dead paths, so the per-element body gets smaller and has fewer branches.
Why it is faster: —
Applies elsewhere: image codecs with one decode loop per pixel format, database scan kernels specialized for "has nulls / no nulls", a shader permutation per material feature set.
Watch out: —
Seen in Mesen2: `PceVdc::DrawScanline` checked sprite state on every pixel. It now dispatches once to `InternalDrawScanline<hasSprites, hasSprite0>`, and lines with no sprites run a loop that has no sprite code at all. The PPU viewers' `GetTileView` used to call `GetTilePixelColor(..., format)` and `GetRgbPixelColor(format, ...)` per pixel, each containing a `switch(format)`. It now switches once into `InternalGetTileView<TileFormat>`, and the commit reports 30–75% faster viewer code in the core. Frame skipping added a third parameter, `skipRender`, to the same scanline template. 6d254cd4 2022-08 “PCE: Optimize DrawScanline using template” · a61e559c 2022-08 “Debugger: Performance improvements for ppu viewers (30-75% faster in core)” · 45287639 2022-09 “PCE: Added support for frame skipping...”

### mesen2-045 — Rebind a function pointer when configuration changes
Lens: branching-fast-paths · Source: Mesen2
Principle: When a per-tick function branches on configuration that rarely changes (hardware variant, speed mode, attached peripherals), compile one specialized version per combination. Store a pointer to the right one and update it only when the configuration changes. The hot path then makes one indirect call and evaluates none of the configuration branches.
Why it is faster: —
Applies elsewhere: picking a SIMD kernel once at startup from CPUID, swapping a logger's write function when the log level changes, a storage engine that installs a different page handler after compression is turned on.
Watch out: —
Seen in Mesen2: `PceMemoryManager::Exec` used to test `FastCpuSpeed`, `_cdrom` and SuperGrafx on every master-clock step. `UpdateExecCallback()` now chooses one of four `ExecTemplate<hasCdRom, isSuperGrafx>` instances and stores it in `_exec`. In slow-CPU mode it stores `ExecSlow` instead, which calls the chosen fast variant three times. The callback is rebuilt on power-on and whenever the speed register changes. f05260ad 2022-08 “PCE: Optimize Exec() logic with templates/logic to avoid calls when not needed, etc.”

### mesen2-046 — Skip idle subsystems with a "needs work" flag
Lens: laziness · Source: Mesen2
Principle: A component that is idle most of the time should not be stepped every tick. Keep a boolean that every state-changing entry point (register write, DMA request, state load) sets to true. The component's step function clears it once nothing is pending, and the scheduler checks the flag before making the call.
Why it is faster: —
Applies elsewhere: game engines that put sleeping physics bodies out of the update loop, UI frameworks that re-layout only dirty widgets, an event loop that drops idle timers from the poll set.
Watch out: —
Seen in Mesen2: `PceCdRom::Exec` called `_scsi.Exec()` and `_adpcm.Exec()` on every clock. It now tests `NeedExec()` first. `PceAdpcm` sets `_needExec = true` in `SetControl`, `Write`, `Read`, DMA requests and deserialization. At the end of `Exec()` it recomputes `_needExec = Playing || ReadClockCounter > 0 || WriteClockCounter > 0 || dmaRequested`. The rarely-needed `Exec` bodies are marked `__noinline`, while the cheap CD-audio tick is force-inlined. f05260ad 2022-08 “PCE: Optimize Exec() logic with templates/logic to avoid calls when not needed, etc.”

### mesen2-047 — Catch up on demand and jump straight to the next event
Lens: laziness · Source: Mesen2
Principle: Don't step a simulated component every cycle. Record when it last ran, and when someone touches it (a register write, the end of a frame), run it forward by the elapsed time. Inside that catch-up loop, advance straight to the earliest pending timer expiry instead of going one tick at a time, and emit output only when it changes.
Why it is faster: —
Applies elsewhere: discrete-event simulators, lazily computed token-bucket rate limiters (refill on request), audio synths that render a block between parameter changes.
Watch out: —
Seen in Mesen2: `PcePsg::Write` calls `Run()` before changing any state, and `PceVpc::SendFrame` calls it once per frame. `Run()` computes `clocksToRun = clock - _lastClock`, finds `minTimer` as the smallest channel timer, advances all six channels by that amount, and calls `blip_add_delta` only when the mixed output differs from `_prevLeftOutput`/`_prevRightOutput`. 8d55f9c2 2022-04 “Implement PSG, fixed VDC timing to be 1365 master clocks per scanline”

### mesen2-048 — Occupancy map to skip empty positions
Lens: branching-fast-paths · Source: Mesen2
Principle: If every output position has to search a small list of objects for one that covers it, mark the covered positions in a byte array once per row. The inner loop then reads one byte and skips the search wherever nothing is present, which is usually most of the row.
Why it is faster: —
Applies elsewhere: a canvas renderer that tracks which tiles any layer touches, a collision broad phase built on a grid bitmap, a text layout engine that marks which columns hold inline objects.
Watch out: —
Seen in Mesen2: `PceVdc::LoadSpriteTiles` clears `bool _xPosHasSprite[1024+16]` and runs `memset(_xPosHasSprite + spr.X, 1, 16)` for each sprite loaded on the line. In `DrawScanline` the sprite loop is now guarded by `if(_state.SpritesEnabled && _xPosHasSprite[_screenOffsetX])`. ebd28918 2022-08 “PCE: Optimize rendering by tracking which pixels potentially have sprites”

### mesen2-049 — Coarse presence table in front of an expensive lookup
Lens: lookup-tables · Source: Mesen2
Principle: When only a few keys in a large space have a handler, keep a small boolean array indexed by a coarse part of the key, such as the high bits. One array load rejects the common case before any virtual call or hash lookup happens.
Why it is faster: —
Applies elsewhere: a Bloom filter in front of a disk lookup, a per-page "has watchpoint" bit in a debugger, an HTTP router with a first-character table ahead of its regex routes.
Watch out: —
Seen in Mesen2: PCE reads and writes called `_mapper->Read/Write` whenever any mapper existed. `IPceMapper` now has `bool _mappedBanks[0x100]`, which the SF2 mapper and the Arcade Card fill with the banks they handle, and the memory manager tests `_mapper && _mapper->IsBankMapped(bank)`. The cheat engine uses the same idea: per-CPU `_hasCheats[cpu]` and `_bankHasCheats[cpu][addr >> GetBankShift(cpu)]` run before the `unordered_map` lookup. f8209045 2022-09 “PCE: Reduce performance overhead of SF2 mapper/arcade card” · a79a13ec 2022-03 “Cheats: Added UI and implemented cheats in core”

### mesen2-050 — Force-inline tiny hot helpers and single-call-site functions
Lens: compiler-build · Source: Mesen2
Principle: The compiler's inliner works from heuristics and often leaves small accessors and once-called stage functions out of line, especially across translation units. Marking them force-inline removes the call overhead and lets the optimizer see through to the caller's constants. A function with exactly one unconditional caller adds no code size when inlined.
Why it is faster: —
Applies elsewhere: `[MethodImpl(AggressiveInlining)]` on .NET span accessors, `#[inline(always)]` on Rust iterator adaptors in hot loops, `static inline` helpers in embedded HAL headers.
Watch out: —
Seen in Mesen2: The PCE CPU's `GetOPCode`, `ReadByte`, `MemoryRead`, `ProcessCpuCycle` and every addressing-mode helper (`GetAbsXAddr`, `GetIndYAddr`, ...) became `__forceinline`, and `FetchOperand` now writes straight into `_operand` instead of returning a value. `PceVdc`'s `ProcessSpriteEvaluation`, `LoadSpriteTiles` and `LoadBackgroundTiles` were changed from `__noinline` to `__forceinline` because each has one unconditional caller. `DebugHud::AddCommand` moved into the header as `__forceinline`. 509d67c5 2022-04 “Performance improvements” · 2c0c5511 2022-04 “PCE performance tweaks” · e2672e36 2022-08 “PCE: Inline functions that are only called once uncondtionally to improve performance” · 8d7f12ef 2022-07 “Debugger: Lua - Performance fixes, naming fixes/changes”

### mesen2-051 — Split cold paths out of a hot function
Lens: compiler-build · Source: Mesen2
Principle: Move large, rarely taken blocks (end-of-line processing, DMA transfers, slow modes) into separate non-inlined functions. The hot function becomes a short test-and-call that fits in fewer cache lines and inlines cleanly into its callers, and the cold code stays out of the instruction cache.
Why it is faster: —
Applies elsewhere: putting error formatting behind a `[[gnu::cold]]` function, the fast-path/slow-path split in allocators and lock implementations, a network receive loop that handles the common packet inline and sends everything else to a handler.
Watch out: —
Seen in Mesen2: `PcePpu::Exec` had the whole h-blank and end-of-scanline logic inline. It now reads `switch(_state.HClock) { case 999: ProcessHBlankStart(); break; case ClockPerScanline: ProcessEndOfScanline(); break; }`, and the SATB DMA moved to `ProcessSatbTransfer()`. In `PceMemoryManager`, `ExecFast` is force-inlined and `ExecSlow` is `__declspec(noinline)`. The later `PceVdc` template change also marked `LoadSpriteTiles` as `__noinline` so that it would not bloat the specialized scanline loops. 2c0c5511 2022-04 “PCE performance tweaks” · 509d67c5 2022-04 “Performance improvements” · d18e5312 2022-05 “PCE: Minor memory manager tweaks” · 6d254cd4 2022-08 “PCE: Optimize DrawScanline using template”

### mesen2-052 — Gather bits from two lanes with one mask
Lens: simd · Source: Mesen2
Principle: If the bits you need sit at fixed distances inside a wider word, shift once and mask with a pattern that selects all of them together, then fold the upper lane down with one more shift. Several shift-mask-or sequences become one, which is SIMD within a register (SWAR).
Why it is faster: —
Applies elsewhere: unpacking planar bitmaps, extracting packed RGB565 channels in two operations, bit-sliced decoding in compression codecs.
Watch out: —
Seen in Mesen2: `PceVdc::GetTilePixelColor` took four separate shift-and-mask terms to build a 4-bit color from two 16-bit plane words. It now does `color = ((chr0 >> shift) & 0x101) | (((chr1 >> shift) & 0x101) << 2); return color | ((color & 0x500) >> 7);`. It also receives the two words by value instead of through an array pointer. 1a95508e 2022-08 “PCE: Optimize GetTilePixelColor”

### mesen2-053 — Flatten multi-step decode into tables indexed by a combined key
Lens: lookup-tables · Source: Mesen2
Principle: If a value comes from a nested table lookup followed by scaling or sign extension, pack the inputs into a single small index and precompute the final answers. Branchy sign handling can likewise become a two-entry table (`sign[bit]`).
Why it is faster: —
Applies elsewhere: mapping an opcode and its addressing mode to an instruction length, mapping a format and flag pair to a pixel stride, turning a sign bit into an OR mask in a fixed-point decoder.
Watch out: —
Seen in Mesen2: `SnesPpu::FetchSpritePosition` read `_oamSizes[mode][large][axis] << 3` and sign-extended X with `((sign | x) << 7) >> 7`. It now uses `oamWidth[16]` and `oamHeight[16]` indexed by `OamMode | (large << 3)`, and computes `X = sign[hi & 1] | oam[i*4]` with `sign[2] = {0x0000, 0xFF00}`. Its argument also changed from a byte address to a sprite index, which removes a shift and its reversal. cfdd8b28 2022-07 “SNES: Minor performance improvements”

### mesen2-054 — Answer "anything pending?" with a single mask test
Lens: branching-fast-paths · Source: Mesen2
Principle: If a status check loops over sources or calls a general query with a wildcard argument, store the state as bitmasks and test `active & ~disabled` directly. This is the check that runs on every cycle, so it should cost one AND and one compare.
Why it is faster: —
Applies elsewhere: an interrupt controller in firmware, `epoll`-style ready masks, the pending-signal check in a scheduler.
Watch out: —
Seen in Mesen2: After every PCE CPU instruction and cycle, the old code called `HasIrqSource((PceIrqSource)0xFF)`. It now calls a force-inlined `HasPendingIrq()` that returns `(_state.ActiveIrqs & ~_state.DisabledIrqs) != 0`. 2c0c5511 2022-04 “PCE performance tweaks”

### mesen2-055 — Cache derived settings in members and refresh them on change events
Lens: laziness · Source: Mesen2
Principle: Don't look up settings or query a slow API from a hot path. Compute the derived booleans once, store them in fields, and recompute only when a change notification arrives. The per-call cost becomes a field read.
Why it is faster: —
Applies elsewhere: caching feature flags in a request handler with invalidation on config push, storing a resolved log level in an atomic, a UI widget that reloads document metadata on file-open instead of on every repaint.
Watch out: —
Seen in Mesen2: `SnesDebugger::ProcessInstruction`, `ProcessRead` and `ProcessPpuCycle` called `_settings->CheckDebuggerFlag(...)` several times on every instruction and PPU cycle. A new `IDebugger::ProcessConfigChange()` fills `_debuggerEnabled`, `_predictiveBreakpoints`, `_runSpc`, `_runCoprocessors` and `_needCoprocessors`, and the hot paths read those fields. In the UI, the register viewer called `EmuApi.GetRomInfo()` on every refresh. It now calls it only on the `GameLoaded` notification. 4f6ce1f6 2022-02 “Debugger: Performance improvements” · 436458f1 2022-03 “Debugger: Register viewer - Fixed performance issue due to calling GetRomInfo on each frame”

### mesen2-056 — Per-category "any?" flags before scanning a list
Lens: branching-fast-paths · Source: Mesen2
Principle: If a hook runs on every operation but only does something when a matching rule exists, keep a precomputed boolean for each operation category, or one summary flag over all fields of a request struct. The common empty case exits after a single indexed load.
Why it is faster: —
Applies elsewhere: event emitters that store a `hasListeners[eventType]` bit, a firewall that keeps a per-protocol "rules present" flag, a tracing system that checks `enabled[category]` before formatting.
Watch out: —
Seen in Mesen2: `BreakpointManager` now stores `_breakpoints` and `_hasBreakpointType` indexed directly by `MemoryOperationType`, covering dummy reads, DMA and PPU rendering reads, instead of collapsing them to read/write/exec. `CheckBreakpoint` begins with `if(!_hasBreakpointType[(int)operationInfo.Type]) return -1;`. `StepRequest` got a `HasRequest` flag, computed when the request is copied, so `ProcessPpuCycle` skips every step and scanline check when no step is active. d5a28f14 2022-02 “Debugger: (WIP) Evaluate breakpoints before an instruction starts executing...” · 3d143131 2022-01 “Debugger: Breakpoint improvements, enable ppu breakpoints for NES, performance tweaks for debugger”

### mesen2-057 — Fetch expensive context only on the path that uses it
Lens: laziness · Source: Mesen2
Principle: Don't build a costly context object, or reset a timer, before a loop that may never need it. Fetch it on first use, then reuse it for the rest of the loop.
Why it is faster: —
Applies elsewhere: opening a database transaction only when the first write happens, building an error message only when validation fails, starting a span timer only when a callback actually runs.
Watch out: —
Seen in Mesen2: `BreakpointManager::InternalCheckBreakpoint` called `GetCpuStateRef` on every check. It now starts with `BaseState* state = nullptr` and fetches the state only when a matching breakpoint has a condition, then reuses it for the remaining breakpoints. `ScriptingContext::InternalCallMemoryCallback` now resets its execution timer only when a registered callback matches the address, through a `needTimerReset` flag, instead of on every memory access. d5a28f14 2022-02 “Debugger: (WIP) Evaluate breakpoints before an instruction starts executing...” · c7ebd57e 2022-07 “Debugger: Lua - Modify Lua core to implement watchdog timer to bypass Lua hooks and improve performance”

### mesen2-058 — Trim per-element bookkeeping to what consumers read
Lens: memory-layout · Source: Mesen2
Principle: For a side array with one record per byte of a large address space, every field costs memory and cache traffic on every access. Drop the counters nobody displays and derive flags from fields you already keep, such as "never written" from `writeStamp == 0`. That shrinks the struct and the store count on every access.
Why it is faster: —
Applies elsewhere: slimming per-row metadata in a columnar store, per-page flags in a garbage collector, per-pixel G-buffers in a renderer.
Watch out: —
Seen in Mesen2: `AddressCounters` held `Address`, `ReadCount`, `WriteCount`, `ExecCount`, `UninitRead` and three 64-bit stamps. It now holds only the three stamps. `ProcessMemoryRead` returns a `ReadResult` (Normal, FirstUninitRead, UninitRead) based on the stamps, so the separate `GetReadCount` call made for the "warn once" log is no longer needed. 581f8ab7 2022-02 “Debugger: Remove memory access count tracking (only keep stamps for memory tools color)”

### mesen2-059 — Turn off runtime hardening in the hot module and default to LTO
Lens: compiler-build · Source: Mesen2
Principle: Stack-cookie checks and control-flow-guard instrumentation add work to every function call and indirect call. In a self-contained compute module that parses no untrusted input in its hot loops, turning them off removes that overhead. Link-time optimization lets the compiler inline across files, so making it the default ensures every build gets it.
Why it is faster: —
Applies elsewhere: building a game engine's inner simulation library with `-fno-stack-protector` and `-flto`, turning on `Optimize` in a .NET release profile, ThinLTO for a database's query executor.
Watch out: —
Seen in Mesen2: `Core.vcxproj` release configurations now set `<BufferSecurityCheck>false</BufferSecurityCheck>` and `<ControlFlowGuard>false</ControlFlowGuard>`. On Linux the makefile changed `ifeq ($(LTO),true)` to `ifneq ($(LTO),false)`, so `-flto` is on by default (the removed comment credited LTO with a 25–30% gain), and it builds the UI with `-p:OptimizeUi="true"`. The UI's release configuration now has `<Optimize>True</Optimize>`. d5a28f14 2022-02 “Debugger: (WIP) Evaluate breakpoints...” · 10894842 2022-08 “Linux: Added OptimizeUi = true flag to makefile” · cc8b2e33 2021-12 “Improve startup performance”

### mesen2-060 — Make the PGO training run look like real use
Lens: compiler-build · Source: Mesen2
Principle: Profile-guided optimization only knows the code paths the training run exercised. Cover every workload family, script the input needed to get past menus into the main loop, and enable optional subsystems that users turn on (such as a debugger) so their code gets profiled too.
Why it is faster: —
Applies elsewhere: running a representative request mix through a server binary for PGO or BOLT, training a compiler on its own bootstrap, recording real browser sessions for Chrome's PGO profiles.
Watch out: —
Seen in Mesen2: `PGOHelper` first added `.nes` and then `.pce` ROMs to its test set, and `PgoRunTest` creates the debugger with `GetDebugger(true)`. A `PgoKeyManager` reports key 10 as pressed on 4 of every 7 frames, and Start is mapped to key 10 for NES, SNES, Game Boy and PCE. Games therefore move past their title screens and CD games actually read from the disc, and the program logs the number of frames each game ran. 7a284976 2022-02 “Project fixes for PGO” · b620d6be 2022-04 “PGO support for PCE games” · 7a6c7dc7 2022-09 “PGO: Improve PGO results by simulating start button presses...”

### mesen2-061 — Compile out development-only checks in release builds
Lens: compiler-build · Source: Mesen2
Principle: Invariant checks that insert into a set or hash a string on every call help during development but are pure overhead once the code is stable. Wrap them in a release-only preprocessor guard so shipped builds don't run them.
Why it is faster: —
Applies elsewhere: `debug_assert!` in Rust, `#if DEBUG` schema validation in serializers, dev-mode prop-type checks in React.
Watch out: —
Seen in Mesen2: Every `Serializer::Stream` call ran `_usedKeys.emplace(key)` to detect duplicate keys. That check now lives in `CheckDuplicateKey` under `#ifndef MESENRELEASE`, and `MESENRELEASE`/`MESENOPTIMIZE` are defined in the release project settings. Runahead saves and loads state every frame, so this speeds it up. 76014f12 2022-08 “Save states: Performance improvements (improves runahead performance)”

### mesen2-062 — Bulk-copy arrays of plain values instead of per-element encoding
Lens: memory-layout · Source: Mesen2
Principle: If elements are trivially copyable and the stored layout matches memory (single bytes, or a little-endian host), one `memcpy` or range insert replaces a per-element loop with per-byte shifts. The compiler can vectorize the copy, and the loop's overhead goes away.
Why it is faster: —
Applies elsewhere: sending typed arrays through `ArrayBuffer` instead of JSON, a `BinaryWriter` writing a whole `Span<byte>`, protobuf packed repeated fields.
Watch out: —
Seen in Mesen2: `Serializer::StreamArray` called `WriteValue` and `ReadValue` on each element. It now uses `if constexpr(sizeof(T) == 1 || !isBigEndian)` to choose `_data.insert(...)` when saving and `memcpy(arrayValues, savedValue.DataPtr, sizeof(T) * elementCount)` when loading. `isBigEndian` became `constexpr`, so the byte-swap mask is a compile-time constant. 15a33e8d 2022-07 “Save States: Improve serialization/deserialization performance” · 76014f12 2022-08 “Save states: Performance improvements (improves runahead performance)”

### mesen2-063 — Serialize only the live part of fixed-capacity buffers
Lens: memory-layout · Source: Mesen2
Principle: A fixed-size array with a count field usually holds far fewer live entries than its capacity. Iterate to the count, not the capacity, and drop fields that can be recomputed. Snapshot size and time then track actual content.
Why it is faster: —
Applies elsewhere: sending only `used` bytes of a ring buffer over the wire, writing a vector's size instead of its capacity, delta snapshots in multiplayer games.
Watch out: —
Seen in Mesen2: In `NesPpu::Serialize`, the loop over `_spriteTiles` now stops at `_spriteCount` instead of 64. `PceVdc::Serialize` limits its loops to `_spriteCount`, `_totalSpriteCount` and `_tileCount` instead of 64, 64 and 100. It also stopped saving `TileData` for evaluated sprites and the `TileAddress`/`LoadSp23` fields. 76014f12 2022-08 “Save states: Performance improvements (improves runahead performance)”

### mesen2-064 — Reuse buffers, presize containers, construct in place
Lens: allocation · Source: Mesen2
Principle: In a loop, allocate a scratch string once and `clear()` it each pass. When the final element count is known, reserve it up front. Use `emplace`/`try_emplace` so an entry is built directly in the container instead of being constructed, copied and destroyed. Each of these removes allocations or rehashes from the loop.
Why it is faster: —
Applies elsewhere: `StringBuilder.Clear()` reuse in .NET, `Vec::with_capacity`, `new Array(n)` or `Map` presizing in parsers, `lua_createtable(L, n, 0)` in C bindings.
Watch out: —
Seen in Mesen2: `Serializer::LoadFrom` now declares one `string key` outside the loop (`key.clear(); key.append(...)`), stores values with `_values.emplace(key, ...)`, and no longer keeps a copy of the key inside each `SerializeValue`. Lua `getScreenBuffer` replaced `lua_newtable` with `lua_createtable(lua, w*h, 0)`, flattened the x/y loops into one, and moved `GetOutputBuffer()` out of the loop. `SetState` uses `map.try_emplace`. On the UI side, `MesenList.Replace` sets `Capacity = items.Count` before calling `AddRange`. 15a33e8d 2022-07 “Save States: Improve serialization/deserialization performance” · 13ef393c 2022-07 “Debugger: Minor performance tweaks for getScreenBuffer/setScreenBuffer” · f61f0504 2022-07 “Debugger: Lua - Refactoring/minor performance improvements” · 6f23f68b 2022-06 “Debugger: Improve debugger window startup speed”

### mesen2-065 — Replace a general-purpose hook with a dedicated countdown
Lens: other · Source: Mesen2
Principle: A generic instrumentation hook can force an interpreter onto a slower dispatch path for every instruction. If you only need "call me every N steps", one decrement-and-test in the fetch loop does the job without enabling the generic machinery.
Why it is faster: —
Applies elsewhere: fuel or gas metering in WebAssembly and blockchain VMs, cooperative-yield counters in green-thread runtimes, a watchdog tick in an embedded main loop.
Watch out: —
Seen in Mesen2: Script timeouts used `lua_sethook(L, hook, LUA_MASKCOUNT, 1000)`. The bundled Lua was patched with `lua_setwatchdogtimer`, which adds `watchdogtimer` and `watchdoghook` to `lua_State`, plus a check in `vmfetch()`: `if (L->watchdogtimer && !--L->watchdogtimer) (*L->watchdoghook)(L);`. The hook re-arms itself every 1000 instructions. c7ebd57e 2022-07 “Debugger: Lua - Modify Lua core to implement watchdog timer to bypass Lua hooks and improve performance”

### mesen2-066 — Pass non-owning raw pointers instead of shared_ptr copies
Lens: allocation · Source: Mesen2
Principle: Every copy of a `shared_ptr` does an atomic increment and later an atomic decrement. If one owner clearly outlives all users, hold the object in a `unique_ptr` and hand out raw pointers. Getters then cost nothing beyond returning the pointer.
Why it is faster: —
Applies elsewhere: passing `&T` instead of `Arc<T>` in Rust, borrowed views in C++ component systems, avoiding `std::function` captures of `shared_ptr` in hot callbacks.
Watch out: —
Seen in Mesen2: `Debugger` now owns `ScriptManager`, `MemoryDumper`, `MemoryAccessCounter`, `CodeDataLogger`, `Disassembler` and `LabelManager` through `unique_ptr`. `GetCodeDataLogger`, `GetEventManager`, `GetCallstackManager` and the other getters return raw pointers, and the per-CPU debuggers and trace loggers were updated to match. 0d78c064 2022-01 “Core: Reduce shared_ptr usage, refactor netplay code to remove static instances”

### mesen2-067 — Bind references to cache entries instead of copying structs
Lens: memory-layout · Source: Mesen2
Principle: Writing `T x = table[i]` in a loop copies the whole struct on every iteration. Bind `T& x = table[i]` instead, and store the container inline in its owner rather than behind another pointer. Each access then saves a copy and an indirection.
Why it is faster: —
Applies elsewhere: `for (const auto& row : rows)` in C++, `ref var` locals over struct arrays in C#, iterating by reference in Rust.
Watch out: —
Seen in Mesen2: `Disassembler::Disassemble` and `GetLineData` copied a `DisassemblerSource` and a `DisassemblyInfo` for every address. They now use `DisassemblerSource& src` and `DisassemblyInfo& disassemblyInfo = src.Cache[addr]`. `Cache` changed from a `vector*` into a separate array to a `vector` member, and address translation calls `_console->GetAbsoluteAddress` directly. eb527172 2021-05 “Disassembler performance tweaks”

### mesen2-068 — Load large files in chunks on first access
Lens: io · Source: Mesen2
Principle: Don't read a multi-hundred-megabyte file into memory at startup. Split it into fixed-size chunks, load each chunk the first time an offset inside it is read, and cache the file size. Startup cost then no longer depends on file size, and memory holds only the parts actually used.
Why it is faster: —
Applies elsewhere: HTTP range requests for video seeking, page caches in SQLite, lazily mapped asset bundles in mobile games.
Watch out: —
Seen in Mesen2: `VirtualFile::ReadByte` used to call `LoadFile()`, which read the entire disc image. It now calls `InitChunks()` to size `vector<vector<uint8_t>> _chunks`, and on a miss opens the file, seeks to `chunkId * 256 KB` and reads that one chunk. `GetSize()` stores the result of `tellg()` in `_fileSize`. 1108bffe 2022-09 “PCE: Read CD-ROM image in 256kb chunks instead of loading the entire file at startup”

### mesen2-069 — Page a large remote array through a windowed accessor
Lens: io · Source: Mesen2
Principle: When code scans a large array that lives on the other side of an interop or RPC boundary, don't copy all of it up front. Wrap it in an indexer that fetches fixed-size pages on demand and keeps the current page. Sequential scans make one boundary crossing per page, and an early exit never pays for the rest.
Why it is faster: —
Applies elsewhere: cursor-based pagination over a REST API, virtualized list data sources, chunked reads of a GPU buffer.
Watch out: —
Seen in Mesen2: Memory search and the "navigate to next code/data/unread byte" commands called `DebugApi.GetMemoryState`, `GetCdlData` or `GetMemoryAccessCounts` for the entire memory type. They now use `VirtualArray<T>`, which loads 0x10000-element pages through a loader delegate. `Get(i)` is `AggressiveInlining` and checks only `_loadedPage != (i & ~(BatchSize-1))`, while `LoadPage` is `NoInlining`. 49da00fb 2022-07 “Debugger: Memory tools - Only fetch data for 64k bytes at a time for searches/navigation (faster in most scenarios)”

### mesen2-070 — Check for consumers before computing an expensive key
Lens: laziness · Source: Mesen2
Principle: A content hash used only to match against candidate files is wasted work when there are no candidates. Enumerate the cheap side first and return early when it is empty.
Why it is faster: —
Applies elsewhere: skipping ETag hashing when no cache layer is configured, avoiding image fingerprinting when no duplicate-detection rule exists, short-circuiting a cache-key build when caching is off.
Watch out: —
Seen in Mesen2: `HdPackLoader::InitializeLoader` computed `romFile.GetSha1Hash()` on every NES ROM load. It now returns before hashing if `hdnPackages.size() == 0`. 7854ea96 2022-02 “NES: Avoid calculating SHA-1 for rom when no hd packs exist”

### mesen2-071 — Keep long-lived device resources across sessions
Lens: gpu-rendering · Source: Mesen2
Principle: Tearing down and rebuilding a renderer, connection or context on each new document repeats setup work that rarely changes. Keep the resource and reset it only when its parameters actually change.
Why it is faster: —
Applies elsewhere: reusing a WebGL context between scenes, connection pooling instead of reconnecting per request, keeping an audio device open across tracks.
Watch out: —
Seen in Mesen2: `VideoRenderer::RenderThread` called `_renderer->Reset()` every time the thread started, which happened on every game load. That call was removed. 81c351fa 2022-02 “Video: Don't reset renderer each time a game is loaded”

### mesen2-072 — Skip the write when serialized output hasn't changed
Lens: io · Source: Mesen2
Principle: Remember the bytes last read or written. If a fresh serialization produces the same bytes, don't touch the disk. Unchanged saves then cost one string comparison instead of file I/O, which also avoids extra flash wear and file-watcher events.
Why it is faster: —
Applies elsewhere: build tools that skip rewriting identical generated files to preserve timestamps, config sync agents that compare content hashes, React's bailout when state is `Object.is`-equal.
Watch out: —
Seen in Mesen2: `Configuration.Deserialize` now keeps the file contents in `_fileData`. `Save()` serializes to `cfgData` and calls `File.WriteAllText` only when `_fileData != cfgData`. This fixed the slowdown when closing the event viewer. Separately, startup stopped calling `Config.Save()` right after applying defaults. 8f96e612 2022-01 “Debugger: Fixed performance issue when closing event viewer + minor refactoring” · 68d2447f 2022-07 “UI: Small tweaks to startup code...”

### mesen2-073 — Propagate only when the value actually changed
Lens: laziness · Source: Mesen2
Principle: Periodic inputs such as polling, timers and pointer-move events often repeat the previous value. Compare with the last value and stop there if nothing changed. This keeps downstream work (bindings, layout, image decoding, shortcut processing) from rerunning on no-op updates.
Why it is faster: —
Applies elsewhere: `distinctUntilChanged` in Rx, `React.memo` and `useMemo`, setters in MVVM that return early on equal values, debounced resize handlers that compare dimensions.
Watch out: —
Seen in Mesen2: Toolbar actions used to decode a new icon `Image` on every refresh. `ContextMenuAction` now tracks `_currentIcon` and calls `ImageUtilities.FromAsset` only when the icon path changes. `MemoryMappingViewModel.UpdateMappings` keeps the old list when the new blocks (now records with value equality) compare equal. The event and tilemap viewers return early from pointer-move handlers when the position equals `_prevMousePos`. `KeyManager.SetKeyState` now returns whether the state changed, and only then is `ShortcutKeyHandler::ProcessKeys` called. `EnumComboBox` was rewritten so it no longer fires change events that pushed wrong settings to the core and then corrected them. 54e9ce65 2022-03 “Debugger: Fixed toolbars causing excessive CPU usage due to icon update” · dba7e1ce 2022-06 “Debugger: Improve performance of memory mappings...” · 101223eb 2021-12 “Tilemap/Event viewers: Fixed high CPU usage when tooltip is shown” · 91ab4bd8 2022-09 “Input: Stop calling ShortcutKeyHandler::ProcessKeys constantly...” · ca164857 2022-08 “UI: Fixed EnumComboBox triggering onchange events that weren't useful...”

### mesen2-074 — Precompute reflection metadata once per type
Lens: lookup-tables · Source: Mesen2
Principle: Reflection lookups such as reading custom attributes off enum values are slow and allocate. If the answer is fixed per value, build a dictionary in a static constructor and replace each reflection call with a hash lookup.
Why it is faster: —
Applies elsewhere: caching `PropertyInfo` accessors in ORMs and serializers, precomputed enum-to-display-string maps, Go struct-tag caches in JSON encoders.
Watch out: —
Seen in Mesen2: `BaseMenuAction.GetIconFile()` called `ActionType.GetAttribute<IconFileAttribute>()` on every toolbar refresh. A static constructor now fills `_iconCache` for every `ActionType` value, and lookups read `_iconCache[ActionType]`. 28177008 2022-08 “Debugger: Various performance fixes + automatically reduce refresh FPS as more windows are opened...”

### mesen2-075 — Batch collection updates behind one change notification
Lens: allocation · Source: Mesen2
Principle: Observable collections that raise an event per `Clear`/`Add` make bound views re-measure for every item. Swap the contents in bulk and raise a single Reset, or build the child list first and attach it once. Likewise, when deleting many items, refresh the dependent indexes once at the end rather than after every deletion.
Why it is faster: —
Applies elsewhere: `DocumentFragment` in the DOM, `beginUpdate`/`endUpdate` on list controls, batched `setState` in React, bulk inserts inside one database transaction.
Watch out: —
Seen in Mesen2: A new `MesenList<T>` provides `Replace(items)`, which raises one cached Reset event. The breakpoint, call stack, watch, label, event and cheat lists use it in place of `Clear()` followed by `AddRange()`. `TooltipEntries` now raises Reset only when an entry is added, not when an existing entry's value changes. `StateGrid` builds `ColumnDefinitions`, `RowDefinitions` and a list of children before assigning them. `LabelManager.DeleteLabels` deletes each label with refresh turned off and then calls `ProcessLabelUpdate()` once. 6f23f68b 2022-06 “Debugger: Improve debugger window startup speed” · c6aa9354 2022-01 “Tooltips: Fixed performance issues when updating entries...” · 68d2447f 2022-07 “UI: Small tweaks to startup code...” · ca96b5eb 2022-03 “...Fixed performance issue when deleting labels”

### mesen2-076 — Build expensive parts on first view, validate on selection
Lens: laziness · Source: Mesen2
Principle: Don't construct every tab's model, load every list or test every option when a window opens. Create each part when it first becomes visible, run data loads after the window shows, and run costly per-item checks only for the item the user picks. Open time then depends on what the user sees, not on everything the window could show.
Why it is faster: —
Applies elsewhere: lazy-loaded route components in SPAs, settings screens that build panes on demand, font pickers that rasterize previews only for visible rows.
Watch out: —
Seen in Mesen2: `ConfigViewModel` used to construct all of the Audio, Video, Emulation, NES, SNES and Game Boy view models. `SelectTab` now creates each one with `??=` when its tab is selected. The debugger's list view models no longer load data in their constructors, and `DebuggerWindowViewModel.Init()` runs those loads later. The font options list used to construct a `Typeface` for every installed font to check `IsFixedPitch`. It now lists all fonts and validates only the selected one. 5db85ae1 2021-05 “Tweak icons, menus, config dialog performance” · 6f23f68b 2022-06 “Debugger: Improve debugger window startup speed” · 7b6c0741 2022-08 “Debugger: Improve debug settings dialog startup performance...”

### mesen2-077 — Overlap startup I/O and skip unchanged extraction
Lens: concurrency · Source: Mesen2
Principle: At startup, run independent work in parallel, for example parsing config on a worker while the main thread unpacks native libraries. Skip an unpack when the target file's size and timestamp already match, and load only the styles the first window needs. Time to first window then becomes the longest single task instead of the sum of all of them.
Why it is faster: —
Applies elsewhere: Electron apps that preload settings in parallel with window creation, installers that compare hashes before overwriting, critical-CSS inlining with deferred full stylesheets.
Watch out: —
Seen in Mesen2: `Program.Main` now starts `Task.Run(() => ConfigManager.LoadConfig())` and then runs `ExtractNativeDependencies()`, which skips a zip entry when `LastWriteTime` and `Length` match the existing file. The app first loads a small `StartupStyles.xaml` through `StyleHelper.LoadStartupStyles()` instead of the full Fluent, DataGrid, Dock and AvaloniaEdit style set, and single-file compression is disabled. Removing the reactive `FontFamilyObject` property, now replaced by a converter, also cut startup time. cc8b2e33 2021-12 “Improve startup performance” · 48fc7b4c 2022-06 “UI: Slightly improve startup performance by removing FontFamilyObject observable property”

### mesen2-078 — Throttle refresh rates and redraw only when something can change
Lens: other · Source: Mesen2
Principle: Periodic redraws should depend on whether anything changed. Invalidate only when an animated element such as a marching-ants selection is present, slow the timer when it isn't, and lower each tool's frame-rate cap as more tools are open. Total UI cost then stays bounded instead of growing with every open window.
Why it is faster: —
Applies elsewhere: `requestAnimationFrame` that stops when idle, dashboards that lower poll rates when many panels are open, rendering background browser tabs at reduced rates.
Watch out: —
Seen in Mesen2: `PictureViewer`'s timer went from 50 ms to 250 ms, and `timer_Tick` calls `InvalidateVisual()` only when `SelectionRect != Rect.Empty`. `ToolRefreshHelper.LimitFps` lowers `maxFps` by 10% for each tool beyond five, down to 30% and never below 5 fps, and removes entries for closed windows. 59d78db8 2021-12 “UI: Fixed DPI scaling issues + fixed performance issues with PictureViewer” · 28177008 2022-08 “Debugger: Various performance fixes + automatically reduce refresh FPS as more windows are opened...”

### mesen2-079 — Delay a state toggle that usually reverts right away
Lens: other · Source: Mesen2
Principle: If an action flips UI state and the opposite action usually follows within milliseconds (resume, then break again when stepping), wait a short time before applying the change and drop it if it has already reverted. Rapid repeated actions then skip two full re-renders each time.
Why it is faster: —
Applies elsewhere: delayed loading spinners that only appear after 200 ms, hover-intent menus, coalescing rapid "saving..." indicators.
Watch out: —
Seen in Mesen2: On resume, the debugger used to set `ConsoleStatus.EditAllowed = false` immediately, then back to true on the next break. It now posts a task that waits 50 ms and disables the fields only if they are still enabled and `!EmuApi.IsPaused()`. The break path sets `EditAllowed` only when it is currently false. a35e307d 2022-06 “Debugger: Improve debugger refresh performance”

### mesen2-080 — Keep heavy views mounted and resize them instead of remounting
Lens: other · Source: Mesen2
Principle: If a complex control is expensive to create and lay out, toggling it in and out of the visual tree with tabs repeats that cost on every switch. Keep it mounted and collapse it to zero height (or hide it) when it isn't in use.
Why it is faster: —
Applies elsewhere: `keep-alive` in Vue, hidden rather than unmounted tabs in React, Android `ViewPager` offscreen page limits.
Watch out: —
Seen in Mesen2: The event viewer's two `TabItem`s were replaced by a `TabStrip` plus two views that stay mounted. Switching tabs sets `ShowViewer` and `ListHeight` (0 or `NaN`), so the `DataBox` list is no longer rebuilt each time the List tab is clicked. 5636ab90 2022-04 “Debugger: Event Viewer - Improve list tab load performance”

### mesen2-081 — Give virtualized lists their real height on the first measure
Lens: other · Source: Mesen2
Principle: A virtualizing list creates as many rows as fit in the height it is measured with. If a parent first measures it with an unbounded or screen-sized height, it creates hundreds of rows and throws them away on the next pass. Pass the actual available size straight through.
Why it is faster: —
Applies elsewhere: fixed-height containers around `react-window` lists, avoiding `ScrollView` inside an unbounded `StackPanel` in WPF or MAUI, setting explicit heights for RecyclerView in nested scrolls.
Watch out: —
Seen in Mesen2: `MesenScrollContentPresenter.MeasureOverride` now checks for a `DataBox` ancestor. If there is one, it calls `Child.Measure(availableSize)` and returns `Child.DesiredSize.Constrain(availableSize)` instead of running the base scroll presenter's measure. ffd4bec3 2022-06 “UI: Fixed performance issues with DataBox in some scenarios”

### mesen2-082 — Hand-written span tokenizer instead of backtracking regexes
Lens: algorithms · Source: Mesen2
Principle: For a simple `key=value,key=value` format, regexes full of optional `.*` groups backtrack and allocate a `Match` and a string for every group. A single pass over a `ReadOnlySpan<char>` that splits on delimiters and parses numbers in place does no allocation and does not depend on field order.
Why it is faster: —
Applies elsewhere: log parsers in Go and Rust that use `bytes.IndexByte`, hand-rolled CSV or querystring splitters, HTTP header parsing in high-throughput servers.
Watch out: —
Seen in Mesen2: `DbgImporter` parsed ca65 `.dbg` lines with `_lineRegex`, `_spanRegex` and `_symbolRegex`. A new `DbgReader.ReadEntry(row, (ref ReadOnlySpan<char> name, ref ReadOnlySpan<char> data) => ...)` walks the fields instead, together with `Int32.Parse(span)` and `ReadIntArray`. The `LineInfo`, `SpanInfo` and related records also became value types. 4b3e425e 2022-01 “Debugger: Performance fixes for DBG import”

### mesen2-083 — Comparator dispatch table and ordinal string comparison
Lens: algorithms · Source: Mesen2
Principle: A multi-column sort comparator that loops over every possible column with local closures allocates and branches on every comparison. Build a column-to-comparer dictionary once per sort and call only the active ones. Compare strings with an ordinal comparison instead of the culture-aware default, which is much slower.
Why it is faster: —
Applies elsewhere: `StringComparer.Ordinal` for dictionary keys, precompiled sort-key extractors in data grids, `localeCompare` only when you actually need linguistic order.
Watch out: —
Seen in Mesen2: The breakpoint, event, label, profiler, sprite and cheat lists had used a `Compare(name, () => ...)` local function called for every column on each comparison. They now look up `comparers[column](a, b)` in a `Dictionary<string, Func<T,T,int>>` and use `string.Compare(..., StringComparison.OrdinalIgnoreCase)` for text columns. 0bfc9ba9 2022-06 “Debugger: Rewrite old sort logic”

### mesen2-084 — Move an optional effect from every producer into one post-pass
Lens: branching-fast-paths · Source: Mesen2
Principle: If every output path carries an "if effect enabled" branch inside its pixel loop, take the effect out into a single pass that runs after the main output and returns immediately when the effect is off. Each main loop becomes a straight copy, and the effect is written once.
Why it is faster: —
Applies elsewhere: moving gamma or tonemapping into one final shader pass, applying watermarking as middleware instead of in each handler, putting log redaction in a single sink.
Watch out: —
Seen in Mesen2: The NES, SNES, Game Boy and PCE default filters, the NTSC and Bisqwit filters and `ScaleFilter` each contained `if(i & 1) ApplyScanlineEffect(...)` branches in their decode loops. These are gone. `ScanlineFilter::ApplyFilter(buffer, w, h, intensity, scale)` now runs after the other filters, starts with `if(scanlineIntensity <= 0) return;`, and darkens every other row as a separate pass. b58cb6fd 2022-05 “Refactor scanline filter”

### mesen2-085 — Skip producing output nobody will see while keeping its side effects
Lens: laziness · Source: Mesen2
Principle: When a frame will be dropped, skip computing its pixels but still run the parts with observable side effects, such as bus-timing windows or collision interrupts. A compile-time flag can strip the output code from a dedicated loop instance.
Why it is faster: —
Applies elsewhere: headless game servers that run physics but not rendering, catching up a video decoder by decoding only reference frames, dry-run modes that still validate inputs.
Watch out: —
Seen in Mesen2: When `PceVpc::IsSkipRenderEnabled()` is true, `LoadBackgroundTiles<true>` only toggles `_allowVramAccess = (i & 1) == 0` and fetches no BAT or CG data. The scanline loop then runs only when sprite-0 collision has to be detected (`InternalDrawScanline<true, true, true>`), with all palette lookups and buffer writes compiled out by `if constexpr(!skipRender)`. 45287639 2022-09 “PCE: Added support for frame skipping, moved frame skipping option to global video settings”

### mesen2-086 — Cap auxiliary stacks that unusual input can grow without bound
Lens: memory-layout · Source: Mesen2
Principle: Bookkeeping that assumes balanced push and pop (calls and returns, open and close tags) can grow forever on input that never pops. Cap it and drop the oldest entries, so per-event cost and memory stay bounded.
Why it is faster: —
Applies elsewhere: bounded breadcrumb buffers in crash reporters, depth limits in recursive-descent parsers, LRU caps on navigation history.
Watch out: —
Seen in Mesen2: When code jumps into functions without matching JSR/RTS, `Profiler` now calls `pop_front()` on `_functionStack`, `_cycleCountStack` and `_stackFlags` once they exceed 100 entries. 80ec8ce6 2022-04 “Fixed draw order in sprite viewer, reduced perf issue in debug builds”

### mesen2-087 — Move rare states out of the hot dispatch into a non-inlined function
Lens: branching-fast-paths · Source: Mesen2
Principle: When a function runs millions of times a second and almost always takes one path, put that path first in a single `if` and move every rare case into a separate function marked never-inline. The hot function stays small enough to inline and keeps its registers. The cold code no longer fills the instruction cache or confuses the inliner's size budget.
Why it is faster: —
Applies elsewhere: request handlers that split error and retry handling into `[[gnu::cold]]` helpers; parsers that keep the fast ASCII path inline and push escape handling out of line; game loops that move pause and menu logic out of the per-frame update.
Watch out: —
Seen in Mesen2: `SnesCpu::Exec` used a three-way `switch(_state.StopState)` with the STP and WAI handling inline. It became `if(StopState == Running) { ProcessInstruction(); RunOp(); } else { ProcessHaltedState(); }`, with `__noinline void ProcessHaltedState()`. The SA-1 CPU got the same split, plus a `__forceinline CheckForInterrupts()`. In the debugger, `Debugger::ProcessInstruction` now tests `IsStepBack()` inline and calls a `__noinline ProcessStepBack()` only when step-back is active. The PCE VDC's rare vertical-mode transition also went into a `__noinline SetVertMode`. 00133c46 2022-09 “SNES: Minor performance improvement” · 2a930149 2022-09 “Minor tweaks to debugger code for performance…” · 203f1112 2023-02 “SNES: Fixed CPU bug where calling WAI just before NMI…” · 5f89c254 2023-09 “PCE: Improved start timing for the SATB DMA transfer…”

### mesen2-088 — Check that your inlining macros mean what you think on every compiler
Lens: compiler-build · Source: Mesen2
Principle: Portability shims that `#define` a compiler keyword can silently replace a strong hint with a weak one on a compiler that already understood the original. When `__forceinline` was redefined as plain `inline` on MSVC, every hot helper became optional to inline, and the release build slowed down with no error or warning. Guard each such shim with the compiler it is meant for.
Why it is faster: —
Applies elsewhere: `likely()`/`unlikely()` macros that expand to nothing on one toolchain; `restrict` macros defined away; `NDEBUG` missing from one build configuration so asserts stay in release.
Watch out: —
Seen in Mesen2: `Utilities/pch.h` defined `__forceinline` as `__attribute__((always_inline)) inline` on GCC/Clang with LTO and as `inline` otherwise. The "otherwise" branch also caught MSVC, where `__forceinline` is a built-in keyword. The fix wraps the whole block in `#ifndef _MSC_VER`. An earlier change reworked the same macros in `Core/pch.h` so that `__noinline` maps to `__declspec(noinline)` or `__attribute__((noinline))`, and `always_inline` is used only when LTO can supply the bodies. 1741f38d 2023-04 “Fixed MSVC performance issue caused by __forceinline being redefined as inline by mistake” · 7473580d 2023-02 “Linux: Fix building without LTO (#9)”

### mesen2-089 — Make sure the optimization flags reach the compiler
Lens: compiler-build · Source: Mesen2
Principle: In make, a variable set on the command line overrides every `+=` in the makefile. A flags variable expanded before the lines that append to it can also miss those additions. Either mistake quietly produces an unoptimized build that still compiles and runs. Look at the compile command lines in CI logs, not only at the makefile.
Why it is faster: —
Applies elsewhere: CMake builds with no `CMAKE_BUILD_TYPE`, which use no optimization flags; `CFLAGS=...` in a Dockerfile that wipes the project's `-O2`; Cargo profiles overridden by environment variables.
Watch out: —
Seen in Mesen2: The Linux makefile builds `MESENFLAGS` up with `+= -O3` and `+= -flto -DHAVE_LTO`. CI ran `make MESENFLAGS="-fdiagnostics-color=always"`, which replaced all of that, so the dev builds had no optimization. One fix removed the command-line assignment. The other moved the `CXXFLAGS`/`CFLAGS` definitions below the conditional blocks that append to `MESENFLAGS`. b7d368d5 2023-02 “Build: Fixed Linux dev builds not getting optimized” · a8644c13 2023-02 “Linux: Tweak makefile to try to fix dev builds not getting optimized”

### mesen2-090 — Turn off safety instrumentation and frame pointers in release builds
Lens: compiler-build · Source: Mesen2
Principle: Compiler security checks (MSVC `/sdl`), inlining limited to explicitly marked functions, and kept frame pointers each cost a little on every call. Together they matter in tight loops. Release builds of a hot interpreter core should allow inlining of any suitable function and omit frame pointers. When profile-guided optimization makes the link slow, look for linker options that speed up profile handling instead of dropping PGO.
Why it is faster: —
Applies elsewhere: `-fomit-frame-pointer` and `-fno-stack-protector` for numeric kernels; turning off `_GLIBCXX_ASSERTIONS` in shipping builds; PGO with `-fprofile-use` for interpreters and databases.
Watch out: —
Seen in Mesen2: In `Core.vcxproj`, the optimized configurations went from `SDLCheck=true` to `false` and from `InlineFunctionExpansion=Disabled` to `AnySuitable`. `OmitFramePointers` became `true` in the configuration that still had it off. `InteropDLL.vcxproj` switched PGO to explicit `/GENPROFILE:NOPATH` and `/USEPROFILE`, which avoided a link step of more than 10 minutes. 2a930149 2022-09 “Minor tweaks to debugger code for performance, set NOPATH option for PGO to avoid 10+ min link time (VS bug?)”

### mesen2-091 — Turn a runtime constant into a template parameter and pick the instance once
Lens: compiler-build · Source: Mesen2
Principle: If a value such as a scale factor is fixed for the lifetime of an object but read in the innermost loop, make it a template parameter. Use one `switch` at construction to create the matching instance. Loop bounds become compile-time constants, so the compiler can fully unroll, use `if constexpr` for special cases, and drop the per-pixel loads of the setting.
Why it is faster: —
Applies elsewhere: image resizers specialized per channel count; codecs with a template per bit depth; matrix kernels instantiated for 2×2, 3×3 and 4×4 sizes; protocol encoders specialized per field width.
Watch out: —
Seen in Mesen2: `HdNesPack` became `HdNesPack<uint32_t scale>`. `HdVideoFilter`'s constructor switches over `hdData->Scale` 1–10 to create the right instance. `DrawColor` now has `if constexpr(scale == 1)` for a single store, an unrolled 2×2 case, and the generic loop for everything else. Reads of `_hdData->Scale` in `DrawCustomBackground` and `OnLineStart` became uses of the template parameter `scale`. 78acaee8 2023-03 “NES: HD Packs - Small performance improvements”

### mesen2-092 — Pass call-site facts as template flags to strip branches from shared hooks
Lens: compiler-build · Source: Mesen2
Principle: When one hook function serves several callers and each caller already knows which subset of the work applies, add a compile-time flag parameter with a default value. Each call site then gets a version that contains only its own branch. The common caller pays nothing for the new case.
Why it is faster: —
Applies elsewhere: allocators with a `template<bool zeroed>` path; logging macros that compile out disabled levels; serializers with a `template<bool isWriting>` instead of a runtime mode flag.
Watch out: —
Seen in Mesen2: `Debugger::ProcessMemoryRead/Write` gained `template<CpuType type, MemoryAccessFlags flags = MemoryAccessFlags::None, typename T>`. The SPC's DSP-side RAM accesses call the `MemoryAccessFlags::DspAccess` instance. `SpcDebugger::ProcessRead<flags>` uses `if constexpr(flags == MemoryAccessFlags::None)` to keep the CPU-access path, with trace logging and exec counters, separate from the light DSP path. 7ca3ae78 2023-01 “Debugger: SPC - Invalidate instruction cache on writes even when option to ignore read/writes is turned on”

### mesen2-093 — Hoist loop-invariant mode checks out of pixel loops
Lens: branching-fast-paths · Source: Mesen2
Principle: A condition that cannot change during a loop, such as "brightness is at its default", should select between two copies of the loop instead of being tested per element. Inside the loop, replace byte extraction with whole-word comparisons when the data layout allows it. For ARGB pixels, `px >= 0xFF000000` means fully opaque.
Why it is faster: —
Applies elsewhere: blitters with separate loops for opaque and blended sprites; audio mixers with a unity-gain fast path; CSV writers with a separate loop when no field needs quoting.
Watch out: —
Seen in Mesen2: `HdNesPack::DrawCustomBackground` used to check `brightness == 255` and read `((uint8_t*)pngData)[3]` for every pixel. It now has two nested-loop variants, picked once by `bgInfo.Brightness == 255`. Each one tests `pixelColor >= 0xFF000000` to copy and `>= 0x01000000` to blend, and steps through rows with `outputBuffer += screenWidth`. 78acaee8 2023-03 “NES: HD Packs - Small performance improvements”

### mesen2-094 — Load raw bytes up front, decode on first use
Lens: laziness · Source: Mesen2
Principle: Startup is slow when you decode every asset before you know which ones will be needed. Keep the compressed file bytes and decode each asset the first time it is actually used, then free the compressed copy. Keep the "already initialized?" test as a tiny inline check and put the decode in a non-inlined function, so the check stays cheap on the hot path.
Why it is faster: —
Applies elsewhere: lazy image decoding in browsers (`decoding="async"`, on-demand textures); fonts that rasterize glyphs on first draw; ORMs that load relations lazily; mobile apps that inflate layouts on demand.
Watch out: —
Seen in Mesen2: HD-pack loading used to `ReadPNG` and premultiply every `<img>` and `<background>` at load time. `HdPackBitmapInfo` now keeps `FileData`, and its `Init()` decodes, premultiplies, and then sets `FileData = {}`. `HdPackTileInfo::Init()` copies its sub-rectangle out of the bitmap the first time `GetMatchingTile` matches the tile. A later commit split this into `__forceinline bool NeedInit()` and `__noinline void Init()`. ccc2bbbc 2023-03 “NES: HD Packs - Decode backgrounds on first use Makes loading HD packs faster” · 6f525ca6 2023-03 “NES: HD packs - Decode images on first use…” · 78acaee8 2023-03 “NES: HD Packs - Small performance improvements”

### mesen2-095 — Warm lazy resources on a background thread, with a lock and a cancel flag
Lens: concurrency · Source: Mesen2
Principle: Lazy loading moves the cost to first use, and first use can land in the middle of a frame and cause a stutter. Start a background worker that initializes the same items in advance. Have each item's init take a per-item lock and re-check its flag (double-checked), so whichever side gets there first does the work exactly once. Give the worker a cancel flag so teardown doesn't wait for it.
Why it is faster: —
Applies elsewhere: games that stream level assets while showing a menu; IDEs that index files in the background; web apps that prefetch routes after first paint (`requestIdleCallback`).
Watch out: —
Seen in Mesen2: `NesConsole::LoadHdPack` starts a detached thread that runs `HdPackData::LoadAsync()`, calling `Init()` on every background and image bitmap and checking `_cancelLoad` between items. `HdPackBitmapInfo::Init()` now takes a `SimpleLock` and re-checks `_initDone` after acquiring it. The console destructor calls `CancelLoad()`, and `_hdData` became a `safe_ptr` so the worker can't outlive the data it uses. e9a94065 2023-03 “NES: HD Packs - Load PNG files in a separate thread to prevent stuttering in some packs that use huge PNG files”

### mesen2-096 — Replace linear find-by-name with a hash map
Lens: algorithms · Source: Mesen2
Principle: Checking each new entry against a list of existing ones is O(n²) over a whole load. Keep a side `unordered_map` from key to entry while loading, which makes the duplicate check O(1).
Why it is faster: —
Applies elsewhere: import scripts that dedupe rows by email; build tools that resolve module names; asset pipelines that share textures by path.
Watch out: —
Seen in Mesen2: `HdPackLoader::ProcessBackgroundTag` looped over all of `_data->BackgroundFileData` comparing `PngName` for every `<background>` line. It now looks the name up in `unordered_map<string, HdPackBitmapInfo*> _backgroundsByName`. ccc2bbbc 2023-03 “NES: HD Packs - Decode backgrounds on first use Makes loading HD packs faster”

### mesen2-097 — Bucket records by the key you filter on
Lens: algorithms · Source: Mesen2
Principle: If every frame scans a big list and skips most entries because a field doesn't match, store the entries in one array per value of that field. Empty buckets then cost nothing, and a non-empty bucket holds only candidates.
Why it is faster: —
Applies elsewhere: event dispatchers keyed by event type; render queues split per layer or material; schedulers with one run queue per priority.
Watch out: —
Seen in Mesen2: `HdPackData::Backgrounds`, a single vector, became `BackgroundsByPriority[BgLayerCount]`. Previously `GetLayerIndex(priority)` walked all backgrounds and did `continue` on a priority mismatch, once per priority level per frame. With thousands of background entries this cost a lot even for layers that had no backgrounds. `HdBgConfig` now stores `BgPriority` next to the index. 11dbb907 2023-03 “NES: HD Packs - Performance tweaks Keep BGs in separate arrays based on their priority…”

### mesen2-098 — Memoize per-key match results, and skip empty ranges in bulk
Lens: caching · Source: Mesen2
Principle: When each item has to be checked against a long rule list, and the same items come up again and again, cache the list of matching rules per item key, including "no matches". After the first time, each item costs one hash lookup instead of a scan of every rule. A tight inner `while` that skips over entries with nothing to process also keeps the outer loop body off the common empty case.
Why it is faster: —
Applies elsewhere: CSS selector matching cached per element signature; firewall rule results cached per flow tuple; routing tables memoized per URL pattern.
Watch out: —
Seen in Mesen2: `HdNesPack::ProcessAdditionalSprites` compared every sprite on every pixel with every `<addition>` rule, including fallback-tile lookups. The first version kept an `unordered_set` of tiles that had no additions. The next replaced it with `unordered_map<HdTileKey, vector<HdPackAdditionalSpriteInfo>> _additionalTilesByKey`, filled on first sight, and added `while(ScreenTiles[j].SpriteCount == 0) ++j;` to skip pixels with no sprites. 9e7a5ab1 2023-03 “NES: HD Packs - Improved performance when a large number of <addition> tags are defined” · 99acfc71 2023-03 “NES: HD Packs - Small performance improvement for additional sprites”

### mesen2-099 — Decode straight into the destination buffer
Lens: allocation · Source: Mesen2
Principle: A decoder that fills a temporary byte vector, which is then `memcpy`'d into the real typed array, allocates and touches every byte twice. Template the decoder on the element type, or hand it the final storage, so it writes in place. Use `resize()` on an existing vector instead of constructing a new zero-filled one.
Why it is faster: —
Applies elsewhere: `read()` into a preallocated `ArrayBuffer` instead of concatenating chunks; image libraries that decode into a caller-provided surface; protobuf parse-in-place.
Watch out: —
Seen in Mesen2: `PNGHelper::ReadPNG`/`DecodePNG` became `template<typename T>` over the output vector and size it as `out_size / sizeof(T)`. `HdPackLoader::ProcessImgTag` now decodes directly into `bitmapInfo.PixelData` (`vector<uint32_t>`), and the `pixelData` temporary and the `memcpy` are gone. `LoadFile` switched from `fileData = vector<uint8_t>(fileSize, 0)` to `fileData.resize(fileSize)`. 21afcb5b 2023-03 “NES: Slightly improve hd pack load times by reducing memory allocations” · e92685c5 2023-03 “NES: HD packs load optimizations”

### mesen2-100 — Scan text in place instead of splitting it into a vector of strings
Lens: io · Source: Mesen2
Principle: Splitting a whole file into `vector<string>` allocates one string per line, then copies each one again in a by-value loop. Walk the buffer with an index, reuse a single line string (`clear()` then `insert`), and test the most frequent record types first in the dispatch chain.
Why it is faster: —
Applies elsewhere: log parsers that use `string_view` over an mmapped file; CSV readers that reuse a row buffer; config loaders that put the common keys first in a tag switch.
Watch out: —
Seen in Mesen2: `HdPackLoader::LoadPack` did `for(string lineContent : StringUtilities::Split(string(data...), '\n'))`. It now runs a manual `while(pos < len)` loop that looks for `'\n'` and fills a reused `lineContent`. The `if/else` chain was reordered so `<tile>`, by far the most common tag, is tested first, followed by `<background>`, `<condition>` and `<img>`. e92685c5 2023-03 “NES: HD packs load optimizations”

### mesen2-101 — Use a virtual type tag and a switch instead of a dynamic_cast chain
Lens: branching-fast-paths · Source: Mesen2
Principle: `dynamic_cast` walks RTTI and may compare type-name strings, and a chain of them runs that walk once per candidate type. Give the base class a virtual `GetType()` that returns an enum, `switch` on it, and use `static_cast` inside each case.
Why it is faster: —
Applies elsewhere: AST visitors keyed on a node-kind enum (as in LLVM's `isa<>`/`dyn_cast<>`); UI event handling that switches on an event-type field; message buses that switch on a discriminator.
Watch out: —
Seen in Mesen2: `HdPackCondition` gained `virtual HdPackConditionType GetConditionType() = 0`, overridden by all eleven condition structs. `ProcessTileTag` and `ProcessConditionTag` replaced their `dynamic_cast<HdPackSpriteNearbyCondition*>` and `dynamic_cast<HdPackBaseTileCondition*>` chains with a `switch(condition->GetConditionType())` and C-style casts. e92685c5 2023-03 “NES: HD packs load optimizations”

### mesen2-102 — Use the OS's native text-conversion API instead of a generic locale facet
Lens: platform · Source: Mesen2
Principle: Standard library conversion facets such as `std::wstring_convert`/`codecvt` are general, deprecated, and often slow. Where the platform has a native converter, call it twice: once to get the output length and once to fill a presized buffer.
Why it is faster: —
Applies elsewhere: `TextDecoder` in browsers instead of manual byte loops; `iconv` or ICU on POSIX; `CFStringCreateWithBytes` on Apple platforms.
Watch out: —
Seen in Mesen2: `utf8::decode` in `Utilities/UTF8Util.cpp` now uses `MultiByteToWideChar(CP_UTF8, …)` on MSVC: one call gets the length, then the result is resized and filled. Other compilers keep the codecvt path. This was part of the HD-pack load work, where every file path goes through this conversion. e92685c5 2023-03 “NES: HD packs load optimizations”

### mesen2-103 — Store snapshots as XOR deltas against periodic keyframes, then compress quickly
Lens: memory-layout · Source: Mesen2
Principle: Consecutive snapshots of a large state are mostly identical. XOR each snapshot with the last full one: unchanged bytes become zeros, which a fast, low-level compressor squeezes to almost nothing. Insert a full keyframe every N snapshots so a restore never has to chain through more than one delta. Put variable-length fields at the end of the snapshot so the fixed-layout bytes stay at the same offsets and the XOR lines up.
Why it is faster: —
Applies elsewhere: video codecs (I-frames and P-frames); undo stacks in editors; database WAL with periodic checkpoints; incremental VM snapshots.
Watch out: —
Seen in Mesen2: `RewindData::SaveState` used to store the raw serialized state. It now serializes uncompressed. For any position where `position % 30 != 0`, it XORs the data with the previous full state (`ProcessXorState`), then calls `CompressionHelper::Compress(data, 1, …)` (miniz level 1). `LoadState` decompresses and XORs back. In every console's `Serialize`, `SV(_controlManager)`, whose input data varies in size, moved to the end. Smaller states let the manager snapshot more often, which shortens the delay when rewinding starts. c3634740 2023-02 “Rewind: Reduce memory usage, increase save state frequency (which reduces delay when rewind starts)”

### mesen2-104 — Cap a cache by bytes, not by entry count
Lens: memory-layout · Source: Mesen2
Principle: When entry sizes vary a lot (different consoles, compressed versus raw data), a limit on item count or duration can mean wildly different memory use. Total the real sizes, newest first, and evict everything older once the byte budget is exceeded.
Why it is faster: —
Applies elsewhere: LRU caches weighted by size (Caffeine's `maximumWeight`); browser HTTP caches; log rotation by size; CDN edge caches.
Watch out: —
Seen in Mesen2: `RewindManager::AddHistoryBlock` used to pop the oldest block while `_history.size() > RewindBufferSize*…`, with the buffer size given in seconds. It now walks `_history` from newest to oldest, adds up `GetStateSize()`, and pops everything before the point where `memoryUsage >> 20` exceeds the configured number of MB (default 300). db2db7dd 2022-12 “UI: Change rewind buffer setting to configure a maximum amount of memory rather than a maximum amount of time”

### mesen2-105 — Don't serialize what you can recompute on load
Lens: memory-layout · Source: Mesen2
Principle: Lookup tables derived from a few registers don't need to be saved. Rebuild them after loading. Snapshots get smaller, which matters most when they are taken constantly, as with rewind or netplay.
Why it is faster: —
Applies elsewhere: leaving indexes out of exported data and rebuilding them on import; storing seeds instead of generated content; saving app state without memoized selectors.
Watch out: —
Seen in Mesen2: `GbMemoryManager::Serialize` wrote five 256-entry arrays (`MemoryType`, `MemoryOffset`, `MemoryAccessType`, `IsRead/WriteRegister`) and replayed `Map()` from them. It now drops them and calls `RefreshMappings()` on load, which rebuilds the mappings from the saved bank registers. 79915cc5 2023-12 “GB: Remove unnecessary data from save states”

### mesen2-106 — Turn off background features in secondary instances that can't use them
Lens: memory-layout · Source: Mesen2
Principle: When you create a second copy of a subsystem for a preview or a sandbox, it inherits every always-on feature from the original configuration. Explicitly turn off the costly ones the copy will never use.
Why it is faster: —
Applies elsewhere: headless test browsers with extensions and GPU disabled; read replicas without some secondary indexes; preview renderers without analytics or autosave.
Watch out: —
Seen in Mesen2: `HistoryViewer::Initialize` creates a second `Emulator` to replay history. It now sets that instance's `RewindBufferSize = 0`, so the replay doesn't record its own rewind history in memory. 0cf91d30 2023-02 “History Viewer: Disable rewind history within the history viewer instance (lowers memory usage)”

### mesen2-107 — Track dirtiness and skip uploads and presents when nothing changed
Lens: gpu-rendering · Source: Mesen2
Principle: A render loop that wakes on a timer redraws and re-uploads overlays even when they are identical to last time. Record what was drawn, compare it with the previous output, and upload only on a change. Skip the whole present when no new frame arrived and no overlay is dirty. Make the wait primitive report whether it was signaled or timed out.
Why it is faster: —
Applies elsewhere: React's virtual-DOM diffing; dashboards that repaint only changed tiles; terminal emulators that redraw damaged lines; e-ink drivers with partial refresh.
Watch out: —
Seen in Mesen2: `DebugHud::Draw(…, clearAndUpdate)` now records draw commands into an `unordered_map<uint32_t,uint32_t>` of offset→color and compares it with `_drawPixels`. It clears and repaints the buffer and returns `IsDirty` only when the map differs. `AutoResetEvent::Wait` returns whether it was signaled. `VideoRenderer::RenderThread` calls `Render` only if `forceRender || _needRedraw || emuHud.IsDirty || scriptHud.IsDirty`. In `Renderer::DrawHud` (D3D11) the `Map`/copy is skipped unless the surface is dirty or the texture was recreated, and SDL later got the same treatment. In the sprite viewer, `SpritePreviewModel` compares each preview with a cached `_rawPreview` and skips the bitmap lock and copy when they match. af2afc4c 2023-10 “Reduced CPU usage for HUDs and when paused” · 9e6f9af4 2024-01 “Linux: Fixed SDL2 blank screen issue…” · 8411896c 2023-12 “Debugger: Sprite Viewer - Slightly improve refresh performance by avoiding redrawing sprites that haven't changed”

### mesen2-108 — Copy a whole surface with one memcpy when the strides match
Lens: memory-layout · Source: Mesen2
Principle: Copying row by row is only needed when the destination pitch differs from the source width. When the strides are equal, the surface is one contiguous block. A single `memcpy` avoids the per-row call overhead and lets the library use its widest copy loop.
Why it is faster: —
Applies elsewhere: uploads into GPU staging buffers, framebuffer blits, image crops that turn out to be full-width, numpy's contiguous fast path.
Watch out: —
Seen in Mesen2: `Renderer::DrawScreen` and `DrawHud` on Windows used a per-row `memcpy` into the mapped D3D11 texture. They now check `if(rowPitch != dd.RowPitch)` to keep the row loop, and otherwise call `memcpy(surfacePointer, videoBuffer, rowPitch * height)`. af2afc4c 2023-10 “Reduced CPU usage for HUDs and when paused”

### mesen2-109 — Double-buffer frames so the producer never waits on the consumer
Lens: concurrency · Source: Mesen2
Principle: If the producer has to hand off a frame synchronously, it stalls until the consumer is done. With two buffers, the producer writes into the back buffer, and the handoff is a pointer swap under a short lock. Use the same path for every mode, so no special case falls back to synchronous handoff.
Why it is faster: —
Applies elsewhere: swap chains in graphics APIs; ping-pong DMA buffers in embedded audio; triple buffering in video players; lock-free SPSC queues of size two.
Watch out: —
Seen in Mesen2: For interlaced frames, `SnesPpu` skipped the output-buffer swap and sent them with `UpdateFrame(frame, true, …)`, a synchronous handoff. Now it always swaps `_currentBuffer` and seeds the new buffer from the previous one with `memcpy` for interlace. Every frame goes through `UpdateFrame(frame, isRewinding, isRewinding)`. The commit message says this speeds up fast-forward when video filters are on. The new `SoftwareRenderer` does `memcpy` into `_textureBuffer[0]` under `_textureLock`, and `Render` just calls `std::swap(_textureBuffer[0], _textureBuffer[1])`. a86f5abf 2023-03 “SNES: Use double buffering for interlaced games (like all other games) Improves fast forward performance when video filters are used…” · 63bac5d4 2023-02 “macOS: Replace SDL renderer with Avalonia…”

### mesen2-110 — Reuse buffers across periodic refreshes in garbage-collected code
Lens: allocation · Source: Mesen2
Principle: A tool window that refreshes 30–60 times a second and allocates new arrays for memory dumps each time causes frequent GC pauses. Keep the arrays as fields, fill them through `ref` parameters with `Array.Resize` (which does nothing when the size is unchanged), and copy from a "core" buffer into a "UI" buffer under a lock. That way the background refresh and the UI thread never share one array. Size per-call scratch buffers to a realistic maximum, not a huge worst case.
Why it is faster: —
Applies elsewhere: `ArrayPool<T>` in .NET; reused `ByteBuffer`s in Java NIO loops; typed-array pools in WebGL apps; Unity code that avoids per-frame `new`.
Watch out: —
Seen in Mesen2: `DebugApi.GetMemoryState(type, ref byte[] dst)` and `GetMemoryAccessCounts(type, ref AddressCounters[])` were added. The tilemap, sprite and tile viewers now keep `_coreData`/`_data` pairs guarded by `_updateLock`. The profiler keeps one `ProfiledFunction[100000]` instead of allocating it on every refresh. `EmuApi.GetLog`, polled every 100 ms, cut its string buffer from 1,000,000 to 100,000 bytes. 126cf052 2023-04 “Debugger: Tilemap viewer - Reduce memory allocations to reduce GC frequency” · 5926ada4 2023-04 “Debugger: Sprite viewer - Reduce memory allocations…” · 9c6c6744 2023-04 “Debugger: Profiler - Reduce memory allocations” · 68e1f0ce 2023-04 “Debugger: Tile viewer - Reduce memory allocations” · fea4df8f 2023-04 “UI: Reduce memory allocations for log window”

### mesen2-111 — Pin with a scoped fixed block instead of allocating a GC handle
Lens: allocation · Source: Mesen2
Principle: Allocating a pinned `GCHandle` creates a handle-table entry, requires an explicit free, and leaks if an exception hits before `Free()`. A C# `fixed` block pins only for its lexical scope, using a stack-tracked reference the JIT can treat specially. It is cheaper and cannot leak.
Why it is faster: —
Applies elsewhere: `Span<T>` with `MemoryMarshal.GetReference` for P/Invoke; JNI `GetPrimitiveArrayCritical` for short critical sections; Rust slices passed as `as_ptr()` within a borrow.
Watch out: —
Seen in Mesen2: `DebugApi.GetExecutionTrace`, `GetProfilerData`, `GetTokenList` and `GetMemoryAccessCounts`, and `InputApi.GetPressedKeys`, all replaced `GCHandle.Alloc(arr, Pinned)` / `AddrOfPinnedObject()` / `Free()` with `unsafe … fixed(T* ptr = arr) { Wrapper((IntPtr)ptr, …); }`. cde0db1d 2023-04 “Replaced GCHandle.Alloc usage with fixed keyword”

### mesen2-112 — Batch UI notifications during bulk changes
Lens: gpu-rendering · Source: Mesen2
Principle: Each "changed" event can trigger a full re-render or re-sort of every listener. During a bulk operation, suspend events with a counter, add children with one `AddRange`, and wrap many selections in a begin/end batch. One notification fires at the end instead of N.
Why it is faster: —
Applies elsewhere: `BeginUpdate`/`EndUpdate` on WinForms lists; batched state updates in React; `DocumentFragment` in the DOM; database bulk inserts with deferred index maintenance.
Watch out: —
Seen in Mesen2: `LabelManager` gained `SuspendEvents()`/`ResumeEvents()`, built on `Interlocked`. `ProcessLabelUpdate` fires `OnLabelUpdated` only when `_suspendEvents == 0`, and `DebugWorkspaceManager.Load` wraps the workspace, symbol and CDL import in a suspend/resume pair. `DataBoxCellsPresenter` builds a `List<DataBoxCell>` and calls `Children.AddRange`. A new `SelectionModelExtensions.SelectIndexes` wraps reselection in `BeginBatchUpdate()`/`EndBatchUpdate()`. That fixed slow breakpoint add/remove when many rows were selected, and five other list view models use it too. 7c482c74 2022-12 “Debugger: Minor performance improvements when reloading a rom with debugger window opened…” · dfee4d92 2023-04 “Debugger: Fixed performance issue when adding/removing breakpoint while a lot of breakpoints are selected in the list”

### mesen2-113 — Tell listeners what kind of reload happened so they can skip unneeded work
Lens: laziness · Source: Mesen2
Principle: A generic "loaded" event makes every subscriber redo its full initialization. Add the reason to the event (fresh load or restart of the same content), and listeners can skip disk reads and reconfiguration whose inputs haven't changed.
Why it is faster: —
Applies elsewhere: hot-reload events that carry the changed module so other modules aren't re-evaluated; Kubernetes informers that tell resync apart from change; config watchers that report which keys changed.
Watch out: —
Seen in Mesen2: `GameLoaded` notifications now carry `GameLoadedEventParams { IsPaused, IsPowerCycle }`. On a power cycle, `DebugWindowManager` no longer calls `DebugWorkspaceManager.Load()` (which reads JSON from disk). It just re-sends breakpoints and calls `LabelManager.RefreshLabels(false)`. `ToolRefreshHelper` skips the CPU-type, `OnGameLoaded` and min/max refresh on a power cycle. 9494aec1 2022-10 “Debugger: Don't reload debugger workspace from disk when power cycling” · 4ca6af71 2022-10 “Debugger: Only update tools cpu type/etc when loading a rom, not when power cycling”

### mesen2-114 — Use virtualizing controls for large or growing content
Lens: gpu-rendering · Source: Mesen2
Principle: Generic text boxes and repeaters build layout for the whole content, so selection, scrolling and updates cost O(total size). Controls that lay out only the visible lines or items, such as a code-editor component or a virtualizing panel, keep each interaction proportional to what is on screen.
Why it is faster: —
Applies elsewhere: react-window or virtual scrollers for long lists; Monaco or CodeMirror for log viewers; `RecyclerView` on Android; `UITableView` cell reuse.
Watch out: —
Seen in Mesen2: The log window and debug log replaced a `TextBox` bound to the whole log string with `MesenTextEditor` (AvaloniaEdit), which fixed slow text selection, and scroll with `ScrollToEnd()`. In the debugger shortcut grid, the `ItemsRepeater` was replaced with an `ItemsControl` using a `VirtualizingStackPanel` items panel, and other `ItemsRepeater` uses were swapped for `ItemsControl`. 3c210ba9 2022-09 “UI: Replaced TextBox with AvaloniaEdit for log window/debug log (fixes performance issues when selecting text)” · 84e27a52 2023-03 “UI: Replaced ItemsRepeater usage with ItemsControl”

### mesen2-115 — Bind once at the container instead of walking ancestors per item
Lens: gpu-rendering · Source: Mesen2
Principle: Bindings that search the visual tree for an ancestor (`$parent[...]`) are resolved per templated item and fail noisily before the tree is attached. Error logging during layout costs real time. Put the binding on the container, which already has the data context, and select children with a style selector.
Why it is faster: —
Applies elsewhere: setting a CSS class on a list's parent instead of on every row; React context instead of prop-drilling lookups; WPF `RelativeSource FindAncestor` replaced by direct DataContext binding.
Watch out: —
Seen in Mesen2: In `DynamicTooltip.axaml`, each inner `TextBox` had `Classes.mono="{Binding $parent[StackPanel].DataContext.UseMonoFont}"`. The classes now go on the outer `StackPanel` (`Classes.mono="{Binding UseMonoFont}"`), and the styles became `StackPanel.mono TextBox`. This removed the binding warnings that caused the slowdown. 771ebff7 2022-10 “Debugger: Fixed binding warnings/errors shown in VS (caused slowdown)”

### mesen2-116 — Compile out parts of a general library your target never uses
Lens: other · Source: Mesen2
Principle: A third-party core written for a superset device runs every channel and feature on every sample. If your hardware variant has only a subset, compile out the extra work with a define. The inner mixing loop shrinks and fewer registers are handled.
Why it is faster: —
Applies elsewhere: building SQLite with `SQLITE_OMIT_*`; trimming codec builds to the formats you need; disabling unused peripherals in an embedded HAL.
Watch out: —
Seen in Mesen2: emu2413 emulates the YM2413's 9 FM channels plus rhythm. The VRC7 has 6 channels. `#define VRC7AUDIO` in `emu2413.cpp` skips `update_short_noise`, the CH7–9 rhythm block and `update_noise`, cuts `mix_output` to `for(i = 0; i < 6; i++)`, and drops register cases `0x16–0x18`/`0x26–0x28`/`0x36–0x38`. Save states serialize 12 slots instead of 18. 5a37dd42 2022-10 “NES: VRC7 audio - Disable processing/output of extra channels that only exist in the YM2413”

### mesen2-117 — Test the cheapest, most selective condition first
Lens: branching-fast-paths · Source: Mesen2
Principle: In a per-access hook, check an inexpensive flag or an emptiness test before a hash lookup or classification function. When the feature is off or the set is empty, the hook costs a single predictable branch.
Why it is faster: —
Applies elsewhere: `if (listeners.length)` before building an event object; `if (logger.isDebugEnabled())` before formatting; `if (!cache.empty())` before a lookup.
Watch out: —
Seen in Mesen2: `FrozenAddressManager::IsFrozenAddress`, which runs on every debugger memory write, returns `_frozenAddresses.size() > 0 && _frozenAddresses.find(addr) != end()`. `MemoryAccessCounter::ProcessMemoryRead` checks `_enableBreakOnUninitRead && counts.WriteStamp == 0` before calling `DebugUtilities::IsVolatileRam`. Before this change, each CPU debugger computed the uninit result first and tested its own flag afterwards. 1df7d9ae 2022-09 “Debugger: Added freeze address option in memory viewer” · 44233413 2023-04 “Debugger: Prevent load state/reset counters from triggering uninit ram read break on all reads”

### mesen2-118 — Filter rules when they are installed, not when they are checked
Lens: laziness · Source: Mesen2
Principle: If a rule property decides which events it applies to, use that property when building the per-event lookup lists. The hot path then iterates only rules that can match and never re-tests the property.
Why it is faster: —
Applies elsewhere: event routers that register handlers per event type; packet filters compiled into per-protocol tables; feature-flag rules pre-partitioned by user segment.
Watch out: —
Seen in Mesen2: A new breakpoint option ignores dummy reads and writes. Instead of checking it on each access, `BreakpointManager::SetBreakpoints` builds the per-operation-type lists and pushes `bp` into `_breakpoints[i]` only if `bp.IsAllowedForOpType(opType)`. 84a265ea 2023-04 “Debugger: Added option to ignore dummy reads/writes in breakpoints”

### mesen2-119 — Skip identity transforms when building lookup tables
Lens: lookup-tables · Source: Mesen2
Principle: When user settings are at their neutral values, the color-adjustment math is a no-op that can still introduce rounding errors. Detect the neutral case and copy the source table as-is. This saves the floating-point round trip and gives exact output.
Why it is faster: —
Applies elsewhere: image pipelines that skip gamma or ICC conversion when the profiles match; audio paths that bypass resampling at equal rates; matrix code with an identity fast path.
Watch out: —
Seen in Mesen2: `NesDefaultVideoFilter::InitLookupTable` always ran 512 RGB→YIQ→RGB conversions. It now does that only `if(Brightness || Contrast || Hue || Saturation)`, and otherwise calls `GetFullPalette(_calculatedPalette, …)` directly. The shared math moved to `BaseVideoFilter::ApplyColorOptions` for the GB, PCE and SNES filters. 2085af03 2023-02 “NES: Fixed colors being slightly different than palette Caused by the code used to apply brightness/etc. settings…”

### mesen2-120 — Release window graphs on close: named handlers, disposed view models
Lens: memory-layout · Source: Mesen2
Principle: A lambda subscribed to a long-lived object's event captures its surroundings and keeps them alive. Windows that open and close repeatedly then pile up retained memory and GC work. Use a static or named handler that gets everything from `sender`, dispose the DataContext on close, and clear references.
Why it is faster: —
Applies elsewhere: removing DOM listeners in a React effect cleanup; weak event patterns in WPF; `[weak self]` in Swift closures; unsubscribing RxJS streams in `ngOnDestroy`.
Watch out: —
Seen in Mesen2: `DebugWindowManager.CreateDebugWindow` used to attach an inline lambda to `wnd.Closed`. It now attaches a static `OnClosedHandler(object? sender, …)`, which disposes `window.DataContext` and unregisters the window. The commit notes that it "fixes memory leaks when closing windows (by setting DataContext to null, etc.)". 716227c0 2023-04 “UI: Update to Avalonia11-preview6 … Fixes memory leaks when closing windows”

### mesen2-121 — Don't route rendering through an indirect or remoted graphics path
Lens: platform · Source: Mesen2
Principle: Indirect GL rendering, which serializes every GL call through the X server, avoids some driver bugs but costs far more CPU per frame. A plain software renderer is often cheaper than hardware rendering forced through such a path.
Why it is faster: —
Applies elsewhere: avoiding remote X forwarding for GPU apps; turning off ANGLE's debug or validation layers in production; preferring a CPU canvas to a WebGL context that falls back to SwiftShader.
Watch out: —
Seen in Mesen2: A Linux workaround for blank SDL output set `LIBGL_ALWAYS_INDIRECT=true` and relaunched the process. It was removed because "it causes it to have much higher CPU usage than normal". Users were pointed to the new software renderer option instead. 1d7285d5 2023-12 “Linux: Remove LIBGL_ALWAYS_INDIRECT patch…”

### mesen2-122 — Turn a runtime mode flag into a template parameter
Lens: compiler-build · Source: Mesen2
Principle: If a hot function takes a boolean that its caller always knows at compile time, make it a template parameter instead. Each instantiation then compiles with the flag as a constant, so dead branches disappear and the remaining code can be scheduled tightly.
Why it is faster: —
Applies elsewhere: a pixel blitter with `blend<bool premultiplied>`, a serializer with `write<bool pretty>`, a DSP routine with `process<bool stereo>`.
Watch out: —
Seen in Mesen2: `GbPpu::Exec(bool singleStep)` ran on every Game Boy PPU tick and checked the flag at run time. It became `template<bool singleStep> void Exec()`, and the call sites call `_ppu->Exec<true>()` or `Exec<false>()`. The same commit force-inlined `ProcessNextCycleStart` and `ExecTimerDmaSerial`, the per-cycle helpers around it. 36d23cc4 2024-01 “GB: Performance tweaks”

### mesen2-123 — Cache "is this hook overridden?" as a plain flag
Lens: branching-fast-paths · Source: Mesen2
Principle: A virtual hook that most subclasses leave empty still costs an indirect call on every invocation. Have each subclass declare once, at setup, whether it uses the hook, and store the answer in a bool. The hot path tests the bool and makes the virtual call only when it is set.
Why it is faster: —
Applies elsewhere: plugin systems that record which callbacks a plugin implements, middleware chains that skip handlers with no-op `onRequest`, game-engine components with an `wantsUpdate` flag.
Watch out: —
Seen in Mesen2: NES mappers gained `EnableCpuClockHook()`, `EnableCustomVramRead()` and `EnableVramAddressHook()`. `BaseMapper::Initialize` caches them into `_hasCpuClockHook` and related fields, and only the few mappers that need a hook return true. PPU VRAM reads now take the inline default unless a mapper opted in: `if(!_hasCustomReadVram) { value = InternalReadVram(addr); } else { value = MapperReadVram(addr, type); }` 4c1579d8 2024-09 “NES: Small performance improvements”

### mesen2-124 — Put hot helper bodies in the header so they can inline
Lens: compiler-build · Source: Mesen2
Principle: The compiler can only inline a function whose body it can see. A tiny function defined in a .cpp file stays an out-of-line call for callers in other translation units, unless you use LTO. Moving the body into the class definition and marking it force-inline removes the call overhead and lets the optimizer fold it into the caller.
Why it is faster: —
Applies elsewhere: getters and bounds checks in a C++ container library, small math helpers in a game engine, `static inline` accessors in C driver headers.
Watch out: —
Seen in Mesen2: The NES `BaseMapper::InternalReadVram` moved from BaseMapper.cpp into BaseMapper.h as `__forceinline`. The GBA `GbaCpu::CheckConditions`, a 16-case switch on ARM condition codes that runs for every ARM instruction, moved from GbaCpu.Arm.cpp into GbaCpu.h as `__forceinline` as well. 4c1579d8 2024-09 “NES: Small performance improvements”; 87ee4da5 2025-05 “GBA: Minor performance optimizations”

### mesen2-125 — Check a pending flag before calling a per-tick subsystem
Lens: branching-fast-paths · Source: Mesen2
Principle: When a subsystem is idle almost all the time, don't call into it every tick only for it to return immediately. Expose an inline "has work" flag and test it at the call site. The common case becomes one load and a well-predicted branch.
Why it is faster: —
Applies elsewhere: a game loop that checks `queue.empty()` before calling the event dispatcher, a network poller that skips TLS processing when no bytes are buffered.
Watch out: —
Seen in Mesen2: `SnesCpu::ProcessCpuCycle` used to call `_dmaController->ProcessPendingTransfers()` unconditionally. It now checks `HasPendingTransfer()` first, which just returns `_needToProcess`. The NES got the same treatment for controller writes: `if(_controlManager->HasPendingWrites()) _controlManager->ProcessWrites();`. 30d28c83 2024-12 “SNES: Performance improvements”; 6a16f729 2024-09 “NES: Improve CPU OUT pins emulation”

### mesen2-126 — Reselect a specialized function pointer when the mode changes
Lens: compiler-build · Source: Mesen2
Principle: If a hot operation takes a parameter that changes rarely, such as a speed setting, generate one unrolled version per value. Store a pointer to the right version, and repoint it only when the setting changes. Each call then does a fixed amount of work with no loop counter and no arithmetic on the parameter.
Why it is faster: —
Applies elsewhere: pick an SSE, AVX or NEON kernel once at startup, swap a codec's inner loop when the bit depth changes, or switch a logger's sink function when the log level changes.
Watch out: —
Seen in Mesen2: SNES memory reads called `IncrementMasterClockValue(_cpuSpeed - 4)`. Now `Read` calls `(this->*_execRead)()`, and `SetCpuSpeed` calls `UpdateExecCallbacks()` only when the speed actually changes. That function picks `IncMasterClock<2/4/6/8/12>`, each a straight-line run of `if constexpr`-expanded `Exec()` calls. The pointers are also rebuilt after loading a save state. 30d28c83 2024-12 “SNES: Performance improvements”

### mesen2-127 — Put a hot loop and its callees in the same translation unit
Lens: compiler-build · Source: Mesen2
Principle: If a driver loop lives in one source file and the per-step functions live in another, every step is an opaque call. Move the loop next to the step functions, mark them force-inline, and the whole loop body can be optimized as one piece.
Why it is faster: —
Applies elsewhere: keep a bytecode interpreter's dispatch loop in the same file as its opcode handlers, and keep a parser's main loop in the same file as its tokenizer helpers.
Watch out: —
Seen in Mesen2: `Spc::Run()` and `Spc::ProcessCycle()` moved from Spc.cpp into Spc.Instructions.cpp, which defines `Exec()`. `ProcessCycle`, `Exec` and `IncCycleCount` are now declared `__forceinline`, so `while(_state.Cycle < targetCycle) ProcessCycle();` inlines down to the opcode step. 30d28c83 2024-12 “SNES: Performance improvements”

### mesen2-128 — Move the early-exit test out of the callee and into an inline predicate
Lens: branching-fast-paths · Source: Mesen2
Principle: A function that starts with "if there is nothing to do, return" still costs a call every time. Split that test into a force-inline predicate and check it at the call site. In the common no-op case the call disappears entirely.
Why it is faster: —
Applies elsewhere: `if(log.enabled(level)) log.write(...)`, and `if(dirty) flush()` in caches and render-state managers.
Watch out: —
Seen in Mesen2: `GbaRomPrefetch::Exec` began with `if(_state.ReadAddr == 0 || IsFull() || ...) return;`. That test became `__forceinline bool NeedExec(bool prefetchEnabled)`, and every caller in the memory manager now writes `if(_prefetch->NeedExec(...)) _prefetch->Exec(...)`. A later commit wrapped this in a `RunPrefetch()` helper. 87ee4da5 2025-05 “GBA: Minor performance optimizations”; aec12b17 2025-06 “GBA: Performance fixes”

### mesen2-129 — Skip a transform when it would leave the input unchanged
Lens: branching-fast-paths · Source: Mesen2
Principle: If a correction step does nothing in the common case, test for that case first with a cheap check, such as an alignment mask, and skip the call.
Why it is faster: —
Applies elsewhere: skip the byte swap when the data is already in host order, skip re-encoding when a string is already ASCII, and skip the unaligned-load path when a pointer is already aligned.
Watch out: —
Seen in Mesen2: GBA 16- and 32-bit reads always called `RotateValue`. The rotation only matters for misaligned addresses, so the check became `if(!(mode & GbaAccessMode::NoRotate) && (addr & 0x03))`, and `addr & 0x01` for halfwords. 87ee4da5 2025-05 “GBA: Minor performance optimizations”

### mesen2-130 — Embed a small lookup table by value in an inline-able helper
Lens: memory-layout · Source: Mesen2
Principle: If a lookup table sits behind a heap pointer and its getter is out of line, every lookup costs a call plus a pointer chase. Put the table in a small struct held by value, define the getter in a header, and the lookup becomes one indexed load from a known offset.
Why it is faster: —
Applies elsewhere: a CRC table as a `static const` array next to an inline update function, and a fixed-size routing table stored inside the router object instead of allocated separately.
Watch out: —
Seen in Mesen2: `_waitStatesLut = new uint8_t[0x400]` and the out-of-line `GetWaitStates`/`GetPrefetchWaitStates` moved into a new header-only `GbaWaitStates` class, which the memory manager holds by value. The ROM prefetcher now holds a `GbaWaitStates*` rather than a pointer to the whole memory manager, which also untangled the header dependencies so the lookup can inline. aec12b17 2025-06 “GBA: Performance fixes”

### mesen2-131 — Split hot code into an inlined fast path and an out-of-line slow path
Lens: branching-fast-paths · Source: Mesen2
Principle: Force-inline the short, common path. Move the rare, bulky path into a separate function marked noinline. Callers stay small and keep their code in the instruction cache, and the rare case still costs only one call.
Why it is faster: —
Applies elsewhere: a small-string-optimized append that calls a `grow()` function out of line, an allocator fast path that falls back to `refill()`, and the `[[unlikely]]` error-handling block in a parser.
Watch out: —
Seen in Mesen2: `GbaMemoryManager::ProcessIdleCycle` became a `__forceinline` function in the header. The rare branch, where idle cycles run in parallel with DMA, was moved out into `ProcessParallelIdleCycle()`. NES controller writes follow the same pattern: `__forceinline bool HasPendingWrites()` is paired with `__noinline void ProcessWrites()`. aec12b17 2025-06 “GBA: Performance fixes”; 6a16f729 2024-09 “NES: Improve CPU OUT pins emulation”

### mesen2-132 — Build immutable dispatch tables once per process
Lens: allocation · Source: Mesen2
Principle: A table that depends only on code, such as an opcode-to-handler map, should be static and filled once. If it is a per-instance member, it gets rebuilt every time a throwaway instance is created, and that cost can dominate when instances are short-lived.
Why it is faster: —
Applies elsewhere: compile a regex once into a static, build a parser's keyword table once, and share one JSON schema validator across requests.
Watch out: —
Seen in Mesen2: The GBA trace logger creates a `DummyGbaCpu` to predict memory accesses. Each instance ran `InitArmOpTable()`/`InitThumbOpTable()` to fill a 4096-entry and a 256-entry handler table. Those tables became `static Func _armTable[0x1000]` and `_thumbTable[0x100]`, filled once by `GbaCpu::StaticInit()` in the `GbaConsole` constructor. 6468f379 2024-03 “Debugger: GBA - Fixed performance issue with trace logger”

### mesen2-133 — Wrap the pixel buffer in place instead of converting it
Lens: gpu-rendering · Source: Mesen2
Principle: High-level image APIs often copy or convert a bitmap before drawing it. Wrap the existing pixel memory in the graphics library's native bitmap type and issue the draw call directly. This saves a full-frame copy and a format conversion on every frame.
Why it is faster: —
Applies elsewhere: `SkBitmap::installPixels` over a decoder's output, an `ImageData` view over a WASM memory buffer, and `CVPixelBuffer` wrapping existing memory on macOS.
Watch out: —
Seen in Mesen2: The software renderer's `SimpleImageViewer` used `context.DrawImage(Source, ...)`. It now issues `context.Custom(new DrawOperation(this))`. The operation calls `_bitmap.InstallPixels(info, lockedBuffer.Address)` and draws through Skia directly with `canvas.DrawBitmap`, choosing nearest-neighbour or high-quality filtering for each draw. f7f9ce76 2024-09 “UI: Improved software renderer performance”

### mesen2-134 — Transfer only the window the viewer shows
Lens: io · Source: Mesen2
Principle: Copying a whole data source to show a small part of it scales with the size of the source, not with what is on screen. Ask for just the visible range, and refetch when that range changes.
Why it is faster: —
Applies elsewhere: virtualized list views, paginated or range-limited database queries, and HTTP Range requests for media scrubbing.
Watch out: —
Seen in Mesen2: The tile viewer called `DebugApi.GetMemoryState(Config.Source, ...)`, which copied the entire memory region. For GBA "CPU Memory" that is very large. It now calls `GetMemoryValues(Source, StartAddress, StartAddress + totalSize - 1)`, where `totalSize = bytesPerTile * rows * columns`, and it refreshes when the start address or layout changes. The tile editor fetches one tile at a time. 5b0173f4 2024-10 “Debugger: Tile viewer - Limit copied data to what is required...”

### mesen2-135 — Count with bit arithmetic instead of if/else
Lens: branching-fast-paths · Source: Mesen2
Principle: When you count elements by category, extract each flag as a 0 or 1 value and add it. Correct for overlaps with a separate AND count. The loop then has no data-dependent branches, so it runs at a steady rate and the compiler can vectorize it.
Why it is faster: —
Applies elsewhere: histogramming flags in a columnar database, counting set permission bits across records, and computing per-category statistics over large byte arrays.
Watch out: —
Seen in Mesen2: The CDL statistics loop used `if(IsCode(i)) codeSize++; else if(IsData(i)) dataSize++;`. It now does `isCode = d & Code; isData = (d & Data) >> 1; codeSize += isCode; dataSize += isData; bothSize += isCode & isData;` and subtracts `bothSize` from `dataSize` at the end. This made it cheap enough to recompute the stats on every refresh while the game runs. be41cdea 2024-12 “Debugger: ... Optimize CDL stats calculation”

### mesen2-136 — Coalesce UI refresh requests while one is pending
Lens: concurrency · Source: Mesen2
Principle: If a producer posts refresh jobs to a UI thread faster than the UI can run them, the queue grows without limit and the UI freezes. Keep a "refresh pending" flag. While a refresh is queued, drop new requests, because the queued one will show the latest state anyway.
Why it is faster: —
Applies elsewhere: `requestAnimationFrame` debouncing in browsers, coalescing `setNeedsDisplay`, and throttling websocket-driven re-renders.
Watch out: —
Seen in Mesen2: The event viewer, sprite viewer, tile viewer and tilemap viewer each gained `_refreshPending`. `RefreshUi` returns early if it is set. Otherwise it sets the flag and posts work to `Dispatcher.UIThread`, and the posted work clears the flag when it finishes. The event viewer's list view was already rate-limited to one refresh every 66 ms. 8ad34343 2024-12 “Debugger: Prevent UI freeze when viewer updates pile up...”

### mesen2-137 — Dispatch once to loop variants specialized on invariant flags
Lens: compiler-build · Source: Mesen2
Principle: If loop-invariant booleans are tested inside a tight nested loop, branch on them once outside the loop and call a version of the loop compiled for each combination. Also move each subexpression up to the outermost loop level where its inputs stop changing.
Why it is faster: —
Applies elsewhere: image filters specialized for "has alpha" or "flip X", and matrix kernels specialized on transpose flags.
Watch out: —
Seen in Mesen2: The SNES tilemap renderer for the debugger took `bool largeTileHeight, largeTileWidth` as arguments and tested them per pixel. A wrapper now dispatches to `RenderTilemap<format, true/false, true/false>`. The Y half of the large-tile offset is computed as `yTileOffset` once per row instead of once per pixel, and `GetTilePixelColor` became `__forceinline`. 8ad34343 2024-12 “Debugger: Prevent UI freeze ... slightly improves performance for the SNES tilemap viewer”

### mesen2-138 — Size a container in one step instead of growing it element by element
Lens: allocation · Source: Mesen2
Principle: A loop of `push_back` calls, even after `reserve`, does a capacity check and a store per element. Constructing the vector with its size zero-fills it in one pass, which compiles to a memset. Skip the allocation entirely when the size is zero.
Why it is faster: —
Applies elsewhere: `new Array(n).fill(0)` or typed arrays in JS, `make([]T, n)` in Go, and `vec![0; n]` in Rust.
Watch out: —
Seen in Mesen2: `MemoryAccessCounter` filled one counter per byte of every memory type with `for(j < memSize) _counters[i].push_back({})`. This runs when the debugger initializes and can mean megabytes of entries. It is now `if(memSize > 0) _counters[i] = vector<AddressCounters>(memSize);`. 9e5d013c 2025-03 “Debugger: Minor performance improvements for debugger initialization”

### mesen2-139 — Lend the loaded buffer instead of copying it out
Lens: memory-layout · Source: Mesen2
Principle: If an object already holds file data in memory, returning a reference to it avoids a second allocation and a full copy. This matters most for large, read-once data.
Why it is faster: —
Applies elsewhere: `std::string_view` and `std::span` accessors, Go's `bytes.Buffer.Bytes()`, and zero-copy parsers that hand out slices of the input.
Watch out: —
Seen in Mesen2: `CodeDataLogger::LoadCdlFile` copied the file with `vector<uint8_t> cdlData; cdlFile.ReadFile(cdlData);`. A new `VirtualFile::GetData()` returns `vector<uint8_t>&` to the already-loaded `_data`, which the loader now reads from directly. 9e5d013c 2025-03 “Debugger: Minor performance improvements for debugger initialization”

### mesen2-140 — Don't invalidate a cache that is already empty
Lens: laziness · Source: Mesen2
Principle: At construction or first use, a cache has nothing to invalidate. Pass a flag so the initialization path skips the reset work that later refreshes still need.
Why it is faster: —
Applies elsewhere: skip `clear()` on freshly created pools, and skip a CDN purge on a first deploy.
Watch out: —
Seen in Mesen2: `CdlManager::RefreshCodeCache()` always called `_disassembler->ResetPrgCache()`. It now takes `bool resetPrgCache = true`, and the `Debugger` constructor passes `false`. 9e5d013c 2025-03 “Debugger: Minor performance improvements for debugger initialization”

### mesen2-141 — Skip expensive serialization sections the consumer never uses
Lens: laziness · Source: Mesen2
Principle: One serializer may feed several consumers with different needs. If one section requires heavy work, such as diffing or patch generation, check which output format is being produced and skip that section when the consumer doesn't need it.
Why it is faster: —
Applies elsewhere: leave large blobs out of debug JSON dumps, and skip computing checksums when serializing for an in-memory cache.
Watch out: —
Seen in Mesen2: Lua's `emu.getState()` goes through the NES serializer. For mappers with flash saves, `SerializeRomDiff` copied the whole PRG ROM and built an IPS patch on every call. It now begins with `if(s.GetFormat() != SerializeFormat::Binary) return; //Skip this completely for Lua`. 57f898da 2025-05 “Debugger: Lua - Fixed performance issues with getState on NES...”

### mesen2-142 — Parallelize the link-time optimization step
Lens: compiler-build · Source: Mesen2
Principle: Whole-program LTO gives the best code, but its link step runs on one thread by default and becomes the build bottleneck. Partitioned, parallel LTO uses every core and generates nearly the same code.
Why it is faster: —
Applies elsewhere: `-flto=auto` with GCC, `-flto=thin` with Clang, and `codegen-units` combined with `lto="thin"` in Rust.
Watch out: —
Seen in Mesen2: The release makefile flags changed from `-O3 -flto` to `-O3 -flto=auto -DHAVE_LTO`. a71ae7fd 2024-08 “Linux: Enable parallel LTO in Makefile (#65)”

### mesen2-143 — Compile the managed front end ahead of time
Lens: compiler-build · Source: Mesen2
Principle: JIT-compiled runtimes spend startup time compiling methods and warming up. Ahead-of-time native compilation removes that cost, makes startup and memory use more predictable, and drops the runtime dependency. The price is that reflection-heavy code has to become trimming-safe.
Why it is faster: —
Applies elsewhere: .NET NativeAOT for CLI tools, GraalVM native-image for Java microservices, and Dart AOT for Flutter release builds.
Watch out: —
Seen in Mesen2: The UI was made compatible with NativeAOT. Config classes and bindings were adjusted for trimming, and CI gained `-p:PublishAot=true` builds for Windows and Linux, with a `USE_AOT=true` make flag on Linux. d208fdc6 2024-06 “UI: Support for NativeAOT”

### mesen2-144 — Give latency-critical polling timers a higher dispatcher priority
Lens: concurrency · Source: Mesen2
Principle: UI frameworks run timer callbacks by priority. A timer at background priority can be starved by layout and rendering work, which adds visible input lag. Raise the priority of timers that sample user input.
Why it is faster: —
Applies elsewhere: raise the dispatcher priority of WPF or Avalonia input timers, schedule input handling at user-blocking priority on the web, and give an input thread higher RTOS priority in embedded systems.
Watch out: —
Seen in Mesen2: The mouse and light-gun polling timer changed from `new DispatcherTimer()`, which defaults to background priority, to `new DispatcherTimer(DispatcherPriority.Normal)`. ed4d97b3 2024-06 “Input: Improve responsiveness for mouse/light gun input devices”

### mesen2-145 — Write a file only if its contents changed
Lens: io · Source: Mesen2
Principle: Keep a snapshot of the data as it was loaded and compare it with the current data before saving. Unchanged data then causes no disk writes, no file creation and no flash wear.
Why it is faster: —
Applies elsewhere: build tools that skip rewriting identical outputs to keep timestamps stable, config editors that save only when the content is dirty, and EEPROM drivers that compare before writing.
Watch out: —
Seen in Mesen2: The PC Engine CD-ROM keeps `_orgSaveRam`, which is filled with `memcpy` right after the battery file is loaded. `SaveBattery` now writes only `if(memcmp(_orgSaveRam, _saveRam, _saveRamSize) != 0)`, so a game that never touches save RAM never gets a .sav file. 2aa9cc59 2024-09 “PCE: ... only create .sav file if game actually writes to save ram”

### mesen2-146 — Advance subsystems lazily, but sync before anyone looks
Lens: laziness · Source: Mesen2
Principle: A component that is advanced only on demand saves work, but its visible state goes stale between catch-ups. Every place that observes that state, such as a debugger break, a register read or the end of a frame, must first force a catch-up.
Why it is faster: —
Applies elsewhere: lazy physics stepping that syncs before a raycast, write-back caches that flush before a snapshot, and deferred layout that runs before `getBoundingClientRect`.
Watch out: —
Seen in Mesen2: The APUs are only run when needed. A new `IDebugger::OnBeforeBreak()` hook is called from `Debugger::SleepUntilResume`, and each console implements it to catch its APU up, for example `_console->GetApu()->Run()` on GBA and `_gameboy->RunApu()` on Game Boy. Register viewers then show current values without the APU being run every cycle. 36bd1611 2025-04 “Debugger: Catch up APU for all consoles when breaking...”

### mesen2-147 — Prune rules that can never match, once at load time
Lens: algorithms · Source: Mesen2
Principle: If some rules in a set can never apply in the current context, remove them once when the set is loaded. Otherwise the matcher re-checks and rejects them on every evaluation.
Why it is faster: —
Applies elsewhere: drop CSS rules for media types that don't apply, compile firewall rules per interface, and filter feature-flag rules per tenant at startup.
Watch out: —
Seen in Mesen2: The HD pack constructor now calls `CleanupInvalidRules()`. It removes, with `remove_if`, the `<addition>` sprite rules whose `IsChrRamTile` doesn't match whether the loaded cartridge uses CHR RAM, so per-frame sprite matching never sees them. 45846435 2024-01 “NES: HD Packs - Remove invalid <addition> rules after loading the pack”

### mesen2-148 — Keep synchronous console logging off by default
Lens: io · Source: Mesen2
Principle: Writing to stdout with `std::endl` flushes on every line and can block when the terminal is slow. Keep logs in memory and mirror them to the console only when explicitly asked.
Why it is faster: —
Applies elsewhere: log-level gates in production services, buffered log appenders, and turning off verbose serial logging in firmware.
Watch out: —
Seen in Mesen2: `MessageManager::Log` always ran `std::cout << message << std::endl`. It now does so only `if(_outputToStdout)`, which is set by a new `--enableStdout` option. Messages about ignored Linux input devices were also put behind a `logInformation` check. a17bb92d 2025-06 “General: Disable stdout output by default...”

### mesen2-149 — Precompute a polyphase filter-coefficient table
Lens: lookup-tables · Source: Mesen2
Principle: Windowed-sinc interpolation is expensive to compute per sample. Quantize the fractional position, say to 256 phases, and precompute a fixed-point coefficient set for each phase. Each output sample then costs a table index plus a short integer dot product.
Why it is faster: —
Applies elsewhere: audio resamplers, image scaling kernels such as Lanczos tables, and software-defined-radio decimators.
Watch out: —
Seen in Mesen2: A new SNES DSP `Sinc` interpolation mode uses `static constexpr int16_t _sincTable[2048]`, which holds 256 phases of 8 taps each, generated offline with a Blackman window and normalized to 16384. A sample is computed as `Clamp16((Σ _sincTable[offset+k] * samples[(pos+k) % 12]) >> 14)`. 21f1f624 2024-12 “SNES: Added sinc audio interpolation option”

### mesen2-150 — Make instrumentation hooks cost one null check when disabled
Lens: branching-fast-paths · Source: Mesen2
Principle: Put tracing and debug hooks in force-inline wrappers that test a single pointer or flag before doing anything. With the feature off, the hot path pays one predictable branch. With it on, compile-time template parameters route each call to type-specific code with no virtual dispatch.
Why it is faster: —
Applies elsewhere: `if(unlikely(tracepoint_enabled))` in kernels, OpenTelemetry no-op tracers, and compile-time-typed profiler macros.
Watch out: —
Seen in Mesen2: `Emulator` wraps every hook as `template<CpuType type, ...> __forceinline void ProcessMemoryRead(...) { if(_debugger) _debugger->ProcessMemoryRead<type, accessWidth, flags>(...); }`. Inside the debugger, a `switch(type)` on the template parameter resolves at compile time to the right console debugger. `_debugger` is a `safe_ptr`, which keeps a raw `_ptr` for the check and a locked `shared_ptr` for lifetime. Cheats use the same pattern with `HasCheats<CpuType>()`. current code: Core/Shared/Emulator.h, Core/Debugger/Debugger.cpp, Utilities/safe_ptr.h

### mesen2-151 — Index a table of template specializations by state bits
Lens: compiler-build · Source: Mesen2
Principle: If a routine has several independent boolean modes, instantiate every combination as a template and store pointers to them in an array indexed by the packed mode bits. At run time the dispatch is one indexed indirect call, and each variant runs with its feature checks compiled out.
Why it is faster: —
Applies elsewhere: shader permutation tables, blitters specialized on format and blend mode, and a query engine choosing a precompiled operator variant.
Watch out: —
Seen in Mesen2: The GBA PPU fills `_colorMathFunc[128]` using `StaticFor<0,128>` with `ProcessColorMath<effect, bg0..bg3Enabled, windowEnabled>` and calls `(this->*_colorMathFunc[(effect << 5) | (windowEnabled << 4) | activeLayers])()`. The GBA APU does the same with `_runFunc[16]` over which of the four channels are enabled. `Utilities/StaticFor.h` is a compile-time loop built on `if constexpr` and `std::integral_constant`. current code: Core/GBA/GbaPpu.cpp, Core/GBA/APU/GbaApu.cpp, Utilities/StaticFor.h

### mesen2-152 — Render in batches, catching up only when state changes
Lens: laziness · Source: Mesen2
Principle: Don't produce output one step at a time. Remember how far you have rendered, and render everything up to "now" only when an input that affects the output is about to change, or at a fixed sync point. Many steps then collapse into one tight loop, and the result is still exact.
Why it is faster: —
Applies elsewhere: batching draw calls until the GPU state changes, flushing a text layout only when a style changes, and computing spreadsheet ranges when a dependency is edited.
Watch out: —
Seen in Mesen2: `GbaPpu::WriteRegister` calls `RenderScanline(true)` before applying a write during active display. `RenderScanline` renders from `_lastRenderCycle` up to the current cycle. Otherwise a whole line is rendered at cycle 1056. `SnesPpu::RenderScanline` keeps `_drawStartX`/`_drawEndX`, `_fetchBgStart`/`_fetchBgEnd` and `_spriteEvalStart`/`_spriteEvalEnd` so that drawing, tile fetching and sprite evaluation each catch up from where they last stopped. current code: Core/GBA/GbaPpu.cpp, Core/SNES/SnesPpu.cpp

### mesen2-153 — Skip rendering frames that would never be displayed
Lens: laziness · Source: Mesen2
Principle: When the simulation runs faster than the display, as in fast-forward, rendering every frame wastes time on pictures that will be overwritten before anyone sees them. Keep simulating, but compose a frame only if enough wall-clock time has passed since the last one shown. Always skip rendering for speculative frames.
Why it is faster: —
Applies elsewhere: games that decouple tick rate from render rate, video players dropping late frames, and dashboards that throttle redraws to the refresh rate.
Watch out: —
Seen in Mesen2: At the start of each frame the GBA, SNES and PCE PPUs set `_skipRender` if frame skipping is enabled, the emulator is not rewinding or recording, the speed is unlimited or above 150%, and `_frameSkipTimer.GetElapsedMS() < 15` (10 ms on SNES). On SNES, run-ahead frames always set `_skipRender = true`, and `VideoDecoder::UpdateFrame` returns early for them. Output buffers are only swapped when a frame is rendered. current code: Core/GBA/GbaPpu.cpp, Core/SNES/SnesPpu.cpp, Core/PCE/PceVpc.cpp

### mesen2-154 — Keep one "next event" deadline and compare against it
Lens: branching-fast-paths · Source: Mesen2
Principle: Don't check many timers on every tick. Keep only the timestamp of the soonest pending event. The per-tick cost is one comparison, and the event handler sets the next deadline.
Why it is faster: —
Applies elsewhere: timer wheels and min-heaps in event loops, tickless kernels that program the next interrupt, and discrete-event simulators.
Watch out: —
Seen in Mesen2: `SnesMemoryManager::Exec()` runs every two master clocks and does `if(_hClock == _nextEventClock) ProcessEvent();`. `ProcessEvent` handles DRAM refresh, HDMA init and similar events, then sets `_nextEvent` and `_nextEventClock` to whatever comes next. The rest of `Exec` uses bit masks on `_hClock` to decide when to tick the PPU and IRQ counters. current code: Core/SNES/SnesMemoryManager.cpp

### mesen2-155 — Map address pages with a pointer table
Lens: memory-layout · Source: Mesen2
Principle: Split the address space into fixed-size pages and keep an array of pointers to the backing memory of each page, or to a handler for it. A memory access becomes a shift, an index and a load. Bank switching rewrites a few table entries instead of adding conditions to every access.
Why it is faster: —
Applies elsewhere: MMU page tables, the slot-to-buffer mapping in a paged file cache, and segment-indexed lookups in a virtual-memory allocator.
Watch out: —
Seen in Mesen2: The Game Boy keeps `uint8_t* _reads[0x100]` and `_writes[0x100]` and reads with `_reads[addr >> 8][(uint8_t)addr]`, after checking `IsReadRegister[addr >> 8]`. NES mappers keep `_prgPages`/`_chrPages` plus access-type arrays at 256-byte granularity. SNES uses `_handlers[addr >> 12]`, and the NES memory manager keeps a handler pointer for each of its 64K addresses. The PCE uses `_readBanks[bank][addr & _bankMask[bank]]`. current code: Core/Gameboy/GbMemoryManager.cpp, Core/NES/BaseMapper.h, Core/SNES/MemoryMappings.cpp, Core/PCE/PceMemoryManager.cpp

### mesen2-156 — Precompute per-region access costs into a small table
Lens: lookup-tables · Source: Mesen2
Principle: If a cost depends on a handful of address bits plus a mode, compute it for every combination once, and again only when a configuration register changes. The hot path then indexes the table with a shift and a mask instead of walking a switch.
Why it is faster: —
Applies elsewhere: NUMA-aware cost tables, shipping-rate lookups by zone and weight class, and precomputed tax brackets.
Watch out: —
Seen in Mesen2: `GbaWaitStates::GenerateWaitStateLut` fills 256 regions × 4 access modes, and it is called again when WAITCNT is written or a save state is loaded. A lookup is `_waitStatesLut[((addr >> 22) & 0x3FC) | (mode & (Word | Sequential))]`. The SNES builds `_masterClockTable[0x800]` once in `GenerateMasterClockTable()`, holding the slow or fast access speed by bank and page. current code: Core/GBA/GbaWaitStates.h, Core/SNES/SnesMemoryManager.cpp

### mesen2-157 — Use static polymorphism (CRTP) for hot customization points
Lens: compiler-build · Source: Mesen2
Principle: When a base algorithm needs per-variant hooks that are called millions of times a second, template the base on the derived class and call the hooks through a cast to it. The compiler can inline them, and empty hooks vanish completely.
Why it is faster: —
Applies elsewhere: Eigen expression templates, policy-based allocators, and CRTP visitors in compilers.
Watch out: —
Seen in Mesen2: `template<class T> class NesPpu` calls `((T*)this)->DrawPixel()`, `StoreTileInformation()` and `ProcessScanline()`. `DefaultNesPpu final : NesPpu<DefaultNesPpu>` defines the HD-pack hooks as empty `__forceinline` bodies and implements `DrawPixel` inline, with the comment "called 3.7 million times per second". `HdNesPpu` supplies real versions of the same hooks. current code: Core/NES/NesPpu.h, Core/NES/DefaultNesPpu.h, Core/NES/HdPacks/HdNesPpu.h

### mesen2-158 — Pick a hardware-configuration-specialized step function at load
Lens: compiler-build · Source: Mesen2
Principle: If a configuration is fixed for a whole session, such as which add-on hardware is present, compile the per-tick function once for each configuration and choose one through a member function pointer. The per-tick code never re-tests the configuration.
Why it is faster: —
Applies elsewhere: choosing a GPU backend's draw path at init, codec paths specialized by profile, and database scan loops specialized per column type.
Watch out: —
Seen in Mesen2: `PceMemoryManager` sets `_exec` to one of `ExecTemplate<hasCdRom, isSuperGrafx>`. It falls back to `ExecSlow` when the CPU is in slow mode, and it keeps `_fastExec` for VDC stalls. The hot entry point is `__forceinline void Exec() { (this->*_exec)(); }`. current code: Core/PCE/PceMemoryManager.h, Core/PCE/PceMemoryManager.cpp

### mesen2-159 — Run a subsystem only when a deadline could be crossed
Lens: laziness · Source: Mesen2
Principle: For a component whose output matters only at certain moments, keep a cheap per-tick check of whether any of those moments falls inside the time not yet run. When none does, the clock just advances. When one does, the component catches up in one batch.
Why it is faster: —
Applies elsewhere: lazy timer evaluation in GUI toolkits, and battery-saving sensors that wake only when a threshold could be reached.
Watch out: —
Seen in Mesen2: `NesApu::Exec()` increments `_currentCycle` and calls `Run()` only if `NeedToRun()` returns true. That happens when DMC is active, when a register write set `_needToRun`, or when `_frameCounter->NeedToRun(cyclesToRun)` or `_dmc->IrqPending(cyclesToRun)` says an event falls inside the pending window. `Run()` then advances every channel by `_currentCycle - _previousCycle` at once. current code: Core/NES/APU/NesApu.cpp

### mesen2-160 — Mix audio only at timestamps where the output changes
Lens: algorithms · Source: Mesen2
Principle: Channels typically change output at sparse moments. Record (time, delta) pairs, then at frame end sort and deduplicate the timestamps and recompute the mix only at those points. A band-limited step synthesizer turns the steps into clean samples, so the cost scales with the number of changes, not with the clock rate.
Why it is faster: —
Applies elsewhere: event-driven logic simulation, sparse time-series aggregation, and MIDI synthesizers that render only between note events.
Watch out: —
Seen in Mesen2: `NesSoundMixer::AddDelta` stores `_channelOutput[channel][time] += delta` and pushes `time`, but only if the delta is non-zero. `EndFrame` sorts the timestamps and removes duplicates with `unique`, then accumulates the channels and calls `blip_add_delta(_blipBufLeft, stamp, current - previous)` for each one. The right channel is mixed only when panning is in use. The Game Boy, PCE and SMS sound code also uses `blip_add_delta`. current code: Core/NES/NesSoundMixer.cpp, Core/Gameboy/APU/GbApu.cpp, Core/PCE/PcePsg.cpp

### mesen2-161 — Store snapshots as XOR deltas against periodic keyframes
Lens: memory-layout · Source: Mesen2
Principle: Consecutive snapshots are mostly identical. XOR each one against the last full keyframe, which turns unchanged bytes into zeros, and then compress it, since zeros compress very well. Keep the current keyframe uncompressed while it is in use, so creating each delta doesn't require decompressing the keyframe first.
Why it is faster: —
Applies elsewhere: video I-frames and P-frames, incremental VM checkpoints, and game replay or undo systems.
Watch out: —
Seen in Mesen2: `RewindData::SaveState` stores a full state every 30 entries (`position % 30`). In between, `ProcessXorState` XORs the state against the last full state and calls `CompressionHelper::Compress(data, 1, ...)` at the fastest level. The full state keeps `_uncompressedData` for the next 30 deltas, as the source comment explains: "this avoids having to decompress the state 30 times". When the next keyframe arrives, the previous one's uncompressed copy is released. current code: Core/Shared/RewindData.cpp

### mesen2-162 — Pipeline frame production across threads with a one-slot handoff
Lens: concurrency · Source: Mesen2
Principle: Run the expensive post-processing of frame N, such as filtering and scaling, on its own thread while the producer computes frame N+1. A single slot with a "busy" flag is enough. If the consumer is still busy, the producer briefly spins rather than paying the latency of a sleep and wake-up.
Why it is faster: —
Applies elsewhere: game engines with separate simulation and render threads, camera capture to encode pipelines, and double-buffered audio callbacks.
Watch out: —
Seen in Mesen2: `VideoDecoder::UpdateFrame` spins `while(_frameChanged)` if the previous decode isn't finished. The source comment explains why: "sometimes Signal() introduces a 25-30ms delay". It then stores the frame, sets `_frameChanged = true` and signals `_decodeThread`. A separate `VideoRenderer` render thread presents the frames, and when nothing arrives it wakes every 32 ms to keep the HUD updated. current code: Core/Shared/Video/VideoDecoder.cpp, Core/Shared/Video/VideoRenderer.cpp

### mesen2-163 — Record raw state in a ring buffer and format it only when viewed
Lens: laziness · Source: Mesen2
Principle: Logging every event as formatted text is slow. Copy the raw structs into a fixed-size circular buffer and turn them into text only for the rows a viewer actually requests. Do the eager formatting only when the log is being streamed to a file.
Why it is faster: —
Applies elsewhere: flight recorders and in-memory trace buffers such as ftrace, and structured logging that defers message templating.
Watch out: —
Seen in Mesen2: `BaseTraceLogger::AddRow` copies `DisassemblyInfo`, the CPU state and the PPU state into arrays of `ExecutionLogSize = 30000` entries and advances `_currentPos = (_currentPos + 1) % ExecutionLogSize`. It builds a text row, using `row.reserve(300)`, only `if(_debugger->GetTraceLogFileSaver()->IsEnabled())`. current code: Core/Debugger/BaseTraceLogger.h

### mesen2-164 — Pre-filter conditional checks by category, and precompile the conditions
Lens: branching-fast-paths · Source: Mesen2
Principle: Keep one flag per operation type that says whether any rule could apply, and bail out right away when it is false. Compile rule conditions once into a postfix (RPN) program, so evaluating one doesn't require parsing.
Why it is faster: —
Applies elsewhere: WAF rules indexed by HTTP method, alerting rules precompiled into bytecode, and event filters with per-type subscriber flags.
Watch out: —
Seen in Mesen2: `BreakpointManager::CheckBreakpoint` is `__forceinline` and starts with `if(!_hasBreakpointType[(int)operationInfo.Type]) return -1;`. Breakpoint conditions are turned into `ExpressionData` RPN lists with `GetRpnList` when breakpoints are set, and `ExpressionEvaluator` keeps an `unordered_map<string, ExpressionData> _cache` of parsed expressions. current code: Core/Debugger/BreakpointManager.h, Core/Debugger/BreakpointManager.cpp, Core/Debugger/ExpressionEvaluator.h

### mesen2-165 — Spin first, then back off to sleeping
Lens: concurrency · Source: Mesen2
Principle: Locks that are held only briefly are cheapest to acquire by spinning on an atomic flag, which avoids a kernel transition. Cap the spinning: after many failed attempts, sleep between retries so a long hold doesn't burn a core. Allowing reentrancy from the owning thread prevents self-deadlock.
Why it is faster: —
Applies elsewhere: adaptive mutexes such as glibc's `PTHREAD_MUTEX_ADAPTIVE_NP`, and spin-then-park in Java and Rust parking_lot.
Watch out: —
Seen in Mesen2: `SimpleLock::WaitForAcquire` loops on `_lock.test_and_set()`. After `++attempts >= 10000` it calls `sleep_for` for 1 ms, or 10 ms when a timeout was given, between tries. `Acquire` just increments `_lockCount` when `_holderThreadID == _threadID`. current code: Utilities/SimpleLock.cpp

### mesen2-166 — Replace number-to-text conversion with a 256-entry string table
Lens: lookup-tables · Source: Mesen2
Principle: Formatting bytes as hex in a hot loop with general-purpose formatting is slow. A table of the 256 two-character strings turns each byte into one index plus a short copy. Build output in a fixed stack buffer instead of a growing heap string.
Why it is faster: —
Applies elsewhere: hex dump tools, UUID formatting, and fast integer-to-string routines that use two-digit tables.
Watch out: —
Seen in Mesen2: `HexUtilities` holds `_hexCache`, a `vector<string>` of "00" through "FF", and `constexpr const char* _hexCharCache[256]`. `FastString` writes into a `char _buffer[1000]` with `memcpy`, and is used by the disassembler and trace text output. current code: Utilities/HexUtilities.cpp, Utilities/FastString.h

### mesen2-167 — Precompute a full color-space conversion table
Lens: lookup-tables · Source: Mesen2
Principle: If a filter compares colors in a perceptual space for every pixel and every neighbour, convert each possible RGB value once at startup. A 24-bit input domain fits in a 64 MB table, and each conversion becomes a single load.
Why it is faster: —
Applies elsewhere: gamma or sRGB lookup tables in image codecs, palette quantizers, and 3D LUTs for color grading.
Watch out: —
Seen in Mesen2: The HQX scaler's initialization fills `uint32_t RGBtoYUV[16777216]` with `(y << 16) + (u << 8) + v` for every 24-bit color. The hq2x, hq3x and hq4x kernels then look up neighbour colors in it when they compare them. current code: Utilities/HQX/init.cpp

### mesen2-168 — Upscale by building one row, then copying it
Lens: memory-layout · Source: Mesen2
Principle: For nearest-neighbour upscaling, each output row is repeated `scale` times. Build the first expanded row pixel by pixel, then fill the remaining copies with bulk `memcpy` from that row, which is much faster than writing every pixel again.
Why it is faster: —
Applies elsewhere: pixel-art scaling in games, thumbnail generation, and LED-matrix framebuffer doubling.
Watch out: —
Seen in Mesen2: `ScaleFilter::ApplyPrescaleFilter` writes each source pixel `_filterScale` times, then runs `for(i = 1; i < _filterScale; i++) memcpy(outputBuffer, outputBuffer - _width*_filterScale, _width*_filterScale*4);`. `UpdateOutputBuffer` reallocates only when the dimensions change. current code: Core/Shared/Video/ScaleFilter.cpp
