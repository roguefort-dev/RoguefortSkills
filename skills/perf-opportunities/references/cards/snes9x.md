# Snes9x — technique cards (165)

Each card is exactly 7 lines. Fetch one with: grep -A 6 "^### snes9x-NNN " snes9x.md

### snes9x-001 — Catch up a subordinate component only when someone observes it
Lens: laziness · Source: Snes9x
Principle: Don't advance a dependent simulation in lockstep with its driver. Add the elapsed time to a debt counter and settle the debt in one batch when its state is read or written, or at a periodic boundary. Many tiny calls become a few large ones, and the inner loop stays tight and hot in cache.
Why it is faster: —
Applies elsewhere: lazy physics or animation updates that run only when an object is queried or rendered. Deferred layout in UI toolkits, settled on read of `offsetHeight`. Accumulated sensor integration on a microcontroller, settled when the host polls.
Watch out: —
Seen in Snes9x: With byuu's SMP and blargg's DSP combined, every SMP tick called `dsp.spc_dsp.run(1)` through a `DSP::enter`/`synchronize_smp` wrapper. The first step removed that wrapper indirection and called `spc_dsp.run(cycles)` directly per opcode. The second step only increments `dsp.clock`. A new `DSP::synchronize()` (`if (clock) { spc_dsp.run(clock); clock = 0; }`) runs before every DSP register `read`/`write` and once in `S9xAPUEndScanline`. 08f5601c 2011-06 “Reduce the sync footprint slightly with less indirection.” · 7c468e51 2011-06 “Remove parts of DSP wrapper we don't need.” · 91e69af0 2011-06 “Loosen SMP<->DSP synchronization.”

### snes9x-002 — Account time per operation from a table, not per sub-step
Lens: algorithms · Source: Snes9x
Principle: If nothing outside can observe the intermediate steps of an operation, run the whole operation at once and charge its cost from a precomputed table. You save one dispatch and bookkeeping round per sub-step. Keep the fine-grained version behind a compile-time switch for cases that need it.
Why it is faster: —
Applies elsewhere: instruction-set simulators that charge per-instruction latency instead of modelling pipeline stages. Network simulators that charge per-packet cost instead of per-byte. Gas metering per opcode in a VM.
Watch out: —
Seen in Snes9x: When it imported byuu's cycle-accurate SMP, Snes9x turned `CYCLE_ACCURATE` off for its own build (`#ifndef SNES9X #define CYCLE_ACCURATE`). `SMP::op_step` now decodes and runs a whole opcode, then does `clock += cycle_count_table[opcode]` and ticks the timers once by that amount. 1af39f38 2011-06 “Clean things up slightly. Switch CYCLE_ACCURATE off, since things seem to work fine without it.”

### snes9x-003 — Collapse a resumable per-step state machine into multi-step chunks
Lens: algorithms · Source: Snes9x
Principle: A state machine that returns after every tiny step pays a switch dispatch and a state save/restore each time. Group consecutive steps that need no interleaving into one case, and advance time by `n` in one call (`tick(n)`). Fewer re-entries and fewer branches do the same work.
Why it is faster: —
Applies elsewhere: coroutine or generator parsers that yield only at real I/O points, not per byte. Protocol handlers that process a whole header in one state. Cooperative schedulers that batch micro-tasks.
Watch out: —
Seen in Snes9x: The per-cycle SMP (`opcycle_*.cpp`) had each opcode as `switch(opcode_cycle++)` with one `op_io()` per case. The “pseudo-cycle” rewrite merged cases (for example, `0x9f` became `op_io(4)` plus the operation), added `SMP::tick(unsigned clocks)`/`op_io(unsigned)`, paired the opcode fetch with the first cycle, and dropped the `opcode_cycle = 0` resets. The result shipped as a third core selectable with `PSEUDO_CYCLE`, alongside the cycle and opcode cores. a19395ee 2011-09 “Start converting some opcodes to cycle-based.” · b97e56d6 2011-09 “Group some MOV opcode cycles together.” · 70159fd9 2011-09 “Pair opcode read with first instruction cycle. Remove opcode_cycle switching from unneeded cases.” · 13c55b92 2011-10 “Allow switching between 3 SMP cores at compile-time.”

### snes9x-004 — Coarsen the interleave between cooperating workers
Lens: algorithms · Source: Snes9x
Principle: When two loops take turns, each switch costs loop setup, condition checks and lost cache locality. Give the secondary worker a bigger slice per turn, as long as correctness tolerates the extra skew. You get fewer switches for the same total work.
Why it is faster: —
Applies elsewhere: raising the batch size in a producer/consumer pipeline. Larger quanta in a green-thread scheduler. Processing N events per poll in an event loop instead of one.
Watch out: —
Seen in Snes9x: `S9xSA1MainLoop` now runs up to 5 SA-1 opcodes per call instead of 3: `for (int i = 0; i < 5 && !(Memory.FillRAM[0x2200] & 0x60); i++)`. 162ccbb1 2017-11 “Increase the number of opcodes executed on the SA1 to 5 per S-CPU opcode.”

### snes9x-005 — Make tiny hot functions inlinable
Lens: compiler-build · Source: Snes9x · Also: ares
Principle: A call to a tiny function costs more than its body: argument setup, a jump and return, spilled registers, and no optimization across the call boundary. Mark per-instruction helpers `inline` or force-inline them so the compiler can fold them into the dispatch loop. This matters most in an interpreter's inner loop, where speed problems appeared as soon as the helpers stopped inlining.
Why it is faster: —
Applies elsewhere: getters and arithmetic helpers in game-engine math libraries. Byte-code VMs (`static inline` opcode helpers). Codec bit-readers.
Watch out: —
Seen in Snes9x: `SMP::tick()` and `Timer::tick()` were declared out-of-line, and one commit marks them `inline` “to fix speed issues”. Another marks the ALU helpers `op_adc`, `op_and`, `op_cmp`, `op_asl` and others `inline`. Later the DSP voice-pipeline stages `VOICE_CLOCK(V3c)` and `VOICE_CLOCK(V4)` in `SPC_DSP.cpp` were also made `inline`. 2b5a54b4 2011-07 “Inline ticks. This should fix speed issues.” · 804d419b 2012-01 “Inline some functions to avoid vtable lookups.” · 97f4067c 2018-04 “Inline a couple functions. ...” • Also seen in ares (ares-146): SH-2 `MOVWI`, `MOVLI` and `MOVA` each consist of a PC computation and one read, and all three had `NALL_NOINLINE`. The commit removed it. ed1e34c74 2025-04 “remove errant NALL_NOINLINE macro from SH2 instructions”

### snes9x-006 — Use non-short-circuit boolean operators on cheap conditions
Lens: branching-fast-paths · Source: Snes9x · Also: zstd
Principle: `&&` and `||` require a conditional jump per operand. When every operand is a cheap, side-effect-free comparison, `&` and `|` compute them all and combine the results without branching. That removes mispredictions on data-dependent conditions in hot paths.
Why it is faster: —
Applies elsewhere: bounds checks in image kernels (`(x>=0) & (x<w)`). Collision tests in game loops. Packet-filter predicates.
Watch out: —
Seen in Snes9x: `S9xCheckInterrupts` in `cpuexec.h` runs after every CPU instruction. Its compound tests were rewritten from `&&`/`||` to `&`/`|`, for example `if (CPU.PrevCycles >= htimepos | CPU.Cycles < htimepos)`. The commit reports an 11% speedup on x64. 2cbb3f6c 2018-03 “11% performance boost on x64 by reducing branches” • Also seen in zstd (zstd-082): a419777eb1 2017-03-01 `HUF_validateCTable` accumulates `bad |= (count[s] != 0) & (CTable[s].nbBits == 0)` over all symbols. Validation takes about 55 ns, down from 90 ns. • 3f457264d1 2017-11-19 Replaces `&&` with `&` where both sides are cheap, for example in the lazy catch-up loop `((start > anchor) & (start - … > base))` and the repcode loop `((ip <= ilimit) & (offset_2>0))`. • a101721f4e 2019-09-14 One `strstr` over a concatenated suffix list replaces up to five `strcmp` calls. • 718f00ff6f 2019-11-25 The four interleaved Huffman streams combine their reload statuses with `&` into one loop condition. cb2abc3dbe folded them into a single `LIKELY(a & b & c & d)`. • 68c14bdff2 2020-12-04 `*hasZeroWeights = (rankVal[0] > 0)` replaces `|= (w == 0)` in the fill loop: about 1-2% faster. • 5e9a6c2fe4 2024-05-29 `ZSTD_index_overlap_check` returns `(U32)((prefixLowestIndex-1) - repIndex) >= 3`, one unsigned compare in place of two compares and two branches.

### snes9x-007 — Rebuild derived data at the writer, not with a check in the reader
Lens: laziness · Source: Snes9x
Principle: A "needs rebuild" flag checked inside a hot loop costs a test on every iteration, but the input changes only rarely. Rebuild eagerly at the few places that change the input, and the hot path is just a lookup.
Why it is faster: —
Applies elsewhere: recompute a cached view-projection matrix in the camera setter, not in every draw call. Rebuild a routing table when config changes, not on each request.
Watch out: —
Seen in Snes9x: The tile renderer's `SELECT_PALETTE()` and Mode 7 macros tested `if (IPPU.DirectColourMapsNeedRebuild) S9xBuildDirectColourMaps();` per tile. That flag was removed. `S9xBuildDirectColourMaps()` now runs directly when brightness changes in `S9xSetPPU`, and on graphics init, PPU reset and state load. 5285ec4e 2018-04 “Move color table generation out of tile renderer.”

### snes9x-008 — Track a dirty flag so an expensive rebuild runs only after a real change
Lens: laziness · Source: Snes9x
Principle: If a commit point triggers a costly recomputation, record whether any input actually changed since the last commit. Skip the rebuild when nothing did.
Why it is faster: —
Applies elsewhere: UI frameworks that skip re-render when props are equal. Build systems that skip unchanged targets. GPU state caches that skip redundant uniform uploads.
Watch out: —
Seen in Snes9x: In `S9xSetBSX`, every write to MMC register 0x0E called `BSX_Map()`, which rebuilds the memory map. Writes to registers 0x01–0x0D now set `BSX.dirty` only when the value differs, and 0x0E remaps only `if (BSX.dirty)`. 54d7fe40 2017-12 “[BS-X] Avoid updating the memory map when it is not needed”

### snes9x-009 — Clear large tables row by row, on first use
Lens: memory-layout · Source: Snes9x
Principle: Zeroing a big 2-D scratch table every pass touches memory that is mostly never used. Keep a small "row touched" bitmap. Clear a row only the first time it is written, and skip untouched rows when you read the table back.
Why it is faster: —
Applies elsewhere: sparse histogram or bucket tables in image processing. Hash tables reset with generation counters. Per-scanline span buffers in software rasterizers.
Watch out: —
Seen in Snes9x: `SetupOBJ` in `gfx.cpp` did `memset(OBJOnLine, 0, sizeof(OBJOnLine))` on a `[SNES_HEIGHT_EXTENDED][128]` array each frame. It now clears only `bool8 AnyOBJOnLine[]`, zeroes `OBJOnLine[Y]` when a sprite first lands on line Y, and skips the per-line 128-sprite scan when `!AnyOBJOnLine[Y]`. c64bfbf6 2013-04 “gfx.cpp: memset less memory at once for FirstSprite+Y, accelerating sprite rendering if not all lines have sprites.”

### snes9x-010 — Put the cheapest rejection test first
Lens: branching-fast-paths · Source: Snes9x
Principle: If an early-exit condition can be decided from raw inputs, test it before computing values that only the non-exit path needs. Rejected calls then do almost no work.
Why it is faster: —
Applies elsewhere: reject fully transparent pixels before blending. Check cache headers before parsing a body. Do an AABB check before a precise collision test.
Watch out: —
Seen in Snes9x: `fx_plot_2bit`/`fx_plot_4bit` (Super FX) computed the dithered colour `c` and then returned early if it was transparent. The transparency test now runs first on `COLR`: `if (!(GSU.vPlotOptionReg & 0x01) && !(COLR & 0xf)) return;`. bd9db572 2018-05 “Test for transparency before dithering.”

### snes9x-011 — Fix a runtime-selectable format at compile time
Lens: compiler-build · Source: Snes9x · Also: ares
Principle: Supporting several data formats through runtime dispatch puts an indirection or branch into every element operation. When a build only ever uses one format, make it a compile-time constant. The macros then reduce to straight shifts and masks the compiler can optimize.
Why it is faster: —
Applies elsewhere: template-specialized pixel or audio-sample formats in media libraries. Feature flags compiled out of embedded firmware. Monomorphized generics instead of trait objects.
Watch out: —
Seen in Snes9x: `GFX_MULTI_FORMAT` was defined unconditionally in `port.h`, so `BUILD_PIXEL` went through runtime-selected builders. It is now defined only by the unix port. `PIXEL_FORMAT` defaults to `RGB565`, GTK passes `-DPIXEL_FORMAT=RGB555`, and libretro requires RGB565 instead of switching at runtime. The commit says the `BuildPixel` calls cost about 3%. 51392652 2018-04 “Don't set GFX_MULTI_FORMAT if not needed. The BuildPixel calls cost ~3% performance.” • Also seen in ares (ares-155): `Accuracy::RSP::SIMD = !SISD`, with `SISD = Reference | !ARCHITECTURE_SUPPORTS_SSE4_1`. Each vector op contains both an `if constexpr(Accuracy::RSP::SISD)` loop and an SSE block. The CPU/RSP `Recompiler` flags and `JitInterleaving` work the same way. current code: ares/n64/accuracy.hpp, ares/n64/rsp/interpreter-vpu.cpp

### snes9x-012 — Produce data in the format the consumer accepts natively
Lens: platform · Source: Snes9x
Principle: Every format mismatch between producer and consumer means a conversion pass, sometimes per pixel per frame. Pick the internal format the downstream API or hardware takes directly, so the data can be uploaded or handed over unchanged.
Why it is faster: —
Applies elsewhere: decode video straight to the texture format the GPU samples. Use RGB565 framebuffers on embedded displays. Choose the audio sample format the device opens natively.
Watch out: —
Seen in Snes9x: The libretro core asks the frontend for `RETRO_PIXEL_FORMAT_RGB565`, the format that OpenGL ES and low-power devices support natively, and renders directly in it. GTK's renderer switched from 15-bit to 16-bit pixels. Its OpenGL driver now uploads with `GL_RGB, GL_UNSIGNED_SHORT_5_6_5` instead of `GL_BGRA, GL_UNSIGNED_SHORT_1_5_5_5_REV`, and the converters and filters (xBRZ, blend masks) were adjusted to 565. d7e3fa64 2012-10 “libretro: use RGB565 if available” · dcfb7c3a 2018-05 “Use 16-bit instead of 15-bit pixel format on Gtk+. ...”

### snes9x-013 — Match rates exactly to skip the resampling stage
Lens: algorithms · Source: Snes9x
Principle: If a producer runs at a rate the consumer can accept directly, report that exact rate so the interpolation stage disappears. Use one canonical constant everywhere, or nearly equal values will still force a ratio that isn't 1.
Why it is faster: —
Applies elsewhere: open audio devices at the decoder's native sample rate. Render at the display's native resolution to avoid scaling. Choose a video frame rate equal to the refresh rate.
Watch out: —
Seen in Snes9x: libretro advertised `sample_rate = 32040.5`. It now reports `32040` so the frontend doesn't need to resample. Elsewhere `apu.cpp` and `msu1.cpp` replaced mixed 32000/32040.5 constants with 32040 for buffer sizing and MSU-1 ratio maths. cf846e0c 2016-12 “Use 32040hz with more consistency.” · bbeb13c2 2016-12 “Use 32040hz sample rate in libretro to avoid resampling.”

### snes9x-014 — Shape per-sample DSP code so the compiler can vectorize it
Lens: simd · Source: Snes9x
Principle: Use the narrowest float type that meets quality needs, fold constant parameters out of the formula, and hoist loop-invariant loads. Compute independent channels into a small array before clamping. The compiler then sees two identical, independent operations it can pack into one SIMD instruction.
Why it is faster: —
Applies elsewhere: stereo or RGB processing in audio plugins and image filters. Struct-of-arrays layouts in particle systems. Any interpolation kernel in a hot loop.
Watch out: —
Seen in Snes9x: `HermiteResampler` switched from `double` to `float`. `hermite()` became `static inline`, and the `tension`/`bias` terms reduced to `m0 = (c - a) * 0.5`. A per-sample `fabs(r_step - 1.0)` branch was dropped, and `max_samples` moved out of the loop. Later the left and right results go into `float hermite_val[2]` before `SHORT_CLAMP`, “so compiler auto-SIMDs”. 2337ab3e 2011-12 “Optimize hermite resampler slightly. ...” · f6c9ea8d 2012-01 “Don't call hermite twice per sample for clamping. Adjust so compiler auto-SIMDs.”

### snes9x-015 — Check output quality before replacing floating point with fixed point
Lens: other · Source: Snes9x
Principle: Fixed point can be faster on FPU-poor hardware, but each multiply-and-shift loses precision. Cubic terms (mu² and mu³) compound the loss. Measure output quality (SNR) before keeping the change. If it falls short, a wider fixed-point format or float is the right choice.
Why it is faster: —
Applies elsewhere: audio codecs on microcontrollers, quantized neural-network inference, fixed-point physics in deterministic games.
Watch out: —
Seen in Snes9x: The Hermite resampler was converted to Q15 integer maths (`mu2 = (mu1 * mu1) >> 15` and so on) for slower-FPU systems. It was reverted, re-applied with signed arithmetic, and finally reverted to `float` because 16-bit fixed point “doesn't have enough accuracy for a decent SNR with hermite”. 2411daa2 2012-01 “Change hermite resampler to use fixed-point arithmetic for speed on slower FPU systems.” · 53defb5b 2012-04 “Go back to floating point hermite resampler. ...”

### snes9x-016 — Pace a producer by nudging its rate, not by blocking
Lens: concurrency · Source: Snes9x
Principle: Blocking until a buffer drains ties throughput to the consumer's jitter. Instead, measure the buffer fill level and scale the production rate by a small factor toward half-full. A simplified closed-form formula keeps the per-call cost to a few integer operations and one division.
Why it is faster: —
Applies elsewhere: adaptive bitrate streaming, network jitter buffers, PID-style flow control between threads, video players syncing audio to the display clock.
Watch out: —
Seen in Snes9x: `S9xUpdateDynamicRate(avail, buffer_size)` sets `dynamic_rate_multiplier`, which `UpdatePlaybackRate` applies to the resampler's `time_ratio`. The ALSA and OSS drivers call it with their fill level. A follow-up collapsed the half-size, delta and direction steps into one expression: `1.0 + (DynamicRateLimit * (buffer_size - 2*avail)) / (double)(1000*buffer_size)`. 39f05664 2017-11 “Add dynamic rate control for audio, based on Retroarch/Themaister's paper ...” · 61851945 2017-11 “Optimize rate control formula and remove placeholder copy-paste.”

### snes9x-017 — Block on an event instead of spinning with zero-length sleeps
Lens: concurrency · Source: Snes9x
Principle: `Sleep(0)`/`yield` polling keeps a core at 100% while waiting. Have the producer signal an event, or at least sleep for a real interval, so the waiting thread costs nothing until there is work.
Why it is faster: —
Applies elsewhere: condition variables or semaphores instead of busy-wait flags. `epoll`/`kqueue` instead of polling sockets. RTOS task notifications in embedded firmware.
Watch out: —
Seen in Snes9x: The Win32 sound-sync wait loop around `S9xSyncSound()` first went from `Sleep(0)` to `Sleep(2)`. Later every audio backend's mix callback (DirectSound, FMOD, FMOD Ex, XAudio2's `OnBufferEnd`) called `SetEvent(GUI.SoundSyncEvent)`, so the main loop could wait on that event instead of sleeping. 6d03d238 2011-01 “Win32: use less aggressive waiting for soundsync” · dce8f6a1 2012-01 “Win32: use event for soundsync”

### snes9x-018 — Yield before a driver call that may busy-wait
Lens: concurrency · Source: Snes9x
Principle: Some blocking APIs spin internally. Yielding the timeslice and flushing pending work first shortens the spin and gives the CPU to other threads.
Why it is faster: —
Applies elsewhere: calling `sched_yield()` before a spin-lock in user space. Flushing command buffers before a GPU fence wait. Back-off before polling a hardware status register.
Watch out: —
Seen in Snes9x: With “sync every frame” enabled, GTK's OpenGL `gl_swap` now calls `usleep(0); glXWaitX();` before `glFinish()`, which some drivers implement as a busy loop. f582cb26 2010-12 “Yield before glFinish to lighten load with frame sync. Helps cases where glFinish is a busy loop.”

### snes9x-019 — Maintain aggregates incrementally instead of rescanning
Lens: memory-layout · Source: Snes9x
Principle: If a comparator or query recomputes a count by scanning the whole collection, a sort or save becomes O(n² log n). Keep a side map of counts and update it on every insert and erase, so each query is a lookup.
Why it is faster: —
Applies elsewhere: keep a running count per category in an e-commerce cart, maintain per-key counters next to a database table, or track per-bucket sizes in a custom allocator.
Watch out: —
Seen in Snes9x: `ConfigFile::GetSectionSize` iterated over every entry with `strncasecmp`. It was called from the `section_then_key_less` comparator used to sort output. A `sectionSizes` map is now incremented and decremented in `LoadFile`, `SetString`, `DeleteKey` and `DeleteSection`, and `GetSectionSize` returns from it. 851839cd 2010-09 “faster config file saving”

### snes9x-020 — Reconfigure an OS resource in place instead of recreating it
Lens: platform · Source: Snes9x
Principle: Destroying and recreating a heavyweight handle (a window, GL context binding, socket or device) costs far more than changing its properties. Use the update API when one exists.
Why it is faster: —
Applies elsewhere: resizing a swap chain instead of recreating the device, `setsockopt` instead of reconnecting, `realloc`/`ftruncate` instead of reallocating and copying.
Watch out: —
Seen in Snes9x: On resize, the GTK OpenGL and Xv drivers' `resize_window` did `XDestroyWindow`, `create_window`, then `glXMakeCurrent` and swap-control setup again. They now call `XConfigureWindow(display, xwindow, CWWidth | CWHeight, &changes)`. 52defdc1 2011-01 “Use XConfigureWindow instead of creating new ones. ...”

### snes9x-021 — Remember applied state so per-frame code can skip redundant updates
Lens: laziness · Source: Snes9x
Principle: Per-frame code that compares wanted state with "current" state only saves work if "current" is actually updated after each change. Otherwise the comparison never matches and the expensive call runs every frame.
Why it is faster: —
Applies elsewhere: GPU state caches, DOM diffing, debounced CSS-property writes, register shadow copies in device drivers.
Watch out: —
Seen in Snes9x: The GTK OpenGL and Xv drivers resized the X11 sub-window when the size differed from `output_window_width/height`, but `resize_window` never stored the new size. The window was therefore reconfigured on every draw. The fix stores `output_window_width = width; output_window_height = height;` after resizing. db6de9a6 2014-02 “Fix reconfiguring the X11 sub-window every draw.”

### snes9x-022 — Filter and debounce system notifications before doing expensive rescans
Lens: io · Source: Snes9x
Principle: Broad OS notifications arrive often and in bursts. Act only on the subtypes that matter, and coalesce bursts with a one-shot timer, so a costly enumeration runs once per real change.
Why it is faster: —
Applies elsewhere: debouncing file-watcher events before rebuilding, throttling `resize` handlers in the browser, coalescing udev hotplug events.
Watch out: —
Seen in Snes9x: Win32's `WM_DEVICECHANGE` handler called `S9xDetectJoypads()` on every message. It was first limited to `DBT_DEVICEARRIVAL`/`DBT_DEVICEREMOVECOMPLETE`. Later it arms `SetTimer(hWnd, TIMER_SCANJOYPADS, 500, NULL)` and rescans once in `WM_TIMER`. 70cf3df1 2015-02 “Win32: only check joypads on arrival/disconnect” · f7c3117f 2018-04 “win32: do hotplugging detection 500ms after arrival/removal”

### snes9x-023 — Skip a processing pass that a later stage already performs
Lens: algorithms · Source: Snes9x
Principle: In a pipeline of optional stages, two stages can do equivalent work. Have each stage declare its capabilities, and run the generic pass only when the selected stage doesn't already cover it.
Why it is faster: —
Applies elsewhere: don't gzip already-compressed assets, skip CPU-side colour conversion when the shader converts, avoid double escaping in templating layers.
Watch out: —
Seen in Snes9x: Win32 rendering gained `GetFilterBlendSupport()`. The new `RenderMethod` wrapper calls `RenderMergeHires` into `BlendBuffer` only when `GUI.BlendHiRes && Src.Width == 512 && !GetFilterBlendSupport(GUI.ScaleHiRes)`. Filters that blend by themselves (Blargg NTSC, Simple1x) skip it. 481bf064 2011-03 “Win32: rework hi-res blending”

### snes9x-024 — Downsample early, when the extra resolution isn't needed
Lens: algorithms · Source: Snes9x
Principle: A filter that writes as many pixels as it reads, carrying state from pixel to pixel, does more work than one that combines independent pairs and halves the output. The smaller result also makes every later stage cheaper.
Why it is faster: —
Applies elsewhere: box-downsampling a thumbnail before applying effects, decimating sensor data before FFT, reducing log volume at the edge before shipping.
Watch out: —
Seen in Snes9x: GTK's `S9xMergeHires` walked 512-wide lines, keeping a running `l`/`r` pair and writing two blended outputs per input pair. It now writes `AVERAGE_1555(input[0], input[1])` once per pair and sets `width >>= 1`. 88176c03 2011-02 “Changed the hi-res filter to be a simple merge between adjacent pixels. ...”

### snes9x-025 — Offload scaling and colour conversion to a hardware or library path
Lens: gpu-rendering · Source: Snes9x
Principle: Hand-written per-pixel scaling loops are slow. Overlay hardware (Xv) or a tuned graphics library can scale far faster. When a colour-space conversion is still needed, precompute it into lookup tables indexed by the whole source pixel.
Why it is faster: —
Applies elsewhere: CSS transforms or `drawImage` scaling instead of JS pixel loops. Hardware video overlays on set-top boxes. SIMD-tuned libraries (libyuv, Skia) instead of custom loops.
Watch out: —
Seen in Snes9x: GTK's software driver removed about 390 lines of hand-written `internal_convert_scale` loops and lets cairo do the scaling. The unix X11 port gained `-xvideo`. Frames go out as YUY2 `XvImage`s, and hardware does the scaling. The RGB→YUV step uses `y_table`/`u_table`/`v_table[1 << 15]` built once in `SetupXvideo`. 6880eacd 2010-11 “Use cairo for scaling in software driver.” · 73967665 2015-12 “Add Xv acceleration support to unix port.”

### snes9x-026 — Precompute a function over the full input range of a quantized difference
Lens: lookup-tables · Source: Snes9x
Principle: When a costly function (sqrt of a weighted sum) depends only on a few small integer inputs, tabulate it for every input once at startup. A 16 M-entry float table indexed by the packed per-channel difference replaces the maths with one load. The xBRZ authors measured 30% faster.
Why it is faster: —
Applies elsewhere: gamma and tone-curve LUTs in image pipelines, precomputed CRC tables, sine tables on microcontrollers, colour-distance tables in palette quantizers.
Watch out: —
Seen in Snes9x: The vendored xBRZ filter (updated to 1.2) added `struct DistYCbCrBuffer`. It is filled with `sqrt(y² + cb² + cr²)` for all 256³ channel-difference combinations, and the comment notes “30% perf boost compared to distYCbCr()”. It also notes that an early `pix1 == pix2` test was 8% slower. 1d140638 2015-01 “Win32: update xbrz to 1.2”

### snes9x-027 — Store snapshots as XOR deltas in a power-of-two ring buffer
Lens: memory-layout · Source: Snes9x
Principle: Consecutive snapshots differ in only a few words. Store `(index, old ^ new)` pairs for the words that changed, separated by sentinels. XOR is its own inverse, so applying the same record restores the previous state. A power-of-two buffer replaces modulo with `& mask`, and swapping the current and previous buffers avoids a copy.
Why it is faster: —
Applies elsewhere: undo history in editors, incremental VM or container checkpoints, game replay and rollback netcode, delta-encoded telemetry.
Watch out: —
Seen in Snes9x: The new `StateManager` (rewind, based on SSNES code by Themaister) freezes state into `in_state`. `generate_delta` pushes `(i << 32) | xor_` for each differing `uint32` into a `uint64_t` ring sized with `nearest_pow2_size`, then swaps the `tmp_state`/`in_state` pointers. `pop()` XORs records back until it reaches the 0 sentinel. 486cde43 2012-03 “Win32: add rewind support”

### snes9x-028 — Sample input as late as possible in the frame
Lens: scheduling · Source: Snes9x
Principle: Where the main loop yields to read input decides how stale that input is when the next frame is computed. Break out right after the frame's output is complete, not at the start of the next frame. User actions then reach the screen one frame sooner at no compute cost.
Why it is faster: —
Applies elsewhere: game engines polling input just before simulation, not at the top of a long frame. VR late-latching of head pose. Audio plugins reading parameter changes per block, not per buffer.
Watch out: —
Seen in Snes9x: `S9xDoHEventProcessing` set `SCAN_KEYS_FLAG` (which exits `S9xMainLoop` to poll input) when the frame counter wrapped. It now sets it right after `S9xEndScreenRefresh()` at the start of VBlank (the “Brunnis lag fix”). 10e0ef00 2017-11 “Change where we break S9xMainLoop/Scan for input ("Brunnis lag fix")”

### snes9x-029 — Force a GPU sync after present, touching as little data as possible
Lens: scheduling · Source: Snes9x
Principle: Drivers queue several frames ahead, which adds latency. Make the CPU wait for the GPU after presenting: call `glFinish`, or lock the back buffer read-only. Lock only a tiny rectangle, so that if the driver reads back the locked region, it transfers almost nothing.
Why it is faster: —
Applies elsewhere: limiting frames in flight in Vulkan or DX12 renderers, `MaximumFrameLatency` settings, low-latency modes in streaming clients.
Watch out: —
Seen in Snes9x: A new Win32 “Reduce Input Lag” option adds `glFinish()` after `SwapBuffers` in OpenGL. In Direct3D, `Present` is now followed by `GetBackBuffer` plus `LockRect(..., D3DLOCK_READONLY)`, with `D3DPRESENTFLAG_LOCKABLE_BACKBUFFER` set. A follow-up limits the lock to `RECT r = {0, 0, 2, 2}`. 113a43c4 2018-05 “win32: Add a reduce input lag option for OpenGL and Direct3D.” · f51734d6 2018-05 “d3d: only lock a small region in case the driver tries to readback.”

### snes9x-030 — Ask the compositor to step aside in fullscreen
Lens: platform · Source: Snes9x
Principle: A desktop compositor adds a copy and often a frame of latency to every presented image. Fullscreen apps can set the window-manager hint that lets the compositor unredirect the window, so frames go straight to scanout.
Why it is faster: —
Applies elsewhere: fullscreen-exclusive or flip-model presentation on Windows, Wayland direct scanout for video players, kiosk apps disabling compositing.
Watch out: —
Seen in Snes9x: On entering fullscreen, GTK's `enter_fullscreen_mode` sets `_NET_WM_BYPASS_COMPOSITOR` to 1 via `XChangeProperty`, and `leave_fullscreen_mode` clears it. A follow-up passes a 32-bit value instead of a `char` and sets the hint on the top-level window. 363fdc98 2018-04 “Try to turn off compositor in full-screen.” · a1f8cf38 2018-05 “Bypass compositor properly.”

### snes9x-031 — Opt into known-endianness fast paths and drop redundant checks
Lens: compiler-build · Source: Snes9x
Principle: Portable code often assembles multi-byte values a byte at a time. When the target's byte order is known, a single native load or store is faster. Also remove bounds checks that the surrounding code's invariants already guarantee.
Why it is faster: —
Applies elsewhere: `memcpy`-based unaligned loads in parsers, `__builtin_bswap` in network code, removing defensive checks in a validated inner loop.
Watch out: —
Seen in Snes9x: `blargg_config.h` now defines `BLARGG_NONPORTABLE 1`, so `GET_LE16`/`SET_LE16` in `blargg_endian.h` become direct `uint16_t` accesses on little-endian hosts. The same commit removed the `out >= m.out_end` overflow-to-`extra` branch from `SPC_DSP`'s sample-write macro and the null-buffer fallback in `set_output`. 287fab5f 2018-04 “Small optimizations to DSP.”

### snes9x-032 — Fold a clamp into a small lookup table
Lens: lookup-tables · Source: Snes9x
Principle: If a value is added and then clamped to a ceiling that changes rarely, build a table indexed by the unclamped sum that already holds the clamped result. Each channel then costs one indexed load and no compare-and-branch. The table only needs rebuilding when the ceiling changes.
Why it is faster: —
Applies elsewhere: audio gain with hard limiting on 8-bit samples, gamma or brightness ramps in image filters, saturating counters on microcontrollers that have no saturating instructions.
Watch out: —
Seen in Snes9x: `COLOR_ADD` in `gfx.h` split two pixels into R/G/B and compared each channel sum against `mul_brightness[PPU.Brightness][31]`. A 64-entry `brightness_cap[]` is now rebuilt in `S9xFixColourBrightness` whenever brightness changes, and the add becomes `BUILD_PIXEL(brightness_cap[r1 + r2], brightness_cap[g1 + g2], brightness_cap[b1 + b2])`. The commit message says it is “a little faster”. 02aa2d0c 2019-03 “Use a lookup table. A little faster.”

### snes9x-033 — Saturating arithmetic on packed fields within one integer (SWAR)
Lens: simd · Source: Snes9x
Principle: Several small fields packed into one machine word can be added or subtracted together if each field has a guard bit above it. The carry or borrow that lands in the guard bit becomes a mask, and multiplying that mask by the field's maximum saturates every overflowing field at once. One integer add replaces three separate unpack, compare and repack sequences, and there are no branches.
Why it is faster: —
Applies elsewhere: RGB565 or RGBA5551 blending on embedded displays, adding packed 8-bit counters in a 64-bit word, software rasterizers on CPUs without SIMD.
Watch out: —
Seen in Snes9x: `COLOR_SUB` used three masked compares, one per channel. `COLOR_ADD` is now done on red and blue together plus green separately, with a carry mask: `int rgbsaturate = (((g & (0x20 << GREEN_SHIFT_BITS)) | rbcarry) >> 5) * 0x1f;`. Subtraction uses the same trick with borrow guards. A later commit turned these into `static fn()` members of `struct COLOR_ADD`/`COLOR_SUB` so they can be passed as template policies. 9721f78d 2019-03 “Select brightness function in tile renderer.” · 3aecd16e 2019-03 “Unmacro DRAW_PIXEL Normal1x1.”

### snes9x-034 — Pick the specialized routine once, outside the hot loop
Lens: branching-fast-paths · Source: Snes9x
Principle: If a slower general case is needed only under some condition, keep the fast routine as the default and switch to the slower one when the condition holds, choosing once per batch (per frame or per scanline). The inner loop then never tests the condition and carries no code for the rare case.
Why it is faster: —
Applies elsewhere: picking a premultiplied or straight-alpha blit routine once per draw call, a codec choosing its SIMD kernel once at startup, a query engine choosing a null-aware or null-free operator at plan time.
Watch out: —
Seen in Snes9x: Brightness-capped color addition is needed only when the screen is not at full brightness. In `S9xSelectTileRenderers`, when `IPPU.MaxBrightness != 0xf`, the renderer index is remapped to new `Add_Brightness_` variants. Otherwise the plain SWAR add runs with no brightness check per pixel. 9721f78d 2019-03 “Select brightness function in tile renderer.”

### snes9x-035 — Turn runtime-configurable formats into compile-time constants
Lens: compiler-build · Source: Snes9x
Principle: If a data format can be chosen at run time through function pointers and global mask variables, every pixel operation becomes an indirect call plus memory loads. Fixing the format at build time makes the masks immediates and the helpers inlinable. Lookup tables that only existed to make the generic path fast can then be deleted as well.
Why it is faster: —
Applies elsewhere: a graphics library built per target pixel format, serializers specialized per schema rather than driven by reflection, DSP code with sample width as a template parameter.
Watch out: —
Seen in Snes9x: Under `GFX_MULTI_FORMAT`, `BUILD_PIXEL` and `DECOMPOSE_PIXEL` expanded to `(*GFX.BuildPixel)(R,G,B)`, and masks such as `MAX_RED` or `RGB_LOW_BITS_MASK` were `extern uint32` variables. Color addition went through the 128 KB `GFX.X2` table. Most of this was removed: the masks are now macros, there are no function pointers, and `GFX.X2` and its setup loop are gone. 76b6342c 2019-03 “Remove most of GFX_MULTI_FORMAT.”

### snes9x-036 — Generate specialized kernels from static policy templates
Lens: compiler-build · Source: Snes9x
Principle: Write a kernel once as a template over small policy types (how to write a pixel, which blend to apply, how to step lines), each exposing static member functions. Every combination becomes its own straight-line function with all calls known at compile time, so the optimizer can inline the whole chain. This is fast like hand-written macro expansion, but stays debuggable and type-checked.
Why it is faster: —
Applies elsewhere: C++ image-processing libraries parameterized on pixel type and blend op, Rust generics over traits for codecs, game engines specializing shaders or CPU skinning per vertex format.
Watch out: —
Seen in Snes9x: `tile.cpp` used to `#include` itself several levels deep and stamp out renderers with `DRAW_TILE`, `DRAW_PIXEL`, `GET_CACHED_TILE` and similar macros. A series of commits replaced this with `struct Normal1x1`/`Normal2x1`/`Hires` exposing `static void Draw(...)`, blend policies such as `typedef REGMATH<COLOR_ADD> Blend_Add` calling `MATHOP::Calc`, and a `CachedTile` class, all instantiated per combination. 7b31af02 2019-04 “Unmacro DRAW_TILE.” · b4142cb1 2019-04 “Remove tile drawing facade functions.” · e59ccab0 2019-04 “Unmacro GET_CACHED_TILE, IS_BLANK_TILE and SELECT_PALETTE.” · e54b9f43 2019-04 “Remove top-level #include.”

### snes9x-037 — Force-inline tiny hot helpers
Lens: compiler-build · Source: Snes9x
Principle: The compiler's inlining heuristics can give up inside large template instantiations or after code was moved out of macros, leaving call overhead on a two-line function in the innermost loop. A portable `alwaysinline` macro (`__attribute__((always_inline))` or `__forceinline`) restores the inlining that macros used to guarantee.
Why it is faster: —
Applies elsewhere: vector math helpers in game engines, byte-swap and checksum helpers in network stacks, accessor functions in embedded HAL layers.
Watch out: —
Seen in Snes9x: Two private macros (`FORCE_INLINE` in `xbrz.cpp`, `alwaysinline` in the APU) were merged into one in `port.h`. It was then applied to `COLOR_ADD::fn`/`fn1_2`, the blend `Calc` functions, the `CachedTile` methods and the per-pixel `Draw`/`Get` policies in `tile.cpp`. c71429c2 2019-04 “Merge `FORCE_INLINE` macro to `alwaysinline` into port.h.” · 3b9820d7 2019-04 “Use alwaysinline with COLOR functions in gfx.h, MATH functions in tile.cpp, and CachedTile class in tile.cpp.” · 1f2b5988 2019-04 “Define alwaysinline to PIXEL functions.”

### snes9x-038 — A rolled loop can beat a manual unroll
Lens: compiler-build · Source: Snes9x
Principle: Hand-unrolling a short fixed-count loop inside a macro that is expanded many times multiplies code size and puts pressure on the instruction cache. A plain `for` loop with a constant trip count lets the compiler decide how far to unroll, and it may vectorize or schedule the loop better than the copy-pasted version.
Why it is faster: —
Applies elsewhere: per-byte loops in parsers, 8-wide pixel loops in blitters, fixed-size matrix kernels. Measure before keeping any hand unroll.
Watch out: —
Seen in Snes9x: The `DRAW_TILE` bodies called `DRAW_PIXEL(0, Pix = bp[0]); … DRAW_PIXEL(7, Pix = bp[7]);` and the reverse order for flipped tiles. They were replaced with `for (int x = 0; x < 8; x++) DRAW_PIXEL(x, Pix = bp[x]);` (and `bp[7 - x]` for flipped tiles). The commit reports a 3–4% speed gain. 2d97d2dd 2019-03 “Use for loops for DRAW_TILE. 3-4% speed benefit.”

### snes9x-039 — Split heavy template instantiations across translation units
Lens: compiler-build · Source: Snes9x
Principle: One source file that instantiates hundreds of template specializations is compiled serially and becomes the critical path of a parallel build. Put the shared template code in a header and the instantiations in several small .cpp files, so the build system can compile them on separate cores.
Why it is faster: —
Applies elsewhere: large Eigen or Boost-heavy files, generated protobuf code, explicit-instantiation files in numerical libraries.
Watch out: —
Seen in Snes9x: The renderer templates moved from `tile.cpp` into `tileimpl.h`. The instantiations were split into `tileimpl-n1x1.cpp`, `tileimpl-n2x1.cpp` and `tileimpl-h2x1.cpp`, which were added to the Unix makefile and the Xcode project. 7603ed09 2019-05 “tile: Divide instantiations. This enables parallel compilation.”

### snes9x-040 — Enable link-time / whole-program optimization
Lens: compiler-build · Source: Snes9x · Also: ares
Principle: Without LTO the compiler cannot inline or specialize across source files, so hot calls between modules (CPU core to memory map, renderer to PPU) remain real calls. LTO (`-flto`, or MSVC `WholeProgramOptimization`) runs optimization again over the whole program at link time.
Why it is faster: —
Applies elsewhere: release builds of browsers and databases, firmware images where code size and speed both matter, Rust `lto = "fat"`.
Watch out: —
Seen in Snes9x: The libretro `Makefile` added `-flto` to CFLAGS/CXXFLAGS/LDFLAGS for Unix, macOS, iOS, Vita, Wii/WiiU and others. It was later removed for toolchains without LTO (PS3), and an override was added. The win32 project turned on `<WholeProgramOptimization>true`. 8064f7f2 2018-08 “Makefile Changes: Use Link-time optimization on GCC, and define __WIN32__ in all MSVC targets” · df9e37d7 2019-02 “win32: Fix poor performance with OpenGL/snes_ntsc.” • Also seen in ares (ares-066): `nall/GNUmakefile` set `lto = true` for every non-debug build type. Tuning dropped `-fwhole-program` and switched to `-flto=auto -fno-fat-lto-objects` for GCC and `-flto=thin` for clang, which gave the Game Boy core about +10 fps on average. LTO was turned off in desktop-ui for a while and then made the default again: "the difference between barely scraping 50-55fps on 32x core to getting constant 60-70fps". cb9f94ae3 2021-07 “nall: enable lto by default on non-debug builds” · 9c5ecf52c 2022-07 “nall: tweak LTO settings for better performance in compiled builds” · 9aa05bcbe 2022-07 “nall: more LTO tuning” · 4da6316d1 2022-12 “desktop-ui: make LTO default again”

### snes9x-041 — Choose optimization levels per compiler, and keep the safe ones
Lens: compiler-build · Source: Snes9x
Principle: The same flag means different things on different compilers. `-O3` helps GCC/Clang, while `-Ofast` breaks IEEE semantics and can change program output. Set the level per toolchain, back out flags that alter results, and fix a single miscompiled function with a scoped pragma instead of lowering the global level. Build scripts should also leave the user's own optimization flags alone.
Why it is faster: —
Applies elsewhere: CMake per-compiler flag blocks, `#pragma GCC optimize` around one known-bad function, distro packaging that passes its own CFLAGS.
Watch out: —
Seen in Snes9x: The libretro build moved from `-O2` to `-O3` but kept `-O2` for MSVC. The Unix `configure` briefly defaulted to `-Ofast` and was reverted to `-O3` the next day. `configure` stopped stripping `-O2`/`-g` from the user's CFLAGS and now appends user flags last. `SetupOBJ` in `gfx.cpp` is wrapped in `#pragma GCC optimize ("no-tree-vrp")` to work around a GCC miscompilation. 1fcfcbb4 2018-12 “libretro: Use -O3 instead of -O2.” · a9a646eb 2019-01 “Don't use O3 for MSVC” · c1042e53 2019-05 “unix: Defaults -Ofast and adds --enable-sse41, --enable-avx and --enable-neon.” · 8452680d 2019-05 “unix: Revert -Ofast to -O3.” · 39601dd1 2018-05 “unix: Don't remove compiler flags for no reason…” · 91518800 2018-06 “Disable tree-vrp on GCC for this function. GCC miscompiles it.”

### snes9x-042 — Tune for the exact target CPU when it is known
Lens: platform · Source: Snes9x
Principle: When a build targets one fixed device, flags such as `-mcpu`/`-mtune` and enabling its SIMD unit let the compiler schedule for that pipeline and auto-vectorize. For general desktop builds, offer the ISA extensions as opt-in configure switches so a portable default binary still exists.
Why it is faster: —
Applies elsewhere: console and phone builds, Raspberry Pi images built with `-mcpu=cortex-a72`, x86-64-v3 package variants.
Watch out: —
Seen in Snes9x: The Nintendo Switch (libnx) target gained `-mcpu=cortex-a57+crc+fp+simd -ffast-math` next to `-march=armv8-a -mtune=cortex-a57`. The Unix `configure` gained `--enable-sse41`, `--enable-avx2` and `--enable-neon`, all off by default. 62e5a201 2018-09 “Clean up libnx makefile and enable more optimizations” · c1042e53 2019-05 “unix: Defaults -Ofast and adds --enable-sse41, --enable-avx and --enable-neon.”

### snes9x-043 — Return early on writes that change nothing
Lens: branching-fast-paths · Source: Snes9x
Principle: If writing a control register sets off expensive recomputation (rescheduling timers, rebuilding derived state), compare the new value with the stored one and return when they match. Programs often rewrite the same value in a loop, and each rewrite would otherwise repeat the full side-effect path.
Why it is faster: —
Applies elsewhere: React `setState` bailing out on equal values, GPU state caches that skip redundant `glEnable`/bind calls, setters that suppress change notifications when the value is unchanged.
Watch out: —
Seen in Snes9x: `S9xSetCPU` handling `$4200` (NMI and IRQ enable) recomputes V/H timer state and IRQ positions. It now begins with `if (Byte == Memory.FillRAM[0x4200]) break;`. db1d5249 2018-06 “Don't do anything when write to $4200 is the same.”

### snes9x-044 — Identity fast path: copy when the transform is a no-op
Lens: algorithms · Source: Snes9x
Principle: If a general transform (resampling, scaling, format conversion) is called with parameters that make it the identity, skip the arithmetic and copy the data straight through. Any helper that reports output size must use the same shortcut so the two stay consistent.
Why it is faster: —
Applies elsewhere: image resizers returning the input when the scale is 1:1, audio mixers bypassing sample-rate conversion when the device rate matches, text transcoders that `memcpy` when source and destination encodings match.
Watch out: —
Seen in Snes9x: `HermiteResampler::read` interpolated every sample even when `r_step == 1.0`. It now does `ring_buffer::pull((unsigned char*)data, num_samples * sizeof(short))` in that case, and `avail()` returns `space_filled()` directly instead of the fractional-step formula. 412c6554 2018-06 “Backport Hermite Resampler bypass code from Libretro version”

### snes9x-045 — Have the producer write straight into the consumer's ring buffer
Lens: memory-layout · Source: Snes9x
Principle: A pipeline that renders into a staging buffer and then copies into a queue touches each sample twice and needs logic for partial copies and overflow. Give the producer a handle to the final ring buffer and push one element at a time. The intermediate buffers and their copy and drop logic disappear.
Why it is faster: —
Applies elsewhere: audio synths writing directly into the device ring buffer, log appenders writing into a lock-free ring instead of a temporary string, network decoders filling the application buffer in place.
Watch out: —
Seen in Snes9x: The S-DSP wrote into `spc::dsp_buffer` and MSU-1 into `msu::mixing_buffer`. `S9xFinalizeSamples` then `push()`ed both into resamplers, with clear and retry logic. After a consolidated `Resampler` class and a single-sample `push(int16_t l, int16_t r)` were added, `SNES::dsp.spc_dsp.set_output(spc::resampler)` and `S9xMSU1SetOutput(msu::resampler)` send output straight into the resampler, and the extra buffers were deleted. 36406a66 2019-02 “Add a consolidated resampler.” · ac03faca 2019-02 “Add single sample push.” · e938225c 2019-02 “APU: Remove extra buffers.”

### snes9x-046 — Wrap ring indices with a compare-and-subtract, not modulo
Lens: branching-fast-paths · Source: Snes9x
Principle: For a ring buffer whose size is not a power of two, `(start + size) % capacity` costs an integer division on every push. Both operands are already below the capacity, so one conditional subtract gives the same answer for a fraction of the cost.
Why it is faster: —
Applies elsewhere: UART receive buffers on microcontrollers without hardware divide, audio delay lines, circular log buffers.
Watch out: —
Seen in Snes9x: In `apu/resampler.h`, `int end = (start + size) % buffer_size;` became `int end = start + size; if (end > buffer_size) end -= buffer_size;`. The new per-sample `push(l, r)` uses the same form. ac03faca 2019-02 “Add single sample push.”

### snes9x-047 — Render the final pass directly to the output surface
Lens: gpu-rendering · Source: Snes9x · Also: Flycast
Principle: In a multi-pass post-processing chain, the last pass doesn't need its own offscreen target that is then drawn again to the screen. Bind the default framebuffer and the final viewport for that pass. This saves a texture allocation, a full-screen draw and a bandwidth-heavy copy every frame.
Why it is faster: —
Applies elsewhere: web canvas filter chains, game post-processing stacks (tonemapping as the last pass), video players applying shaders before present.
Watch out: —
Seen in Snes9x: `GLSLShader::render` allocated a texture and FBO for every pass, and the GTK OpenGL driver then drew the result to the window. Now, for `lastpass`, it does `glBindFramebuffer(GL_FRAMEBUFFER, 0)` and sets the window viewport, and `S9xOpenGLDisplayDriver::update` just calls `gl_swap()`. A later commit saves and restores whatever framebuffer was bound so this also works under Wayland subsurfaces. 21dd926a 2018-05 “Output directly to framebuffer on last pass.” • Also seen in Flycast (flycast-075): `RenderFrame` in `gles.cpp` always called `fullscreenQuadCreateTemporaryFBO` and `DrawFullscreenQuad`. Now, when `HorizontalResolution` and `VerticalResolution` are both 100, it sets `glViewport` to the screen and draws directly. Below 100%, it renders to a smaller FBO and scales up. 8571be39f 2019-01 “Rendering resolution lowering (horizontal and vertical) is possible from the configuration menu”

### snes9x-048 — Set GPU state once, not every frame
Lens: gpu-rendering · Source: Snes9x · Also: Flycast
Principle: If the draw is always the same full-screen quad, bind vertex attribute pointers, enable arrays and upload vertex data once at setup or on resize. Re-specifying state every frame costs driver validation and CPU time and achieves nothing.
Why it is faster: —
Applies elsewhere: WebGL apps that rebind attributes per frame, UI toolkits re-uploading static geometry, Vulkan pipelines rebuilt instead of cached.
Watch out: —
Seen in Snes9x: `S9xOpenGLDisplayDriver::update` called `glUseProgram`, `glBindBuffer`, `glEnableVertexAttribArray` and `glVertexAttribPointer` (or `glVertexPointer` on legacy GL) before each `glDrawArrays`, then disabled them again. These now run once in `opengl_defaults`/`update_texture_size`, and the per-frame path is just `glDrawArrays(GL_TRIANGLE_STRIP, 0, 4)`. The coordinate VBO is re-uploaded only on non-legacy paths, and only on size change. 1cd82509 2018-11 “GTK+: OpenGL: Don't reset state we don't change.” · b6858adf 2018-11 “GTK+: OpenGL: Only do this part on non-legacy.” • Also seen in Flycast (flycast-142): `drawQuad()` in `core/rend/gles/quad.cpp` built a 4-vertex array on the stack each time, with the V coordinate depending on `swapY`, and uploaded it. `initQuad()` now creates `quadBuffer` and `quadBufferSwapY` with `GL_STATIC_DRAW`, each with its own VAO. The draw just binds `swapY ? quadVertexArraySwapY : quadVertexArray`. 9781d8971 2022-12 “gl: use static buffers for quad vertices”

### snes9x-049 — Stream uploads through write-only, invalidated pixel buffers
Lens: gpu-rendering · Source: Snes9x
Principle: Write each frame into a mapped pixel-unpack buffer and upload from it asynchronously instead of passing a CPU pointer to `glTexSubImage2D`. Map it write-only with the invalidate flag (`GL_MAP_INVALIDATE_BUFFER_BIT`) so the driver can hand over fresh memory instead of waiting for the GPU to finish with the old contents or reading them back. Never map for read-write if you only write.
Why it is faster: —
Applies elsewhere: video players uploading decoded frames, WebGL texture streaming, map-tile renderers. In Vulkan or D3D12, ring-buffered upload heaps play the same role.
Watch out: —
Seen in Snes9x: GTK enabled PBOs by default on GL ≥ 1.5 and switched to `glMapBufferRange(..., GL_WRITE_ONLY | GL_MAP_INVALIDATE_BUFFER_BIT)` on GL ≥ 3.0. In the win32 `COpenGL::Render`, the PBO had been mapped `GL_READ_WRITE`, which made OpenGL with the NTSC filter slow. It is now mapped `GL_WRITE_ONLY`, and the texture internal format changed from `GL_RGB` to `GL_RGBA`. ffdf886a 2018-11 “GTK+: GL: Use PBOs by default, disable on GL < 1.5” · 175b9baf 2019-01 “GTK+: OpenGL: Use MapBufferRange on GL >= 3.0.” · df9e37d7 2019-02 “win32: Fix poor performance with OpenGL/snes_ntsc.”

### snes9x-050 — Keep pixels in their native format all the way to the display
Lens: memory-layout · Source: Snes9x
Principle: Each format conversion on the way to the screen (16→24 bit, byte swizzling) is a full pass over the frame in memory. If the destination API accepts the source format (RGB565 textures, 16-bit surfaces), give it that format directly and drop the conversion step.
Why it is faster: —
Applies elsewhere: uploading YUV planes and converting in a shader, passing camera frames in their native format to the encoder, embedded LCD drivers that take RGB565 directly.
Watch out: —
Seen in Snes9x: The GTK OpenGL driver had a 24-bit PBO mode that ran `S9xConvert(..., 24)` into the mapped buffer. It was removed as “actually useless”, leaving 16-bit (`GL_UNSIGNED_SHORT_5_6_5`) and 32-bit uploads. The NTSC filter's hires path and the libretro build were changed to emit 16-bit/565 output that matches the frontend's format. 94c2c4c4 2018-10 “Remove 24-bit PBO format. It's actually useless.” · 456f9036 2018-05 “Use 16-bit with NTSC filter in hires mode.” · bd7669e7 2019-03 “libretro: Use 565 format for NTSC filter.”

### snes9x-051 — Wrap an existing buffer instead of copying into a library object
Lens: memory-layout · Source: Snes9x
Principle: Many graphics APIs can build an image object over memory you already own, given a pointer, format and stride. Wrapping the frame buffer that way removes a per-frame allocation, a conversion pass and a copy.
Why it is faster: —
Applies elsewhere: `cairo_image_surface_create_for_data`, `new ImageData(clampedArray, w, h)` on the web, `CGBitmapContextCreate` over app memory, NumPy views over a C buffer.
Watch out: —
Seen in Snes9x: The GTK software driver converted each frame to 24-bit with `S9xConvert` into a reallocated buffer behind a `GdkPixbuf`, then drew it with `gdk_cairo_set_source_pixbuf`. It now calls `cairo_image_surface_create_for_data((unsigned char *) src, CAIRO_FORMAT_RGB16_565, width, height, src_pitch)` on the emulator's buffer and draws that surface. f1b508de 2018-10 “GTK+: Faster cairo software output path.”

### snes9x-052 — Poll a GPU fence instead of blocking on glFinish
Lens: concurrency · Source: Snes9x
Principle: `glFinish` blocks the thread until the GPU is idle, and the UI event loop stops with it. Insert a fence after the swap, then return to the event loop and check the fence with a zero timeout before starting the next frame. If a presentation counter is available (e.g. `GLX_OML_sync_control`), poll that instead. Sleep briefly between polls rather than spinning.
Why it is faster: —
Applies elsewhere: games limiting frames in flight with `glFenceSync` or Vulkan fences, browsers pacing compositor frames, video players waiting on presentation timestamps.
Watch out: —
Seen in Snes9x: `swap_buffers` in the GTK OpenGL driver did `usleep(0); glFinish();`. It first gained a blocking `glClientWaitSync`. The fence was then kept across frames, with `is_ready()` checking `glClientWaitSync(fence, ..., 0) == GL_TIMEOUT_EXPIRED`, and `S9xIdleFunc` returns early while the driver is not ready. glFinish and fences became separate options. Fences were later replaced by `GTKGLXContext::ready()`, which compares `glXGetSyncValuesOML` counters, and a `usleep(100)` stops the idle loop from spinning. dc4de5b7 2018-12 “GTK+: OpenGL: Add fences as glFinish alternative.” · 8106c284 2018-12 “GTK+: OpenGL: Use fence to allow GTK+ to run during wait.” · 3eea5065 2018-12 “GTK+: OpenGL: Separate glFinish and glFenceSync options.” · 7b310a0c 2019-03 “GTK: Swap glFenceSync for sync control extension.” · 51edec75 2019-04 “GTK: OpenGL: Don't spin with sync_control option.”

### snes9x-053 — Let the compositor pace windowed output
Lens: platform · Source: Snes9x
Principle: In a composited window, driver vsync together with the compositor's own frame clock can add latency or judder because two clocks are involved. Turn off the swap interval in windowed mode and wait on the compositor's flush. Use real vsync only in exclusive fullscreen, where nothing else composites.
Why it is faster: —
Applies elsewhere: Windows apps calling `DwmFlush`, Wayland clients pacing on frame callbacks, macOS apps using `CVDisplayLink`.
Watch out: —
Seen in Snes9x: In win32 `S9xDeinitUpdate` with OpenGL and `GUI.DWMSync`, it now calls `SetSwapInterval(GUI.Vsync ? 1 : 0)` only in fullscreen or when DWM is off. Otherwise it sets interval 0, presents, and syncs to DWM. 23a0e125 2019-03 “win32:opengl: Use DWM sync in windowed, Vsync in fullscreen exclusive.”

### snes9x-054 — Fuse a follow-up pass into the producing loop
Lens: algorithms · Source: Snes9x
Principle: If a second pass over an image only derives extra output from values the first pass just computed (darkened scanlines, a doubled row), emit both outputs from the first loop while the value is still in a register. This removes a full read and write of the frame and the scratch layout the second pass needed.
Why it is faster: —
Applies elsewhere: computing a thumbnail while decoding, writing a checksum while serializing, writing both mip level 0 and 1 in a single compute shader.
Watch out: —
Seen in Snes9x: Blargg's NTSC filter wrote half-height output, and a second loop in `RenderBlarggNTSC` walked it bottom-up to double lines and darken alternates. A new `snes_ntsc_blit_scanlines` uses `PIXEL_OUT(x)` to write `line_outa[x] = value; line_outb[x] = value - (value >> snes_ntsc_scanline_offset & snes_ntsc_scanline_mask);` in one pass. GTK adopted it first, then win32. 61cced47 2018-11 “GTK+: Use root snes_ntsc implementation.” · df9e37d7 2019-02 “win32: Fix poor performance with OpenGL/snes_ntsc.”

### snes9x-055 — Split image filters into row bands with halo rows
Lens: concurrency · Source: Snes9x
Principle: Give each thread a horizontal band of the image so a neighborhood filter scales across cores. Each band must read a few rows beyond its edges (a halo) but write only its own rows, otherwise the seams come out wrong. Size output offsets by the vertical scale factor, and keep the join logic in one shared wait helper.
Why it is faster: —
Applies elsewhere: tiled convolution in image editors, parallel video scalers, GPU compute tiles with shared-memory aprons.
Watch out: —
Seen in Snes9x: `internal_threaded_filter` in `gtk_display.cpp` computed destination offsets from `coverage * yscale`, and three copies of the busy-wait loop became `wait_for_jobs_to_complete()` (polling with `sched_yield`). For multithreaded runs, `xBRZ` converts `height + 4` rows starting two rows above the band and scales rows `2..height+2`, so each band has context across the seam. 93a99a48 2019-01 “GTK+: Fix threaded filters.” · 70484ba6 2019-03 “GTK: Fix xBRZ scaler when multithreading enabled.”

### snes9x-056 — Skip reloading a resource whose key hasn't changed
Lens: laziness · Source: Snes9x
Principle: Remember the key (path, hash) of the expensive resource currently loaded and return immediately when asked to load the same key again. Settings dialogs and reinit paths often request a reload without the input having changed.
Why it is faster: —
Applies elsewhere: asset managers deduplicating texture loads, config hot-reload that compares mtime or hash, memoized compilation of the same shader source.
Watch out: —
Seen in Snes9x: `COpenGL::SetShaders` on win32 recompiled the Cg/GLSL shader chain every time it was called. It now keeps `currentShaderFile[MAX_PATH]` and returns early when `lstrcmp(file, currentShaderFile) == 0`. 47dce096 2019-02 “win32: only reload shader if filename changes”

### snes9x-057 — Parse in a single pass instead of a chain of filter passes
Lens: algorithms · Source: Snes9x
Principle: Running several separate passes over the same line list (expand includes, strip pragmas, parse names, parse formats) costs repeated scans and, with `vector::erase`, quadratic shifting. Handle every directive in one loop as each line is read, and append only the lines that survive.
Why it is faster: —
Applies elsewhere: config and template preprocessors, log parsers with several extractors, build tools scanning sources for includes and annotations.
Watch out: —
Seen in Snes9x: Loading a shader called `read_shader_file_with_includes`, then `strip_parameter_pragmas` (erasing from a `std::vector`), then `slang_parse_pragmas`. These were merged into a single reader that handles `#include`, `#pragma parameter`, `#pragma name` and `#pragma format` inline and uses `continue` to drop those lines. According to the commit, this “improves loading time of crt-royale significantly.” fc31f47f 2019-01 “Shaders: Scan source code in as few passes as possible.”

### snes9x-058 — Fast state restore: load in place and invalidate only caches
Lens: memory-layout · Source: Snes9x
Principle: A robust snapshot loader reads into temporary copies, resets everything, then copies back, which suits untrusted or partial loads. When the caller restores state many times per second (rewind, run-ahead, netplay rollback), offer a trusted mode that deserializes directly into the live buffers and only invalidates derived caches, without a full reset.
Why it is faster: —
Applies elsewhere: game rollback netcode, database page restore from a trusted WAL, an undo stack restoring memento blobs in place.
Watch out: —
Seen in Snes9x: With `Settings.FastSavestates` (set from libretro's `RETRO_ENVIRONMENT_GET_AUDIO_VIDEO_ENABLE` bit), `S9xUnfreezeFromStream` calls `UnfreezeBlock(stream, "VRA", Memory.VRAM, …)` instead of `UnfreezeBlockCopy` into a temporary buffer. Blocks for chips that aren't present are skipped, and `S9xResetPPUFast()` (just clearing the tile caches and setting dirty flags) replaces `S9xReset()`. d2183af3 2018-08 “libretro: add fast snapshot support”

### snes9x-059 — Give hot, narrow access paths their own accessor
Lens: branching-fast-paths · Source: Snes9x
Principle: If a generic read or write routine checks for special regions (I/O, overlays), every access pays for those checks even when the address can never land there. For paths whose address range is known, such as stack accesses or opcode fetch, add a direct accessor that skips the dispatch.
Why it is faster: —
Applies elsewhere: fast paths for aligned or in-bounds accesses in VMs, separate stack push/pop helpers in interpreters, ORMs that skip hooks on bulk internal loads.
Watch out: —
Seen in Snes9x: In the SPC700 core (`apu/bapu/smp/core.cpp`), `op_readstack()` and `op_writestack()` were macros over `op_read`/`op_write`, which check for MMIO (`0x00f0`) and IPL ROM. They are now `alwaysinline` functions accessing `apuram[0x0100 | ++regs.sp]` directly. The first attempt, which also specialized `op_readpc`, was reverted and then re-added for the stack accessors. f24ca1f8 2019-02 “APU: Seemingly obvious optimization.” · 8c9d900c 2019-02 “Add back optimizations.”

### snes9x-060 — Swap an overlay into memory on toggle instead of checking it on every read
Lens: memory-layout · Source: Snes9x
Principle: If a small ROM or overlay sometimes shadows part of RAM, checking the overlay flag on every read taxes the common path. Instead, copy the overlay into the backing array when it is enabled and save the shadowed bytes aside, then reverse that when it is disabled. Reads become plain array loads, and the cost moves to the rare toggle and to writes into that window.
Why it is faster: —
Applies elsewhere: copy-on-enable feature patches, page-table remapping instead of software checks, swapping a precomputed table into a hot array.
Watch out: —
Seen in Snes9x: `SMP::op_read` checked `if (addr >= 0xffc0 && status.iplrom_enable) return iplrom[...]`. In `mmio_write` for `$F1`, changes to `iplrom_enable` now `memcpy` the IPL ROM into `apuram[0xffc0]` and stash the RAM bytes in `highmem[64]` (or restore them). Writes at `≥ 0xffc0` are redirected, and save/load handle the swap. This was committed, reverted, re-added and then reverted again (“Something's not quite right with it”). 7dd2f760 2019-02 “IPL rom mapping optimization.” · 8c9d900c 2019-02 “Add back optimizations.” · 8624a3d0 2019-03 “Revert IPL rom optimization.”

### snes9x-061 — Don't redraw the whole frame on expose events
Lens: laziness · Source: Snes9x
Principle: Window systems send expose/draw signals for many reasons, and with a GPU swap chain the last frame is usually still valid. Handle the signal yourself and stop it propagating, and re-render only when the backend actually needs it (e.g. software output). The toolkit then doesn't redraw the widget on top of your output.
Why it is faster: —
Applies elsewhere: games ignoring WM_PAINT while their swap chain is valid, canvas apps throttling resize-driven redraws, native video views that skip toolkit painting.
Watch out: —
Seen in Snes9x: The GTK `event_drawingarea_draw`/`expose` handlers now return `TRUE` (stop propagation) instead of `FALSE`. `Snes9xWindow::expose` no longer calls `S9xRealDeinitUpdate` unconditionally, only on Wayland with `HWA_NONE`. 6d0eddf0 2018-10 “GTK+: Don't pass draw signal on OpenGL/Wayland.”

### snes9x-062 — Don't std::move a local return value
Lens: compiler-build · Source: Snes9x
Principle: Writing `return std::move(local);` prevents named return value optimization, so the compiler must move-construct instead of building the object directly in the caller's slot. A plain `return local;` lets copy elision happen, and the language still falls back to a move if it can't.
Why it is faster: —
Applies elsewhere: any C++ factory function returning a string, vector or buffer. Clang's `-Wpessimizing-move` catches this.
Watch out: —
Seen in Snes9x: `canonicalize()` in `shaders/shader_helpers.cpp` returned `std::move(filename_string)`. It now returns `filename_string`. 0e559cd0 2019-01 “Don't use move on return value.”

### snes9x-063 — Replace a periodic modulo with a counter that resets
Lens: branching-fast-paths · Source: Snes9x
Principle: If a counter is only ever incremented in one place, "every Nth time" can be tested with `++n == N` followed by `n = 0`, not with `n % N == 0`. That turns an integer division into a compare. The saving is large on CPUs that have no hardware divide.
Why it is faster: —
Applies elsewhere: logging or metrics every N requests, flushing a batch every N items, a "once per second" check in a frame loop on a microcontroller.
Watch out: —
Seen in Snes9x: `S9xStartScreenRefresh` in gfx.cpp computed rendered-FPS statistics with `++IPPU.FrameCount % Memory.ROMFramesPerSecond == 0`. Because the counter is reset to 0 inside that branch, it now uses `++IPPU.FrameCount == (uint32)Memory.ROMFramesPerSecond`. The commit message names pre-ARMv7 cores, which have no divide instruction. 305cafc8 2022-02 “gfx: Use simpler equality test for IPPU.FrameCount”

### snes9x-064 — Use count-leading-zeros instead of a shift loop to normalise
Lens: branching-fast-paths · Source: Snes9x
Principle: Normalising a fixed-point number (shifting left until the top bit is set and counting the shifts) is a data-dependent loop. A single count-leading-zeros instruction gives the shift amount directly, so the operation becomes constant-time and branch-free.
Why it is faster: —
Applies elsewhere: software floating point on MCUs, finding the highest set bit for power-of-two rounding, log2 bucketing in histograms and allocators, bit-scan in a priority bitmap scheduler.
Watch out: —
Seen in Snes9x: In dsp1.cpp, `DSP1_Inverse`, `DSP1_Normalize` and `DSP1_NormalizeDouble` used `while (Coefficient < 0x4000) { Coefficient <<= 1; Exponent--; }`-style loops. Under GCC they now compute `__builtin_clz(n) - (8 * sizeof(int) - 15)`, and the loop is kept as the fallback for other compilers. Negative inputs are handled by complementing first (`m < 0 ? ~m : m`). 104769ce 2022-02 “dsp1: Use clz CPU instruction when normalising fixed-point numbers”

### snes9x-065 — Make local lookup tables static const
Lens: compiler-build · Source: Snes9x
Principle: A `const` array declared inside a function without `static` has automatic storage. The compiler may rebuild it on the stack on every call. Adding `static` places it once in read-only data, and the per-call initialisation copy goes away.
Why it is faster: —
Applies elsewhere: CRC or tables inside C helpers, lookup arrays declared inside a JS function body (hoist them to module scope), constant maps rebuilt inside request handlers.
Watch out: —
Seen in Snes9x: `MaxAZS_Exp[16]` in `DSP1_Parameter`, `HiAdd`/`LoAdd` in `DSP3_OP1E_D1`, and `div_lut[64]` and `OP0A_Values` in dsp4.cpp all became `static const`. In fxemu.cpp, `avHeight`/`avMult` also gained `const`. 423361e4 2022-02 “dsp: Mark function-scope const arrays as static” · 242238a6 2022-02 “fx: Optimise copying registers on LE platforms”

### snes9x-066 — Load and store whole words instead of assembling bytes
Lens: memory-layout · Source: Snes9x
Principle: Building a 16-bit value with `p[0] | (p[1] << 8)` costs two loads, a shift and an OR. On a little-endian host that is just one 16-bit load. Wrap the access in a macro that becomes a single word load or store where the byte order matches and falls back to byte assembly elsewhere.
Why it is faster: —
Applies elsewhere: parsing binary network or file headers, reading registers on a memory-mapped peripheral, `DataView.getUint16(o, true)` in place of byte arithmetic in JS.
Watch out: —
Seen in Snes9x: `fx_readRegisterSpace`/`fx_writeRegisterSpace` copied the 16 SuperFX registers, plus SFR and CBR, one byte at a time. They now use `GSU.avReg[i] = READ_WORD(p)` and `WRITE_WORD(p, ...)`. `DSP1SetByte` made the same change for every command's parameters and outputs, for example `DSP1.Op00Multiplicand = (int16) READ_WORD(&DSP1.parameters[0])`. That commit also fixed a long-standing Op11 bug. 242238a6 2022-02 “fx: Optimise copying registers on LE platforms” · 96059dd4 2022-02 “dsp1: Optimise copying registers on LE platforms”

### snes9x-067 — Read only the section of a file that you need
Lens: laziness · Source: Snes9x
Principle: If a container format stores tagged blocks, a viewer that needs one block can skip the others by reading their length headers and seeking past them. A full load would also decode, allocate and apply every block.
Why it is faster: —
Applies elsewhere: reading only EXIF thumbnails from photos, reading only the ZIP central directory, reading only the moov/thumbnail atoms of an MP4, fetching only a Parquet footer.
Watch out: —
Seen in Snes9x: The Windows save/load preview dialog used to call `FreezeUnfreezeSlot(slot, FALSE)`, which restored the entire emulator state just to get its screenshot. The new `S9xUnfreezeScreenshotFromStream` in snapshot.cpp checks the header, then calls `SkipBlockWithName` for "CPU", "REG", "PPU", "DMA", "VRA", "RAM", "SRA" and the rest until it reaches the screenshot block. The dialog calls `UnfreezeScreenshotSlot` instead. 3c2ef2aa 2019-12 “Add function to load only screenshot from snapshot file” · 04a15d9a 2019-12 “win32: only load screenshots in preview dialogs”

### snes9x-068 — Grow buffers to the actual size, not a worst-case guess
Lens: allocation · Source: Snes9x
Principle: Pre-allocating a huge "safe" buffer for input of unknown length wastes memory and page-faults in pages that are mostly unused. Reading in small chunks into a growable container allocates only what the input needs. It also removes the arbitrary size cap and the manual frees on every error path.
Why it is faster: —
Applies elsewhere: reading an HTTP body of unknown length, slurping stdin, decompressing a stream with no declared size.
Watch out: —
Seen in Snes9x: The UPS patcher in memmap.cpp did `new uint8[8 * 1024 * 1024]` for every patch it tried. It now calls `ReadStreamUntilEOF`, which fills a 4 KB stack buffer and appends it to a `std::vector`. The eight `delete[] data; return false;` exits became plain returns. 08bf7a86 2023-02 “Don't allocate 8MB for patching.”

### snes9x-069 — Keep long-lived buffers alive across UI events
Lens: allocation · Source: Snes9x
Principle: Tearing down and rebuilding large buffers or GPU resources whenever a dialog opens or a setting changes costs time and can stall the frame. Keep them alive, and recreate only the part whose inputs actually changed.
Why it is faster: —
Applies elsewhere: rebuilding only a font atlas instead of the whole GL context when font size changes, keeping canvas buffers across modal dialogs, reallocating a texture only when its size changes.
Watch out: —
Seen in Snes9x: In win32/wsnes9x.cpp, opening some dialogs called `S9xGraphicsDeinit(); S9xGraphicsInit();`. Those calls were removed. In the Qt port, changing the OSD message mode or font size called `recreateCanvas()`, which rebuilt the whole OpenGL/Vulkan canvas. It now calls the new `recreateUIAssets()`, which only reinitialises ImGui with the new font. The Vulkan `Texture::from_buffer` only calls `destroy()`/`create()` when `width`/`height` differ from the existing image. 6e738950 2023-11 “win32: Don't realloc graphics buffers when dialog shown.” · 8bb1e774 2023-07 “UI element resize without canvas recreate...” · 259dfd07 2022-06 “Vulkan support.”

### snes9x-070 — Load independent assets in parallel
Lens: concurrency · Source: Snes9x · Also: H2O, Kafka
Principle: If a pipeline needs N files that do not depend on each other, start every read or compile at once and join on the results. Start-up time then approaches the slowest single item, not the sum of all of them.
Why it is faster: —
Applies elsewhere: `Promise.all` over fetches, compiling shader variants on a thread pool at game start, parallel module loading in a bundler.
Watch out: —
Seen in Snes9x: `SlangPreset::load_preset_file` in vulkan/slang_preset.cpp called `shader.load_file()` on each pass one after another. It now starts `std::async(std::launch::async, ...)` for each pass and ANDs the `future.get()` results together. 8e38e14e 2023-02 “Vulkan/Slang: Load shader files asynchronously.” • Also seen in H2O (h2o-100): 59df385f27 2017-11-21 "Parallel http join" in the mruby layer. • d99c4be20b 2017-11-30 A channel class and a task method for parallel processing in mruby. • Also seen in Kafka (kafka-109): e1ff387605 2025-07-14 `DelayedRemoteFetch` launches remote reads for all tiered partitions of a fetch and completes when all are done, cancelling the rest on expiry. • be368b3b6a 2025-12-03 `onPartitionsDeleted` collects offset-deletion and share-state futures into one list and waits once instead of two sequential `allOf().get()`.

### snes9x-071 — Move blocking output onto its own thread
Lens: concurrency · Source: Snes9x
Principle: A blocking `write()` to a device stalls the producer until the device drains. Put the device writes on a dedicated consumer thread that waits on a condition variable. The producer then only appends to a buffer under a short lock and never blocks on I/O.
Why it is faster: —
Applies elsewhere: async loggers, a serial or UART TX thread on embedded Linux, a network sender thread that decouples a game loop from socket back-pressure.
Watch out: —
Seen in Snes9x: The Unix OSS sound path in unix/unix.cpp gained `S9xAudioOutput`. When threaded, `Write()` appends to `m_Buffering` under a mutex and signals `m_hasBuffer`. The thread swaps the buffer out and calls `WriteImpl`, which loops on `write(m_FD, ...)`. `GetFreeBufferSize()` still feeds dynamic rate control. e2c1abb7 2020-08 “audio output thread”

### snes9x-072 — Run the hot loop on a dedicated thread and send commands to it
Lens: concurrency · Source: Snes9x
Principle: If a compute loop runs from a UI toolkit's timer or idle callback, every UI event can delay it. Put the loop on its own thread. UI actions then become closures queued onto that thread, so state is touched from one thread only and no lock is needed on the hot path.
Why it is faster: —
Applies elsewhere: a Web Worker running a simulation that receives `postMessage` commands, a game logic thread with a command queue, an actor or event-loop design in servers.
Watch out: —
Seen in Snes9x: The Qt frontend drove emulation from a zero-interval `QTimer` (`idleLoop`). `EmuApplication` now creates an `EmuThread` running `mainLoop()`. Input and settings go through `emu_thread->runOnThread([&, b, active] { core->reportBinding(b, active); })`, and `suspendThread`/`unsuspendThread` provide a counted pause. Later, paint events on the OpenGL canvas are forwarded to the thread that owns the context with `BlockingQueuedConnection`. a438d3fa 2023-07 “Threading.” · e02add8f 2026-09 “qt/opengl: Run drawing for Qt paintEvent on OpenGL canvas on OpenGL thread.”

### snes9x-073 — Remove a coarse lock that serialises producer and presenter
Lens: concurrency · Source: Snes9x
Principle: A single mutex held around both "produce the whole frame" and "draw the frame" makes the UI thread wait for an entire emulated frame, and the reverse. If the presenter no longer touches the producer's state, drop the lock so each side runs at its own pace.
Why it is faster: —
Applies elsewhere: removing a global lock between a render thread and a simulation thread once they exchange data through a queue or a double buffer.
Watch out: —
Seen in Snes9x: In macosx/mac-os.mm, `EmulationLoop` wrapped `S9xMainLoop()` in `pthread_mutex_lock(&renderLock)`, and `drawRect:` took the same lock to call `S9xPutImage`. With the Metal renderer, `drawRect:` no longer draws the image, and `renderLock` was deleted. b12ca3e9 2019-12 “Mac: Remove render lock and fix freeze/defrost screen”

### snes9x-074 — Single-producer/single-consumer ring buffer with separate indices
Lens: concurrency · Source: Snes9x
Principle: A ring buffer that keeps a shared `size` field needs both threads to write that field, which is a data race. If the producer owns only `end` and the consumer owns only `start`, and fullness is derived from the two (keeping one slot free), each index has a single writer. Acquire/release atomics are then enough, with no mutex on the audio callback.
Why it is faster: —
Applies elsewhere: audio callbacks (Web Audio worklets, CoreAudio), lock-free queues between an ISR and the main loop, log ring buffers in kernels.
Watch out: —
Seen in Snes9x: apu/resampler.h replaced `int size` with `volatile int end` beside `start`, plus `space_filled()` computed as `end - start` with wrap. The cubeb callback zero-fills when samples are short. Later, common/audio/atomic_ring_buffer.hpp introduced `std::atomic_int start, end`, with `end.store(..., std::memory_order_release)` after the two-part `memcpy`, for the cubeb and SDL3 drivers. c13e4d83 2023-06 “Try to fix resampler to be thread-safe.” · 991380a2 2026-03 “Qt: Use safer ring buffer for sound drivers.”

### snes9x-075 — Batch output once per frame instead of on many small callbacks
Lens: io · Source: Snes9x
Principle: Pushing data downstream every time a small chunk is ready costs a callback for each chunk. Collect everything produced during a unit of work and hand it off in one call at the end, using a reusable buffer that grows to the high-water mark.
Why it is faster: —
Applies elsewhere: coalescing small socket writes (corking), flushing DOM updates once per animation frame, batched telemetry.
Watch out: —
Seen in Snes9x: The libretro core registered `S9xAudioCallback` via `S9xSetSamplesAvailableCallback`, which fired about eight times per frame with a 256-sample static buffer. Now no callback is registered. `retro_run` calls `S9xGetSampleCount()` once, resizes a `static std::vector<int16_t>` if needed, and calls `audio_batch_cb` once. A later commit moved this inside the frame callback. 0a206817 2022-03 “libretro: Send all samples at frame end.” · 6628042f 2022-04 “libretro: Get audio samples during frame callback.”

### snes9x-076 — Sleep coarsely, then spin for the last stretch
Lens: platform · Source: Snes9x
Principle: OS sleeps overshoot by about a millisecond or more, and a pure busy-wait burns a core. Sleep on a high-resolution waitable timer until shortly before the deadline, then yield-spin on a monotonic clock for the remaining time. The deadline is met precisely and the CPU cost stays low.
Why it is faster: —
Applies elsewhere: frame pacing in games and video players, real-time audio schedulers, precise rate limiters, test harnesses that need µs timing on Windows.
Watch out: —
Seen in Snes9x: The Windows throttle first went from timer messages to a QueryPerformanceCounter loop (`ThrottleTimer`). Then `CreateWaitableTimer`/`SetWaitableTimer` was added for long waits and `Sleep(0)` for short ones, and the split was tuned. The shared `Throttle::wait_for_frame` in std_chrono_throttle.cpp does the same: if more than 2 ms remain, it waits on a waitable timer for `time_to_wait - 2000` µs (`sleep_for(time_to_wait - 1ms)` on other platforms), then runs `while (remaining() > 0) std::this_thread::yield();`. e9233823 2023-02 “win32: Replace timer-based throttling.” · daf4a60b 2023-02 “win32: Fix CPU usage with new throttle.” · 1907b9e2 2023-02 “win32: Tweak busy loop/sleep balance a bit.” · faaf900e 2023-07 “More precise std::chrono throttle implementation.” · f1837aa0 2023-07 “Proper sleep implementation for Windows.”

### snes9x-077 — Pace with an accumulating deadline, not "now + period"
Lens: algorithms · Source: Snes9x
Principle: If each wait is computed from the current time, the error of every wait adds up. Advance a target timestamp by a fixed period each frame (`then += period`) and wait until it arrives. Errors no longer accumulate, catching up after a short stall is automatic, and the target is re-based only after a long stall.
Why it is faster: —
Applies elsewhere: fixed-timestep game loops, cron-like schedulers, sampling loops in data acquisition firmware.
Watch out: —
Seen in Snes9x: `ThrottleTimer` runs `while ((PCEnd - PCStart) >= PCFrameTime) { ... PCStart += PCFrameTime; }` and flags a frame adjustment once the backlog reaches two frames. The `Throttle` struct has `advance()` (`then += frame_duration_us`) and `reset()`. `wait_for_frame` resets when it is more than 10% of a frame late. e9233823 2023-02 “win32: Replace timer-based throttling.” · b82edaac 2023-02 “vulkan: Add a throttler before present for precise frame rates.” · f1837aa0 2023-07 “Proper sleep implementation for Windows.”

### snes9x-078 — Skip the costly output stage when running behind
Lens: laziness · Source: Snes9x
Principle: When a loop runs behind its deadline, skip the optional stage (rendering) for the next iteration but still do the required work. Measure how many periods late you are: skip at one or more, and reset the clock at two or more.
Why it is faster: —
Applies elsewhere: video players dropping frames to keep A/V sync, adaptive rendering in a canvas animation, dropping telemetry samples under load.
Watch out: —
Seen in Snes9x: `Throttle::get_late_frames()` returns `remaining() / -frame_duration_us`. In gtk_s9x.cpp, `S9xSyncSpeed` in `THROTTLE_TIMER_FRAMESKIP` mode sets `IPPU.RenderThisFrame = false` when at least one frame late, calling `advance()` below two frames and `reset()` at two or more. Qt's `EmuCanvas` skips `draw` the same way. a23d76b5 2025-09 “qt,gtk: Fix timer with frameskip to actually frameskip.”

### snes9x-079 — Don't throttle when the user asked for maximum speed
Lens: laziness · Source: Snes9x
Principle: A pacing step that sleeps to reach a target rate should be bypassed entirely in modes that are meant to run as fast as possible. A rate limit applied by accident caps throughput.
Why it is faster: —
Applies elsewhere: disabling rate limiting for bulk backfills, turning off vsync or frame caps in benchmark mode.
Watch out: —
Seen in Snes9x: In win32_display.cpp, `WinThrottleFramerate` returned early only when `Settings.SkipFrames != AUTO_FRAMERATE`. It now also returns early when `Settings.TurboMode` is set. The earlier QPC throttle already treated turbo, frame advance and netplay as "run immediately". f62eb40a 2024-05 “win32: skip framerate throttling in turbo mode (#853)” · e9233823 2023-02 “win32: Replace timer-based throttling.”

### snes9x-080 — Throttle right before present, inside the output driver
Lens: gpu-rendering · Source: Snes9x
Principle: Frame pacing done far from the swap (in the main loop) gets jitter from all the work in between. Put the wait immediately before the present call in each display backend, so the time between frames as displayed is what gets regulated.
Why it is faster: —
Applies elsewhere: frame limiters that wait just before `SwapBuffers`/`Present` in game engines, pacing a video encoder right before it pushes to the network.
Watch out: —
Seen in Snes9x: A `Throttle` was added to `Vulkan::Swapchain` (`set_max_frame_rate(60.0988...)`). On win32, `WinThrottleFramerate()` moved into `CDirect3D`, `COpenGL` and `CVulkan` just before they present. Interlaced alternate frames got the same call. b82edaac 2023-02 “vulkan: Add a throttler before present for precise frame rates.” · 803125ff 2023-02 “win32: Move frame throttle into drivers for precision.”

### snes9x-081 — Bound frames in flight separately from swapchain size
Lens: gpu-rendering · Source: Snes9x
Principle: Allocating per-frame CPU resources (command buffers, fences, descriptor sets) for every swapchain image ties latency to however many images the driver hands back. Use a fixed small number of frames in flight, indexed round-robin, and size the swapchain images separately. Latency is capped and fewer resources are allocated.
Why it is faster: —
Applies elsewhere: any Vulkan/D3D12/Metal renderer ("MAX_FRAMES_IN_FLIGHT = 2–3"), triple-buffered upload rings for uniform data.
Watch out: —
Seen in Snes9x: vulkan_swapchain.cpp split `num_frames` into `num_swapchain_images` and `max_latency`. Command buffers and `frames` are sized by `max_latency`, and `current_frame = (current_frame + 1) % max_latency`. `ShaderChain` sizes its descriptor pool with `queue_size = 3` and rotates its own `current_frame_index`. `SlangPipeline::frame[4]` became `frame[3]`. 23cf7ec0 2023-01 “Restrict max latency to 3 frames.”

### snes9x-082 — Wait on a specific fence instead of idling the whole device
Lens: gpu-rendering · Source: Snes9x
Principle: A device-wide idle wait drains every queue and stalls the pipeline. Wait only on the fence for the resource you are about to reuse, so unrelated work keeps flowing. If you can toggle a setting on a live object, you also avoid recreating it.
Why it is faster: —
Applies elsewhere: per-resource fences in GPU upload code, waiting on one future instead of a thread-pool barrier, per-row locks instead of a table lock.
Watch out: —
Seen in Snes9x: With `VK_EXT_swapchain_maintenance1`, the swapchain is created with a `SwapchainPresentModesCreateInfoEXT` list. `set_vsync` then changes the present mode per present instead of recreating the swapchain, and `device.waitIdle()` before recreation became `wait_on_frames()`. In Qt's "reduce input lag" path, `context->wait_idle()` became `swapchain->wait_on_frames()`. A follow-up attached the present fence to the swapchain image rather than to the frame slot, so `begin_frame` waits only on `frame.fence`. 5949bbab 2024-06 “Vulkan: Utilize VK_EXT_swapchain_maintenance1.” · 2e25b70c 2024-06 “Vulkan: Associate new fence with swapchain image, not frame resources.”

### snes9x-083 — Choose the presentation mode for the latency goal
Lens: gpu-rendering · Source: Snes9x · Also: Flycast
Principle: FIFO blocks the producer on vblank. Mailbox replaces the queued image without blocking. Immediate presents with tearing. Pick per mode: FIFO for vsync, immediate or else mailbox when vsync is off, and mailbox where FIFO is known to stall (for example, occluded windows on some compositors).
Why it is faster: —
Applies elsewhere: the vsync options in any game settings menu, video players choosing a presentation mode, low-latency remote desktop clients.
Watch out: —
Seen in Snes9x: vulkan_swapchain.cpp went from `vsync ? eFifo : eImmediate` to using a `tearing_present_mode` that prefers mailbox. A later commit limited mailbox and immediate to the `!vsync` branch. On Wayland, mailbox is used because "FIFO can stall when other windows obscure the main window". 58bdb3ef 2023-10 “Vulkan: Use mailbox when available...” · 99990af3 2024-06 “Vulkan: Don't use mailbox for vsync case.” · 191cdf46 2025-01 “vulkan/wayland: Use mailbox present mode.” • Also seen in Flycast (flycast-199): DX11 checks `DXGI_FEATURE_PRESENT_ALLOW_TEARING`, creates the swap chain with `DXGI_SWAP_CHAIN_FLAG_ALLOW_TEARING`, and presents with `DXGI_PRESENT_ALLOW_TEARING` when vsync is off. Vulkan falls back from FIFO to `vk::PresentModeKHR::eMailbox` when immediate isn't supported, which fixed fast-forward on mobile. b7a2e605f 2024-05 “dx11: use ALLOW_TEARING flag to disable vsync if available” · 75feafbbf 2026-02 “vk: use mailbox present mode if immediate mode isn't available”

### snes9x-084 — Wait for actual presentation to keep CPU and display in lockstep
Lens: gpu-rendering · Source: Snes9x
Principle: Fences only report when the GPU has finished. The image may still sit in a queue before it reaches the screen. Present-wait or anti-lag driver hooks let the CPU block until a given present has reached the display, or tell the driver where input sampling and presentation happen. The CPU then starts the next frame later, with fresher input.
Why it is faster: —
Applies elsewhere: NVIDIA Reflex and AMD Anti-Lag in games, `DXGI` frame-latency waitable objects, `requestAnimationFrame`-timed input sampling.
Watch out: —
Seen in Snes9x: The Vulkan context enables `VK_KHR_present_id`/`VK_KHR_present_wait` when they are available. Each present carries `presentation_id++`, and `Swapchain::present_wait()` calls `device.waitForPresentKHR(..., presentation_id, ...)`, except on Wayland. The extension was removed once and re-added. Later, `VK_AMD_anti_lag` was queried through a `getFeatures2` chain, and `Context::update_anti_lag_stage` calls `device.antiLagUpdateAMD` when "Reduce Input Lag" is on. c8895c8c 2024-09 “vulkan: Add support for VK_KHR_present_wait” · 66373834 2025-01 “qt,gtk/vulkan: Add VK_EXT_present_wait support again.” · 2e3fce37 2026-05 “vulkan: Add support for VK_AMD_anti_lag.”

### snes9x-085 — Make pipeline barriers as narrow as possible
Lens: gpu-rendering · Source: Snes9x
Principle: A barrier that names `ALL_GRAPHICS` stages and broad access masks forces the GPU to drain more work than it needs to. Name the exact producer and consumer stages (transfer to vertex/fragment, fragment to bottom-of-pipe) and the exact access types, so unrelated stages can overlap.
Why it is faster: —
Applies elsewhere: Vulkan/D3D12 resource transitions, memory fences in lock-free code (use acquire/release, not seq_cst, where enough).
Watch out: —
Seen in Snes9x: In vulkan_pipeline_image.cpp and vulkan_shader_chain.cpp, layout transitions with `eAllGraphics` and `eColorAttachmentWrite`/`eMemoryRead` were replaced with `eVertexShader | eFragmentShader`, `eTransfer` and `eBottomOfPipe`, using `eShaderRead`, `eTransferRead` or `eNone` as appropriate. The single barrier batch was split into two barriers at the correct stages. 39c0f841 2024-10 “Vulkan: Use recommended access flags for image layout transitions.”

### snes9x-086 — Pick the GPU's native pixel layout
Lens: gpu-rendering · Source: Snes9x
Principle: Surfaces and swapchains on most desktop GPUs are natively BGRA. Choosing RGBA can force a swizzle, or a conversion copy, on every blit or present. Prefer the native format when you have a choice.
Why it is faster: —
Applies elsewhere: `BGRA` textures for video frames on Windows, choosing `kCVPixelFormatType_32BGRA` on Apple platforms, canvas `ImageData` layout choices.
Watch out: —
Seen in Snes9x: In vulkan_slang_pipeline.cpp, the fallback render-target format for shader passes changed from `VK_FORMAT_R8G8B8A8_UNORM`/`_SRGB` to `VK_FORMAT_B8G8R8A8_UNORM`/`_SRGB`. 56a6396c 2023-02 “Favor BGRA over RGBA.”

### snes9x-087 — Size textures to the image, not to a padded power of two
Lens: gpu-rendering · Source: Snes9x
Principle: Allocating a fixed 1024×1024 texture for a 256×224 image wastes memory and makes texture-coordinate maths depend on padding. Any modern GPU supports non-power-of-two textures, so allocate the exact size.
Why it is faster: —
Applies elsewhere: WebGL2 textures for video or canvas frames, sprite uploads in 2D engines.
Watch out: —
Seen in Snes9x: gtk_display_driver_opengl.cpp removed the `npot` flag and the 1024-wide fallback (`coords[...] = width / 1024.0f`). It now always calls `glTexImage2D(..., width, height, ...)` at the image size, with unit texture coordinates. 1b10ca7a 2023-03 “Gtk/OpenGL: Remove npot variable.”

### snes9x-088 — Declare the real row alignment of uploaded pixel data
Lens: gpu-rendering · Source: Snes9x
Principle: OpenGL assumes 4-byte row alignment by default. With 16-bit pixels and odd widths, that assumption is wrong. The driver then either mis-reads the data or falls back to a slower repacking path. Set `GL_UNPACK_ALIGNMENT` to the true alignment before uploading.
Why it is faster: —
Applies elsewhere: uploading RGB24 or single-channel images in WebGL (`gl.pixelStorei(gl.UNPACK_ALIGNMENT, 1)`), video frame upload in media players.
Watch out: —
Seen in Snes9x: The GTK and Qt OpenGL drivers call `glPixelStorei(GL_UNPACK_ALIGNMENT, 1)` before the texture uploads. 01e408e4 2023-10 “(Qt,Gtk)/OpenGL: Specify unpack alignment.”

### snes9x-089 — Copy into a persistent output image instead of wrapping foreign memory
Lens: memory-layout · Source: Snes9x
Principle: Wrapping a producer's buffer in a toolkit image object each frame couples their lifetimes. It can also force the toolkit to deep-copy, or to read memory that the producer is overwriting. Keep one reusable image, reallocate it only when size or format changes, and copy rows into it with `memcpy`.
Why it is faster: —
Applies elsewhere: reusing a `Bitmap` or `ImageData` across frames, a staging buffer for camera frames.
Watch out: —
Seen in Snes9x: In qt/src/EmuCanvasQt.cpp, a new `QImage` was built over `output_data.buffer` each frame. Now `qimage` is recreated only when width, height or format differ. Each row is copied into `qimage.bits()` with `memcpy`, respecting the source `bytes_per_line`. 03c9a10c 2023-08 “Qt: Use copy for software output path.”

### snes9x-090 — Use a vectorised library for pixel-format conversion
Lens: simd · Source: Snes9x
Principle: Colour-space conversions such as RGB565 to planar YUV are simple per-pixel arithmetic over large buffers. A SIMD-optimised library does them several times faster than a scalar loop. Detect it at build time and keep the scalar path as a fallback.
Why it is faster: —
Applies elsewhere: libyuv in WebRTC and camera pipelines, sharp/libvips for image resizing, using ffmpeg's swscale instead of hand-written loops.
Watch out: —
Seen in Snes9x: The X11 port added the Xvideo I420 format. If configure finds `libyuv.h` (`-DUSE_LIBYUV`), x11.cpp converts each frame with `libyuv::RGB565ToI420(...)`. Otherwise it uses `rgb565ToRgb24` plus `rgb24ToI420`, which use integer coefficients (`(66*r + 129*g + 25*b) >> 8`). bd996da4 2021-11 “Add I420 Xvideo image format support, with both libyuv hardware acceleration and software conversion support.”

### snes9x-091 — Blend packed colour channels with one multiply (SWAR)
Lens: simd · Source: Snes9x
Principle: In a packed pixel format, mask the channels apart so that each has headroom bits above it. One integer multiply-add then weights several channels at once, and masking again repacks them. In RGB565, red and blue are interpolated together (`& 0xf81f`) and green separately.
Why it is faster: —
Applies elsewhere: alpha blending in software rasterisers on MCUs with RGB565 LCDs, fast cross-fades in 2D engines, SWAR byte arithmetic in parsers.
Watch out: —
Seen in Snes9x: The hqx filter (filter/hq2x.cpp) was replaced with an adaptation of FFmpeg's version, changed to RGB565. Its `interp_2px`/`interp_3px` compute `((c1 & 0xf81f) * w1 + (c2 & 0xf81f) * w2) >> s` for red and blue in a single operation. A precomputed 64K-entry `yuvtable` drives `yuv_diff`. 046f348d 2023-03 “Update hqxx filters to adapted version from FFmpeg.”

### snes9x-092 — Replace a huge unrolled switch with compact rules, then force inlining
Lens: compiler-build · Source: Snes9x
Principle: Thousands of lines of generated case code blow out the instruction cache. A short set of mask-and-compare rules is much smaller, and forcing the compiler to inline the small helpers into the main loop recovers specialisation (constants folded per call site) without code bloat.
Why it is faster: —
Applies elsewhere: replacing generated state-machine tables with bitmask tests, and adding `__attribute__((always_inline))` or `[[gnu::always_inline]]` to tiny hot helpers that the compiler declined to inline.
Watch out: —
Seen in Snes9x: hq2x.cpp shrank from about 12,200 lines to about 460. It now classifies the 3×3 neighbourhood into a bitmask `k` and tests patterns with `P(m, r) ((k_shuffled & (m)) == (r))`. A follow-up changed `rgb2yuv`, `yuv_diff`, `interp_2px`/`interp_3px`, `hq2x_interp_1x1` and `hqx_filter` from `static inline` to `static alwaysinline`. 046f348d 2023-03 “Update hqxx filters to adapted version from FFmpeg.” · fea60278 2023-03 “Use alwaysinline with hq2x.”

### snes9x-093 — Precompute filter weights per output phase in fixed point
Lens: lookup-tables · Source: Snes9x
Principle: For integer-ratio upscaling, each output pixel lands at one of only scale² fractional positions. Compute the kernel weights, and the reciprocal of their sum, for each position once. The inner loop then does integer multiply-adds and a multiply-shift instead of float kernel evaluation and division. Gamma conversion becomes a table lookup.
Why it is faster: —
Applies elsewhere: image resizers (libvips, Pillow) precompute their filter taps, polyphase audio resamplers, fixed-point DSP on MCUs with no FPU.
Watch out: —
Seen in Snes9x: The new interpolation filters built `weights[scale][scale][4][4]` (bicubic) and `w_table[2][scale*scale*k*k]` (lanczos) from fixed-point `hermite_weight`/`lanczos2_weight_fp`, with `inv_weight_sums = (65536 + sum/2) / sum` for normalisation. `gamma_r_encode[32]`, `gamma_g_encode[64]` and `gamma_decode[256]` are built once. In sharpbilinear, the float `smoothstep` per pixel became `static constexpr uint16_t W[4] = {0, 40, 128, 216}` in 8.8 fixed point. 9aa08f4b 2025-07 “Interpolation filters” · 15d86b61 2025-08 “Update filter_sharpbilinear_flexible.cpp”

### snes9x-094 — Loop over source blocks and reuse separable partial results
Lens: algorithms · Source: Snes9x
Principle: When upscaling by an integer factor, iterate over source texels rather than destination pixels. Fetch and decode each 2×2 neighbourhood once per output block. Do the horizontal blend once per output column and reuse it for every output row of that block, which is a separable filter.
Why it is faster: —
Applies elsewhere: separable Gaussian blur, box-filter mipmap generation, 2D convolution in image pipelines and ML preprocessing.
Watch out: —
Seen in Snes9x: `ApplySharpBilinear4x` looped `for y in dst_height, for x in dst_width`. Each output pixel computed float source coordinates, clamped them, unpacked four gamma-encoded neighbours and evaluated smoothstep. It now loops `for sy, for sx` and unpacks the 2×2 block once. It fills `rtop[dx]`/`rbot[dx]` once for the four columns, then blends vertically with `(rtop[dx]*iwy + rbot[dx]*wy + 128) >> 8` for each of the four rows. 15d86b61 2025-08 “Update filter_sharpbilinear_flexible.cpp”

### snes9x-095 — Drop to a low-frequency timer while idle
Lens: platform · Source: Snes9x
Principle: A main loop that runs as an "always-idle" callback spins a core even when nothing is happening. When paused, switch to a timer callback every few milliseconds. Switch back to the always-run callback only when real work resumes.
Why it is faster: —
Applies elsewhere: lowering `requestAnimationFrame` work when a tab is idle, adaptive polling in daemons, tickless idle in embedded RTOSes.
Watch out: —
Seen in Snes9x: In gtk/src/gtk_s9x.cpp, `idle_func` calls `switch_to_low_performance_idle_loop()` when `Settings.Paused`. That stops sound and registers `Glib::signal_timeout().connect(idle_func, timeout)` with an 8 ms timeout (2 ms for animated backgrounds). `switch_to_high_performance_idle_loop()` restores `Glib::signal_idle()` on unpause. 033a23fe 2025-06 “Gtk: Rework base idle and timeouts.”

### snes9x-096 — Ask the audio server for small, early refills
Lens: io · Source: Snes9x
Principle: Letting the audio server pick its default request size can produce large, late callbacks, which cost latency and cause underrun spikes. Request a small minimum refill and early notifications. Measure free space after writing, not before, and don't sleep in the producer.
Why it is faster: —
Applies elsewhere: configuring the period size for ALSA or JACK, Web Audio `latencyHint`, the buffer size in real-time voice apps.
Watch out: —
Seen in Snes9x: The PulseAudio driver changed `buffer_attr.minreq` from -1 to `pa_usec_to_bytes(3000, &ss)`, used `PA_STREAM_EARLY_REQUESTS` instead of `PA_STREAM_ADJUST_LATENCY`, and dropped a `usleep` in the write path. The ALSA and PortAudio drivers now query `snd_pcm_avail`/`Pa_GetStreamWriteAvailable` after writing to feed dynamic rate control. e55b1331 2023-04 “Gtk: Tweak sound drivers for better performance.”

### snes9x-097 — Poll input at a higher rate than you consume it
Lens: io · Source: Snes9x
Principle: If input is sampled on a slow timer, a press can wait up to one full interval before the consumer sees it. Polling more often than the frame rate cuts that worst-case latency, and polling itself costs very little.
Why it is faster: —
Applies elsewhere: gamepad polling in browser games, 1 kHz USB mouse polling, sensor sampling ahead of a control loop.
Watch out: —
Seen in Snes9x: In qt/src/EmuApplication.cpp, `poll_input_timer->setInterval(4)` became `setInterval(1)`. 2462dc98 2026-06 “Qt: Poll joysticks more often.”

### snes9x-098 — Look up once and reuse the result
Lens: memory-layout · Source: Snes9x
Principle: Calling `map.find(key)` several times on the same path repeats the hash or tree walk each time. Look up once and keep the result in a local. When writing, insert directly instead of finding and then assigning.
Why it is faster: —
Applies elsewhere: repeated `dict[key]` access in Python hot loops, repeated `Map.get` in JS, repeated DOM queries.
Watch out: —
Seen in Snes9x: In controls.cpp, `S9xReportPointer` and `S9xReportAxis` called `keymap.find(id)->second` three or four times each. They now do `auto mapping = keymap.find(id)->second;` once. The mapping setter uses `keymap.insert({id, mapping})`. eb949b47 2026-09 “controls: Reduce number of map lookups.”

### snes9x-099 — Swap the whole dispatch table when a mode bit changes
Lens: branching-fast-paths · Source: Snes9x
Principle: If a few rarely-changing mode flags alter what many handlers do, build one specialized handler table per mode combination and switch a single table pointer when a flag changes. Each handler then runs straight-line code with no per-call test of the mode.
Why it is faster: —
Applies elsewhere: a parser that swaps its token-handler table when it enters a string or comment state; a codec with separate inner loops for 8- and 16-bit samples, picked once per stream.
Watch out: —
Seen in Snes9x: The 65c816 has emulation mode plus 8/16-bit accumulator (M) and index (X) widths. `cpuops.cpp` defines five 256-entry tables (`S9xOpcodesE1`, `M1X1`, `M1X0`, `M0X1`, `M0X0`), and `S9xFixCycles()` in `cpuexec.h` repoints `ICPU.S9xOpcodes` and `ICPU.S9xOpLengths` only when P changes. Handlers like `Op69M1` just call `ADC(Immediate8(READ))`, with no width check at run time. current code: cpuops.cpp, cpuexec.h (S9xFixCycles), globals.cpp (S9xOpLengths*)

### snes9x-100 — Generate handler variants from one macro body
Lens: compiler-build · Source: Snes9x
Principle: When dozens of handlers differ only in an addressing function, operand width and operation, write the body once as a macro (or template) and pass those three parts in. Every variant is fully inlined and specialized, and the source does not become thousands of hand-copied functions.
Why it is faster: —
Applies elsewhere: generating typed array kernels (`add_i8`, `add_f32`…) from one template; SQL engines that stamp out a comparison function per column type.
Watch out: —
Seen in Snes9x: `cpumacro.h` defines `rOP8`/`rOP16`/`wOPC`/`mOPM`/`bOP`. A line like `rOP8 (75E1, DirectIndexedXE1, WRAP_BANK, ADC)` expands to a `static void Op75E1()` that inlines the addressing mode, the read and the ALU op. Addressing modes in `cpuaddr.h` are `static inline` and take an `AccessMode` constant, so `if (a & READ)` folds away at compile time. current code: cpumacro.h, cpuaddr.h, cpuops.cpp

### snes9x-101 — Pair every fast handler with a general fallback
Lens: branching-fast-paths · Source: Snes9x
Principle: Write each operation twice: a fast version that assumes the common preconditions hold, and a slow version that checks everything. The dispatcher picks the table once per step based on a single cheap test, so the fast path never pays for rare cases.
Why it is faster: —
Applies elsewhere: `memcpy` with an aligned bulk loop and a byte-wise tail; a JSON decoder with an ASCII-only fast path that falls back to full UTF-8 handling.
Watch out: —
Seen in Snes9x: Fast handlers fetch operands straight from `CPU.PCBase[Registers.PCw]` (`Immediate8`). `*Slow` handlers go through `S9xGetByte` and test the M/X flags at run time. `S9xMainLoop` uses `S9xOpcodesSlow` whenever `CPU.PCBase` is NULL (code running from I/O or a special chip) or the instruction could cross a 4 KB map block. current code: cpuexec.cpp (S9xMainLoop), cpuaddr.h (Immediate8 vs Immediate8Slow), cpuops.cpp (S9xOpcodesSlow)

### snes9x-102 — Cache a direct pointer to the current code page
Lens: memory-layout · Source: Snes9x
Principle: If the program counter almost always stays inside one memory region, resolve that region to a raw host pointer once and index it directly. Re-resolve only on jumps that leave the region, so per-fetch address translation disappears.
Why it is faster: —
Applies elsewhere: a bytecode VM that keeps `uint8_t *ip` into the current function's code array; a stream parser that holds a pointer into the current buffer chunk instead of calling `read()` per byte.
Watch out: —
Seen in Snes9x: `S9xSetPCBase()` stores `CPU.PCBase`, a pointer pre-biased so `PCBase[PCw]` addresses the byte. It also caches `CPU.MemSpeed` and `MemSpeedx2` for that region. Branches call it only when the target leaves the 4 KB block: `if ((Registers.PCw & ~MEMMAP_MASK) != (newPC.W & ~MEMMAP_MASK)) S9xSetPCBase(...)`. current code: getset.h (S9xSetPCBase), cpumacro.h (bOP), cpuexec.cpp

### snes9x-103 — Check a block boundary once per item, not once per byte
Lens: lookup-tables · Source: Snes9x
Principle: If a variable-length item might straddle a buffer boundary, look up its length in a table and test the boundary once before decoding. The decoder can then read every byte with an unchecked pointer access.
Why it is faster: —
Applies elsewhere: network packet parsers that check `remaining >= header_len[type]` once; UTF-8 decoders that look up the sequence length from the lead byte and bounds-check once.
Watch out: —
Seen in Snes9x: Each mode has a 256-entry `S9xOpLengths*` table. The main loop tests `(Registers.PCw & MEMMAP_MASK) + ICPU.S9xOpLengths[Op] >= MEMMAP_BLOCK_SIZE`. Only in that rare case does it re-resolve `PCBase` and switch to the slow table. current code: cpuexec.cpp, sa1cpu.cpp, globals.cpp

### snes9x-104 — Store flag inputs, derive the flags on demand
Lens: laziness · Source: Snes9x
Principle: Instead of computing every status bit after every operation, store the raw result value(s) that determine them and compute a bit only when it is read. The common operations then do one store instead of several masks and shifts.
Why it is faster: —
Applies elsewhere: keeping the last comparison operands and computing `<`/`==` lazily; storing a byte count and deriving "empty" and "full" only when queried.
Watch out: —
Seen in Snes9x: `SetZN(uint8 Work8)` is just `ICPU._Zero = Work8; ICPU._Negative = Work8;`. Z is "`_Zero == 0`" and N is bit 7 of `_Negative`. They are packed into P only by `S9xPackStatus()`, when P is pushed or read. The SuperFX core does the same: `vSign`, `vZero`, `vCarry` and `vOverflow` hold raw values (e.g. `// v & 0x8000`). current code: cpumacro.h (SetZN), cpuexec.h (S9xPackStatus/S9xUnpackStatus), fxinst.h

### snes9x-105 — Overlay byte and word views on one register
Lens: memory-layout · Source: Snes9x
Principle: Model registers that are accessed both as a whole and in parts as a union whose layout follows host endianness. Partial reads and writes become plain loads and stores, with no shift-and-mask code.
Why it is faster: —
Applies elsewhere: IP-address structs with `uint32`/`uint8[4]` views; packed RGBA pixels accessed as one word or as channels.
Watch out: —
Seen in Snes9x: `65c816.h` defines `pair` (`B.l`/`B.h`/`W`) and `PC_t`, which gives `xPBPC` (24-bit bank:PC), `W.xPC` and `B.xPB`. Both are laid out under `#ifdef LSB_FIRST`. `Registers.AL &= ...` touches only the low byte. Page-wrap uses `a.B.xPCl++` and bank-wrap uses `a.W.xPC++`. current code: 65c816.h, getset.h (S9xGetWord wrap cases)

### snes9x-106 — Precompute shifted forms of registers
Lens: memory-layout · Source: Snes9x
Principle: If a value is always used shifted or combined (bank << 16), store the shifted copy alongside it and update both on write. Every use site saves the shift.
Why it is faster: —
Applies elsewhere: caching `row * stride` for the current row in image code; storing a page base address next to the page number.
Watch out: —
Seen in Snes9x: `ICPU.ShiftedPB` and `ICPU.ShiftedDB` hold the program and data bank pre-shifted to bits 16–23. Address helpers then do `ICPU.ShiftedDB + Registers.Y.W`, and the MVN/MVP handlers set `ICPU.ShiftedDB = Registers.DB << 16` on each write. current code: cpuexec.h (SICPU), cpuops.cpp (Op54*/Op44*), getset.h (S9xSetPCBase)

### snes9x-107 — Page table whose entries are either pointers or tags
Lens: memory-layout · Source: Snes9x
Principle: Split the address space into fixed pages and keep one pointer per page. Real memory maps to a host pointer. Special regions map to small integer tags below a sentinel. One comparison sends the common case to a direct load, and only special pages fall into a switch.
Why it is faster: —
Applies elsewhere: software MMUs and TLBs; tagged pointers in language runtimes (small ints vs heap pointers); a memory-mapped I/O dispatcher in an embedded simulator.
Watch out: —
Seen in Snes9x: `Memory.Map[]` has 4096 entries for 4 KB blocks (`MEMMAP_SHIFT 12`). `S9xGetByte` does: `if (GetAddress >= (uint8 *) CMemory::MAP_LAST) { byte = *(GetAddress + (Address & 0xffff)); ... }`. Anything else is an enum tag (`MAP_PPU`, `MAP_LOROM_SRAM`, `MAP_C4`, …) handled by `switch ((pint) GetAddress)`. current code: memmap.h, getset.h (S9xGetByte/S9xSetByte)

### snes9x-108 — Pre-bias base pointers so lookup needs no subtraction
Lens: memory-layout · Source: Snes9x
Principle: When filling a translation table, store each entry already offset by the page's own base address. The hot path is then `table[page] + full_address`, with no "minus page start". Mirroring and remapping are also resolved when the table is built.
Why it is faster: —
Applies elsewhere: segment-relative addressing in runtimes; texture atlases that store `atlas_ptr - tile_origin` so tile-local coordinates index directly.
Watch out: —
Seen in Snes9x: `CMemory::map_lorom` sets `Map[p] = ROM + map_mirror(size, addr) - (i & 0x8000);`, so the read side can add the raw 16-bit offset. `map_mirror()` folds non-power-of-two ROM mirroring into the table once, at load time. current code: memmap.cpp (map_lorom, map_hirom, map_mirror)

### snes9x-109 — Separate read and write translation tables
Lens: memory-layout · Source: Snes9x
Principle: Keep distinct page tables for reads and writes. Read-only regions can map to real memory for reads and to a "discard" tag for writes, so write protection costs nothing extra on either path.
Why it is faster: —
Applies elsewhere: copy-on-write page tables; separate fast paths for loads and stores in a software cache; read-only mmap with a write-fault handler.
Watch out: —
Seen in Snes9x: `CMemory` holds `Map[]` and `WriteMap[]`. `map_WriteProtectROM()` points ROM blocks at `MAP_NONE` in `WriteMap`, so `S9xSetByte` drops ROM writes through the same tag test. current code: memmap.h, memmap.cpp (map_WriteProtectROM), getset.h (S9xSetByte)

### snes9x-110 — Apply bank switches by rewriting the page table
Lens: laziness · Source: Snes9x
Principle: When a control register remaps memory, update the affected page-table entries at that moment. Accesses never have to consult the bank register, and the cost lands on rare writes instead of frequent reads.
Why it is faster: —
Applies elsewhere: swapping a handler pointer when a feature flag flips instead of testing the flag per request; remapping a ring buffer's views on resize.
Watch out: —
Seen in Snes9x: `S9xSA1SetBWRAMMemMap()` loops over banks and sets `SA1.Map[c + 6]`/`[c + 7]` (and `WriteMap`) to `MAP_BWRAM` or `MAP_BWRAM_BITMAP2`. It also recomputes `SA1.BWRAM` once. `S9xSetSA1MemMap()` does the same for ROM banks. current code: sa1.cpp (S9xSA1SetBWRAMMemMap, S9xSetSA1MemMap)

### snes9x-111 — Classify by bit tests instead of range comparisons
Lens: branching-fast-paths · Source: Snes9x
Principle: When the region boundaries line up with bit patterns, classify an address with a few AND masks (and an add that turns a range into one bit) instead of a table or chained range comparisons. This keeps the code small, inlinable and free of memory loads.
Why it is faster: —
Applies elsewhere: testing for private IPv4 ranges with masks; checking power-of-two alignment classes; bucketing sizes with `clz`.
Watch out: —
Seen in Snes9x: `memory_speed()` in `getset.h` returns the access cost: `if (address & 0x408000) { if (address & 0x800000) return CPU.FastROMSpeed; return SLOW_ONE_CYCLE; } if ((address + 0x6000) & 0x4000) return SLOW_ONE_CYCLE; if ((address - 0x4000) & 0x7e00) return ONE_CYCLE; return TWO_CYCLES;`. In the 1.52 import this was a per-block `Memory.MemorySpeed[block]` load. current code: getset.h (memory_speed) debd16c1

### snes9x-112 — Merge two narrow accesses when no boundary is crossed
Lens: memory-layout · Source: Snes9x
Principle: A multi-byte read normally goes through the byte path twice. Check once whether the access stays inside one page. If it does, do one native wide load. Only boundary-straddling accesses take the split path.
Why it is faster: —
Applies elsewhere: reading a little-endian `u32` straight from a packet buffer when it fits in the current segment; vectorized string scanning that falls back near page ends.
Watch out: —
Seen in Snes9x: `S9xGetWord` computes a wrap-dependent `mask`. When `(Address & mask) != mask` it skips the two-byte route and does `word = READ_WORD(GetAddress + (Address & 0xffff))`. On LSB-first hosts `READ_WORD` is `*(uint16 *)(s)`. current code: getset.h (S9xGetWord/S9xSetWord), port.h (READ_WORD)

### snes9x-113 — One "next event" timestamp instead of polling every source
Lens: algorithms · Source: Snes9x
Principle: Keep a single `next_event` time for the earliest pending event. The hot loop compares one counter against it. Only when it passes does the code work out which event fired and schedule the next. A fixed cyclic event order lets rescheduling be a constant-time switch.
Why it is faster: —
Applies elsewhere: game loops and RTOS tick handlers with a "next deadline" variable; timer wheels; an event loop that sleeps until the earliest timer.
Watch out: —
Seen in Snes9x: `AddCycles(n)` is `CPU.Cycles += n; while (CPU.Cycles >= CPU.NextEvent) S9xDoHEventProcessing();`. `S9xReschedule()` walks a fixed ring: HBLANK_START → HDMA_START → HCOUNTER_MAX → HDMA_INIT → RENDER → WRAM_REFRESH, setting `CPU.NextEvent` from precomputed `Timings.*`. IRQ timing is likewise precomputed into `Timings.NextIRQTimer` and compared once per instruction. current code: cpuops.cpp (AddCycles), cpuexec.cpp (S9xReschedule, S9xDoHEventProcessing, S9xMainLoop)

### snes9x-114 — Run a dependent subsystem only when its state is observed
Lens: laziness · Source: Snes9x
Principle: Let a secondary component lag behind and bring it up to date only when someone reads or writes its shared state, or at a coarse checkpoint. The two components never step in lockstep, and each runs in long uninterrupted bursts.
Why it is faster: —
Applies elsewhere: lazily recomputing derived spreadsheet cells on read; flushing a write-behind cache only on fsync or a read of the same key; catching up a physics simulation only when rendering asks for positions.
Watch out: —
Seen in Snes9x: The SPC700 runs only inside `S9xAPUExecute()`. That is called from `S9xAPUReadPort`/`S9xAPUWritePort` ("write_port will run the APU until given clock before writing value") and from `S9xAPUEndScanline()`. The DSP is in turn caught up by `SNES::dsp.synchronize()` once per line. current code: apu/apu.cpp (S9xAPUExecute, S9xAPUReadPort, S9xAPUWritePort, S9xAPUEndScanline), ppu.cpp (0x2140 case)

### snes9x-115 — Exact rational clock conversion with a carried remainder
Lens: algorithms · Source: Snes9x
Principle: To convert between two clocks with an awkward ratio, multiply by the integer numerator, divide by the denominator, and carry the remainder into the next conversion. This uses integer math only, never drifts, and needs no per-tick work.
Why it is faster: —
Applies elsewhere: audio sample-rate bookkeeping (44.1 kHz vs 48 kHz); Bresenham line stepping; converting monotonic ticks to frames at 30000/1001 fps.
Watch out: —
Seen in Snes9x: `S9xAPUGetClock()` returns `(ratio_numerator * (cpucycles - reference_time) + remainder) / ratio_denominator`, with `APU_NUMERATOR_NTSC 15664` / `APU_DENOMINATOR_NTSC 328125`. `S9xAPUExecute()` saves the `%` remainder and resets `reference_time`. A speed hack just scales the denominator in `S9xAPUTimingSetSpeedup()`. current code: apu/apu.cpp

### snes9x-116 — Interleave co-processors in bounded bursts
Lens: algorithms · Source: Snes9x
Principle: Run a second processor in batches sized by a time budget derived from the primary clock, instead of alternating one instruction each. Context switches and synchronization checks happen per batch, not per instruction.
Why it is faster: —
Applies elsewhere: cooperative schedulers with time slices; a game engine that runs N physics substeps per frame; batching GPU command submission.
Watch out: —
Seen in Snes9x: After each main-CPU instruction, `S9xSA1MainLoop()` runs the SA-1 in a loop `for (; SA1.Cycles < cycles && ...;)` with `cycles = CPU.Cycles * 3`. The 1.52 code ran a fixed 3 instructions per call. The SuperFX runs one per-scanline budget per call: `FxEmulate(... SuperFX.speedPerLine ...)` from the H-event handler. current code: sa1cpu.cpp (S9xSA1MainLoop), fxemu.cpp (S9xSuperFXExec), cpuexec.cpp 56af48c0

### snes9x-117 — Detect idle spin loops and skip to the next event
Lens: branching-fast-paths · Source: Snes9x
Principle: Programs often busy-wait by re-reading a memory location in a tight loop. Recognize a backward branch that returns to an instruction which read RAM and wrote nothing, and fast-forward time to the next scheduled event instead of executing the loop.
Why it is faster: —
Applies elsewhere: OS idle detection that halts the CPU in a spin-wait; VM hypervisors that trap PAUSE/HLT loops; replacing a polling loop with a blocking wait.
Watch out: —
Seen in Snes9x: In the 1.52 import, under `CPU_SHUTDOWN`: `S9xGetByte` recorded `CPU.WaitAddress = CPU.PBPCAtOpcodeStart` when reading a RAM block, and every `S9xSetByte` cleared it. The `bOP` branch macro called `CPUShutdown()`. When `Registers.PBPC == CPU.WaitAddress` was hit repeatedly, that function set `CPU.Cycles = CPU.NextEvent` and ran the APU. An SA-1 variant set `SA1.Executing = FALSE`. code at d1666860: cpuops.cpp (CPUShutdown), cpumacro.h (bOP), getset.h (WaitAddress)

### snes9x-118 — Compile one source file twice with renamed symbols
Lens: compiler-build · Source: Snes9x
Principle: To run two instances of the same engine against different state, include the implementation twice, with macros that rename its globals and entry points the second time. Each copy addresses its own globals directly, with no "this" pointer or indirection on the hot path.
Why it is faster: —
Applies elsewhere: generating float and double variants of a math library from one file; building a signed and an unsigned codec from one source.
Watch out: —
Seen in Snes9x: `sa1cpu.cpp` does `#define CPU SA1`, `#define Registers SA1Registers`, `#define S9xGetByte S9xSA1GetByte`, `#define S9xOpcodesM1X1 S9xSA1OpcodesM1X1`, … and `#define SA1_OPCODES`, then `#include "cpuops.cpp"`. Inside, `AddCycles` becomes a plain `SA1.Cycles += (n)`. current code: sa1cpu.cpp, cpuops.cpp (SA1_OPCODES)

### snes9x-119 — Batch rendering until the state actually changes
Lens: laziness · Source: Snes9x
Principle: Record which output lines are pending, and render the whole pending span in one pass only when a state change would make the next lines differ. Setup cost is paid once per span of identical state, not once per line.
Why it is faster: —
Applies elsewhere: UI toolkits that accumulate dirty regions and paint once per frame; text renderers that draw runs of identically styled glyphs together; batching draw calls until a material change.
Watch out: —
Seen in Snes9x: `RenderLine()` only snapshots per-line scroll and Mode 7 matrices into `LineData[]`/`LineMatrixData[]` and advances `IPPU.CurrentLine`. PPU register writes call `FLUSH_REDRAW()` (`if (IPPU.PreviousLine != IPPU.CurrentLine) S9xUpdateScreen();`). `S9xUpdateScreen` then draws lines `PreviousLine..CurrentLine-1` in one go. current code: gfx.cpp (RenderLine, S9xUpdateScreen), ppu.h (FLUSH_REDRAW), ppu.cpp

### snes9x-120 — Ignore writes that don't change anything
Lens: laziness · Source: Snes9x
Principle: Before invalidating caches or flushing work in response to a write, compare the new value with the stored one. Programs often rewrite registers with the same value, and that should cost one compare.
Why it is faster: —
Applies elsewhere: React-style `setState` bailing out on equal values; skipping a DB UPDATE when the row is unchanged; GPU state caches that elide redundant `glBindTexture`.
Watch out: —
Seen in Snes9x: Most PPU cases in `S9xSetPPU` are wrapped as `case 0x2105: if (Byte != Memory.FillRAM[0x2105]) { FLUSH_REDRAW(); ... }`, using `FillRAM` as a shadow of the last written value. OAM writes in `REGISTER_2104` flush and set `OBJChanged` only `if (lowbyte != PPU.OAMData[addr] || highbyte != PPU.OAMData[addr + 1])`. current code: ppu.cpp (S9xSetPPU), ppu.h (REGISTER_2104)

### snes9x-121 — Decode-on-demand cache with write invalidation
Lens: laziness · Source: Snes9x · Also: ares
Principle: Keep decoded copies of source data in a cache, each with a valid flag. Decode an entry on first use. When the source is written, clear only the flags for the entries that cover the written bytes.
Why it is faster: —
Applies elsewhere: glyph atlases rasterized on first use; compiled-template caches invalidated by file mtime; texture upload caches keyed by a dirty bit.
Watch out: —
Seen in Snes9x: Planar SNES tiles are converted to one byte per pixel into `IPPU.TileCache[]`. `CachedTile::GetCachedTile()` calls `BG.ConvertTile` only `if (!BG.Buffered[TileNumber])`. Every VRAM write in `REGISTER_2118` clears `IPPU.TileCached[TILE_2BIT][address >> 4]`, `[TILE_4BIT][address >> 5]`, `[TILE_8BIT][address >> 6]` and the hi-res odd/even neighbours. current code: tileimpl.h (CachedTile), tile.cpp (ConvertTile*), ppu.h (REGISTER_2118/2119) • Also seen in ares (ares-013): The new scanline PPU added `PPU::Cache::tile_2bpp/4bpp/8bpp`, which check `tilevalid[bpp][tile]` and on a miss decode the planar tile to 8-bit pixels at `tiledata[bpp] + (tile << 6)`. Background and sprite renderers then read pixels with `*tiledata++`. In total the new PPU was 22–34% faster than the old one. 0bf6c40d1 2010-09 “Update to v068r07 release.” · f1009ec63 2010-09 “Update to v068r08 release.”

### snes9x-122 — Remember "empty" so whole items can be skipped
Lens: branching-fast-paths · Source: Snes9x
Principle: While decoding or building an item, OR all of its data together and record whether the result was zero. Consumers check that one flag and skip the item entirely, with no per-element tests.
Why it is faster: —
Applies elsewhere: sparse-tile flags in map renderers; zero-page detection in VM memory snapshots; skipping all-transparent sprites in a 2D engine.
Watch out: —
Seen in Snes9x: Each `ConvertTile*` accumulates `non_zero |= p1 | p2` and returns `non_zero ? TRUE : BLANK_TILE`. The returned value is stored as the cache flag itself, so `DrawTile16::Draw` starts with `if (cache.IsBlankTile()) return;`. current code: tile.cpp, tileimpl.h (IsBlankTile)

### snes9x-123 — Spread bits with a nibble lookup table
Lens: lookup-tables · Source: Snes9x
Principle: To turn bit-planar data into packed per-pixel values, precompute a table that maps each 4-bit nibble to four bytes with that plane's bit in the right position. Each pixel row is then built from a few table ORs, not 8×planes bit extractions.
Why it is faster: —
Applies elsewhere: 1-bpp bitmap font expansion to 8-bit masks; Morton/bit-interleave encoders; converting packed bitsets into byte arrays for SIMD.
Watch out: —
Seen in Snes9x: `S9xInitTileRenderer()` fills `pixbit[8][16]`, where each entry places nibble bits in four bytes, shifted by the plane index, with endianness handled at init. `DOBIT(n, i)` then does `p1 |= pixbit[i][pix >> 4]; p2 |= pixbit[i][pix & 0xf];` and skips zero bytes. `hrbit_odd`/`hrbit_even` pre-split even and odd pixels for hi-res. current code: tile.cpp

### snes9x-124 — Policy templates for pixel kernels, picked once per batch
Lens: compiler-build · Source: Snes9x
Principle: Express each kernel variant as a template combining small policy types (blend op × pixel plotter × tile walker). The compiler emits a fully inlined function for each combination. A table of function pointers is indexed once per layer, so the per-pixel loop has no mode branches.
Why it is faster: —
Applies elsewhere: image libraries (Skia, pixman) that instantiate blitters per format pair; `std::sort` with an inlined comparator; shader permutations compiled per feature set.
Watch out: —
Seen in Snes9x: `tileimpl.h` defines blend policies (`NOMATH`, `REGMATH<COLOR_ADD>`, `MATHF1_2`, `MATHS1_2`…), plotters (`Normal1x1`, `Normal2x1`, `Hires`, `Interlace`) and walkers (`DrawTile16`, `DrawClippedTile16`, `DrawMode7BG1`…). `Renderers<TILE, PIXEL>::Functions[9]` holds all blend variants. `S9xSelectTileRenderers()` sets `GFX.DrawTileMath = DT[i]` once. The 1.52 code produced the same matrix by re-including `tile.cpp` under different macros. Instantiations are now split across `tileimpl-*.cpp` files. current code: tileimpl.h, tile.cpp (S9xSelectTileRenderers), tileimpl-n1x1.cpp, tileimpl-n2x1.cpp, tileimpl-h2x1.cpp 867ee9c6 7603ed09

### snes9x-125 — Hoist loop-invariant cases out of the inner loop
Lens: branching-fast-paths · Source: Snes9x
Principle: If a condition is fixed for a whole block (orientation, wrap mode), branch on it once and run a separate inner loop for each case. The inner loops become simple enough to unroll and vectorize.
Why it is faster: —
Applies elsewhere: image rotation with one loop per angle; a CSV writer with separate loops for quoted and unquoted columns; separate kernels for clamped vs. wrapped texture sampling.
Watch out: —
Seen in Snes9x: `DrawTile16::Draw` has four copies of the row loop, for no flip, H-flip, V-flip and both. Each indexes `bp[x]` or `bp[7 - x]` and steps `bp` up or down. `DrawTileNormal` (Mode 7) tests `PPU.Mode7Repeat` once per line and runs either a masked `& 0x3ff` loop or an out-of-bounds-aware loop. current code: tileimpl.h (DrawTile16, DrawClippedTile16, DrawTileNormal)

### snes9x-126 — Resolve layer priority with a depth buffer
Lens: algorithms · Source: Snes9x
Principle: Instead of sorting layers or testing priorities per pixel in a fixed order, give each draw a depth value and keep a per-pixel depth buffer. Layers can be drawn in any convenient order, and each write is one compare.
Why it is faster: —
Applies elsewhere: the GPU z-buffer; 2D engines that z-sort UI with a depth attachment; map renderers merging overlapping layers.
Watch out: —
Seen in Snes9x: Each background/priority pair gets a depth (`DO_BG(0, 0, 4, FALSE, FALSE, 15, 11, 0)`, with Zh/Zl added to base `D`). The plotter does `if (Z1 > GFX.DB[Offset + N] && (M)) { GFX.S[...] = MATH::Calc(...); GFX.DB[Offset + N] = Z2; }`. Sprites use separate test and write depths (`Z1`/`Z2`). The buffers are cleared once per frame with `memset`. current code: gfx.cpp (RenderScreen, S9xStartScreenRefresh), tileimpl-n1x1.cpp

### snes9x-127 — Coalesce adjacent rows with identical parameters
Lens: algorithms · Source: Snes9x
Principle: When per-row parameters usually repeat, scan ahead to find how many consecutive rows share them and process that run as one block. Per-row setup becomes per-run setup.
Why it is faster: —
Applies elsewhere: run-length encoding of scanlines in a rasterizer; grouping log lines with the same prefix; batching DB rows with the same foreign key.
Watch out: —
Seen in Snes9x: `DrawBackground()` extends `Lines` while `LineData[Y + Lines].BG[bg]` has the same `VOffset` and `HOffset`, up to `GFX.LinesPerTile - VirtAlign`. Each tile is then drawn for `Lines` rows in one `DrawTile(Tile, Offset, VirtAlign, Lines)` call. current code: gfx.cpp (DrawBackground, DrawBackgroundMosaic, DrawBackgroundOffset)

### snes9x-128 — Represent masks as span lists, then choose a kernel per span
Lens: algorithms · Source: Snes9x
Principle: Turn a per-pixel mask into a short list of [left, right) spans with a mode each. The renderer iterates spans, selects the right kernel for each span once, and runs it without any per-pixel mask test.
Why it is faster: —
Applies elsewhere: scanline polygon fill with span buffers; clip regions as rectangles in windowing systems; interval-based text selection highlighting.
Watch out: —
Seen in Snes9x: `S9xComputeClipWindows()` uses `CalcWindowMask` and `region_map` to build `GFX.Clip[bg].Left[]`, `Right[]` and `DrawMode[]`. It reruns only when `PPU.RecomputeClipWindows` is set. `DrawBackground` loops `for (clip = 0; clip < GFX.Clip[bg].Count; clip++)` and picks `GFX.DrawTileMath` or `GFX.DrawTileNomath` per span. current code: clip.cpp, gfx.cpp (DrawBackground, S9xUpdateScreen)

### snes9x-129 — Swap the lookup table instead of testing a flag per element
Lens: branching-fast-paths · Source: Snes9x
Principle: If a mode only changes what a per-element lookup returns, point the lookup at a different table (for example an all-black palette) instead of testing the mode for each element.
Why it is faster: —
Applies elsewhere: a "disabled" UI theme implemented as an alternate palette; muting audio by swapping in a zero gain table; locale-specific collation tables.
Watch out: —
Seen in Snes9x: `CachedTile::SelectPalette()` ends with `GFX.ScreenColors = GFX.ClipColors ? BlackColourMap : GFX.RealScreenColors;`. The plotter always does `GFX.ScreenColors[Pix]`. Direct-colour mode likewise selects one of the precomputed `DirectColourMaps[8]`. current code: tileimpl.h (SelectPalette), gfx.cpp (S9xBuildDirectColourMaps)

### snes9x-130 — Saturating per-channel arithmetic in one integer (SWAR)
Lens: simd · Source: Snes9x
Principle: Pack colour channels with a spare bit above each, add or subtract them all in one integer operation, then turn each channel's overflow bit into a saturation mask with a shift and multiply. Three branchy clamps become a few ALU ops.
Why it is faster: —
Applies elsewhere: RGB565 blending on microcontrollers; packed-byte saturating add without SIMD intrinsics; checksum folding.
Watch out: —
Seen in Snes9x: `COLOR_ADD::fn`: `int rb = (C1 & (RED_MASK|BLUE_MASK)) + (C2 & ...); int rbcarry = rb & ((0x20 << RED_SHIFT_BITS) | 0x20); ... int rgbsaturate = (((g & (0x20 << GREEN_SHIFT_BITS)) | rbcarry) >> 5) * 0x1f;`. `COLOR_SUB::fn` pre-sets the guard bits and ANDs with the borrow mask. Before this, the 1.52 code went through the 64 K `GFX.X2` table for addition and had per-channel `if (mC1 > mC2)` for subtraction. current code: gfx.h (COLOR_ADD, COLOR_SUB) 9721f78d

### snes9x-131 — Whole-value lookup tables for awkward per-channel ops
Lens: lookup-tables · Source: Snes9x · Also: Flycast
Principle: When an operation on a packed 16-bit value is hard to express branch-free, precompute its result for all 65536 inputs (or for a small sum range) once. Each use becomes a single indexed load.
Why it is faster: —
Applies elsewhere: gamma tables for 8/16-bit images; CRC byte tables; 16-bit RGB→YUV tables in video converters.
Watch out: —
Seen in Snes9x: `S9xGraphicsInit()` builds `GFX.ZERO[0x10000]`, which clears every channel whose high bit is unset. `COLOR_SUB::fn1_2` uses it as `GFX.ZERO[((C1 | RGB_HI_BITS_MASKx2) - (C2 & RGB_REMOVE_LOW_BITS_MASK)) >> 1]`. `brightness_cap[64]` clamps channel sums to the current brightness ceiling in `COLOR_ADD_BRIGHTNESS`. current code: gfx.cpp (S9xGraphicsInit), gfx.h, ppu.cpp (S9xFixColourBrightness) • Also seen in Flycast (flycast-030): `decoded_colors[3][65536]` in `softrend.cpp` is filled once for 565, 1555 and 4444 using `REP_32/REP_64/REP_16` bit replication. The texture path reads `decoded_colors[textype][pdata[u+v*width]]`. e6a9d3e66 2015-08 “softrend: Resembling a pixel pipeline, textures, sort, cull”

### snes9x-132 — Keep palettes pre-converted to the output format
Lens: lookup-tables · Source: Snes9x · Also: ares
Principle: Convert source colours to the final display format, with global adjustments like brightness already applied, when the palette or the adjustment changes. Per-pixel output is then a single indexed load.
Why it is faster: —
Applies elsewhere: indexed-colour framebuffers with a hardware CLUT; terminal emulators caching ANSI-to-RGB maps; theme colours resolved at load time.
Watch out: —
Seen in Snes9x: On a CGRAM write, `REGISTER_2122` computes `IPPU.ScreenColors[PPU.CGADD] = BUILD_PIXEL(IPPU.Red[...], ...)` through `IPPU.XB` = `mul_brightness[PPU.Brightness]`. A brightness change (0x2100) calls `S9xFixColourBrightness()` and `S9xBuildDirectColourMaps()` to rebuild all 256 colours. current code: ppu.h (REGISTER_2122), ppu.cpp (S9xFixColourBrightness), globals.cpp (mul_brightness), gfx.cpp • Also seen in ares (ares-014): The new PPU allocates `light_table[16][32768]`, which maps every 15-bit color at each of 16 brightness levels to output, and indexes `light_table[display_brightness][color]`. The video path was likewise reworked so each core's `video.generate_palette()` builds the native-to-output table once, and the UI blits with `*dp++ = palette[color]`. f1009ec63 2010-09 “Update to v068r08 release.” · 616372e96 2012-04 “Update to v088r03 release.” • Also seen in ares (ares-161): The SNES PPU registers `screen->colors(1 << 19, color)`. `refreshPalette()` fills `_palette[_colors]` lazily (`if(_palette) return;`), and the output loops are `*target++ = _palette[*source++]`. current code: ares/ares/node/video/screen.cpp, ares/sfc/ppu-performance/ppu.cpp

### snes9x-133 — Choose the pixel format at compile time
Lens: compiler-build · Source: Snes9x
Principle: If the output format is fixed for a build, make channel shifts and masks compile-time constants. All colour packing and blending code then folds to immediate operands.
Why it is faster: —
Applies elsewhere: embedded display drivers built for RGB565 or RGB888; serialization code generated per endianness.
Watch out: —
Seen in Snes9x: `pixform.h` defines `BUILD_PIXEL_RGB565`, `BUILD_PIXEL_RGB555`, … and selects them with `BUILD_PIXEL(R, G, B)` → `CONCAT(BUILD_PIXEL_, PIXEL_FORMAT)`. `COLOR_ADD` uses `#if GREEN_SHIFT_BITS == 6` to add the 565 green fix-up only when needed. current code: pixform.h, gfx.h

### snes9x-134 — Incremental fixed-point stepping for affine transforms
Lens: algorithms · Source: Snes9x
Principle: For an affine mapping across a row, compute the start coordinate once per row in fixed point, then add a constant delta per pixel. A per-pixel matrix multiply becomes two integer adds.
Why it is faster: —
Applies elsewhere: software texture mapping and rotozoomers; DDA line drawing; resampling with a fixed-point phase accumulator.
Watch out: —
Seen in Snes9x: Per line, `DrawTileNormal::Draw` computes `BB`/`DD` and `AA = l->MatrixA * startx + ...` in 8.8 fixed point. The inner loop is `for (x = Left; x < Right; x++, AA += aa, CC += cc) { int X = ((AA + BB) >> 8) & 0x3ff; ... }`. Horizontal flip is handled by negating `aa`/`cc`, not by a per-pixel test. current code: tileimpl.h (DrawTileNormal, DrawTileMosaic)

### snes9x-135 — Rebuild derived lists only when their source is dirty
Lens: laziness · Source: Snes9x
Principle: If a derived structure (per-row object lists, clip spans) depends on state that rarely changes, keep a dirty flag set by the writers and rebuild only when it is set and someone needs the result.
Why it is faster: —
Applies elsewhere: memoized layout in UI frameworks; spatial indexes rebuilt only after entity moves; build systems keyed on file hashes.
Watch out: —
Seen in Snes9x: `SetupOBJ()` rebuilds `GFX.OBJLines[]`, the sprites visible on each scanline in priority order, only `if (IPPU.OBJChanged || IPPU.InterlaceOBJ)`. OAM writes set `IPPU.OBJChanged = TRUE`. The per-line list ends with `OBJ[j].Sprite = -1`, so `DrawOBJS` walks it without a count. current code: gfx.cpp (SetupOBJ, DrawOBJS, S9xUpdateScreen), ppu.h (REGISTER_2104)

### snes9x-136 — Initialize large scratch arrays lazily, per row
Lens: memory-layout · Source: Snes9x
Principle: Instead of clearing a big 2-D scratch array up front, keep a small per-row "touched" flag and clear a row the first time it is written. Untouched rows cost nothing, and the flag also lets later passes skip them.
Why it is faster: —
Applies elsewhere: sparse-set tricks for O(1) clear; hash tables with generation counters instead of memset; per-tile dirty flags in image editors.
Watch out: —
Seen in Snes9x: In `SetupOBJ`'s FirstSprite+Y path, `uint8 OBJOnLine[SNES_HEIGHT_EXTENDED][128]` is left uninitialized. `bool8 AnyOBJOnLine[]` is memset instead, and `if (!AnyOBJOnLine[Y]) { memset(OBJOnLine[Y], 0, ...); AnyOBJOnLine[Y] = TRUE; }` runs on first use. The collection loop then skips lines with `if (AnyOBJOnLine[Y])`. current code: gfx.cpp (SetupOBJ) c64bfbf6

### snes9x-137 — Start in the cheap representation, upgrade retroactively
Lens: laziness · Source: Snes9x
Principle: Produce output in the cheaper format by default. If a feature that needs the richer format appears mid-way, convert what has already been produced in place and continue in the rich format. Most frames never pay for the upgrade.
Why it is faster: —
Applies elsewhere: strings stored as Latin-1 until a wide char appears (CPython, V8); small-vector inline storage promoted to the heap; ASCII-only fast paths in text buffers.
Watch out: —
Seen in Snes9x: Frames render at 256 px wide (`Normal1x1`, "for speedhacking modes"). When hi-res or pseudo-hi-res first appears, `S9xUpdateScreen` runs "Have to back out of the regular speed hack". It widens the lines already drawn (`*q = *(q + 1) = *p`) and sets `IPPU.DoubleWidthPixels = TRUE`. Interlace doubles height the same way with `memmove`. current code: gfx.cpp (S9xUpdateScreen), tile.cpp (S9xSelectTileRenderers)

### snes9x-138 — Skip passes whose output cannot be seen
Lens: branching-fast-paths · Source: Snes9x
Principle: Before running an expensive stage, check cheaply whether its result can affect the output, and skip it if not. Do the same for whole frames when the consumer will drop them.
Why it is faster: —
Applies elsewhere: culling off-screen widgets; skipping shadow passes when no light casts shadows; dropping video frames under load instead of decoding them.
Watch out: —
Seen in Snes9x: `S9xUpdateScreen` calls `RenderScreen(TRUE)` for the subscreen only when hi-res, or when colour math actually uses the subscreen (`(FillRAM[0x2130] & 2) && (FillRAM[0x2131] & 0x3f) && (FillRAM[0x212d] & 0x1f)`). Under forced blanking it fills black and skips all layers. With frame skip, `IPPU.RenderThisFrame` is false and `RenderLine()` only keeps sprite overflow flags up to date. current code: gfx.cpp (S9xUpdateScreen, RenderLine, S9xStartScreenRefresh)

### snes9x-139 — Specialize bulk transfers by destination
Lens: io · Source: Snes9x
Principle: For bulk copies into a device with side effects, branch once on the destination and source kind and run a tight loop specialized for that pair. The generic byte path (full address decode and dispatch per byte) is kept only for exotic cases.
Why it is faster: —
Applies elsewhere: `sendfile`/zero-copy paths chosen per fd type; database bulk loaders with per-column-type inner loops; DMA engines with memory-to-memory vs. peripheral modes.
Watch out: —
Seen in Snes9x: `S9xDoDMA` resolves `base = S9xGetBasePointer(...)` and transfers block by block (`count` limited to the 4 KB map block). In the "DMA FAST PATH" it switches on `d->BAddress`. VMDATAL runs `REGISTER_2118_linear` or `REGISTER_2118_tile` loops chosen once by `PPU.VMA.FullGraphicCount`, and OAM, CGRAM and WRAM have loops of their own. Without `base`, the "DMA SLOW PATH" calls `S9xGetByte`/`S9xSetPPU` per byte. current code: dma.cpp (S9xDoDMA), ppu.h (REGISTER_2118_linear/_tile)

### snes9x-140 — Resumable unrolled loops via switch-into-loop
Lens: branching-fast-paths · Source: Snes9x
Principle: When a loop body is a fixed cycle of N steps that may stop at any step and resume later, unroll the cycle inside a `do` loop and jump into it with a `switch` on the saved step index (Duff's device). There is no modulo or step dispatch per iteration, and resuming is free.
Why it is faster: —
Applies elsewhere: protothreads/coroutines in C; incremental parsers that resume mid-token; byte-at-a-time protocol encoders.
Watch out: —
Seen in Snes9x: Multi-register DMA modes in `dma.cpp` use `switch (b) { default: do { ... S9xSetPPU(Work, 0x2100 + d->BAddress); if (--count <= 0) { b = 1; break; } case 1: ... } while (1); }`, commented "a variation on Duff's Device". The DSP does the same in `SPC_DSP::run`: `switch (phase) { loop: #define PHASE(n) if (n && !--clocks_remain) break; case n:` over the 32-clock sample schedule. current code: dma.cpp, apu/bapu/dsp/SPC_DSP.cpp (run)

### snes9x-141 — Cache resolved pointers across repeated small transfers
Lens: memory-layout · Source: Snes9x
Principle: If a stream of small accesses walks sequentially through memory, resolve the address to a host pointer once, advance the pointer, and invalidate it only when the walk crosses a region boundary or a slow case occurs.
Why it is faster: —
Applies elsewhere: iterator objects over chunked buffers (ropes, deques); cursor-based readers in columnar formats; reusing a prepared statement handle.
Watch out: —
Seen in Snes9x: `HDMAMemPointers[8]` holds one host pointer per HDMA channel, set via `S9xGetMemPointer` in `HDMAReadLineCount`. The per-line transfer does `S9xSetPPU(*HDMAMemPointers[d]++, 0x2100 + p->BAddress)`, and the pointer is set to NULL to force the generic `S9xGetByte` path near block ends. current code: dma.cpp (S9xDoHDMA, HDMAReadLineCount)

### snes9x-142 — Decompress a whole block up front, then stream from the buffer
Lens: laziness · Source: Snes9x
Principle: When a consumer reads compressed data through a byte-wise interface, decode the whole requested range into a plain buffer at the start and hand the consumer that buffer. The decoder runs in one hot burst, and the consumer gets the fast memory path.
Why it is faster: —
Applies elsewhere: inflating a gzip HTTP body into memory before parsing; decoding a texture into a staging buffer before upload.
Watch out: —
Seen in Snes9x: For S-DD1, `S9xDoDMA` calls `SDD1_decompress(sdd1_decode_buffer, in_ptr, d->TransferBytes)` and sets `base = in_sdd1_dma; p = 0; count = rem;`, so the normal DMA fast path copies from the decoded buffer. SA-1 character-conversion DMA likewise converts a whole block into a buffer before the transfer. current code: dma.cpp (S9xDoDMA), sdd1emu.cpp

### snes9x-143 — Chunked on-demand decoding into a power-of-two ring
Lens: laziness · Source: Snes9x
Principle: For sequential reads of a compressed stream, decode a chunk only when the output ring runs empty, and size the ring as a power of two so wraparound is a mask. Seeking forward simply decodes and discards.
Why it is faster: —
Applies elsewhere: streaming audio decoders that refill half a buffer at a time; zlib's inflate window; log tailers that decompress incrementally.
Watch out: —
Seen in Snes9x: `SPC7110Decomp::read()` refills via `mode0/1/2(false)` only `if (decomp_buffer_length == 0)` ("decompress at least (decomp_buffer_size / 2) bytes"). It then reads with `decomp_buffer_rdoffset &= decomp_buffer_size - 1`, and the header notes the size "must be a power of two". current code: spc7110dec.cpp, spc7110dec.h

### snes9x-144 — Split bit permutations into per-byte lookup tables
Lens: lookup-tables · Source: Snes9x
Principle: An arbitrary bit permutation of a 16- or 32-bit word decomposes into one table per input byte, combined by addition or OR. Four 256-entry lookups replace a 32-step bit loop.
Why it is faster: —
Applies elsewhere: bit reversal tables; Morton-code encoding for spatial indexes; DES permutation tables.
Watch out: —
Seen in Snes9x: The SPC7110 decoder builds `morton16[2][256]` and `morton32[4][256]` in its constructor. `morton_4x8(data)` returns `morton32[0][data & 255] + morton32[1][(data >> 8) & 255] + morton32[2][...] + morton32[3][...]`. In the S-DD1 decoder, `GetCodeword()` maps codeword bits to run lengths through `run_table[]`. current code: spc7110dec.cpp (morton_2x8, morton_4x8), sdd1emu.cpp (run_table)

### snes9x-145 — Fold prefix state into the dispatch index
Lens: lookup-tables · Source: Snes9x
Principle: When prefix instructions change how the next opcode behaves, OR the prefix bits into the dispatch index and use a larger table. One indexed call replaces a lookup followed by a branch on the prefix.
Why it is faster: —
Applies elsewhere: x86 decoders with tables indexed by (prefix, opcode); key-binding tables indexed by modifier mask plus keycode.
Watch out: —
Seen in Snes9x: SuperFX `FX_STEP` calls `(*fx_OpcodeTable[(GSU.vStatusReg & 0x300) | vOpcode])()`, where the ALT1/ALT2 status bits select one of four 256-entry banks. FROM/TO prefixes are kept as register pointers `GSU.pvSreg`/`pvDreg`, so handlers do `DREG = ... SREG ...` with no prefix test. current code: fxinst.h (FX_STEP, SREG, DREG, CLRFLAGS), fxinst.cpp

### snes9x-146 — Patch the dispatch table when configuration changes
Lens: laziness · Source: Snes9x
Principle: If an operation's behaviour depends on a configuration register, install the specialized implementation into the dispatch table when the register is written. The handler never re-reads the configuration.
Why it is faster: —
Applies elsewhere: hot-swapping a logger implementation on config reload; installing an optimized `memcpy` into a function pointer at CPU-feature detection time.
Watch out: —
Seen in Snes9x: `fx_readRegisterSpace()` picks `GSU.pfPlot = fx_PlotTable[GSU.vMode]` and `GSU.pfRpix = fx_PlotTable[GSU.vMode + 5]` for the colour depth. It writes them into `fx_OpcodeTable[0x04c]`, `[0x14c]`, `[0x24c]` and `[0x34c]`, so PLOT dispatches straight to `fx_plot_2bit`/`4bit`/`8bit`. current code: fxemu.cpp (fx_readRegisterSpace), fxinst.cpp (fx_PlotTable)

### snes9x-147 — Precompute row and column base addresses for tiled layouts
Lens: lookup-tables · Source: Snes9x
Principle: For a framebuffer stored in an irregular tiled layout, precompute arrays of base addresses per tile row and per tile column whenever the layout changes. Each pixel address becomes two lookups and an add, with no multiply or layout branching.
Why it is faster: —
Applies elsewhere: swizzled texture addressing; line-start tables for framebuffers with odd strides; precomputed offsets in a struct-of-arrays layout.
Watch out: —
Seen in Snes9x: `fx_computeScreenPointers()` fills `GSU.apvScreen[32]` and `GSU.x[32]` for each height and mode. It runs only `if (GSU.vMode != GSU.vPrevMode || GSU.vPrevScreenHeight != GSU.vScreenHeight || GSU.vSCBRDirty)`. The plot then does `a = GSU.apvScreen[y >> 3] + GSU.x[x >> 3] + ((y & 7) << 1);`. current code: fxemu.cpp (fx_computeScreenPointers), fxinst.cpp (fx_plot_*)

### snes9x-148 — Fixed-point trig from quantized angle tables
Lens: lookup-tables · Source: Snes9x · Also: Flycast
Principle: When angles come from a small integer domain, store sin/cos in fixed-point tables indexed by angle. Rotation and scale become table lookups, integer multiplies and shifts, with no floating-point calls.
Why it is faster: —
Applies elsewhere: embedded motor control; retro-style game engines; DSP oscillators with wavetables.
Watch out: —
Seen in Snes9x: `c4emu.cpp` holds `C4SinTable[512]` and `C4CosTable[512]` as `int16`. The scaling/rotation op computes `A = (int16) SAR(C4CosTable[READ_WORD(Memory.C4RAM + 0x1f80) & 0x1ff] * XScale, 15);`. current code: c4emu.cpp • Also seen in Flycast (flycast-051): SH4 `fsca` takes a 16-bit angle. Both JITs translate it into a lookup in `sin_table`. ARM64 emits `Add(x1, x1, Operand(reg, UXTH, 3))` and then two post-indexed `Ldr`s. x64 emits `movzx rax, reg16` and indexes `[rcx + rax*8]`. 18a16f83a 2019-01 “arm64: more native opcodes implemented” · 31163b8d7 2019-01 “x64 dynarec: various fixes and optimizations — implemented fsca” • Also seen in Flycast (flycast-225): `f32_x2 sin_table[0x10000]` holds sin and cos pairs. Both the interpreter (`ctx->fr[n + 0] = sin_table[pi_index].u[0];`) and the dynarec canonical FSCA read from it. The `sinf`/`cosf` path remains only under a compile-time switch. `sh4_rom.cpp` was already present in the 2013 import (1c16baeb0). current code: core/hw/sh4/sh4_rom.cpp, core/hw/sh4/interpr/sh4_fpu.cpp, core/hw/sh4/dyna/shil_canonical.h

### snes9x-149 — Advance timers by N ticks in one call
Lens: algorithms · Source: Snes9x · Also: ares
Principle: When several idle ticks happen back-to-back, advance every clocked counter by the whole count in one call instead of calling the single-tick routine N times. Overflow handling stays the same because the threshold is checked once.
Why it is faster: —
Applies elsewhere: simulation engines that jump idle entities forward by elapsed time; token-bucket rate limiters refilling by elapsed ms; game timers updated by delta time.
Watch out: —
Seen in Snes9x: The SPC700 core uses `op_io(2)`/`op_io(4)` for multi-cycle internal operations (e.g. `case 0x9f: op_io(4); ...`). These call `SMP::tick(unsigned clocks)`, which runs `Timer<f>::tick(clocks)` (`stage1_ticks += clocks; if (stage1_ticks < cycle_frequency) return;`) and bumps `clock`/`dsp.clock` by the count. current code: apu/bapu/smp/core.cpp, apu/bapu/smp/timing.cpp, apu/bapu/smp/core/oppseudo_*.cpp a19395ee • Also seen in ares (ares-003): GBA `CPU::step` used to run `for(n < clocks) timer_tick()`, testing a prescaler mask every cycle. `timer_step(clocks)` now does `timer.counter -= clocks; while(timer.counter <= 0) { timer.counter = timer.period(); ... }` and skips disabled or cascaded timers. The changelog reports Mr. Driller 2's title going from 170 to 400 fps. 303a0a67d 2012-04 “Update to v087r21 release.” • Also seen in ares (ares-095): `M32X::SH7604::step` looped `while(--cycles) { frt.run(); wdt.run(); }`. `FRT::run(cycles)` now computes `increments = counter / frequency` and does `frc += increments`. It sets `ovf` from `frc < prevFrc` and the compare-match flags from `u32(ocra - 1 - prevFrc) < increments`. `WDT::run` is similar. ab40ecbb9 2025-02 “md: slight performance boost with loop free 32x timers”

### snes9x-150 — Paste opcode bodies into one switch, build as one unit
Lens: compiler-build · Source: Snes9x
Principle: Put all opcode cases in a single `switch` by including generated fragment files, and compile the component's sources as one translation unit. The compiler sees every helper, so it can inline them and emit a single jump table.
Why it is faster: —
Applies elsewhere: SQLite's amalgamation build; unity builds in game engines; generated `switch`-based interpreters (Lua, CPython's ceval).
Watch out: —
Seen in Snes9x: `SMP::op_step()` is `switch (opcode_number) { #include "core/oppseudo_misc.cpp" #include "core/oppseudo_mov.cpp" ... }`. `smp.cpp` `#include`s `algorithms.cpp`, `core.cpp`, `memory.cpp` and `timing.cpp`, and `sdsp.cpp` includes `SPC_DSP.cpp`. The SMP runs with `while (clock < 0) op_step();`. current code: apu/bapu/smp/smp.cpp, apu/bapu/smp/core.cpp, apu/bapu/dsp/sdsp.cpp

### snes9x-151 — Branch-free clamp to a signed 16-bit range
Lens: branching-fast-paths · Source: Snes9x
Principle: Check for overflow by casting to the narrow type and comparing. If it overflowed, the saturated value is `(x >> 31) ^ 0x7FFF`, which yields 0x7FFF or 0x8000 from the sign. The common no-overflow case is one compare.
Why it is faster: —
Applies elsewhere: audio mixers summing PCM streams; fixed-point DSP filters; image convolution writing to int16.
Watch out: —
Seen in Snes9x: `SPC_DSP.cpp`: `#define CLAMP16( io ) { if ( (int16_t) io != io ) io = (io >> 31) ^ 0x7FFF; }`, used after every voice, echo and FIR sum. `Resampler::short_clamp` uses the same expression. current code: apu/bapu/dsp/SPC_DSP.cpp, apu/resampler.h

### snes9x-152 — Derive many periodic counters from one global counter
Lens: algorithms · Source: Snes9x
Principle: Instead of decrementing a separate counter per rate, run one shared counter and work out whether a given rate fires as `(counter + offset[rate]) % period[rate] == 0`. Per-tick work stays constant regardless of how many rates are in use.
Why it is faster: —
Applies elsewhere: cron-like schedulers that test `tick % interval`; LFOs sharing a master phase; game systems firing every N frames off one frame counter.
Watch out: —
Seen in Snes9x: `SPC_DSP::run_counters()` just decrements `m.counter` over `simple_counter_range` (30720). Envelope code calls `read_counter(rate)` = `((unsigned) m.counter + counter_offsets[rate]) % counter_rates[rate]`, with the 32-entry rate and offset tables. current code: apu/bapu/dsp/SPC_DSP.cpp

### snes9x-153 — Order branches by measured frequency
Lens: branching-fast-paths · Source: Snes9x · Also: H2O
Principle: Arrange if/else chains so the most frequent case is tested first and falls through, and record the measured frequencies in the code. The common path stays short and well-predicted.
Why it is faster: —
Applies elsewhere: HTTP routers checking hot routes first; tokenizers testing letters before rare punctuation; `likely()` annotations in kernels.
Watch out: —
Seen in Snes9x: `SPC_DSP::run_envelope` is annotated with frequencies: `if ( v->env_mode == env_release ) // 60%`, `if ( m.t_adsr0 & 0x80 ) // 99% ADSR`, `if ( v->env_mode >= env_decay ) // 99%`, `if ( v->env_mode == env_decay ) // 1%`. current code: apu/bapu/dsp/SPC_DSP.cpp (run_envelope) • Also seen in H2O (h2o-019): ab5eefc377 2014-11-17 Header handling tests host, then content-length, transfer-encoding, expect and upgrade, ordered by how often each is seen. • 59a3998cb9 2018-01-18 `parse_hostport` first matches a dotted-quad IPv4 plus port with `sscanf` and builds `sockaddr_in` directly. It falls back to `getaddrinfo` only if that fails. • 047451fc38 2020-12-01 `quicly_send` tries the 1-RTT epoch before 0-RTT, "as the availability of the two epochs are mutually exclusive". • 4faf073191 2022-10-05 When there is a single cookie header, the proxy adds it directly by reference, with no join and no copy.

### snes9x-154 — Wraparound ring buffer with split block copies
Lens: io · Source: Snes9x
Principle: For a producer/consumer FIFO, copy at most two contiguous runs per operation (up to the end of the array, then from the start) with `memcpy`, instead of moving elements one at a time with a modulo per element.
Why it is faster: —
Applies elsewhere: audio callbacks pulling from a ring; network socket buffers; lock-free SPSC queues in embedded firmware.
Watch out: —
Seen in Snes9x: `Resampler::pull()` does `memcpy(dst, buffer + start, min(num_samples, first_block_size) * 2); if (num_samples > first_block_size) memcpy(dst + first_block_size, buffer, ...);`, and `add_silence`/`push` work the same way. `S9xAPUEndScanline()` forwards samples only after `space_filled() >= APU_SAMPLE_BLOCK` (48). current code: apu/resampler.h, apu/apu.cpp

### snes9x-155 — Debounce persistence of dirty data
Lens: io · Source: Snes9x
Principle: Mark data dirty on write and start a countdown. Flush to disk only after the countdown expires. Bursts of writes cost one flag store each and produce a single save.
Why it is faster: —
Applies elsewhere: editor autosave; debounced localStorage writes; write-behind caches with a flush delay.
Watch out: —
Seen in Snes9x: SRAM writes in `S9xSetByte` set `CPU.SRAMModified = TRUE`. `S9xEndScreenRefresh` loads `CPU.AutoSaveTimer = Settings.AutoSaveDelay * Memory.ROMFramesPerSecond` and calls `S9xAutoSaveSRAM()` only when the timer reaches zero. current code: getset.h (S9xSetByte), gfx.cpp (S9xEndScreenRefresh)

### snes9x-156 — XOR-delta history in a power-of-two ring
Lens: allocation · Source: Snes9x
Principle: To keep many snapshots cheaply, store only the words that changed as (index, old XOR new) pairs, with a zero sentinel between snapshots. XOR makes each delta its own inverse, so stepping back reapplies it, and unchanged words cost nothing.
Why it is faster: —
Applies elsewhere: undo stacks for large documents; incremental VM checkpoints; delta-encoded game replays and rollback netcode.
Watch out: —
Seen in Snes9x: `StateManager::generate_delta()` compares 32-bit words of the serialized state and pushes `buffer[top_ptr] = (i << 32) | xor_` into a `uint64_t` ring masked by `buf_size_mask` (from `nearest_pow2_size`). `pop()` applies `tmp_state[addr] ^= xor_` back to the sentinel. `push()` swaps the `tmp_state`/`in_state` buffers instead of copying. current code: statemanager.cpp 8e1b726b

### snes9x-157 — Skip unchanged input with a shadow copy
Lens: laziness · Source: Snes9x
Principle: Keep a copy of the previous input frame. Compare it word by word and regenerate output only where the input changed. Static regions cost one compare per word.
Why it is faster: —
Applies elsewhere: VNC/RDP dirty-rectangle detection; incremental compilation by content hash; e-ink displays refreshing only changed areas.
Watch out: —
Seen in Snes9x: `S9xBlitPixSimple2x2` and the smooth/TV filters keep `XDelta`. The inner loop does `if (currentPixel != lastPixel) { ... dP1[0] = currentPixA; ...; *xP = *bP; }` and compares two 16-bit pixels per 32-bit word. `S9xBlitClearDelta()` forces a full redraw. current code: filter/blit.cpp

### snes9x-158 — Pad the edges so the inner loop has no boundary checks
Lens: memory-layout · Source: Snes9x
Principle: Write a temporary sentinel past the end of a row (duplicating the edge pixel), or allocate guard rows around a buffer. Neighbourhood reads at the border then need no special case, and the original value is restored afterwards.
Why it is faster: —
Applies elsewhere: image convolution with replicate-border padding; parsers that put a NUL sentinel at the end of the input; ghost cells in stencil computations.
Watch out: —
Seen in Snes9x: `S9xBlitPixSmooth2x2` saves `savePixel = *(uint16 *)(srcPtr + lineBytes)`, overwrites it with the last real pixel, and mirrors it into the delta buffer so the look-ahead `nextPixel = *bP++` is always valid. `S9xGraphicsInit` allocates `MAX_SNES_HEIGHT + 64` lines and sets `GFX.Screen = &GFX.ScreenBuffer[GFX.RealPPL * 32]`. The gtk xBRZ path feeds two extra rows above and below. current code: filter/blit.cpp (S9xBlitPixSmooth2x2), gfx.cpp (S9xGraphicsInit), gtk/src/filter_xbrz.cpp

### snes9x-159 — Blend packed colour channels with masked multiplies
Lens: simd · Source: Snes9x
Principle: In 565 or 555 colour, the non-adjacent channels (R and B) can be weighted together in one multiply after masking out the middle channel, since the gap absorbs the carry. Two multiplies blend all three channels.
Why it is faster: —
Applies elsewhere: alpha blending on 16-bit LCD controllers; packed-byte lerp (`(a & 0xff00ff) * w`) in software rasterizers.
Watch out: —
Seen in Snes9x: `hq2x.cpp` `interp_2px`: `((((c1 & 0x07e0) >> 5) * w1 + ((c2 & 0x07e0) >> 5) * w2) << (5 - s)) & 0x07e0) | (((c1 & 0xf81f) * w1 + (c2 & 0xf81f) * w2) >> s) & 0xf81f)`. Halving uses `(((C1 & RGB_REMOVE_LOW_BITS_MASK) + (C2 & ...)) >> 1) + (C1 & C2 & RGB_LOW_BITS_MASK)` (`COLOR_ADD::fn1_2`). current code: filter/hq2x.cpp, gfx.h (fn1_2), filter/blit.cpp

### snes9x-160 — Pack derived values into one word and compare lanes
Lens: lookup-tables · Source: Snes9x
Principle: Precompute a costly per-colour transform (RGB to YUV) for every input value, packed into one integer. A similarity test is then one table load per pixel plus masked lane comparisons, and the equal-pixel case is skipped before any lookup.
Why it is faster: —
Applies elsewhere: perceptual image diffing; colour quantization with cached Lab values; packed feature vectors in similarity search.
Watch out: —
Seen in Snes9x: `hq2x.cpp` fills `yuvtable[65536]` as `(y << 16) + (u << 8) + v`. `yuv_diff` tests `ABSDIFF(yuv1 & YMASK, yuv2 & YMASK) > (48 << 16) || ...`. Pattern building checks `w[4] != w[0] ? yuv_diff(...) : 0` first. xBRZ's `distYCbCrBuffered` goes further with a 16 M-entry float table of colour distances ("30% perf boost"). current code: filter/hq2x.cpp, filter/xbrz.cpp (distYCbCrBuffered)

### snes9x-161 — Exploit symmetry: one kernel, permuted inputs
Lens: algorithms · Source: Snes9x
Principle: If outputs in different quadrants or orientations are mirror images of one rule, write the rule once and call it with the neighbourhood indices permuted (or rotate a packed state). Code size and instruction-cache footprint shrink by the symmetry factor.
Why it is faster: —
Applies elsewhere: board-game engines evaluating one rotation and mapping the others; FFT butterflies; marching-squares lookups using rotated cases.
Watch out: —
Seen in Snes9x: The hq2x loop calls the same `hq2x_interp_1x1(r2y, pattern, w, ...)` four times with index orders `0..8`, `2,1,0,5,4,3,8,7,6` ("vert mirrored") and so on. xBRZ stores four 2-bit blend types per pixel in one byte and rotates them with `rotateBlendInfo<ROT_90>` = `((b << 2) | (b >> 6)) & 0xff`. current code: filter/hq2x.cpp, filter/xbrz.cpp

### snes9x-162 — Precompute linear-filter responses per input value
Lens: lookup-tables · Source: Snes9x
Principle: If a filter is linear, its output is a sum of each input sample's response. Precompute every possible input's response kernel once. Filtering becomes adding a few table rows, with all channels packed in one integer and clamped together with bit tricks.
Why it is faster: —
Applies elsewhere: audio FIR filters with precomputed polyphase tables; CRT/composite shaders baked into LUTs; image-blur LUTs for fixed kernels.
Watch out: —
Seen in Snes9x: `snes_ntsc_t` holds `table[snes_ntsc_palette_size][snes_ntsc_entry_size]`. `SNES_NTSC_RGB_OUT_14_` sums six rows (`kernel0[x] + kernel1[(x+12)%7+14] + ...`) of values packed as `(r) << 21 | (g) << 11 | (b) << 1`. `SNES_NTSC_CLAMP_` saturates all three fields with masks (`io |= clamp; clamp -= sub; io &= clamp;`). current code: filter/snes_ntsc.h, filter/snes_ntsc_impl.h, filter/snes_ntsc.c

### snes9x-163 — Fixed scale factor → constant weights, separable passes
Lens: algorithms · Source: Snes9x
Principle: With an integer scale factor, the interpolation weights repeat, so precompute them as fixed-point constants. Do the horizontal blend once per output column and reuse it across all output rows. Mark pointers `restrict` so the compiler can keep values in registers.
Why it is faster: —
Applies elsewhere: 2×/4× thumbnail generators; mipmap generation; integer-ratio audio upsampling with fixed polyphase coefficients.
Watch out: —
Seen in Snes9x: `ApplySharpBilinear4x` uses `static constexpr uint16_t W[4] = { 0, 40, 128, 216 }` (smoothstep in 8.8). It computes `rtop[dx]`/`rbot[dx]` once per 4×4 block, then blends vertically for each `dy`. Gamma conversions go through `gamma_r_encode[32]`/`gamma_decode[256]` tables, and parameters are declared `uint8_t* __restrict`. current code: filter/sharpbilinear_flexible.cpp

### snes9x-164 — Data-parallel post-processing by horizontal bands
Lens: concurrency · Source: Snes9x · Also: ares
Principle: Split an image filter into horizontal bands and run them on a persistent thread pool. Give each band the extra neighbour rows it reads, and keep stateful filters single-threaded. The pool is created once and reused every frame.
Why it is faster: —
Applies elsewhere: tile-parallel image processing in photo editors; parallel video scaling (FFmpeg slice threads); map/reduce over row ranges.
Watch out: —
Seen in Snes9x: `internal_threaded_filter()` in `gtk/src/gtk_display.cpp` queues `num_threads` jobs with `job_height = height / num_threads & ~3` on a `threadpool`, and skips threading for NTSC ("has internal state"). `xbrz::scale(..., yFirst, yLast)` takes a row range, so `win32/render.cpp` runs it on several threads. current code: gtk/src/gtk_display.cpp, gtk/src/threadpool.cpp, filter/xbrz.cpp, win32/render.cpp • Also seen in ares (ares-012): The XShm video driver's `refresh()` scaling loop got `#pragma omp parallel for` over `y`, and the build added `-fopenmp`. The changelog reports 101 to 111 fps at 4x scale. fc8eba133 2015-05 “Update to v094r18 release.”

### snes9x-165 — Optional hand-written assembly backends behind build flags
Lens: platform · Source: Snes9x
Principle: Keep a portable implementation as the reference, and let a build flag swap in hand-tuned assembly for the hottest subsystems on a specific CPU. Platforms without the assembly still build and run correctly.
Why it is faster: —
Applies elsewhere: OpenSSL and libjpeg-turbo per-arch assembly; video codecs with C fallbacks and x86/ARM kernels.
Watch out: —
Seen in Snes9x: The 1.52 import shipped `i386/fxemu2*.asm`, `i386/c4.asm` and `i386/zsnes*.asm` (from ZSNES). These were selected by `#ifdef ZSNES_FX` in `fxemu.h`/`cpuexec.cpp` and `#ifdef ZSNES_C4` in `c4.h`/`c4.cpp`, with the C++ cores as the default. code at d1666860: i386/*.asm, fxemu.h, c4.h, cpuexec.cpp
