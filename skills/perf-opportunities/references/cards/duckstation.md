# DuckStation — technique cards (347)

Each card is exactly 7 lines. Fetch one with: grep -A 6 "^### duckstation-NNN " duckstation.md

### duckstation-001 — Batch draw calls and flush only when state really changes
Lens: gpu-rendering · Source: DuckStation · Also: Flycast
Principle: Every draw call costs driver and CPU time that does not depend on how many triangles it carries. Collect primitives into one vertex buffer and submit it only when a state change that the batch cannot represent arrives, or when the buffer is full. Check for those changes at the moment the state is set, not once per primitive.
Why it is faster: —
Applies elsewhere: sprite batchers in 2D engines (one draw per texture atlas), canvas/WebGL renderers that merge rects, database bulk inserts that flush at N rows or on a schema change.
Watch out: —
Seen in DuckStation: The first OpenGL renderer flushed for every polygon. Quads are now joined into one triangle strip by repeating the last vertex, which creates degenerate triangles: `m_batch_vertices.push_back(m_batch_vertices.back())`. A flush happens only when `m_batch_command.bits != rc.bits` or the batch reaches `MAX_BATCH_VERTEX_COUNT`. Later, the texture-window and drawing-area checks moved out of the per-primitive path and into the GP0 commands that change them (`SetTextureWindow()` calls `FlushRender()` only if the value differs). 273f010d1 2019-09 “GPU: Use degenerate triangles to split strips and batch them” · a58b68735 2019-09 “GPU: Cap batch sizes at 1024 vertices, flush if exceeded” · 7f5c6f8b4 2020-04 “GPU/HW: Move some flush checks to command time” • Also seen in Flycast (flycast-144): `PolyParam::equivalent` compares masked `pcw`/`isp`/`tsp`/`tcw` words and the matrices. A 2022-02 fix made the `tcw` comparison exact so that a palette difference blocks a merge. Naomi 2 translucent polygons started being merged in the same commit series. Polygons whose culling differs are merged by setting `dupe_next_vtx` to repeat a vertex. Merging was later restricted to modes where order inside a pass doesn't matter (per-strip and per-pixel). In per-strip mode, `sortPolyParams` runs first so that polygons sorted next to each other can be merged. 20d25059b 2022-02 “naomi2: remove geom shader. Merge transparent polys. bump mapping fix.” · 0362fd107 2022-02 “naomi2: disable elan when net rollbacking, optimizations. ggpo: RTT fix” · f2374982b 2022-02 “pvr: don't merge polys with != palette. 2nd pass sometimes ignored” · 4c7df6c7f 2022-12 “pvr: don't merge polys in per-triangle. merge polys in per-strip”

### duckstation-002 — Move per-draw parameters into vertex data so batches stay intact
Lens: gpu-rendering · Source: DuckStation
Principle: If a parameter differs between draws and lives in a uniform, each change forces a new draw call. Put that parameter in a vertex attribute, or apply it on the CPU when you build the vertices, and draws with different values can share one batch.
Why it is faster: —
Applies elsewhere: per-instance colour or atlas UVs in instanced rendering, bindless texture indices carried in vertex data, text renderers that bake glyph offsets into vertices.
Watch out: —
Seen in DuckStation: The texture page and palette base used to be uniforms (`u_texture_page_base`, `u_texture_palette_base`), so a texture page change meant a flush. They are now packed into an `a_texpage` attribute and decoded in the vertex shader to a `flat ivec4 v_texpage`. The drawing offset was added on the CPU (`m_drawing_offset.x + vp.x`) instead of through a UBO field. The mask-bit depth value is now stored per vertex (`m_current_depth`) instead of being computed from a UBO ID, which “significantly reduces the number of mask bit refreshes”. c0a3a4176 2019-09 “GPU: Enable batching of polygons with the different texture pages” · e2e2b573c 2020-03 “GPU/HW: Move drawing offset to CPU-side” · a08c398d4 2020-06 “GPU/HW: Provide depth in vertex rather than computing”

### duckstation-003 — Decode data on the fly in the shader instead of building derived copies
Lens: gpu-rendering · Source: DuckStation · Also: Flycast
Principle: Building an intermediate resource for each source configuration (for example, a decoded texture per palette and page) costs passes and forces work to be split up. Sample the raw source directly and decode in the shader. One resource then serves every configuration, and draws that use different configurations can still be batched.
Why it is faster: —
Applies elsewhere: palette lookup in a fragment shader for indexed-colour images, YUV-to-RGB in the shader instead of a conversion pass, decoding compressed vertex formats in the vertex shader.
Watch out: —
Seen in DuckStation: The hardware renderer first rendered a "page texture" for every texture page/palette combination it used, and later created it with `texelFetch`. It now samples a single copy of VRAM in `SampleFromVRAM()`. The shader handles 4-bit and 8-bit indexed texels (`index_coord.x /= 4`, then a palette fetch) and direct 16-bit texels, so no per-page textures are needed. b5d51f47c 2019-09 “GPU: Use texel fetch for creating page textures” · 792ec27b1 2019-09 “GPU: Improve batching by sampling a VRAM copy” • Also seen in Flycast (flycast-122): The texture format table gained a "Palette (8b)" converter column (`texPAL4PT_TW`, `texPAL8PT_TW`). `IsGpuHandledPaletted(tsp, tcw)` chooses this path. The GL renderers sample a palette texture with a `palette_index` uniform, and `palette_updated` makes them re-upload only that texture. 7449230af 2020-07 “handle some palette textures on the GPU”

### duckstation-004 — Shadow API state to skip redundant driver calls
Lens: gpu-rendering · Source: DuckStation
Principle: Graphics drivers often validate state on every call, even when the value has not changed. Keep a copy of the last value you set (bound program, viewport, scissor) and compare against it before calling the API.
Why it is faster: —
Applies elsewhere: WebGL state caches in three.js and Babylon, OpenGL wrappers in game engines, and skipping DOM style writes when the value is unchanged.
Watch out: —
Seen in DuckStation: `GL::Program::Bind()` keeps a static `s_last_program_id` and returns immediately if the program is already bound. The drawing area is only re-applied when `left/top/right/bottom` actually change. A pair of `ResetGraphicsAPIState()`/`RestoreGraphicsAPIState()` calls sets up GL state once, before and after UI drawing, so it is not reset on every draw. 4fa79f150 2019-10 “GL: Eliminiate most redundant state setting calls at draw time”

### duckstation-005 — Group draw constants in one buffer and upload it only when dirty
Lens: gpu-rendering · Source: DuckStation
Principle: Setting many separate uniforms costs one API call each. Put them in one struct that mirrors a uniform/constant buffer, mark it dirty when a field changes, and upload it once before the next draw that needs it.
Why it is faster: —
Applies elsewhere: per-frame and per-material constant buffers in D3D11/Vulkan, React-style dirty flags before committing a render, batching config changes into one RPC.
Watch out: —
Seen in DuckStation: Individual `glUniform` calls were replaced by a `UBOBlock` (`u_pos_offset`, `u_texture_window_mask/offset`, alpha factors). It is filled into `m_batch_ubo_data` and uploaded from a stream buffer only when `m_batch_ubo_dirty` is set. The same scheme was extended to all shaders and became the D3D11 constant-buffer path. 91c99f022 2019-11 “GPU/HW: Use uniform blocks for batch rendering”

### duckstation-006 — Track a dirty rectangle instead of invalidating everything
Lens: laziness · Source: DuckStation
Principle: When a cached copy of a large 2D resource goes stale, record the bounding box of what was written. Refresh the copy only if a reader's area overlaps that box, and then copy just that region.
Why it is faster: —
Applies elsewhere: partial screen updates in VNC/RDP and e-ink displays, browser paint invalidation regions, tile caches in map viewers.
Watch out: —
Seen in DuckStation: The hardware renderer samples textures from a separate copy of VRAM, because it cannot read the framebuffer it is rendering to. Fills, writes, copies and the drawing area are merged into `m_vram_dirty_rect.Include(...)`. The copy is refreshed only when `GetTexturePageRectangle()` or `GetTexturePaletteRectangle()` `Intersects` the dirty area, and only the dirty sub-rectangle is copied. Later, rectangles were tracked by the area each primitive actually covered (clamped to the drawing area) instead of the whole drawing area, which fixed a performance regression. afbec85f8 2019-11 “GPU/HW: Track dirty area of VRAM shadow texture via drawing area” · da51d49d1 2019-11 “GPU/HW: Track VRAM fills/writes in dirty rectangle instead of invalidating” · cc7483ad5 2019-11 “GPU/HW: Only copy the dirty area to the VRAM read texture” · 9ac7b002e 2020-02 “GPU/HW: Improve dirty rectangle tracking”

### duckstation-007 — Stream uploads through a ring buffer instead of temporary allocations
Lens: gpu-rendering · Source: DuckStation
Principle: Allocating a temporary array for each upload and then handing it to the driver costs an allocation plus an extra copy. Write directly into a ring buffer the GPU can read: persistent-mapped where supported, or with discard/no-overwrite maps. Rewind only when the buffer wraps.
Why it is faster: —
Applies elsewhere: dynamic vertex buffers in game engines, `GL_MAP_PERSISTENT_BIT` and `D3D11_MAP_WRITE_NO_OVERWRITE` streaming, WebGPU staging belts, network ring buffers such as io_uring and DPDK.
Watch out: —
Seen in DuckStation: CPU→VRAM writes used to fill a `std::vector<u32> rgba_data` and then call `glTexSubImage2D`. They now `Map()` a `GL::StreamBuffer`, write pixels in place, and pass the buffer offset. The GL stream buffer gained a persistent-mapped (buffer storage) implementation. The D3D11 version uses `WRITE_DISCARD` at position 0 and `NO_OVERWRITE` otherwise, after checking `MapNoOverwriteOnDynamicConstantBuffer`. Profiling showed `glBufferSubData` was slower everywhere except NVIDIA, so the fallback became a `glBufferData` variant. 2b17cfd36 2019-11 “GPU: Use streaming buffers for CPU->VRAM transfers” · 26c22f003 2019-11 “Common: Persistent-mapped stream buffer implementation” · 99d42c07c 2020-03 “D3D11/StreamBuffer: Use WRITE_DISCARD at position 0” · 2249b873f 2020-03 “D3D11/StreamBuffer: Check feature support for MAP_NO_OVERWRITE before using” · ea1066f76 2020-06 “GL/StreamBuffer: Use glBufferData() on all drivers”

### duckstation-008 — Upload raw data and convert it on the GPU
Lens: gpu-rendering · Source: DuckStation
Principle: Converting pixel formats on the CPU doubles the bytes sent and runs serially. Upload the compact source format into a texel or storage buffer and let a shader unpack it while it writes the destination. That way the CPU never needs to touch the data again.
Why it is faster: —
Applies elsewhere: uploading compressed or indexed images and expanding them in a shader, GPU-side YUV unpacking for video, compute-shader decoding of packed vertex data.
Watch out: —
Seen in DuckStation: VRAM writes were expanded from RGBA5551 to RGBA8 on the CPU. The 16-bit words now go into an `R16UI` texture buffer, or an SSBO when texel buffers are too small or missing, and `GenerateVRAMWriteFragmentShader()` unpacks them per fragment. Writes that wrap around VRAM edges used to fall back to a CPU round trip. They now pass `u_end_x/u_end_y` to the shader, which handles the wrap. 414346935 2019-11 “GPU: Use texture buffer/fragment shader for VRAM writes” · 08ef8c1e8 2020-06 “GPU/HW: Support SSBOs instead of texture buffers for VRAM writes” · 209d7c67e 2020-07 “GPU/OpenGL: Prefer SSBOs over small texel buffers+subimage” · d1a2ebd8f 2020-07 “GPU/HW: Eliminate CPU round trip on oversized VRAM writes”

### duckstation-009 — Shrink data on the GPU before reading it back
Lens: gpu-rendering · Source: DuckStation
Principle: GPU→CPU readbacks stall the pipeline and are limited by bus bandwidth. Run a small pass that encodes the result into the final compact format first, so fewer bytes cross the bus and the CPU has no conversion loop to run.
Why it is faster: —
Applies elsewhere: encoding screenshots or video frames to NV12 on the GPU before download, reducing to a histogram or a single value in compute before reading it, packing depth into 16 bits for picking.
Watch out: —
Seen in DuckStation: For VRAM readbacks, the renderer now draws the upscaled RGBA8 VRAM into a `m_vram_encoding_texture`, packing pairs of RGBA5551 pixels, and copies that to a staging texture. The result lands in a CPU-side `m_vram_shadow`, and later reads are served from that shadow. 3998b9684 2019-11 “GPU/HW: GPU-based RGBA8->RGB5551 conversion for readbacks”

### duckstation-010 — Process buffers where they lie instead of copying them first
Lens: memory-layout · Source: DuckStation
Principle: Copying data into a staging buffer "to be safe" costs bandwidth and cache space for every byte. Let the consumer read straight from the producer's memory (a pointer plus a length). Fall back to buffering only for the rare partial or leftover piece.
Why it is faster: —
Applies elsewhere: `std::span`/`string_view` parsers, zero-copy network stacks (`sendfile`, `splice`), Node streams that parse `Buffer` slices without concatenating them.
Watch out: —
Seen in DuckStation: GPU DMA used to push every word into `m_GP0_command` and then parse it. `HandleGP0Command(command_ptr, command_size)` now parses straight from the DMA source. Only a partial trailing command is copied into `m_GP0_buffer`. DMA itself now reads and writes RAM through `Bus::GetRAM()` without an intermediate copy (“Easy 5% performance improvement”). VRAM readback and save states read `m_vram_ptr` directly instead of filling temporary `std::vector`s. 7448cbaf9 2019-10 “GPU: Copy-less processing from DMA” · 9ea7a8418 2019-11 “GPU: Eliminate temporary buffer when reading back” · 07e8ddcae 2020-04 “DMA: Elide intermediate copy where possible”

### duckstation-011 — Present a sub-rectangle of the source instead of copying it out
Lens: gpu-rendering · Source: DuckStation
Principle: If the next stage can read a region of an existing resource, pass it the source plus a rectangle instead of copying that region into a new resource. Handle vertical flips with a negative height.
Why it is faster: —
Applies elsewhere: texture atlases with UV rects, `drawImage(src, sx, sy, sw, sh, ...)` in canvas, ffmpeg frame cropping by changing pointers and strides.
Watch out: —
Seen in DuckStation: For plain 15-bit, non-interlaced output, scanout used to `CopySubresourceRegion`/`glCopyImageSubData` the visible area into `m_display_texture`. It now calls `SetDisplayTexture(m_vram_texture, offset_x, offset_y, width, -height, ...)` and the host display samples VRAM directly. a397979d7 2019-11 “GPU/HW: Elide buffer copy in scanout and fix flipped display”

### duckstation-012 — Never read back from write-combined memory
Lens: memory-layout · Source: DuckStation
Principle: Mapped GPU upload buffers are usually write-combining and uncached. Writes are fast, but each read goes over the bus and is extremely slow. Keep any value you need to read again in normal memory and only write into the mapped buffer.
Why it is faster: —
Applies elsewhere: filling a mapped Vulkan/D3D upload heap, framebuffer memory in embedded displays, memory-mapped device registers. Never do `*dst++ = dst[-1]` on them.
Watch out: —
Seen in DuckStation: Duplicating a vertex for strip restarts used `memcpy(ptr, ptr - 1, ...)`, which read back from the mapped vertex buffer. `AddVertex()` now builds the vertex in `m_batch_last_vertex` (ordinary RAM), copies it out, and duplicates from that local. Culling rewinds the pointer instead of re-reading. The commit message reports “+100fps in recompiler modes”. 504510c24 2020-01 “GPU: Fix reading of write-combined GPU buffers”

### duckstation-013 — Use the dedicated copy primitive instead of a general-purpose path
Lens: gpu-rendering · Source: DuckStation
Principle: APIs often offer both a general operation (blit or draw with scaling, filtering and state) and a narrow one (raw copy). The narrow one skips state changes and validation, and can use DMA engines.
Why it is faster: —
Applies elsewhere: `memcpy` over element-by-element loops, `copyTextureToTexture` in WebGPU instead of a fullscreen quad, `CopyResource` in D3D.
Watch out: —
Seen in DuckStation: VRAM→VRAM copies used `glBlitFramebuffer`, which required disabling and re-enabling the scissor test and binding a read FBO. They now use `glCopyImageSubData` (GL 4.3, or `EXT_copy_image`), and keep the blit only as a fallback. 771a15156 2019-11 “GPU/OpenGL: Use CopyImageSubData for VRAM copies”

### duckstation-014 — Cache compiled shaders on disk, keyed by source hash
Lens: io · Source: DuckStation
Principle: Compiling shaders at startup or when settings change is slow and causes hitches. Hash the source, store the compiled bytecode or program binary under that hash, and load it on later runs. Shaders that never change can be precompiled into the executable.
Why it is faster: —
Applies elsewhere: Vulkan `VkPipelineCache`, Chrome's GPU program cache, ccache/sccache for C++ builds, Python `.pyc` files.
Watch out: —
Seen in DuckStation: `D3D11::ShaderCache` and `GL::ShaderCache` keep an index file of `{source_hash_low/high, length, file_offset, blob_size}` entries plus a blob file. GL uses `glGetProgramBinary`/`glProgramBinary` and recompiles if loading the binary fails. The display shaders that never change became precompiled bytecode headers (`display_vs.hlsl.h`). The Vulkan pipeline cache is only rewritten when its size changes, which avoids rewriting it on every exit. 82da9a450 2020-01 “D3D11: Add a shader bytecode cache class” · a83cad587 2020-01 “GPU/D3D11: Use shader bytecode cache” · 784a5c789 2020-02 “Qt: Use precompiled D3D11 display shaders” · ef4808bbd 2020-04 “GL/Program: Add program binary interface” · ca2f691db 2020-04 “Common: Add GL Shader Cache class” · d0be5618e 2020-04 “GPU/HW/OpenGL: Use shader cache” · 9265dd72b 2020-06 “Vulkan/ShaderCache: Skip writing pipeline cache when size matches”

### duckstation-015 — Use the modern presentation model and allow tearing when unsynced
Lens: platform · Source: DuckStation
Principle: Legacy blit-model presentation adds a copy and extra compositor latency. Flip-model swap chains hand the buffer to the compositor directly. When vsync is off, the allow-tearing flag lets the frame show immediately instead of waiting for the compositor.
Why it is faster: —
Applies elsewhere: games and video players on Windows (DXGI flip-discard), `VK_PRESENT_MODE_MAILBOX/IMMEDIATE`, low-latency or "desynchronized" canvas contexts in browsers.
Watch out: —
Seen in DuckStation: The swap chain moved from `DXGI_SWAP_EFFECT_DISCARD` to `FLIP_DISCARD`, falling back if creation fails. If `IDXGIFactory5::CheckFeatureSupport(DXGI_FEATURE_PRESENT_ALLOW_TEARING)` succeeds, the swap chain is created with `ALLOW_TEARING` and `Present(0, DXGI_PRESENT_ALLOW_TEARING)` is used when vsync is off. 452417257 2019-11 “Frontend: Use flip model swap chains in D3D” · 46033898b 2020-01 “Frontends: Use DXGI_PRESENT_FLAG_ALLOW_TEARING for presentatation”

### duckstation-016 — Keep debug layers and validation out of release runs
Lens: platform · Source: DuckStation
Principle: API debug callbacks, validation layers and debug devices check every call and can cut throughput several-fold. Turn them off by default and make them opt-in.
Why it is faster: —
Applies elsewhere: Vulkan validation layers, `D3D11_CREATE_DEVICE_DEBUG`, React development builds, verbose ORM query logging in production.
Watch out: —
Seen in DuckStation: The OpenGL display had `KHR_debug` output hard-enabled (`#if 1`). It was disabled “for speed”. The D3D11 device was created with `debug = true` and break-on-warning. That became `false`, and later a user setting (“Use Debug Device”). c1e9617c7 2019-12 “Frontend: Disable KHR_debug for speed” · 46033898b 2020-01 “Frontends: Use DXGI_PRESENT_FLAG_ALLOW_TEARING for presentatation”

### duckstation-017 — Translate hot guest code into native blocks once and cache them
Lens: jit-interpreters · Source: DuckStation
Principle: An interpreter pays fetch, decode and dispatch on every instruction. A block JIT pays that cost once per basic block and emits straight-line native code. Code pages are tagged, so writes to them invalidate the affected blocks.
Why it is faster: —
Applies elsewhere: JavaScript and JVM JITs, LuaJIT, regex engines that compile patterns to machine code, eBPF JIT in the Linux kernel, QEMU TCG.
Watch out: —
Seen in DuckStation: `CPU::CodeCache` compiles blocks with xbyak (x64) and later vixl (AArch64) into a `JitCodeBuffer`. RAM pages that contain code are flagged in `m_ram_code_bits`, so a store to a flagged page calls `InvalidateBlocksWithPageIndex()`. Instructions were added over the next weeks (ALU, loads/stores, branches, GTE). By mid-2020 the recompiler was the default mode. Today's code cache uses page-indexed LUTs (`g_code_lut`, `s_block_lut`) for dispatch and fastmem with fault-handler backpatching for loads and stores. 1d6c4a3af 2019-11 “CPU: Basic recompiler implementation for x64 (lui, ori, addiu)” · 7aafaeacb 2019-11 “CPU/Recompiler: Implement lb/lbu/lh/lhu/lw/sb/sh/sw instructions” · eec37df1e 2020-06 “Settings: Make the recompiler the default CPU mode”

### duckstation-018 — Revalidate cached artefacts by content before throwing them away
Lens: laziness · Source: DuckStation
Principle: A write to an area marked as cached does not always change the data. Mark the cache entry invalid and, when it is next used, compare the stored input against the current input. Rebuild only if they differ.
Why it is faster: —
Applies elsewhere: HTTP ETag/If-None-Match, Make-style rebuilds that check content hashes instead of timestamps, webpack's persistent cache checksums.
Watch out: —
Seen in DuckStation: Writes to a code page used to flush and delete every block on it. Now `InvalidateBlocksWithPageIndex` only sets `block->invalidated`. When the block runs next, `RevalidateBlock()` compares each `cbi.instruction.bits` with memory and recompiles only on a mismatch. This matters for games that write data next to their code. 9e82afac7 2019-11 “CPU/Recompiler: Support block revalidation instead of flushing”

### duckstation-019 — Link translated blocks to skip the dispatcher
Lens: jit-interpreters · Source: DuckStation
Principle: Going back to a central dispatcher after every block means a hash lookup and an indirect branch. Remember each block's observed successors and jump straight to them. A loop that branches back to its own start needs no lookup at all.
Why it is faster: —
Applies elsewhere: trace linking in QEMU and Dynamo, inline caches in JS engines, threaded code in Forth interpreters.
Watch out: —
Seen in DuckStation: The execution loop first checked whether `next_block_key == block->key` and, if no interrupt was pending, did `goto reexecute_block`. Later, every block kept `link_successors`/`link_predecessors` lists, so a block could be found from its predecessor without a map lookup. Unlinking a block on invalidation keeps those lists correct. 9e3bb6221 2019-11 “CPU/CodeCache: Fast path for self-linking blocks” · b8de55b9b 2019-11 “CPU/Recompiler: Implement simple block linking”

### duckstation-020 — Move cold paths out of line, into a separate code region
Lens: jit-interpreters · Source: DuckStation
Principle: Exception and slow-path code mixed into the hot path wastes instruction-cache space and adds taken branches. Emit it into a separate "far" area and reach it with one conditional branch that is rarely taken. Keep that area close enough for short PC-relative jumps.
Why it is faster: —
Applies elsewhere: GCC/Clang `__builtin_expect` with `.text.unlikely` sections, `[[unlikely]]`, profile-guided hot/cold splitting, V8 deopt stubs.
Watch out: —
Seen in DuckStation: `JitCodeBuffer` gained a far-code region (32 MB near plus 32 MB far). Exception exits after loads and stores and other rare paths are written there, “Keeps the hot path nice and clean”. On AArch64, the jumps to far code changed from `mov reg, imm64; br reg` to a single PC-relative `EmitBranch()`. 11966e4ca 2019-11 “CPU/Recompiler: Write exception exits to far code buffer” · 620284f7d 2019-12 “CPU/Recompiler/AArch64: Use PC-relative branches to far code”

### duckstation-021 — Keep guest state in host registers and write it back lazily
Lens: jit-interpreters · Source: DuckStation
Principle: A naive translator loads guest registers from memory and stores them back for every instruction. A register cache keeps values in host registers across instructions and writes back only dirty values, at points where something external can observe them. Saving and restoring the cache state lets a side path run without forcing a flush on the main path.
Why it is faster: —
Applies elsewhere: register allocation in compilers, write-back CPU caches, dirty-field tracking in ORMs that write only changed columns.
Watch out: —
Seen in DuckStation: The register cache now tracks `pc` too, which “Reduces the number of loadstores after each instruction”. The current PC is synced only when an instruction can actually raise an exception: `GenerateExceptionExit` passes `epc` as a constant. `PushState()`/`PopState()` let branch and exception side paths flush registers without disturbing the hot path. The interpreter load-delay slot is checked only when `m_load_delay_dirty` is set. 6c6bf8714 2019-11 “CPU/Recompiler: Only check interpreter load delay when it's dirty” · 20c7aaf74 2019-12 “CPU/Recompiler: Support pushing/popping the register cache state” · aa52dbfeb 2019-12 “CPU/Recompiler: Use register cache for managing pc” · d118377c9 2019-12 “CPU/Recompiler: Reduced forced PC flushes”

### duckstation-022 — Emit three-operand forms to avoid extra moves
Lens: jit-interpreters · Source: DuckStation
Principle: On ISAs that have `op dst, src1, src2`, copying the left operand into the destination first wastes an instruction. Use the three-operand encoding whenever the source is already in a register.
Why it is faster: —
Applies elsewhere: x86 `lea` for non-destructive adds, AVX VEX encodings over SSE two-operand forms, ARM/RISC-V code generators.
Watch out: —
Seen in DuckStation: `AddValues`/`SubValues`/shift helpers used to do `EmitCopyValue(res, lhs); EmitAdd(res, rhs)`. They now emit `EmitAdd(res, lhs.host_reg, rhs)` when `lhs.IsInHostRegister()`, which “eliminates the extra move on ARM”. 44c76f3bf 2019-12 “CPU/Recompiler: Support three-operand basic operations”

### duckstation-023 — Inline the common case and call out only for special cases
Lens: jit-interpreters · Source: DuckStation
Principle: Calling a generic helper for every access costs argument setup, a call and spills. Generate direct code for the common case (a plain field load or store) and call the helper only for the few cases that need special logic.
Why it is faster: —
Applies elsewhere: JIT intrinsics for `Math.*`, inlined property access with a slow-path call in V8, fast-path/slow-path splitting in allocators such as tcmalloc's thread cache.
Watch out: —
Seen in DuckStation: GTE (coprocessor 2) instructions used to make the recompiler fall back to the interpreter. They became native code that calls thunks. Register reads now emit `EmitLoadCPUStructField(offsetof(Core, m_cop2.m_regs.r32[index]))` for most indices. Only IRGB/ORGB reads and a few sign-extending writes still call `Thunks::ReadGTERegister`/`WriteGTERegister`. a5afb250e 2019-12 “CPU/Recompiler: Don't require fallback for GTE instructions” · 37933e967 2019-12 “CPU/Recompiler: Read GTE registers directly from struct”

### duckstation-024 — Pick the cheaper equivalent instruction
Lens: jit-interpreters · Source: DuckStation
Principle: Several instruction sequences often compute the same thing with different latency, throughput, size or macro-fusion behaviour. Choose the cheaper form, for example `test`+`jz`, which fuses into one micro-op, over `bt`+`jnc`.
Why it is faster: —
Applies elsewhere: `xor reg,reg` to zero a register, `lea` for small multiplies, compiler peephole passes, hand-tuned SIMD kernels.
Watch out: —
Seen in DuckStation: The x64 `EmitBranchIfBitClear` emitted `bt reg, bit; jnc`. For bits below 8 it now emits `test reg, 1<<bit; jz`, which is the same size and fuses on modern cores. b00f50c92 2020-06 “CPU/Recompiler: Emit test/jz instead of bt/jnc for bits < 8”

### duckstation-025 — Replace lock-step polling with an event scheduler
Lens: algorithms · Source: DuckStation
Principle: Updating every component after every time slice wastes work on components that have nothing to do. Instead, each component schedules its next deadline in a queue sorted by time. The main loop runs until the earliest deadline and services only the events that are due.
Why it is faster: —
Applies elsewhere: discrete-event simulation, OS timer wheels, the Node.js/libuv timer heap, game-engine coroutine schedulers.
Watch out: —
Seen in DuckStation: `System::Synchronize()` called `Execute(pending_ticks)` on GPU, timers, CD-ROM, pad, SPU, MDEC and DMA after every CPU slice. It was replaced by `TimingEvent` objects in a heap (`std::push_heap` with `CompareEvents`). The CPU downcount is set to the nearest event (`UpdateCPUDowncount()`). Today the active events live in an intrusive doubly linked list kept sorted by `m_next_run_time` (`SortEvent()`), which suits the handful of events that reschedule themselves. 1b9609ef6 2020-01 “Implement event-based scheduler instead of lock-step components”

### duckstation-026 — Count up toward a deadline instead of updating two counters
Lens: branching-fast-paths · Source: DuckStation
Principle: Maintaining both "elapsed" and "remaining" counters doubles the stores on a very hot path. Keep only the elapsed count and compare it with a target that changes rarely.
Why it is faster: —
Applies elsewhere: rate limiters that compare `now` with `next_allowed`, game loops comparing an accumulator with a frame budget, loop-trip counts compared against an end pointer.
Watch out: —
Seen in DuckStation: `AddTicks()` did `m_pending_ticks += t; m_downcount -= t;` for every instruction and memory access. Now only `m_pending_ticks` is incremented, and the loop condition is `while (m_pending_ticks < m_downcount)`. The recompiler no longer emits the downcount subtraction (“Saves a few cycles decrementing the downcount”). eeea5125f 2019-12 “CPU: Use pending ticks as downcount comparison”

### duckstation-027 — Check rare conditions only where they can arise
Lens: branching-fast-paths · Source: DuckStation
Principle: A validity check on every iteration costs a compare and a branch each time. If the bad state can only be created at a few sites, check it there, and keep a debug assertion in the hot loop.
Why it is faster: —
Applies elsewhere: validate input at API boundaries instead of in inner loops, check bounds when constructing an iterator instead of on each `operator*`, sanitize once on deserialization.
Watch out: —
Seen in DuckStation: The interpreter checked PC alignment on every instruction fetch. Only branches to a register can produce an unaligned PC, so the `AdEL` check moved into `Branch()`, and fetch keeps a `DebugAssert`. b4a6c98bb 2019-10 “CPU: Move alignment check to branch instead of fetch”

### duckstation-028 — Force-inline tiny accessors on hot paths
Lens: compiler-build · Source: DuckStation · Also: ares
Principle: Compilers sometimes decline to inline small helpers, especially across translation units or in less-optimised builds. Then a one-instruction bitfield extract costs a call. An `ALWAYS_INLINE` attribute guarantees these helpers disappear.
Why it is faster: —
Applies elsewhere: `__forceinline`/`[[gnu::always_inline]]` on vector math types, `#[inline(always)]` in Rust on newtype getters, MSVC `/Ob1` builds.
Watch out: —
Seen in DuckStation: Every `BitField` operator, `SignExtend`/`ZeroExtend`, the instruction decoders (`imm_sext32()`, `CommonOp()`) and the `CPU::Core` tick/register accessors gained `ALWAYS_INLINE`. The DebugFast configuration was set to `OnlyExplicitInline` so these still inline there. c7a7d682b 2019-10 “Common: Force inlining of BitField/SignExtend/ZeroExtend functions” · 1adb1d14a 2019-10 “Build: Enable inlining for __inline functions in DebugFast” · bedc305b6 2019-10 “CPU: Inline instruction accessor functions” · 9fa8eb239 2019-12 “CPU: Force-inline some accessor functions” • Also seen in ares (ares-024): The accuracy PPU headers converted `vram_read/write`, `oam_read`, `cgram_read`, `get_color`, `hoffset/voffset`, `add_clocks`, `run()` and others to `alwaysinline`. The changelog expects the accuracy profile to be about 7.5% faster. 39ca8a2fa 2015-05 “Update to v094r17 release.” • Also seen in ares (ares-060): All of `nall/recompiler/amd64`'s `emitter` functions (`byte`, `dword`, `rex`, `modrm`, `sib`) and its encoder instructions became `alwaysinline`, which sped up code generation. The SH-2's `call()` template moved into the header as `alwaysinline`. The Mega Drive's `Bus::read`/`write`/`waitRefresh*` changed from `inline` to `alwaysinline` "as intended". 1e4683fba 2021-07 “nall:: speed up recompiler codegen” · 99ef67b4a 2021-07 “sh2: slight speed up 'call' generation” · 3051af6e2 2022-12 “md: make sure bus functions are always inlined as intended”

### duckstation-029 — Compile diagnostics out of hot functions
Lens: compiler-build · Source: DuckStation
Principle: Even untaken logging or assertion code makes a function bigger. That can stop the compiler from inlining it and puts pressure on registers. Use `#ifdef` or debug-only macros to compile it out of release builds entirely.
Why it is faster: —
Applies elsewhere: `NDEBUG`-gated `assert`, Rust `debug_assert!`, log macros that compile to nothing below a level, such as spdlog's `SPDLOG_ACTIVE_LEVEL`.
Watch out: —
Seen in DuckStation: The interpreter's trace and log hooks in `ExecuteInstruction()` had their `#ifdef _DEBUG` commented out, which “prevented ExecuteInstruction() from being inlined”. Restoring the guard fixed the regression. `FIFOQueue` bounds checks changed from `Assert` to `DebugAssert`. 201be8aa9 2019-11 “CPU: Fix interpreter speed regression from recompiler” · 5923129ec 2020-04 “Common/FIFOQueue: Make bounds checks debug only”

### duckstation-030 — Choose build flags for throughput: LTO on, unneeded mitigations off
Lens: compiler-build · Source: DuckStation
Principle: Link-time optimisation lets the compiler inline and specialise across files. Spectre mitigations (retpolines, speculation barriers) slow down indirect branches and loads. They are worth paying for in sandboxes that run untrusted code, but not in a local application.
Why it is faster: —
Applies elsewhere: `-flto`/`/GL /LTCG`, Rust `lto = "fat"`, dropping `/Qspectre` or `-mretpoline` for trusted compute-heavy binaries, profile-guided optimisation.
Watch out: —
Seen in DuckStation: `<SpectreMitigation>false</SpectreMitigation>` was set on every project, which gave “a roughly 20-25% boost … even with the recompiler”. The CMake Windows Release build turned on `CMAKE_INTERPROCEDURAL_OPTIMIZATION`. 3daa6b469 2020-04 “Build: Explicitly disable spectre mitigations” · 0ad6adc90 2020-07 “build: Enable LTO/LTCG on Windows Release builds”

### duckstation-031 — Specialise inner loops with template flags
Lens: compiler-build · Source: DuckStation
Principle: Runtime flags tested inside a per-pixel or per-element loop add branches and block vectorisation. Make the flags template parameters, so each combination is a separate branch-free function. Choose the function pointer once per call.
Why it is faster: —
Applies elsewhere: shader permutations, `std::sort` with an inlined comparator, image-codec kernels specialised per pixel format, generic Rust functions over `const bool`.
Watch out: —
Seen in DuckStation: The software renderer's `DrawTriangle(rc, ...)` checked `rc.shading_enable`, `texture_enable` and others per pixel. It became `template<bool shading, bool texture, bool raw_texture, bool transparency, bool dithering>`. `GetDrawTriangleFunction()` returns the matching instantiation, and the same was done for rectangles and lines. Today this has grown into function tables (`DrawTriangleFunctionTable`) with a separate AVX2 build of the rasterizer. c8d3b2c54 2019-12 “GPU/Software: Use templates for render functions”

### duckstation-032 — Copy bulk transfers with a single memcpy
Lens: memory-layout · Source: DuckStation
Principle: Moving a block word by word through a general access function pays dispatch, bounds and side-effect checks on every word. Check once whether the whole range is plain memory. If so, copy it with one `memcpy` and compute the cost arithmetically.
Why it is faster: —
Applies elsewhere: `System.arraycopy` versus loops, `readv`/`writev`, batched DMA descriptors, bulk `COPY` in Postgres instead of row-by-row `INSERT`.
Watch out: —
Seen in DuckStation: DMA channels used to call `DispatchAccess` once per word. `Bus::ReadWords`/`WriteWords` now check whether the range lies in RAM and, if so, do `std::memcpy(words, &m_ram[address], 4*count)` and return `word_count + (word_count+15)/16` ticks. GPU, SPU, MDEC and CD-ROM gained matching multi-word DMA entry points. ec8c5d4bb 2019-10 “DMA: Batch multi-word transfers together”

### duckstation-033 — Use fixed-point arithmetic instead of float round trips
Lens: other · Source: DuckStation
Principle: Converting integers to float and back for a multiply costs conversions, rounding logic and branches. When the data is fixed-point already, a multiply followed by a shift (`(a*b)>>15`) is exact, cheap and needs no branches.
Why it is faster: —
Applies elsewhere: audio DSP on microcontrollers (Q15/Q31), fixed-point image filters, game physics on hardware without an FPU.
Watch out: —
Seen in DuckStation: SPU voice volume used to convert with `S16ToFloat` (a branch on the sign), multiply in float, and convert back with `FloatToS16`. It now uses `ApplyVolumeUnsaturated(sample, vol) = (sample * s32(vol)) >> 15` and a `constexpr` `Clamp16`. f3cd100b1 2019-10 “SPU: Float-less volume application”

### duckstation-034 — Use bit-scan intrinsics instead of bit loops
Lens: platform · Source: DuckStation
Principle: Counting leading zeros or ones with a loop costs up to 32 iterations, and each one is a branch. `lzcnt`/`clz` do it in one instruction. Count leading ones by inverting the value first.
Why it is faster: —
Applies elsewhere: floating-point normalisation, priority queues built on bitmaps, `log2` for bucket sizing in allocators.
Watch out: —
Seen in DuckStation: The GTE's `CountLeadingBits` (used for LZCR and division) had two 32-step loops. It is now `if (v & 0x80000000) v ^= ~0u; return v ? CountLeadingZeros(v) : 32;` using `bitutils.h` intrinsics. The commit calls it a “Super tiny micro-optimization”. d1e841f55 2020-04 “GTE: Use intrinsic CountLeadingZeros()”

### duckstation-035 — Move blocking I/O to a background thread with read-ahead
Lens: concurrency · Source: DuckStation
Principle: Synchronous file reads on the main thread stall it for the full disk latency. Have a worker thread fetch the next block you expect to need, so the main thread usually finds it already loaded.
Why it is faster: —
Applies elsewhere: asset streaming in open-world games, database buffer-pool prefetching, video players decoding ahead, `posix_fadvise(WILLNEED)`.
Watch out: —
Seen in DuckStation: `CDROMAsyncReader` took over disc access from `CDROM`. `QueueReadSector(lba)` wakes a reader thread. When the emulated drive needs the sector it waits on the result, which is normally already there. The thread is optional (`cdrom_read_thread`). A later commit only logs a warning when the wait exceeds 1 ms. 959a55527 2020-02 “CDROM: Implement asynchronous disc reading”

### duckstation-036 — Preload a whole dataset into RAM when it fits
Lens: io · Source: DuckStation
Principle: If the working set fits in memory, read it once with large sequential reads. After that, random accesses are memory lookups instead of seeks, which removes stalls on slow or compressed storage.
Why it is faster: —
Applies elsewhere: loading a SQLite database with `:memory:` or mmap, game level packs read in one go, ML datasets cached in RAM between epochs.
Watch out: —
Seen in DuckStation: `CDImage::CreateMemoryImage()` copies every sector and subchannel of any image format (including CHD) into a memory-backed `CDImage`, with a progress callback. The “Preload Image To RAM” option swaps it in when a game boots, and later also when discs change and states load. 3187d07d0 2020-07 “CDImage: Add memory CD image class” · 9496c992f 2020-07 “System: Add an option to preload CD image to RAM”

### duckstation-037 — Coalesce and debounce persistent writes
Lens: io · Source: DuckStation
Principle: Writing a file on every small change multiplies syscalls, fsyncs and SSD wear. Keep a dirty flag and write once after a quiet period, or skip the write when nothing changed. Always flush on shutdown.
Why it is faster: —
Applies elsewhere: autosave timers in editors, debounced `localStorage` writes, write-back caches, Redis AOF `everysec`.
Watch out: —
Seen in DuckStation: Memory cards used to rewrite the whole file whenever a sector write completed. Now `QueueFileSave()` arms a 5-second timing event, and `SaveIfChanged()` writes only when `m_changed` is set. The destructor also calls it. The SDL ini is saved only when `m_dirty`. Qt settings saves are pushed back by a 1-second single-shot timer. b0c846388 2020-02 “SDL: Only save settings when changes have been made” · 3c45603cb 2020-04 “MemoryCard: Batch sector writes together 5 seconds at a time” · 2fb00faa6 2020-07 “Qt: Delay settings save by one second”

### duckstation-038 — Cache derived metadata keyed by file path, size and mtime
Lens: io · Source: DuckStation
Principle: Re-parsing many files to rebuild an index on every launch is slow. Store the extracted metadata together with each file's size and modification time, and rescan only the entries whose stat data changed.
Why it is faster: —
Applies elsewhere: media library scanners (Plex, Lightroom), `git`'s index stat cache, IDE symbol indexes, the Python `__pycache__` mtime check.
Watch out: —
Seen in DuckStation: The game list opened every disc image to read the game code and region. `GameList` now writes a binary cache of entries, each with its file size and modification time (`FILESYSTEM_STAT_DATA`). On refresh it uses cached entries whose stat data still matches. c03c1451f 2020-01 “GameList: Support caching entries”

### duckstation-039 — Block idle threads instead of polling
Lens: concurrency · Source: DuckStation
Principle: A thread that loops calling "process events with timeout 0" burns a whole core while doing nothing. When idle, block on the event loop or a condition variable and wake it explicitly. If you must poll, use coarse timers so the OS can merge wakeups.
Why it is faster: —
Applies elsewhere: `epoll_wait` instead of busy loops, `requestIdleCallback`, coarse `setTimeout` clamping in browsers, Android `JobScheduler` batching.
Watch out: —
Seen in DuckStation: While paused or powered off, the Qt emulation thread called `processEvents(..., 0/16)` in a loop. It now runs `m_worker_thread_event_loop->exec()` and is woken by `wakeThread()`, which calls `quit()` on that event loop. The background controller-polling timer was changed to `Qt::CoarseTimer` to fix high idle CPU usage. 1c2c4c848 2020-01 “Qt: Fix emulation thread using 100% CPU when idle” · dac114fa4 2020-04 “Qt: Use coarse timer for background controller polling”

### duckstation-040 — Use one ring buffer between producer and consumer, and notify outside the lock
Lens: concurrency · Source: DuckStation
Principle: A queue of fixed-size buffers adds latency, because data waits until a whole buffer is full, and it needs bookkeeping. A single ring buffer lets the consumer take exactly what it needs. Call out to other code (callbacks, notifications) only after releasing the mutex, so the other side never blocks on it.
Why it is faster: —
Applies elsewhere: audio callbacks in PortAudio/JACK, lock-free SPSC queues, signalling a condition variable after unlocking, log appenders.
Watch out: —
Seen in DuckStation: `AudioStream` replaced its `m_buffers` queue of fixed-size buffers with one `FIFOQueue` ring (`GetWritePointer()`, `GetContiguousSpace()`), which reduced minimum latency. In `EndWrite`, the `PushRange` now happens inside a scoped lock and `FramesAvailable()` is called after it is released. 531c3ad5f 2020-06 “AudioStream: Replace buffer queue with ring buffer” · 444a09efd 2020-06 “AudioStream: Don't call FramesAvailable() while locked”

### duckstation-041 — Initialise subsystems on first use
Lens: laziness · Source: DuckStation
Principle: Starting every subsystem at launch costs startup time and resources, even for features that are never used. Initialise each one the first time it is needed, and keep a reference count so you can shut it down cleanly.
Why it is faster: —
Applies elsewhere: lazy imports in Python, `React.lazy`, on-demand DLL loading, Java holder-class singletons.
Watch out: —
Seen in DuckStation: `SDL_Init(VIDEO|AUDIO|GAMECONTROLLER|HAPTIC)` at startup was replaced by `FrontendCommon::EnsureSDLInitialized()`. The audio stream and controller interface each call `SDL_InitSubSystem` for what they need, and `SDL_QuitSubSystem` when they are done. 06f4d7263 2020-02 “FrontendCommon: Use SDL_InitSubSystem() for lazy initialization”

### duckstation-042 — Make string literals free: no copy, no refcount
Lens: allocation · Source: DuckStation
Principle: Wrapping a literal in a ref-counted string object usually costs an allocation, a copy, and atomic increments and decrements. Mark static data as read-only so copies just share the pointer and never touch a refcount. Drop the atomics if the type is never shared across threads.
Why it is faster: —
Applies elsewhere: `std::string_view` and `constexpr` strings, interned strings in JVMs, Rust `&'static str`/`Cow`, Swift's immortal string literals.
Watch out: —
Seen in DuckStation: `StaticString` was a class that built a `StringData` at runtime. It became a macro producing a `static constexpr String::StringData` with `ReadOnly = true`. Copy constructors skip `StringDataAddRef` when the data is read-only, and `ReferenceCount` changed from `std::atomic<s32>` to a plain `s32`. 86094b808 2020-02 “Common/String: Don't copy StaticStrings when not writing”

### duckstation-043 — Apply targeted updates instead of re-running global reconfiguration
Lens: other · Source: DuckStation
Principle: Routing a frequent UI control through a generic "settings changed" path can rebuild or re-check many subsystems on every tick of the control. Send that control's value straight to the component that uses it.
Why it is faster: —
Applies elsewhere: fine-grained reactive signals instead of a full re-render, PATCH instead of PUT, hot-reloading one module instead of restarting the app.
Watch out: —
Seen in DuckStation: The volume slider was bound with `BindWidgetToIntSetting`, so every change went through the full settings-apply path and slowed emulation while dragging. It now stores the value and calls `QtHostInterface::setAudioOutputVolume()`, which sends one queued call to the worker thread to run `m_audio_stream->SetOutputVolume()`. b2595a3b3 2020-07 “Qt: Fix volume slider slowing down emulation”

### duckstation-044 — Raise OS timer resolution for sleep-based pacing
Lens: platform · Source: DuckStation
Principle: On Windows the default timer granularity is about 15.6 ms, so `Sleep(1)` can oversleep by a whole frame. Request 1 ms resolution while the application is pacing frames, and release it afterwards.
Why it is faster: —
Applies elsewhere: media players and games calling `timeBeginPeriod`, high-resolution waitable timers (`CREATE_WAITABLE_TIMER_HIGH_RESOLUTION`), `nanosleep` with `SCHED_FIFO` on Linux for audio threads.
Watch out: —
Seen in DuckStation: `SDLHostInterface` calls `timeBeginPeriod(1)` in its constructor and `timeEndPeriod(1)` in its destructor, “Hopefully will prevent FPS fluctuations due to sleep variation”. This later moved to the base class behind an “Increase Timer Resolution” option. 55550798e 2019-11 “Frontend: Call timeBeginPeriod() to increase timer resolution”

### duckstation-045 — Order struct fields to remove padding
Lens: memory-layout · Source: DuckStation
Principle: The compiler pads each field to its alignment, so interleaving small and large fields wastes bytes and spreads hot data over more cache lines. Group fields by size, largest first, and keep fields used together next to each other.
Why it is faster: —
Applies elsewhere: `pahole`-driven reordering in the Linux kernel, Go `fieldalignment` lint, Rust's automatic field reordering, packing ECS components.
Watch out: —
Seen in DuckStation: In `GPU::DrawMode`, the raw `mode_reg`/`palette_reg`/`texture_window_value` fields were moved ahead of the decoded byte fields, which tightened the struct the renderer reads on every primitive. f5d65448d 2019-12 “GPU: Improve structure alignment”

### duckstation-046 — Dispatch through a table indexed by address instead of a hash map
Lens: jit-interpreters · Source: DuckStation · Also: Flycast
Principle: When the keys are dense integers in a known range, such as program counters, a flat array of function pointers indexed by `key >> shift` replaces a hash lookup with one load and an indirect call. Fill empty slots with a "compile and install" stub so the miss path needs no branch either.
Why it is faster: —
Applies elsewhere: opcode dispatch tables in interpreters, syscall tables, URL router tries flattened into arrays, interrupt vector tables on microcontrollers.
Watch out: —
Seen in DuckStation: The recompiler used to look up the next block in a `std::unordered_map<u32, CodeBlock*>`. `s_fast_map` now has one `HostCodePointer` per word of RAM and BIOS, and each slot starts out pointing at `FastCompileBlockFunction`. The loop body became `s_fast_map[GetFastMapIndex(pc)]();`, and the commit message reports "up to 15%". Today it is a two-level, cache-line-aligned `g_code_lut` with function pointers pre-offset so the caller does not need to subtract. b1377fe0d 2020-08 “CPU/Recompiler: Add a fast block lookup array” • Also seen in Flycast (flycast-055): The dispatchers find the next block with `ubfx w1, w29, #1, #24` (32 MB RAM) and `ldr x0, [x2, x1, lsl #3]; br x0`. The ARM32 JIT uses the same idiom (`UBFX(r1, r4, 1, 24)`). A later commit sizes `FPCB_SIZE` and the ARM7 `EntryPoints[ARAM_SIZE_MAX/4]` for the maximum RAM so the mask (`and $0x7FFFFC`) is fixed. bfbb01d47 2015-08 “naomi: 32MB-aware block lookups from arm-jit” · 75a04d6d2 2019-01 “arm64 dynarec: implement tail calling and block linking” · 91cfd4b2f 2019-01 “Reserve and allocate maximum RAM/VRAM/ARAM in all cases” • Also seen in Flycast (flycast-205): `Sh4RCB` places `void* fpcb[FPCB_SIZE]` directly before the padded `Sh4Context`, so the dynarec's context register reaches the table at a negative offset. Lookup is `#define FPCA(x) ((DynarecCodeEntryPtr&)p_sh4rcb->fpcb[(x>>1)&FPCB_MASK])`. ARM64 indexes it with `Ubfx(w1, w29, 1, 24)` plus a scaled load. On arm32, `FPCB_PAD` is chosen so the subtraction fits in one immediate. The design dates back to the 2013 import (1c16baeb0). current code: core/hw/sh4/sh4_if.h, core/hw/sh4/dyna/blockmanager.cpp

### duckstation-047 — Write the hot dispatch loop in generated code
Lens: jit-interpreters · Source: DuckStation · Also: Flycast
Principle: A C++ loop that calls into JIT code pays for a call frame, callee-saved register spills and an indirect return on every block. Emitting the dispatcher itself as machine code lets it pin the state pointer in a register, inline the scheduler check, and jump straight from block to block.
Why it is faster: —
Applies elsewhere: threaded-code interpreters (computed goto), LuaJIT's hand-written assembly VM, eBPF's JIT prologue, trampolines in coroutine libraries.
Watch out: —
Seen in DuckStation: `CodeGenerator::CompileDispatcher()` emits the outer loop for x64 and AArch64. It loads `&g_state` into RBP once, tests `frame_done`, checks `sr & cause & 0xFF00` for interrupts, reloads `downcount` from the head timing event, then indexes the fast map with `pc` and calls it. A follow-up commit inlined the "is the head event due?" comparison into this generated loop so that servicing an interrupt no longer goes through `TimingEvents::UpdateCPUDowncount()`. a6f8dde79 2020-10 “CPU/Recompiler: Faster ASM dispatcher” · 11faf6ccf 2020-10 “CPU/Recompiler: Skip calling event update on interrupts” • Also seen in Flycast (flycast-056): `ngen_mainloop` on ARM64 and x64 was a C `while (sh4_int_bCpuRun) { do { bm_GetCode(pc)(); } while (cycle_counter > 0); ... }`. It is now inline assembly. On ARM64 it saves x19–x28 and s8–s15, sets `mov x28, ctx` and `mov w27, SH4_TIMESLICE`, and loops `bl bm_GetCodeInternal; blr x0; cmp w27,#0; b.gt`. The x64 version also gets SEH directives on Win32. a9a2aad8f 2019-01 “arm64: use register spans allocation...” · a2cd9d7af 2019-01 “x64 dynarec: use register alloc spans...” · cd4e4cbdc 2019-01 “x64 dynarec: ... fully naked main loop in win32 with proper seh directives”

### duckstation-048 — Fold rare checks into an existing countdown
Lens: branching-fast-paths · Source: DuckStation
Principle: If a hot loop already compares a counter against a limit, signal a rare condition by forcing the limit to zero. You don't need a second check on every iteration. The rare condition then costs nothing until it happens, and is handled on the slow path at the loop exit.
Why it is faster: —
Applies elsewhere: a "budget exhausted" flag in a cooperative scheduler, cancellation tokens checked through a shared deadline, game loops that end a tick early by zeroing the remaining time.
Watch out: —
Seen in DuckStation: The inner execution loop used to call `HasPendingInterrupt()` before every block. Now anything that can raise an interrupt (mtc0, rfe, IRQ line changes) calls `CheckForPendingInterrupt()`, which does `if (HasPendingInterrupt()) g_state.downcount = 0;`. The block loop then exits on its usual `pending_ticks >= downcount` test, and the interrupt is dispatched once in the outer loop. cb351a7db 2020-10 “CPU: Move interrupt check out of inner-most exec loop”

### duckstation-049 — Map guest memory with the MMU and let faults handle the exceptions
Lens: memory-layout · Source: DuckStation · Also: Flycast
Principle: Reserve a large virtual range and map the backing memory, including its mirrors, at fixed offsets. A memory access then becomes `base + address` with no range checks. Accesses to anything that isn't plain memory fall into unmapped pages, and a fault handler deals with them. The common case runs at native load/store speed.
Why it is faster: —
Applies elsewhere: WebAssembly runtimes (V8, Wasmtime) using guard pages instead of bounds checks, JVM implicit null checks, memory-mapped files, sandboxing via reserved address space.
Watch out: —
Seen in DuckStation: A `Common::MemoryArena` backs the 2MB of RAM, and `UpdateFastmemViews()` maps views of it at `0x00000000`, `0x80000000` and `0xA0000000` inside a 4GB region. Generated loads became `mov eax, dword [RMEMBASE + addr_reg]`, with the base pointer loaded into a reserved host register only for blocks that contain loads or stores. A later commit also routed plain RAM loads through this path in place of global-address loads. 7566c45f6 2020-10 “CPU/Recompiler: Implement fastmem” · 6a4a4c62d 2020-10 “CPU/Recompiler: Use fastmem instead of global for RAM loads” • Also seen in Flycast (flycast-111): `_vmem` now tries a 4 GB reservation first, then falls back to 512 MB. A table maps AICA RAM, VRAM and main RAM with their mirrors at P0/U0 offsets (`{0x0C000000, 0x10000000, MAP_RAM_START_OFFSET, RAM_SIZE, true}`), and the MMU path shares the same space. e3d95f225 2019-05 “wince: unify _vmem and vmem32. Use 4GB virtual space on 64-bit arch”

### duckstation-050 — Use a software page table where you can't remap memory
Lens: lookup-tables · Source: DuckStation
Principle: If the platform can't reserve or remap address space (32-bit hosts, restricted OSes), a flat table of per-page host pointers is the next best thing. A lookup is one shift and one load, and a null entry means "take the slow path". Keep a separate write table so read-only pages can sit in the same structure.
Why it is faster: —
Applies elsewhere: software TLBs in emulators and hypervisors, page-indexed buffer pools in databases, sparse arrays in game worlds keyed by chunk coordinates.
Watch out: —
Seen in DuckStation: LUT fastmem allocates `u8** m_fastmem_lut` with `FASTMEM_LUT_NUM_PAGES` read entries followed by the same number of write entries. `SetLUTFastmemPage()` sets the write slot to `nullptr` for protected pages. The ARM emitters do `lsr r, addr, #12`, load the page pointer and add the offset. Fastmem mode became a choice of Disabled, MMap or LUT, and the enum still exists today. bf2e38aed 2020-11 “CPU/Recompiler: Implement LUT-based fastmem”

### duckstation-051 — Emit the optimistic path, then patch it after a fault
Lens: jit-interpreters · Source: DuckStation
Principle: Generate only the fast inline sequence and record where it is. If it faults, rewrite those bytes as a jump to an out-of-line slow path. Count the faults for each site: once a site keeps failing, demote it to the slow path permanently so it stops paying for repeated traps.
Why it is faster: —
Applies elsewhere: inline caches in JavaScript engines, HotSpot uncommon traps and deoptimization, Linux kernel `alternatives` patching, speculative devirtualization with guard fallbacks.
Watch out: —
Seen in DuckStation: Each fastmem access records a `LoadStoreBackpatchInfo`, and on x64 is padded with `nop`s to at least 5 bytes so a relative `jmp` fits. Writes that fault because they hit a protected code page used to invalidate the page and carry on, every time. With `CODE_WRITE_FAULT_THRESHOLD_FOR_SLOWMEM = 10`, a site that faults 10 times is backpatched to slowmem. The commit message reports this helped Driver 2. Today the backpatch info is held in a `std::map` keyed by host PC. 7566c45f6 2020-10 “CPU/Recompiler: Implement fastmem” · 576914cd2 2020-11 “CPU/CodeCache: Backpatch code writes after they exceed 10 faults”

### duckstation-052 — Propagate constants at compile time to pick the access path
Lens: jit-interpreters · Source: DuckStation · Also: ares
Principle: If a code generator tracks which register values it can know at compile time, even speculatively, it can resolve addresses early. Constant loads from RAM or ROM become direct loads from a host pointer. Accesses predicted to hit I/O skip the fast path entirely and avoid a guaranteed fault.
Why it is faster: —
Applies elsewhere: SQL planners folding constant predicates to pick an index, C++ `constexpr` dispatch, partial evaluation in GraalVM, shader compilers specializing on uniform values.
Watch out: —
Seen in DuckStation: `GetDirectReadMemoryPointer()` maps a constant guest address to `&g_ram[...]`, the scratchpad, or BIOS, and gives its tick cost, so the JIT emits `EmitLoadGlobal` with no call. Speculative constants (`SpeculativeReadReg`/`SpeculativeWriteMemory`) also follow `lui/ori/addiu` chains and earlier stores. A load whose speculative address is MMIO then has `use_fastmem = false` and goes straight to the slow path. 0afdc04d8 2020-10 “CPU/Recompiler: Optimize constant reads (and some writes)” · b704c37e9 2020-09 “CPU/Recompiler: Implement speculative constants” • Also seen in ares (ares-118): `ConstRegs::track()` follows LUI/ORI/ADDIU/shift/logic results, with a bit mask of which registers are known. Vector loads and stores then check `constRegs.has(Rsn)`. With a known address they bounds-check it during compilation (`address > 0x0ff7`), and the LRV/LPV/LUV helpers receive a precomputed DMEM pointer and size in place of a lookup-table pointer. cbf9f79e5 2026-04 “n64: add constant tracking in RSP JIT” · fef2ab490 2026-04 “n64: in RSP, add JIT for lqv/sqv” · 219dff38a 2026-04 “n64: refactor load/store opcodes to make better use of constant addresses”

### duckstation-053 — Peephole-specialize trivially decidable instructions
Lens: jit-interpreters · Source: DuckStation · Also: ares
Principle: Look for operations whose outcome can be decided at compile time, and for cheaper encodings of the same operation. A branch comparing a register with itself is unconditional. A comparison with a negative immediate can be encoded as an add-compare of its negation.
Why it is faster: —
Applies elsewhere: compiler peephole passes (`x*2`→`x<<1`), regex engines turning literal prefixes into `memchr`, query engines dropping `WHERE 1=1`.
Watch out: —
Seen in DuckStation: `beq $zero, $zero, target`, which BIOS code uses as an unconditional jump, now compiles to `DoBranch(Condition::Always, ...)`, with no register reads and no compare. On AArch64, constants such as `-1` that don't fit `cmp`'s immediate are compared with `cmn reg, #-imm`, so the constant doesn't need to be loaded into a scratch register first. f071497be 2020-10 “CPU/Recompiler: Optimize beq zero, zero, addr to unconditional branch” · 70767dc6b 2020-08 “CPU/Recompiler/AArch64: Use cmn for negative constant comparisons” • Also seen in ares (ares-117): `BEQ r,r` becomes `emitKnownConditionalTake(1)`, and `BNE r,r` becomes not-taken. `ADDIU rt,r0,imm` becomes `mov32(mem(Rt), imm(i16))`, and a store from `r0` uses an immediate 0 in place of a register load. 26d5ac3a8 2026-04 “n64: specialize R0 source propagation in RSP JIT”

### duckstation-054 — Place generated code near its callees so short calls reach
Lens: jit-interpreters · Source: DuckStation
Principle: If the code buffer lives inside the program image, for example as a static array, every runtime helper is within ±2GB (x64) or ±128MB (AArch64). Calls can then use the short PC-relative encoding instead of loading a 64-bit address into a register and calling through it. Calling helpers directly instead of through thunks saves another hop.
Why it is faster: —
Applies elsewhere: `-mcmodel=small`, linker relaxation, putting JIT arenas near `libc` in V8/Wasmtime, branch islands/veneers on ARM.
Watch out: —
Seen in DuckStation: The JIT buffer became `alignas(...) static u8 s_code_storage[...]`, allocated in program space. The x64 emitter uses `call rel32` for helpers with no arguments, AArch64 uses `bl` when the target fits in 26 bits, and GTE functions are called directly. To make these direct calls possible, subsystems (GPU, SPU, DMA, timers and others) were turned into globals so their addresses are link-time constants. b6f871d2b 2020-07 “JIT optimizations and refactoring (#675)”

### duckstation-055 — Return error codes from slow-path helpers so the fast path stores no context
Lens: jit-interpreters · Source: DuckStation
Principle: If a helper needs caller context only when it fails, such as which instruction faulted, have it return a status and let the caller supply the context on the failure branch. The common path then has no stores, and the generated code no longer depends on where it is.
Why it is faster: —
Applies elsewhere: errno-free APIs that return `Result<T,E>`, lazy stack-trace capture, exception tables in place of per-call setjmp.
Watch out: —
Seen in DuckStation: `ReadMemoryWord(u32 pc, u32 address)` used to store `g_state.current_instruction_pc = pc` on every call. It is now `ReadMemoryWord(u32 address)` and returns `-static_cast<s64>(Exception::DBE)` on error. The generated code tests the sign bit and raises the exception with the right PC only on that branch. f6e88353e 2020-08 “CPU/Recompiler: Make generated code invariant to virtual PC”

### duckstation-056 — Make rarely needed exactness opt-in
Lens: other · Source: DuckStation
Principle: If a correctness guarantee matters only to edge cases that real workloads never hit, turn it off by default and keep it behind a setting. The guarantee is often expensive, because it forces state to be tracked on every operation. The default path then drops all of that bookkeeping.
Why it is faster: —
Applies elsewhere: `-ffast-math`, relaxed fsync in databases (`synchronous=NORMAL`), disabling denormals, sampled rather than exhaustive tracing.
Watch out: —
Seen in DuckStation: Memory-access exceptions were disabled in the recompiler by default: "no games rely on this". Loads call `UncheckedReadMemoryWord()`, which has no alignment or bus-error checks. A follow-up stopped tracking the branch-delay-slot and branch-taken state per instruction unless `cpu_recompiler_memory_exceptions` is on. 1d5f810a4 2020-08 “CPU/Recompiler: Disable memory access exceptions by default” · a16c2f546 2020-10 “CPU/Recompiler: Skip tracking some more interpreter state”

### duckstation-057 — Spend registers carefully around calls
Lens: jit-interpreters · Source: DuckStation
Principle: Values cached in caller-saved registers must be saved before any call. Write them back to memory once, ahead of the call, rather than pushing and popping them around it. Save and restore callee-saved registers in pairs where the ISA allows it (`stp/ldp`). Rewrite sequences that need many temporaries so that they work in place.
Why it is faster: —
Applies elsewhere: hand-written assembly hot loops, LLVM's spill placement, GPU shaders tuned for occupancy, embedded ISRs that minimize stacked registers.
Watch out: —
Seen in DuckStation: `RegisterCache::FlushCallerSavedGuestRegisters()` runs before each slow-path memory call. The epilogue now uses `EmitPopHostRegPair` as well as push pairs. In `lwl/lwr/swl/swr`, the new `AndValueInPlace`/`OrValueInPlace` helpers reduce the number of scratch registers needed. 60eb22537 2020-08 “CPU/Recompiler: Flush caller-saved regs before calling functions” · 7a82cae2b 2020-10 “CPU/Recompiler: Use register pairs for pop callee saved too” · cdd0a5525 2020-11 “CPU/Recompiler: Reduce register pressure of lwl/lwr/swl/swr”

### duckstation-058 — Compile the remaining fallback operations natively
Lens: jit-interpreters · Source: DuckStation · Also: Flycast
Principle: In a mixed JIT/interpreter, every operation that falls back to the interpreter flushes all cached registers and makes a generic call. Each such operation splits the surrounding code. Giving it a native code path keeps the whole block in registers.
Why it is faster: —
Applies elsewhere: PyPy/LuaJIT closing their "NYI" lists, database engines adding vectorized kernels for operators that used row-at-a-time fallbacks.
Watch out: —
Seen in DuckStation: The recompiler gained native `Compile_Divide`/`Compile_SignedDivide`, which emit the MIPS divide-by-zero semantics inline, and `Compile_LoadLeftRight`/`Compile_StoreLeftRight` for unaligned word access. Later, PGXP CPU mode stopped forcing the interpreter: the recompiler now emits `EmitFunctionCall(&PGXP::CPU_ORI, ...)` next to the native operation. 8d4216068 2020-08 “CPU/Recompiler: Implement div/divu” · 51eff82eb 2020-10 “CPU/Recompiler: Implement lwl/lwr/swl/swr” · bc021ddfd 2021-02 “CPU/Recompiler: Implement PGXP CPU mode” • Also seen in Flycast (flycast-001): In `ngen_compile_opcode` (`rec-ARM/arm_dyna.cpp`), ops such as `shop_sbc`, `shop_shld`, `shop_shad`, `shop_setpeq` and `shop_mul_u16/s16` (via `UXTH`/`SXTH`+`MUL`) and the 64-bit multiplies used to fall back to the canonical C handlers. They now emit ARM instructions directly, and the emitter gained the encodings they need: `SBC`, `UMULL`/`SMULL`, shifted-register `ADD`/`ORR`, and `MOV` with an immediate. a60498894 2013-12 “DYNAREC: Attempt to add a few more opcode to dynarec” · ae9439711 2014-01 “DYNAREC: more opcodes to dynarec” · 79e9c61ee 2014-01 “DYNAREC: More work on some opcodes” • Also seen in Flycast (flycast-161): The SH4 decoder used to return `false`, falling back to the interpreter, for `ldc/stc SR/FPSCR`, `tas.b`, `div1`, and any FPU op with `FPSCR.PR=1`. These now emit SHIL ops (`shop_readm` plus `shop_sync_sr`, and so on), and `fmov/flds/flts` stay native in double-precision mode. A new option allocates 64-bit FPU registers (`FMT_F64`) as host register pairs in the ARM, ARM64 and x64 back ends. ce674a872 2022-12 “dyna: implement missing ops: ldc/stc sr/fpscr, tas, div1” · 62085539a 2022-12 “dynarec: reg alloc 64-bit regs. avoid some interpreter fallbacks”

### duckstation-059 — Pass inputs and let the callee recompute derived values
Lens: compiler-build · Source: DuckStation
Principle: At a call boundary, every argument costs a register move and keeps a value alive across the call. If the callee can cheaply derive a value from the other arguments, such as an operation's result from its operands, recompute it inside. Call sites get shorter and register pressure drops.
Why it is faster: —
Applies elsewhere: logging APIs that take format arguments rather than preformatted strings, hash-map APIs that compute the hash internally, reducing ABI argument counts below the register limit.
Watch out: —
Seen in DuckStation: `PGXP::CPU_ADD(instr, new_value, lhs, rhs)` became `CPU_ADD(instr, lhs, rhs)`, and likewise for all ALU and shift hooks. `CPU_MULT`/`CPU_DIV` now compute hi and lo themselves. The commit message says this "generates shorter code for the recompiler". The small validation helpers were marked `ALWAYS_INLINE_RELEASE` at the same time. 209827b67 2021-02 “PGXP: Compute PSX values on demand for CPU instructions” · a722fd6b5 2021-02 “PGXP: Make mult/div compute results instead of parameters”

### duckstation-060 — Keep a sorted linked list when reschedules move items only a little
Lens: algorithms · Source: DuckStation
Principle: A binary heap pays O(log n) and a re-heapify on every change. With a handful of events whose deadlines change incrementally, a doubly linked list kept in order is cheaper: the next event is always at the head (O(1)), and a reschedule usually moves an item by zero or one position.
Why it is faster: —
Applies elsewhere: kernel timer lists for small n, LRU lists, game entity update queues, the "move toward neighbor" pattern of insertion sort on nearly sorted data.
Watch out: —
Seen in DuckStation: `std::vector<TimingEvent*>` plus `std::make_heap` was replaced by `s_active_events_head/tail`. `SortEvent()` walks backward or forward from the event's current position and relinks it. `UpdateCPUDowncount()` reads only `s_active_events_head->GetDowncount()`. The current `timing_event.cpp` still uses `active_events_head`. f187ee497 2020-08 “TimingEvents: Switch from heap to sorted linked list”

### duckstation-061 — Use plain function pointers with a context pointer instead of std::function
Lens: compiler-build · Source: DuckStation
Principle: `std::function` type-erases through a heap-allocated or small-buffer callable and an extra indirect call. The compiler can't see through it. A raw `void(*)(void*, ...)` plus a `void*` context is one indirect call with no allocation, and captureless lambdas convert to it for free.
Why it is faster: —
Applies elsewhere: C callback APIs (`qsort`, libuv), `function_ref`-style non-owning delegates, embedded HALs registering ISR handlers.
Watch out: —
Seen in DuckStation: Timing events built with `std::bind(&CDROM::ExecuteCommand, this)` now pass `[](void* param, TickCount t, TickCount late){ static_cast<CDROM*>(param)->ExecuteCommand(); }, this`. The page-fault handler `Callback` changed from `std::function` to a function pointer in the fastmem commit. 11992bde4 2021-01 “TimingEvents: Use function pointers instead of std::function” · 7566c45f6 2020-10 “CPU/Recompiler: Implement fastmem”

### duckstation-062 — Compute the exact next wake-up instead of polling on a short interval
Lens: algorithms · Source: DuckStation · Also: Flycast
Principle: An event scheduler is only as cheap as its estimate of when something will next happen. If a component returns "unknown" or a short default, it gets woken repeatedly to do nothing. Work out the real deadline, including wraparound, and the spurious wake-ups go away.
Why it is faster: —
Applies elsewhere: tickless kernels (NO_HZ), `epoll` timeouts computed from the nearest timer, cron daemons sleeping until the next job, low-power MCU RTC alarms.
Watch out: —
Seen in DuckStation: `Timers::GetTicksUntilNextInterrupt()` used to give up when the counter was already past the target, so it returned "max" and fell back to 100ms polling. It now computes `(0xFFFF - counter) + target` for the wrapped case and skips one-shot timers that have already fired. The commit message reports a significant speedup in Fromage. ae4921525 2021-02 “Timers: Better predict next event time” • Also seen in Flycast (flycast-015): `sched_chan_tick` in `sh4/modules/tmu.cpp` now asks `sh4_sched_request(tmu_sched[ch], TCOR<<shift)`, or `-1` to disable, whenever a channel is started, stopped or written. `sched_tmu_cb` raises the interrupt and returns the next period. All three channels are registered, where before only one was. The scheduler was later fixed so expired callbacks don't fire again: it compares `sh4_sched_remaining(i, fztime)` against the tick length. c5cc2f1e5 2015-01 “Document sh4_shced, fix tmu a bit” · 20f7a41fe 2015-05 “sched: Do not re-fire expired callbacks, use sh4_sched_remaing on sh4_sched_tick”

### duckstation-063 — Use coarse time slices by default and fine ones only when needed
Lens: laziness · Source: DuckStation
Principle: Interleaving two simulations at fine granularity is accurate but costs a context switch each time. Run with large slices normally, and drop to small slices only while a timing-sensitive interaction is actually happening.
Why it is faster: —
Applies elsewhere: adaptive polling (busy-poll under load, interrupts when idle), TCP Nagle and delayed-ACK trade-offs, physics engines substepping only for fast objects.
Watch out: —
Seen in DuckStation: `DMA::GetTransferSliceTicks()` returns `g_pad.IsTransmitting() ? 100 : m_max_slice_ticks`, and `GetTransferHaltTicks()` does the same for halt time. Linked-list DMAs run in big chunks, except while the controller port is mid-transfer, when they would otherwise break serial timing. This made a batch of per-game DMA overrides unnecessary. 845cd3783 2021-01 “DMA: Determine slice size based on whether pad is transmitting”

### duckstation-064 — Detect no-op requests and skip them early
Lens: branching-fast-paths · Source: DuckStation
Principle: Workloads often issue operations that change nothing: copying a region onto itself, zero-sized fills, zero opcodes. A cheap check at the entry point skips the work and the pipeline flushes or synchronization that come with it.
Why it is faster: —
Applies elsewhere: `memmove` returning when `src == dst`, React bailing out on equal state, `UPDATE` statements that skip rows whose values are unchanged, file-sync tools comparing mtimes.
Watch out: —
Seen in DuckStation: The interpreter returns at once for `inst.bits == 0` (nop). `GP0(02h)` fills with zero width or height are skipped. VRAM copies with `src == dst` and no mask bit are skipped, along with the `FlushRender()` they forced. Most games issue such a copy every frame. `GP1(05h)` now updates only the display parameters, not the whole CRTC configuration. daa72aba1 2020-08 “CPU: Skip executing nop instructions” · 61a496038 2020-10 “GPU: Skip zero-width or height fills” · c03c4cb95 2021-02 “GPU: Elide VRAM copies when they will have no effect” · a1453ee8a 2020-10 “GPU: Skip full CRTC update on GP1(05h)”

### duckstation-065 — Precompute derived masks when state is written, not when it is read
Lens: lookup-tables · Source: DuckStation
Principle: If a configuration value is written rarely but read for every pixel or element, do the arithmetic in the setter. Store the final AND and OR masks, so the hot path is `(x & and_mask) | or_mask`.
Why it is faster: —
Applies elsewhere: precomputed CSS transforms, cached compiled regexes, baking blend factors into constants, precomputed CRC tables.
Watch out: —
Seen in DuckStation: Setting the texture window used to store the raw mask and offset, and every sample in both the shader and the software renderer computed `(x & ~(mask*8)) | ((offset & mask)*8)`. `texture_window_and_x/or_x` are now computed once when `GP0(E2h)` is written, and the shader becomes `(coords.x & u_texture_window_and.x) | u_texture_window_or.x`. cacf7a7cc 2020-09 “GPU: Precompute texture window expression [SAVEVERSION+]”

### duckstation-066 — Pad buffers with history so boundary checks disappear
Lens: branching-fast-paths · Source: DuckStation
Principle: A filter that reads `i-1..i-3` needs a branch at the start of each block to fetch samples from the previous one. Instead, reserve those slots at the front of the buffer and copy the tail of the previous block into them. Indexing becomes unconditional.
Why it is faster: —
Applies elsewhere: ghost cells in stencil computations, FIR filter delay lines, convolution with padded image borders, ring buffers that mirror a few bytes past the end.
Watch out: —
Seen in DuckStation: `SPU::Voice::SampleBlock(index)` had `if (index < 0) return previous_block_last_samples[index+3];`. Now `current_block_samples` is `3 + 28` entries long. Decoding a block first copies the last three samples to slots 0–2, and the Gaussian interpolation reads `current_block_samples[s - 3]` directly. 245edd94d 2020-10 “SPU: Eliminiate conditional in interpolation”

### duckstation-067 — Force-inline hot leaf helpers and keep cold paths out of line
Lens: compiler-build · Source: DuckStation
Principle: Inlining small helpers called per element removes call overhead and lets the compiler optimize across what were call boundaries. The inverse matters too: forcing rarely used paths inline bloats the hot function and wastes instruction cache. Mark each side explicitly rather than relying on heuristics.
Why it is faster: —
Applies elsewhere: `__attribute__((always_inline))` / `noinline,cold`, Rust `#[inline(always)]`/`#[cold]`, Kotlin `inline fun`, splitting error handling into separate functions in Go.
Watch out: —
Seen in DuckStation: `SPU::ExecuteVoice`, the GTE helpers (`TruncateAndSetMACAndIR`, `PushSXY`, `UNRDivide`…) and the software renderer's `GetPixel/SetPixel` became `ALWAYS_INLINE`/`ALWAYS_INLINE_RELEASE`. That macro now expands to `inline` in debug builds as well. In the other direction, the rarely accessed EXP1/EXP2 bus handlers lost their `ALWAYS_INLINE`. 325f83065 2020-10 “SPU: Always inline SPU::ExecuteVoice” · d4f52f1ec 2021-01 “GTE: Force inline a few of the helper functions” · b2ef93177 2020-10 “Minor gpu_sw optimizations and warning fixes” · ca723d699 2020-08 “Make ALWAYS_INLINE_RELEASE specify inline in debug too” · f832dca97 2021-01 “Bus: Don't force inline EXP1/EXP2 access”

### duckstation-068 — Hoist rarely true or invariant work out of the inner loop
Lens: branching-fast-paths · Source: DuckStation
Principle: If a check inside a per-sample or per-block loop is true only once, or depends on a mode that can't change mid-loop, move it outside. Guard it with a single test, or split the loop into one specialized version per mode.
Why it is faster: —
Applies elsewhere: loop unswitching in compilers, separate render loops per blend mode, checking the "logging enabled" flag once per batch.
Watch out: —
Seen in DuckStation: SPU key-on/key-off bit-shifting ran for all 24 voices on every output sample. It now runs only `if (i == 0 && (m_key_off_register != 0 || m_key_on_register != 0))`. The cached interpreter's block loop also lost its per-block `if (s_use_recompiler)` branch once the recompiler got a loop of its own. 4e802e90c 2021-01 “SPU: Move voice key off/on outside sample loop” · d3f530f96 2020-08 “CPU/CachedInterpreter: Very slight optmization”

### duckstation-069 — Merge duplicated branches into one data-dependent expression
Lens: branching-fast-paths · Source: DuckStation
Principle: If two branches run the same arithmetic with different constants, select the constant (which compiles to a cmov) and run the arithmetic once. Compute min and max as a pair rather than with two independent comparisons.
Why it is faster: —
Applies elsewhere: `rate = isPal ? 50 : 60` before a shared formula, `std::minmax`, SIMD blend/select instead of an if/else per lane.
Watch out: —
Seen in DuckStation: `GPU::SystemTicksToCRTCTicks` had two copies of the 64-bit multiply, divide and modulo for NTSC and PAL. It is now `mul *= !m_console_is_pal ? 715909 : 709379;` followed by a single divide and modulo. Polygon bounding-box code uses a `MinMax()` helper that returns both values from one comparison. f5f12fd76 2020-12 “Small optimization to GPU::SystemTicksToCRTCTicks” · 92975b2e8 2020-08 “GPU/HW: Tiny performance optimization in polygon loading”

### duckstation-070 — Convert every primitive type to triangles so draws batch together
Lens: gpu-rendering · Source: DuckStation
Principle: Each primitive topology (lines, points, triangles) needs its own pipeline or draw call. Expanding lines into quads on the CPU puts everything into one triangle stream, so consecutive primitives batch into a single draw, and geometry-shader or wide-line hardware paths are no longer needed.
Why it is faster: —
Applies elsewhere: 2D canvas renderers tessellating strokes, UI toolkits (Skia, ImGui) emitting only triangles, sprite batchers, map renderers turning polylines into triangle strips.
Watch out: —
Seen in DuckStation: `GPU_HW::DrawLine` now writes four `BatchVertex`es with a 1-pixel extent perpendicular to the major axis, and degenerate lines become points. The geometry-shader line expansion and the `glLineWidth`/`fillModeNonSolid` requirements were removed. The commit message says this gives "fewer batches, consistent upscaling". 96ba9198e 2020-08 “GPU/HW: Expand lines into triangles before rendering”

### duckstation-071 — Shadow graphics API state and skip redundant calls and queries
Lens: gpu-rendering · Source: DuckStation
Principle: Driver calls cost CPU time even when they change nothing, and `glGet*` queries can stall the pipeline. Keep a CPU-side copy of the current state, issue a call only when the state differs, and don't read back state you could have tracked yourself.
Why it is faster: —
Applies elsewhere: state caches in game engines and WebGL wrappers (three.js `WebGLState`), skipping `setState` calls when nothing changed, avoiding `getComputedStyle` in layout loops.
Watch out: —
Seen in DuckStation: The OpenGL batch renderer now tracks `m_current_transparency_mode`, `m_current_render_mode` and `m_current_check_mask_before_draw`, and calls `SetBlendMode()`/`SetDepthFunc()` only when one of them changes. The ImGui GL backend stopped backing up and restoring around 15 pieces of state through `glGetIntegerv` every frame and binds a persistent VAO instead. The commit message reports 25% faster on NVIDIA. b3c5c0f85 2020-10 “GPU/OpenGL: Eliminate some redundant per-draw state calls” · fdb770a59 2020-10 “ImGui: Get rid of serializing GL calls in OpenGL renderer”

### duckstation-072 — Specialize shaders and vertex formats to the features actually enabled
Lens: gpu-rendering · Source: DuckStation
Principle: Generic shaders that clamp, filter or carry extra attributes "just in case" pay in bandwidth and ALU on every vertex and fragment. Decide at pipeline-creation time which features are active, and drop the unused attributes and code.
Why it is faster: —
Applies elsewhere: Unity/Unreal shader keyword variants, Vulkan specialization constants, stripping unused vertex attributes in glTF pipelines, feature-flagged code paths compiled out.
Watch out: —
Seen in DuckStation: UV-limit clamping is needed only with PGXP or texture filtering, so `ShouldUseUVLimits()` sets `m_using_uv_limits`, and the shader generator, the D3D11 input layout (`num_attributes - 1`) and the Vulkan vertex attributes all leave the `a_uv_limits` attribute out when it is unused. 5c6e92cdf 2020-08 “GPU/HW: Disable coordinate clamping when unnecessary” · fba2488bc 2020-08 “GPU/D3D11: Don't add UV limits to input layout when unused” · 2130d3b00 2020-08 “GPU/Vulkan: Don't add UV limits as attribute when unused”

### duckstation-073 — Pack narrow data into wider texels for readback
Lens: gpu-rendering · Source: DuckStation
Principle: A GPU-to-CPU readback is limited by bandwidth and staging-buffer size. If the data is 16-bit, write two values into each 32-bit RGBA8 texel, so the transfer is half as wide and half the bytes.
Why it is faster: —
Applies elsewhere: packing depth into RGBA for WebGL readPixels, half-float or RGB10A2 render targets, bit-packing booleans for network transfer.
Watch out: —
Seen in DuckStation: The Vulkan VRAM readback staging texture and render pass are now `VRAM_WIDTH / 2` wide. The readback shader encodes two 16-bit VRAM pixels per RGBA8 texel, and a full VRAM read now moves 1 MB instead of 2 MB. e21fc9e25 2020-09 “GPU/Vulkan: Use half width framebuffer for VRAM readbacks”

### duckstation-074 — Keep upload/download buffers persistently mapped
Lens: memory-layout · Source: DuckStation
Principle: Mapping and unmapping a GPU buffer for every transfer costs driver round-trips and can implicitly synchronize. Map it once and keep it mapped. Where coherent memory is slow or unavailable, use non-coherent mappings and flush only the byte range you wrote.
Why it is faster: —
Applies elsewhere: Vulkan/D3D12 persistently mapped ring buffers, `GL_MAP_PERSISTENT_BIT` streaming, `mmap` with `msync` on the dirty range, DMA buffers in embedded drivers.
Watch out: —
Seen in DuckStation: `Vulkan::StagingTexture::PrepareForAccess()` used to unmap and remap around each copy. It now asserts the buffer is mapped and only invalidates or flushes. This also fixed corruption on Mali. `GL::StreamBuffer::Create(target, size, coherent)` can drop `GL_MAP_COHERENT_BIT` and call `glFlushMappedBufferRange(m_target, m_position, used_size)` per write. dd0ae0fc9 2020-09 “Vulkan/StagingTexture: Keep mapped throughout transfers” · 011df33fc 2021-02 “GL/StreamBuffer: Support non-coherent mappings”

### duckstation-075 — Keep snapshots on the GPU and reuse their storage
Lens: gpu-rendering · Source: DuckStation
Principle: If saved state only needs to be restored, never inspected, copy it GPU-to-GPU instead of reading it back to the CPU. Reuse the destination textures between snapshots so you aren't allocating and freeing GPU memory on every frame.
Why it is faster: —
Applies elsewhere: render-target pools in engines, double-buffered undo snapshots in image editors, video encoders keeping reference frames on the device, CUDA memory pools.
Watch out: —
Seen in DuckStation: Memory-only save states, used for rewind and runahead, pass a `HostDisplayTexture**` into `GPU::DoState`, so VRAM is copied with `glCopyImageSubData`/`CopyFramebufferForState` instead of downloaded. The next commit reuses the existing texture when width, height and sample count match. OpenGL ES also got a `glCopyImageSubDataOES` path for VRAM copies, which previously required a draw. 516d685dd 2021-01 “System: Add memory-only save states and rewind” · 3f96473fb 2021-01 “System: Reuse existing texture/buffers for runahead/rewind” · e7139fe0d 2020-11 “OES_copy_image support”

### duckstation-076 — Refresh derived copies only when the dirty region overlaps what's needed
Lens: laziness · Source: DuckStation
Principle: If a secondary buffer mirrors part of a primary one, track a dirty rectangle on the primary. Recopy only when the dirty area intersects the region about to be read.
Why it is faster: —
Applies elsewhere: browser compositors repainting damaged rects, spreadsheet recalculation of dependent cells, incremental builds, invalidating GPU texture atlases by region.
Watch out: —
Seen in DuckStation: Before sampling VRAM for display, the Vulkan renderer now checks `m_vram_dirty_rect.Intersects(Rectangle::FromExtents(display_vram_left, top, width, height))` and only then calls `UpdateVRAMReadTexture()`. a0a24c775 2021-01 “GPU/Vulkan: Only update sample texture when render area dirty”

### duckstation-077 — Allocate immutable texture storage
Lens: gpu-rendering · Source: DuckStation
Principle: If you declare a texture's size, format and mip count once with immutable storage, the driver can lay it out up front and skip per-use completeness and validation checks. Mutable `glTexImage` allocations get none of that.
Why it is faster: —
Applies elsewhere: WebGL2 `texStorage2D`, Vulkan/Metal images (always immutable), preallocated fixed-size arenas instead of growable vectors.
Watch out: —
Seen in DuckStation: `GL::Texture::Create` calls `glTexStorage2D`/`glTexStorage2DMultisample` when `ARB_texture_storage` or GLES 3.x is available, and falls back to `glTexImage2D` otherwise. 4fe1c07b0 2020-10 “GL/Texture: Use texture storage where available”

### duckstation-078 — Defer destroying GPU objects until the frame's fence passes
Lens: laziness · Source: DuckStation
Principle: Destroying a resource that in-flight GPU work may still use requires waiting for the GPU to go idle. Queue the destruction on the current frame's cleanup list and run it when that frame's fence signals. The CPU never stalls.
Why it is faster: —
Applies elsewhere: RCU in the Linux kernel, epoch-based reclamation in lock-free structures, deferred-release queues in D3D12 and Vulkan engines.
Watch out: —
Seen in DuckStation: `Vulkan::Context::DeferPipelineDestruction(VkPipeline)` pushes a `vkDestroyPipeline` lambda onto `m_frame_resources[m_current_frame].cleanup_resources`, matching the existing deferred buffer and image destruction. 666d7fbb9 2020-09 “Vulkan/Context: Add pipeline destruction deferring”

### duckstation-079 — Convert pixel formats with SIMD directly into mapped memory
Lens: simd · Source: DuckStation
Principle: Per-pixel channel swizzles are ideal for 128-bit SIMD, which handles 8 pixels per instruction. Write the converted output straight into the mapped destination rather than into an intermediate buffer, which saves a full-frame copy. Pick a destination format the host accepts natively (such as RGB565) so less data moves.
Why it is faster: —
Applies elsewhere: libyuv and ffmpeg swscale, image decoders writing into canvas buffers, video players uploading straight into PBOs, framebuffer blitters in embedded displays.
Watch out: —
Seen in DuckStation: `CopyOutRow16<RGBA5551>` swaps the R and B fields with `_mm_and_si128`/`_mm_srli_epi16`/`_mm_slli_epi16` on x64 and `vandq_u16` etc. on AArch64, 8 pixels per iteration. Software scanout calls `m_host_display->BeginSetDisplayPixels(format, w, h, &dst_ptr, &stride)` and writes into the mapped texture, which removes one copy. The display can be 16-bit. d3d881aa6 2020-10 “GPU/Software: Reduce number of copies by one, enable 16-bit scanout”

### duckstation-080 — Offload heavy consumers through a single-producer ring buffer
Lens: concurrency · Source: DuckStation
Principle: Split a subsystem into a producer that only records variable-sized commands into a ring buffer and a consumer thread that executes them. The producer avoids locks by using atomic read and write indices, and it wakes the consumer only once enough work has built up, not on every command.
Why it is faster: —
Applies elsewhere: render threads in game engines, LMAX Disruptor, audio callback queues, logging backends (spdlog async), io_uring submission queues.
Watch out: —
Seen in DuckStation: The software renderer was split into a `GPU_SW` frontend and a `GPU_SW_Backend`. `GPUBackend::AllocateCommand(size)` carves commands out of `m_command_fifo_data`, writing a `Wraparound` dummy command at the end of the buffer. `PushCommand` does `m_command_fifo_write_ptr.fetch_add(cmd->size)` and calls `WakeGPUThread()` only when `GetPendingCommandSize() >= THRESHOLD_TO_WAKE_GPU`. With the thread disabled, commands run inline. 2595e3157 2020-11 “GPU: Split software to frontend/backend”

### duckstation-081 — Move blocking, non-critical work off the frame thread
Lens: concurrency · Source: DuckStation
Principle: Image compression, file writes and swap-chain presentation can block for milliseconds while the main loop has nothing to wait for. Copy out the data the job needs and hand it to a worker thread. The frame loop continues immediately.
Why it is faster: —
Applies elsewhere: web workers for image encoding, background thumbnail generation, asynchronous fsync, offloading `present` to a dedicated thread in browsers and compositors.
Watch out: —
Seen in DuckStation: `HostDisplay::WriteTextureToFile` now downloads the texture and passes the `std::vector<u32>` and an open file to `CompressAndWriteTextureToFile` on a `std::thread`, and screenshots use it by default. Vulkan "threaded presentation" queues `m_queued_present`, signals `m_present_queued_cv`, and has `DoSubmitCommandBuffer`/`DoPresent` run on a present thread. It is disabled with vsync, and the commit message says it can give "a significant speedup for fast forward". 125dfa7c4 2020-11 “HostDisplay: Support resizing/compressing images on worker thread” · 24fef2048 2020-11 “CommonHostInterface: Compress screenshots in background/use worker thread” · 702ed2120 2020-12 “HostDisplay: Add threaded presentation for Vulkan renderer”

### duckstation-082 — Hold locks only for the hand-off, and skip them when there is nothing to take
Lens: concurrency · Source: DuckStation
Principle: Do expensive work, such as resampling or formatting, outside the critical section, and lock only to move data in or out. On the consumer side, a cheap unlocked emptiness check avoids taking the mutex at all in the common case. Then drain the shared queue into a private one in a single locked pass.
Why it is faster: —
Applies elsewhere: double-buffered event queues in UI frameworks, swap-under-lock patterns (`std::swap(local, shared)`), Go's "copy then unlock", log aggregators batching under a short lock.
Watch out: —
Seen in DuckStation: `AudioStream` now unlocks `m_buffer_mutex` before resampling and notifying. OSD messages are split into `m_osd_posted_messages` (shared) and `m_osd_active_messages` (render thread only). `AcquirePendingOSDMessages()` checks `empty()` after an `atomic_thread_fence` and locks only if the queue is non-empty, which avoids "99.99% of the unnecessary lock attempts". 1ae4b94eb 2021-01 “AudioStream: Don't hold lock while resampling” · c3c418107 2021-02 “optimize locking mechanism for OSD messages, use EnumerateOSDMessages more generally”

### duckstation-083 — Use a condition variable rather than a pipe for in-process signaling
Lens: concurrency · Source: DuckStation
Principle: Signaling between threads with a pipe and `poll()` costs syscalls, file descriptors and kernel buffer traffic. A mutex plus condition variable, or a futex, stays in user space when there is no contention and wakes the waiting thread directly.
Why it is faster: —
Applies elsewhere: replacing self-pipe tricks with `eventfd` or futexes, `std::condition_variable` work queues, `SemaphoreSlim` over kernel semaphores in .NET.
Watch out: —
Seen in DuckStation: `Common::Event` on Linux/macOS replaced `pipe2(m_pipe_fds, O_NONBLOCK)` plus `poll(POLLRDNORM)` with a mutex, a condvar and an atomic `m_signaled`. Windows switched from event objects to `CRITICAL_SECTION` plus `CONDITION_VARIABLE`. dc5320956 2020-10 “Common/Event: Replace poll event with lock/condvar”

### duckstation-084 — Pace frames against absolute deadlines with precise sleeps
Lens: platform · Source: DuckStation
Principle: "Sleep for the remaining time" accumulates drift, and default OS sleeps round up to whole milliseconds. Keep an absolute `next_deadline += period`, sleep with a high-resolution timer, and spin only for the last stretch. Where something else already paces the loop (vsync), don't sleep at all.
Why it is faster: —
Applies elsewhere: game loops with fixed timesteps, audio schedulers, `clock_nanosleep(TIMER_ABSTIME)`, `CREATE_WAITABLE_TIMER_HIGH_RESOLUTION` on Windows, embedded RTOS periodic tasks.
Watch out: —
Seen in DuckStation: Windows throttling moved from `Sleep(ms)` to `SetWaitableTimer` with 100 ns units. `Timer::HybridSleep` sleeps in small chunks and checks the clock between them, and Android uses it. The new pacer keeps `s_next_frame_time += s_frame_period` and calls `Timer::SleepUntil(s_next_frame_time, true)`, so it catches up on lost time instead of drifting. When syncing to the host refresh rate, the speed limiter is turned off and vsync does the pacing. d44de3a9d 2021-01 “System: Use SetWaitableTimer() for throttling on Windows” · e79dff173 2021-01 “System: Use hybrid sleep on Android” · 4bb3fb48f 2021-01 “System: Add a new throttler/pacer which can catch up on lost time” · 16ad54143 2021-01 “HostInterface: Don't sleep when syncing to host refresh”

### duckstation-085 — Rate-limit synchronous device writes to changes plus a heartbeat
Lens: io · Source: DuckStation
Principle: If a device call blocks (Bluetooth, USB HID, a serial port), sending it every frame lets I/O latency cap your frame rate. Send only when the value changes, plus a periodic refresh so the device doesn't time out.
Why it is faster: —
Applies elsewhere: debounced autosave, MQTT publish-on-change with keepalive, LED or display drivers that skip identical frames, telemetry that sends deltas.
Watch out: —
Seen in DuckStation: `UpdateControllerRumble` compares each motor's strength with `rumble.last_strength[i]` and calls `update_callback` only when something `changed` or `UPDATE_FREQUENCY` has passed since `last_update_time`. The commit message says synchronous Bluetooth rumble updates had been limiting fast-forward speed. cbee8fab6 2021-01 “CommonHostInterface: Throttle rumble updates to changes or every 100ms”

### duckstation-086 — Pin the critical thread to dedicated cores
Lens: platform · Source: DuckStation
Principle: On big.LITTLE and mobile SoCs, the scheduler may migrate a latency-critical thread onto a slow core or share its core with background work. If the OS exposes cores reserved for the foreground app, bind the hot thread to them.
Why it is faster: —
Applies elsewhere: audio threads on Android (`getExclusiveCores`), DPDK pinning poll-mode drivers, game consoles reserving cores, `taskset` for latency-sensitive servers.
Watch out: —
Seen in DuckStation: `EmulationThread.setExclusiveCores()` calls `Process.getExclusiveCores()` (API 24+) and passes the result to native `setThreadAffinity`, which builds a `cpu_set_t` and calls `sched_setaffinity(gettid(), ...)`. 35dc530b9 2021-01 “Android: Pin emu thread to exclusive cores where supported”

### duckstation-087 — Keep heap allocation off hot or optional paths
Lens: allocation · Source: DuckStation
Principle: Short-lived conversions such as path encoding can use the stack. Large tables that only an optional feature needs should be allocated when the feature turns on, not reserved statically. Static reservations cost page commits, binary size and, on some platforms, slower addressing.
Why it is faster: —
Applies elsewhere: small-string optimization, `alloca`/`SmallVector`, lazy singletons, allocating GPU resources only when a feature is enabled.
Watch out: —
Seen in DuckStation: On Windows, `FileSystem::FileExists`/`DirectoryExists` converted UTF-8 to a `std::wstring`. They now `MultiByteToWideChar` into an `alloca` buffer. PGXP's 6MB `static PGXP_value Mem[...]` and its vertex cache became heap pointers, allocated in `PGXP::Initialize()` only when `gpu_pgxp_enable` is set. The commit message says the static arrays had caused "performance issues on ARM". f1086b18f 2020-09 “FileSystem: Remove heap allocations from FileExists/DirectoryExists” · a2eaaf0e8 2020-09 “PGXP: Allocate large storage dynamically”

### duckstation-088 — Key caches on fast content hashes, with runtime CPU dispatch
Lens: simd · Source: DuckStation
Principle: To find a cache entry by content, hash the bytes with a modern non-cryptographic hash (XXH3) and use the hash as the map key. Let the hash library pick SSE2, AVX2 or AVX-512 at runtime, so one portable binary still gets the widest vector path.
Why it is faster: —
Applies elsewhere: deduplicating storage and CDNs, asset pipelines caching by content hash, build systems (Bazel, ccache), simdjson and zlib-ng doing runtime ISA dispatch.
Watch out: —
Seen in DuckStation: Texture replacement hashes each VRAM write with `XXH3_128bits(pixels, w*h*2)` and looks it up in `m_vram_write_replacements`. Loaded textures are cached and `PurgeUnreferencedTexturesFromCache()` evicts the stale ones. On x86, `xxh_x86dispatch.h` is included and the build sets `XXH_X86DISPATCH_ALLOW_AVX`. 8db961042 2020-12 “GPU: Support replacing VRAM writes with new textures” · a69474ea9 2020-12 “TextureReplacements: Use x86 dispatch for xxhash” · 25847494d 2021-01 “dep/xxhash: Set XXH_X86DISPATCH_ALLOW_AVX on x86/x64”

### duckstation-089 — Cache expensive derived resources with a bound
Lens: allocation · Source: DuckStation
Principle: Decoding an image or formatting a string every time it is drawn wastes CPU, while caching everything forever wastes memory. Put decoded resources in an LRU cache with a capacity limit, and compute strings that rarely change once, when their inputs change.
Why it is faster: —
Applies elsewhere: browser image caches, Glide/Picasso on Android, memoized React selectors, prepared-statement caches, i18n strings resolved once at locale change.
Watch out: —
Seen in DuckStation: A generic `LRUCache<K,V>` (a `std::map` plus a `last_access` counter) was added. The fullscreen UI's cover map now stores only paths, and the textures live in `s_texture_cache` with `SetMaxCapacity(128)`. The save-state selector builds `m_load_legend` and the other legends in `RefreshHotkeyLegend()` when it opens or settings change, instead of on every frame. e540ab703 2021-02 “Common: Add LRUCache class” · 47e26b1ad 2021-02 “FullscreenUI: Put cover images through the texture cache” · 78f9136fb 2021-02 “Save State UI: Cache legend strings”

### duckstation-090 — Patch direct jumps between generated code blocks
Lens: jit-interpreters · Source: DuckStation
Principle: A JIT that returns to a central dispatcher after every block pays for a hash or table lookup and an indirect call each time. When a block ends in a branch whose target is known at compile time, the JIT can overwrite the exit with a direct jump to the target block once that block exists. Hot loops then run entirely in generated code. Each link has to be recorded so it can be reverted when its target is invalidated.
Why it is faster: —
Applies elsewhere: inline caches in JS engines (V8, SpiderMonkey), lazy symbol binding in the PLT (patched after the first call), trampoline-to-direct-call patching in kernel live-patching and in eBPF tail calls.
Watch out: —
Seen in DuckStation: Each `CodeBlock` now stores `LinkInfo{block, host_pc, host_resolve_pc, host_pc_size}`. `LinkBlock()` calls `CodeGenerator::BackpatchBranch(host_pc, size, to->host_code)` with the code buffer briefly made writable. `UnlinkBlock()` patches predecessors back to the resolver stub. A block that is invalidated again within `INVALIDATE_THRESHOLD_TO_DISABLE_LINKING` (10) frames gets `can_link = false`, so self-modifying code doesn't spend all its time patching. That logic was later split into a reusable `InvalidateBlock()`. The current code cache still keeps a `BlockLinkMap` and `CreateBlockLink()`. 21938e14c 2021-05 “CPU/Recompiler: Implement block linking” · c44059378 2021-12 “CPU/CodeCache: Add InvalidateAll() method”

### duckstation-091 — Stop optimizing code that keeps changing
Lens: jit-interpreters · Source: DuckStation
Principle: Compilation only pays off if the compiled result gets reused. Count how often each unit is recompiled within a time window. Past a threshold, mark it "don't compile" and run it on the slow general path, which beats compiling it again on every execution.
Why it is faster: —
Applies elsewhere: JVM/V8 deoptimization limits, where a function that deopts too often is never optimized again. Query planners stop caching plans for statements whose parameters keep invalidating them. A CDN can stop caching objects that are rewritten faster than they are read.
Watch out: —
Seen in DuckStation: In `RevalidateBlock()`, if a block was recompiled more than `RECOMPILE_COUNT_TO_FALL_BACK_TO_INTERPRETER` (20) times within 100 frames, `FallbackExistingBlockToInterpreter()` stores `nullptr` in `s_blocks` for its key, and from then on the block runs through the interpreter. The heuristic was briefly disabled with `&&false` and then re-enabled. Games known to rewrite code constantly were also given a database trait that forces the LUT-based (rather than mmap) fastmem. Today the threshold is `RECOMPILE_COUNT_FOR_INTERPRETER_FALLBACK = 3`. f34a04800 2021-04 “CPU/CodeCache: Fall back constantly-modified blocks to interpreter” · 8c7282046 2021-05 “CPU/CodeCache: Re-enable interpreter fallback heuristic” · 1e44c2f3d 2021-04 “GameSettings: Add trait to force LUT fastmem”

### duckstation-092 — Bake values known at compile time into the generated code
Lens: jit-interpreters · Source: DuckStation · Also: ares
Principle: If a code generator already knows a value, such as an instruction's address or a field offset, it should emit that value as an immediate. Tracking it in a runtime register means extra loads, adds and stores on every execution.
Why it is faster: —
Applies elsewhere: constant propagation in shader specialization (Vulkan specialization constants), template parameters instead of runtime ints, prepared statements with inlined literals, `constexpr` lookup offsets.
Watch out: —
Seen in DuckStation: The code generator used to keep `m_pc_offset` and emit `load regs.pc; add offset` to get the current PC. It now tracks `m_pc` and `m_pc_valid` at compile time, and `CalculatePC()` returns `Value::FromConstantU32(m_pc + offset)`. A follow-up stopped storing `current_instruction_pc` on every branch path and wrote it only where an exception could need it. Separately, `offsetof(State, regs.r[index])` with a runtime index was replaced by `constexpr GPRRegisterOffset()`. 6081e2415 2021-07 “CPU/Recompiler: Simplify PC addressing” · d9d41de81 2021-07 “CPU/Recompiler: Skip updating current_instruction_pc” · acda42be1 2021-05 “CPU/Recompiler: Get rid of non-constant offsetofs” • Also seen in ares (ares-121): The RSP JIT now updates `branch.pc/nextpc` only inside delay slots. A commit writes `mov32(mem(IpuReg(pc)), imm(nextpc))`, where `nextpc` is `u12(address + 4)` computed by the emitter. 6e7cf80f6 2026-04 “n64: virtualize RSP JIT block PC progression”

### duckstation-093 — Emit nothing for no-op inputs
Lens: jit-interpreters · Source: DuckStation
Principle: Filler instructions and no-op operations still go through the general translation path unless they are recognized. Detect them early and emit only the minimal bookkeeping, or nothing. This matters most when each translated operation also carries instrumentation overhead.
Why it is faster: —
Applies elsewhere: SQL optimizers dropping `WHERE 1=1`, bundlers stripping empty modules, compilers deleting dead stores, log pipelines discarding records filtered at DEBUG before they are formatted.
Watch out: —
Seen in DuckStation: `CompileInstruction()` now checks `IsNopInstruction(cbi.instruction)` (`bits == 0`) and emits only the prologue and epilogue (cycle counting). Before this, every `sll $zero,$zero,0` also went through PGXP hooks. The commit reports a 35% speedup in PGXP CPU mode in Racing Lagoon. baad1a4b2 2021-05 “CPU/Recompiler: Optimize away nops completely”

### duckstation-094 — Fold branches whose outcome is decided by a constant
Lens: branching-fast-paths · Source: DuckStation
Principle: A conditional branch on a value that is fixed (for example, a hardwired zero register) always goes the same way. Turn it into an unconditional jump and skip loading the operand. This saves the register read and the compare, and removes a branch the predictor would otherwise track.
Why it is faster: —
Applies elsewhere: dead-branch elimination on `if (DEBUG)` constants in JS minifiers, `if constexpr`, folding feature flags at build time.
Watch out: —
Seen in DuckStation: MIPS `bgez $zero, target` is an idiom for "branch always". The recompiler now sets `Condition::Always` when `rs == Reg::zero` and loads `lhs` only when the condition isn't Always. e8ac1fca8 2021-06 “CPU/Recompiler: Optimize bgez zero, addr to unconditional”

### duckstation-095 — Settle deferred bookkeeping only on the slow path
Lens: jit-interpreters · Source: DuckStation
Principle: When a counter is batched and normally committed at the end of a block, only the rare path that actually reads it (a fallback or a fault handler) needs it to be exact. Add the pending amount there and subtract it again afterwards. The common path then never touches memory for the counter.
Why it is faster: —
Applies elsewhere: per-CPU counters reconciled only when read, deferred reference counting, batched metrics flushed only when scraped, lazy FPU state saving on context switch.
Watch out: —
Seen in DuckStation: Fastmem slow paths (which run after a page fault and backpatch) used to call `AddPendingCycles(true)` before the access, so every load and store paid for flushing the tick count. Now the backpatched thunk does `pending_ticks += m_delayed_cycles_add`, makes the call, and then subtracts it again. A follow-up removed leftover `AddPendingCycles` calls that had caused an AArch32/64 performance regression. 2113405c7 2021-06 “CPU/Recompiler: Elide tick flush completely where possible” · 7b87b386a 2021-06 “CPU/Recompiler: Fix performance regression in AArch32/64”

### duckstation-096 — Move cold paths out of line, and compile them only when enabled
Lens: jit-interpreters · Source: DuckStation
Principle: Keep the code that usually runs straight-line and contiguous, and branch out to a separate "far" region for rare cases. That keeps the hot path's instruction-cache footprint and register-state handling small. Code for a feature that is switched off shouldn't be generated at all.
Why it is faster: —
Applies elsewhere: `__builtin_expect`/`[[unlikely]]` with `.text.cold` sections, Go's out-of-line stack-growth check, exception tables instead of inline checks, debug assertions compiled out in release builds.
Watch out: —
Seen in DuckStation: On writes to the DCIC (debug breakpoint) register, the update-and-exit sequence was emitted inline between `PushState` and `PopState`. It now does `EmitBranch(GetCurrentFarCodePointer()); SwitchToFarCode();`. The DCIC check is also emitted only when `g_settings.cpu_recompiler_memory_exceptions` is set. Libcrypt games left breakpoints armed, and that had forced them into the interpreter for no reason. 2d8806c13 2021-04 “CPU/Recompiler: Emit DCIC exit path in far code” · cfb4b058d 2021-04 “CPU/Recompiler: Gate debug exceptions behind memory exception flag”

### duckstation-097 — Unroll checks whose trip count is known when the code is generated
Lens: jit-interpreters · Source: DuckStation
Principle: If a loop's bounds are known when the code is generated, emit a straight-line sequence of specialized tests instead of calling a generic helper. Each iteration then uses constant addresses and offsets, there are no loop-control branches, and the call overhead disappears.
Why it is faster: —
Applies elsewhere: generated serializers that unroll per-field code (protobuf, serde), regex engines compiling small fixed loops, SIMD kernels unrolled for a known tile size.
Watch out: —
Seen in DuckStation: On ARM, the per-block instruction-cache simulation used to call a C++ helper. `EmitICacheCheckAndUpdate()` now loops over `m_block->icache_line_count` at compile time and emits one `ldr`/`cmp`/`b.eq`/`str`/`add` group per line, each with a constant tag offset. Lines with zero fill cost are skipped, and blocks in uncached KSEG1 just add a constant number of ticks. 5ff978362 2021-07 “CPU/Recompiler/ARM: Optimize icache check”

### duckstation-098 — Order operations to keep fewer values live at once
Lens: jit-interpreters · Source: DuckStation
Principle: Register pressure depends on how many values are live at the same time. Compute the cheap, independent values (such as masks) before the expensive load, and release temporaries as soon as they are consumed. Fewer registers are then held across the call, which avoids spills. On register-poor targets it also avoids running out of registers entirely.
Why it is faster: —
Applies elsewhere: instruction scheduling in compilers, writing shader code to keep VGPR counts low so GPU occupancy stays high, freeing large buffers early in a GC'd function.
Watch out: —
Seen in DuckStation: For LWL/LWR/SWL/SWR, the recompiler used to load guest memory first and then build shift masks while holding both the memory value and the guest register. Now it computes `shift`/`mask` first, then calls `EmitLoadGuestMemory`, then does `mask.ReleaseAndClear()`. This also fixed a crash from register exhaustion in Ape Escape on AArch32. The x64 `div` emitter similarly writes the quotient and remainder straight to their destination registers when neither is RAX or RDX. 9628f0d9d 2021-02 “CPU/Recompiler: Reduce register usage of LWL/LWR/SWL/SWR” · d2e90b051 2021-09 “CPU/Recompiler: Reduce register usage of SWL/SWR” · 663dc496b 2021-03 “CPU/Recompiler: Tiny optimization for div on x64”

### duckstation-099 — Two-level sparse table with pre-biased pointers
Lens: lookup-tables · Source: DuckStation
Principle: A flat table covering a huge, sparse key space wastes memory, and folding several regions into it needs compares and conditional moves. Split the key into a high part that picks a second-level table and a low part that indexes it. Store each second-level pointer pre-offset by minus its base key, so the full key indexes it directly with no subtraction. A lookup is then one shift, two loads and no branches.
Why it is faster: —
Applies elsewhere: CPU page tables, Unicode property tables (two-stage lookup), sparse arrays in V8 and ICU, radix-tree routing tables, the pointer-biasing trick in arena allocators.
Watch out: —
Seen in DuckStation: The JIT dispatcher computed both `(pc & RAM_MASK) >> 2` and a BIOS index, then chose one with `cmp`/`cmovge`. It now emits `rcx = s_fast_map[pc >> 16]; call [rcx + rax*2]`. The tables are allocated only for the RAM and BIOS ranges, and `EncodeFastMapPointer()` subtracts `slot << 17` so the full PC can be used as the index. Unmapped slots point to `InvalidCodeFunction`. The current code cache has the same structure, called `LUT_TABLE_*`. 033d85cd9 2021-07 “CPU/Recompiler: Simplify fast map addressing”

### duckstation-100 — Invalidate caches on rollback instead of discarding them
Lens: laziness · Source: DuckStation
Principle: After a state restore, most cached derived data (compiled code, prepared plans) is probably still valid. Mark the entries stale and revalidate each one cheaply (for example by hash) the next time it's used. Throwing everything away forces a full rebuild on every restore. The same reasoning applies to fast paths that were disabled because they seemed incompatible with rollback.
Why it is faster: —
Applies elsewhere: HTTP `ETag`/`If-None-Match` revalidation instead of purging, incremental builds that check content hashes after `git checkout`, React reconciliation instead of remounting.
Watch out: —
Seen in DuckStation: Runahead and rewind restore a memory save state every frame. `DoState()` used to call `CodeCache::Flush()`. With `is_memory_state` it now calls `InvalidateAll()`, which marks blocks invalidated so `RevalidateBlock()` can compare them against RAM. Once that was in place, the forced switch from recompiler to interpreter and the forced switch from mmap fastmem to LUT fastmem under runahead/rewind were both removed. 4513c7a23 2021-12 “System: Invalidate instead of flushing code cache on memory state” · 25ea24853 2021-12 “HostInterface: Don't disable recompiler for runahead” · fa9e50d7c 2021-12 “HostInterface: Don't disable fastmem for runahead”

### duckstation-101 — Decode addresses with masks instead of a switch
Lens: branching-fast-paths · Source: DuckStation
Principle: When address regions are aligned powers of two, a lookup can be done with AND, compare and shift. A `switch` on the top byte plus nested range checks and a modulo is slower and branchier. Write it with the platform's mask constants and the function becomes small enough to inline.
Why it is faster: —
Applies elsewhere: memory-mapped I/O decoding in firmware, slab or size-class selection with `clz`, hash-table bucket selection with `& (n-1)` instead of `%`.
Watch out: —
Seen in DuckStation: `PGXP_ConvertAddress()` switched on `addr >> 24` and used `% 0x200000` for RAM mirrors. The replacement checks `(addr & DCACHE_LOCATION_MASK) == DCACHE_LOCATION` for scratchpad, otherwise `paddr = addr & PHYSICAL_MEMORY_ADDRESS_MASK; if (paddr < RAM_MIRROR_END) return &Mem[(paddr & RAM_2MB_MASK) >> 2];`. df26792c1 2021-05 “PGXP: Small optimization in memory lookup”

### duckstation-102 — Remove dead fields from per-element structs
Lens: memory-layout · Source: DuckStation
Principle: In large arrays of small structs, every unused byte costs cache and memory bandwidth on each access. Remove debug counters and flags that nobody reads. Force-inline the tiny helpers. Allocate optional side tables only when their feature is turned on.
Why it is faster: —
Applies elsewhere: ECS components, shrinking DOM node or V8 hidden-class sizes, packing database row headers, removing per-object debug tags in release builds.
Watch out: —
Seen in DuckStation: `PGXP_value`, which shadows every word of RAM and every register, lost its `count`, `gFlags`, `lFlags` and `hFlags` fields along with the code that maintained them. `MakeValid`/`Validate` became `ALWAYS_INLINE_RELEASE`. The 4096x4096 `vertexCache` is now `calloc`'d only if `gpu_pgxp_vertex_cache` is enabled. The commit reports up to 8% in Racing Lagoon with CPU mode enabled. d3d92226a 2021-06 “PGXP: Further optimizations”

### duckstation-103 — Specialize hot loops with templates, but limit how many variants you create
Lens: compiler-build · Source: DuckStation
Principle: Turning runtime flags into template parameters takes per-pixel branches out of the inner loop, and a table of function pointers picks the variant once per call. Every extra dimension multiplies the variant count, though. Past a point the code bloat costs more in instruction-cache and branch-target-buffer misses than the removed branches saved, especially on small cores. Specialize only on flags that change the inner loop a lot.
Why it is faster: —
Applies elsewhere: image codecs with per-format kernels, shader permutation explosion in game engines, C++ `std::visit` code bloat, generic-heavy Rust binaries.
Watch out: —
Seen in DuckStation: The software rasterizer's `DrawTriangle`/`DrawRectangle` were specialized over `[2][9][5][2][2]` combinations (texture mode × transparency mode × dither × check_mask), with the tables generated by `scripts/gen_sw_functions.py`. This caused a performance regression on 32-bit ARM. The fix reduced the parameters to booleans (`texture_enable`, `raw_texture_enable`, `transparency_enable`, …) and moved the exact mode back to runtime, which deleted about 1,300 lines of tables. Today's `gpu_sw_rasterizer.inl` keeps the bool-template approach, with a separate AVX2 build. 1405ba8a6 2021-07 “GPU/SW: Specialize rasterizing functions further” · e9374646e 2021-07 “GPU/SW: Fix performance regression on 32-bit ARM”

### duckstation-104 — Hide symbols and drop unused language runtime features
Lens: compiler-build · Source: DuckStation
Principle: With default visibility, a shared object's calls to its own functions go through the PLT and cannot be inlined, because ELF allows another library to interpose those symbols. `-fvisibility=hidden` plus `-fno-semantic-interposition` turns them into direct, inlinable calls. `-fno-exceptions` removes unwind tables and landing pads, so code is smaller and the optimizer has more freedom.
Why it is faster: —
Applies elsewhere: Python built with `-fno-semantic-interposition` (about 10–30% faster), Chromium and Android builds with hidden visibility, embedded firmware built with `-fno-exceptions -fno-rtti`.
Watch out: —
Seen in DuckStation: CMake got `CMAKE_CXX_VISIBILITY_PRESET hidden` and `-fno-semantic-interposition`, with a comment saying the goal is to avoid the PLT for intra-library calls. The flag was later applied unconditionally, and `-fno-exceptions` was added on non-x86 targets (xbyak needed exceptions on x86). The current CMakeLists uses `-fno-exceptions -fno-rtti` everywhere. 02f75f25d 2021-06 “Build: Default symbol visibility to hidden” · d84a02a2c 2023-01 “CMake: Build tweaks”

### duckstation-105 — Mark leaf classes final so calls can be devirtualized
Lens: compiler-build · Source: DuckStation
Principle: The compiler can turn a virtual call into a direct (and inlinable) call only if it can prove nothing overrides the method. Marking leaf classes `final`, and not declaring helpers virtual when they never need to be, gives it that proof.
Why it is faster: —
Applies elsewhere: `sealed` classes in C#/Kotlin/Java (JIT devirtualization), Swift `final`, Rust preferring generics to `dyn Trait` in hot paths.
Watch out: —
Seen in DuckStation: In `byte_stream.h`, `AtomicUpdatedFileByteStream` and the memory streams became `final`. Redundant `virtual` was removed in favor of `override`, and helpers that didn't need dynamic dispatch were made non-virtual. This was done just before zstd-streamed save states, which make many small `Read`/`Write` calls through these classes. 0154a594c 2022-08 “ByteStream: Devirtualize where unnecessary”

### duckstation-106 — Make hot-path assertions debug-only and force-inline only in release builds
Lens: compiler-build · Source: DuckStation
Principle: An always-on assertion inside a per-instruction or per-sample function costs a compare, a branch and a call to a cold panic path, and it can stop the function from being inlined. Use a debug-only assertion for internal invariants. Use a force-inline macro that applies only in release builds, so debug builds stay easy to step through.
Why it is faster: —
Applies elsewhere: `debug_assert!` vs `assert!` in Rust, `NDEBUG`, a `DCHECK` vs `CHECK` split, contracts disabled in production builds.
Watch out: —
Seen in DuckStation: In the interpreter's per-instruction path, `Assert(g_state.next_load_delay_reg == Reg::count)` became `DebugAssert`. In the SPU, `ExecuteFIFOReadFromRAM`/`WriteToRAM` changed from `ALWAYS_INLINE` to `ALWAYS_INLINE_RELEASE`, and their mode `Assert` became `DebugAssert`. 53299e3c7 2021-07 “CPU/Interpreter: Load delay check should be a debug assertion” · 1aa3be70c 2023-01 “SPU: Bypass FIFO for direct memory writes”

### duckstation-107 — Mark state dirty only when the value actually changed
Lens: laziness · Source: DuckStation
Principle: Setting a "dirty" flag on every assignment triggers uploads, redraws or notifications even when nothing changed. Compare against the stored value first and set the flag only on a real difference. When values are being restored in bulk, assign the fields directly instead of calling setters that each fire an update.
Why it is faster: —
Applies elsewhere: React `setState` bailout on `Object.is`, `INotifyPropertyChanged` guarded by equality, OpenGL state caches that skip `glUniform` for unchanged values, Redux selectors with memoization.
Watch out: —
Seen in DuckStation: In `GPU_HW`, changing the draw mode used to set `m_batch_ubo_dirty = true` unconditionally. Now it does `m_batch_ubo_dirty |= (old_src_alpha != new || old_dst_alpha != new)`, and the same for `u_set_mask_while_drawing`, so the uniform buffer is uploaded only when a value changes. `ProgressCallback::PopState` now assigns `m_status_text`/`m_progress_value` directly instead of calling four virtual setters that each redraw the UI. 75e6c175d 2021-06 “GPU/HW: Elide redundant UBO updates on draw mode change” · 58f05498e 2021-10 “Common/ProgressCallback: Elide some redundant updates”

### duckstation-108 — Use clear commands for solid fills instead of drawing
Lens: gpu-rendering · Source: DuckStation
Principle: Filling a rectangle with one color by drawing a quad means binding a pipeline and running shaders. A clear command with a rectangle list often goes through dedicated fast-clear hardware or compression metadata. Use the draw only when the fill needs shader logic (wrap-around, interlacing, masking).
Why it is faster: —
Applies elsewhere: `vkCmdClearAttachments`/`ClearRenderTargetView` with rects, `glClear` with a scissor rectangle, `memset` instead of a per-pixel loop, `ctx.clearRect` over filling with a transparent color.
Watch out: —
Seen in DuckStation: `GPU_HW_D3D12::FillVRAM` now checks `!IsVRAMFillOversized() && !IsInterlacedRenderingEnabled()`, and in that case calls `ClearRenderTargetView` and `ClearDepthStencilView` with a scaled `D3D12_RECT` and returns before setting up any pipeline. 89a11519f 2021-07 “D3D12: Use clears instead of draws for simple VRAM fills”

### duckstation-109 — Handle edge cases on the GPU instead of reading data back to the CPU
Lens: gpu-rendering · Source: DuckStation
Principle: A GPU-to-CPU readback stalls the pipeline and moves the whole surface across the bus twice. Edge cases like wrap-around can usually be handled in the shader with modular coordinates and a discard. Write a shader variant for the edge case so every input stays on the fast path.
Why it is faster: —
Applies elsewhere: texture wrap modes instead of CPU tiling, compute-shader prefix sums instead of reading back for a CPU pass, WebGL apps avoiding `readPixels`.
Watch out: —
Seen in DuckStation: VRAM fills that wrapped past 1024×512 used to do `ReadVRAM(0,0,VRAM_WIDTH,VRAM_HEIGHT)`, fill on the CPU, and upload again. Now `GenerateVRAMFillFragmentShader(wrapped, interlaced)` produces four variants with `u_dst_x`/`u_end_x` computed modulo VRAM size, in all backends. 70209db40 2021-07 “GPU/HW: Implement oversized VRAM fills in hardware”

### duckstation-110 — Tell the GPU when a render target's old contents don't matter
Lens: gpu-rendering · Source: DuckStation
Principle: On tile-based mobile GPUs, starting a render pass without a clear or invalidate means the previous contents are loaded from memory into tile storage. An explicit invalidate or discard skips that load, which saves bandwidth and power.
Why it is faster: —
Applies elsewhere: `VK_ATTACHMENT_LOAD_OP_DONT_CARE`, Metal `MTLLoadActionDontCare`, WebGL2 `invalidateFramebuffer`, `MADV_DONTNEED` on memory you will overwrite.
Watch out: —
Seen in DuckStation: When the display framebuffer will be fully overwritten (non-interlaced output), the GL renderer now calls `glInvalidateFramebuffer(GL_DRAW_FRAMEBUFFER, 1, {GL_COLOR_ATTACHMENT0})` if GL 4.3 or GLES 3.0 is available. 51fb23c85 2021-06 “GPU/OpenGL: Invalidate display framebuffer where possible”

### duckstation-111 — Give the shader compiler what it needs to know
Lens: gpu-rendering · Source: DuckStation
Principle: Shader compilers optimize better when they know buffers don't alias (`readonly restrict`), when loops have a fixed count (`[unroll]`), and when a variant contains no unconditional `discard`. A discard can disable early depth testing for the whole draw. Generate only the variants that are actually needed.
Why it is faster: —
Applies elsewhere: `restrict` in C kernels, `#pragma unroll` in CUDA, specialization constants to remove dead branches, avoiding `discard`/`clip` in opaque passes.
Watch out: —
Seen in DuckStation: The VRAM-write SSBO became `readonly restrict buffer`. MSAA resolve loops use `FOR_UNROLL`, which expands to `[unroll] for` in HLSL. Before this change, untextured transparent batches compiled shaders with a `discard` that always or never fired. They now take a separate `#elif TRANSPARENCY` path with no test, and `NeedsTwoPassRendering()` requires texturing. 0cbba59e8 2021-03 “GPU/ShaderGen: Add readonly/restrict qualifier to VRAM write SSBO” · 98af6e722 2021-07 “GPU/ShaderGen: Use [unroll] for resolve loops” · 08876d132 2021-05 “GPU/HW: Don't generate always/never-discarding discard shaders”

### duckstation-112 — Stream through a few large ring buffers backed by a sub-allocator
Lens: allocation · Source: DuckStation
Principle: Per-frame upload data should go through ring buffers big enough that they rarely wrap into memory the GPU is still reading. Every wrap forces a fence wait or a command-buffer flush. Allocate the backing memory from a pooled sub-allocator instead of making many separate driver allocations. Prefer one shared ring over several special-purpose ones.
Why it is faster: —
Applies elsewhere: Vulkan Memory Allocator and D3D12MA, Metal ring buffers with semaphores, the Linux network stack's large ring buffers, log shippers with bounded ring buffers.
Watch out: —
Seen in DuckStation: The vertex buffer grew from 1 MB to 4 MB and the uniform buffer from 512 KB to 2 MB. Vulkan switched to VMA (`CreateAllocator()`), and the texture stream buffer doubled to 32 MB. Texture replacements now upload through that shared buffer instead of a dedicated `m_texture_replacment_stream_buffer`, which had its own "executing command buffer while waiting for space" stall. 82749d1ba 2021-03 “GPU/HW: Increase streaming buffer sizes to 4MB/2MB” · 9d27f7095 2022-09 “Vulkan: Use Vulkan Memory Allocator” · 5ef0ad1ec 2022-09 “GPU/Vulkan: Get rid of additional stream buffer”

### duckstation-113 — Keep and reuse GPU textures and contexts instead of recreating them
Lens: gpu-rendering · Source: DuckStation
Principle: Creating textures and contexts involves driver allocation, validation and sometimes a pipeline flush. For data that is streamed every frame, keep a texture of the right size and format and write into it through a mapped pointer. Recreate it only when its dimensions change. Don't tear down a context that already meets the requirement.
Why it is faster: —
Applies elsewhere: object pools in game engines, reusing `<canvas>`/`ImageBitmap` instead of recreating them, keep-alive HTTP connections, keeping a long-lived `AudioContext`.
Watch out: —
Seen in DuckStation: The software renderer called `g_host_display->SetDisplayPixels(...)` every frame. `GPU_SW::GetDisplayTexture(w,h,fmt)` now caches a texture created with the streaming flag, writes into it via `BeginUpdate`/`EndUpdate`, and recreates it only when the size or format changes. A related change stopped switching to the software renderer from requesting a new GL context when a GLES context was already active. 0b3461338 2022-09 “HostDisplay: Use streaming for sw renderer display” · 8f39ad355 2022-10 “GPU/OpenGL: Don't recreate context on sw switch when using GLES”

### duckstation-114 — Choose transfer paths per driver based on measurements
Lens: platform · Source: DuckStation
Principle: The "fast" upload or readback path in an API spec is not fast on every driver. Pixel buffer objects, cached-vs-coherent memory and blit-vs-shader copies each lose on some GPUs. Profile on each target family and switch paths by vendor or driver ID, keeping the general path as the default.
Why it is faster: —
Applies elsewhere: browser GPU blocklists and workarounds (ANGLE, Chromium `gpu_driver_bug_list`), database engines picking `O_DIRECT` per filesystem, JIT tuning per CPU microarchitecture.
Watch out: —
Seen in DuckStation: On Broadcom (Raspberry Pi), texture streaming through PBOs added 2–3 ms per `glTexImage2D`, so those GPUs now join Qualcomm in disabling PBOs. On Adreno with Vulkan, readback buffers now ask for `HOST_CACHED | HOST_COHERENT`, because cached non-coherent memory made `vkInvalidateMappedMemoryRanges` very expensive. An attempt to replace GL framebuffer blits with a shader copy was reverted three days later because it was slower on Mali. 9d26a8596 2021-06 “OpenGLHostDisplay: Disable PBOs for streaming on Broadcom GPUs” · 6af5a2486 2022-10 “VulkanHostDisplay: Use coherent memory type for downloads on Adreno” · 2ed482132 2021-04 “GPU/OpenGL: Use shader instead of framebuffer blit” · 327e8f3f5 2021-04 “GPU/OpenGL: Switch back to framebuffer blits over shader copy”

### duckstation-115 — Shorten the presentation queue to cut latency
Lens: gpu-rendering · Source: DuckStation
Principle: Each extra swap-chain buffer or queued frame adds up to one frame of input-to-display latency. When the producer is paced by its own timer, two buffers and a maximum frame latency of 1 are enough.
Why it is faster: —
Applies elsewhere: NVIDIA Reflex and AMD Anti-Lag, the low-latency mode in Android's `SurfaceView`, audio drivers with small period counts, reducing `TCP_NOTSENT_LOWAT` for interactive streams.
Watch out: —
Seen in DuckStation: Vulkan now requests `minImageCount` images instead of `minImageCount + 1`. D3D11 and D3D12 use `BufferCount = 2`, and D3D11 also calls `IDXGIDevice1::SetMaximumFrameLatency(1)`. 468827798 2022-08 “SwapChain: Reduce buffer count from 3 to 2”

### duckstation-116 — Spin briefly before blocking on a worker handoff
Lens: concurrency · Source: DuckStation
Principle: If the next piece of work usually arrives within microseconds, putting the thread to sleep costs a context switch plus wakeup latency, which can be more than the wait itself. Busy-poll for a short window (about 1 ms) after the last command, then block. A counting semaphore as the blocking primitive avoids the separate wait and reset steps of a manual-reset event.
Why it is faster: —
Applies elsewhere: adaptive mutexes (glibc `PTHREAD_MUTEX_ADAPTIVE_NP`, Windows critical-section spin counts), DPDK/io_uring polling modes, Go's scheduler spinning Ms before parking.
Watch out: —
Seen in DuckStation: The GPU worker thread in `GPUBackend::RunGPULoop` now records `last_command_time` and keeps polling the FIFO until `SPIN_TIME_NS` (1 ms) has passed, unless the sync command was sent with `allow_sleep = true`. Later, `Common::Event m_sync_event` with `Wait()` then `Reset()` was replaced by `Threading::KernelSemaphore` with `Wait()`/`Post()`. e4d5d9f04 2021-04 “GPU/SW: Spin for 1ms rather than immediately sleeping” · eb166220b 2022-07 “GPU/SW: Replace event with semaphore”

### duckstation-117 — Hit precise deadlines by sleeping first and spinning at the end
Lens: platform · Source: DuckStation
Principle: OS sleep timers routinely wake up late by anywhere from a fraction of a millisecond to several. To hit a deadline precisely, request a high-resolution timer, sleep until slightly before the deadline (for example 0.5 ms early), and spin on a monotonic clock for the remainder. Use relative waits so wall-clock adjustments can't shift the wake time.
Why it is faster: —
Applies elsewhere: frame limiters in games (RTSS), audio render loops, high-frequency trading pacing, `CREATE_WAITABLE_TIMER_HIGH_RESOLUTION` on Windows, `clock_nanosleep` with `CLOCK_MONOTONIC`.
Watch out: —
Seen in DuckStation: `Timer::SleepUntil(value, exact)` first switched to `CreateWaitableTimerEx(..., CREATE_WAITABLE_TIMER_HIGH_RESOLUTION)` with a 1 ms spin at the end. The same pattern was then ported to POSIX. The Windows due time became relative (`-one_hundred_nanos_diff`) instead of derived from `GetSystemTimeAsFileTime`. Finally, both platforms settled on `wake_at = value - 0.5ms; SleepUntil(wake_at); do current = GetCurrentValue(); while (current < value);`, which is the code today. b074c0d1e 2022-07 “System: Make inter-frame sleep slightly more precise” · ae0d60fcd 2022-08 “Timer: Spin for last 1ms on non-windows too” · f8cb48077 2022-12 “Timer: Improve SleepUntil()” · 3b038fd27 2023-01 “System: Further improve frame timing consistency”

### duckstation-118 — Keep a parallel CPU replica of GPU-side state to avoid readbacks
Lens: concurrency · Source: DuckStation
Principle: If the CPU occasionally needs data that the GPU produces, every readback stalls both sides. When the data can be recomputed cheaply, feed the same commands to a CPU implementation on another core and read from that copy. You trade spare cores for removing GPU-to-CPU synchronization.
Why it is faster: —
Applies elsewhere: CPU-side collision meshes mirroring GPU-skinned geometry, client-side prediction mirroring server state, keeping a write-through cache alongside the database instead of reading back after every write.
Watch out: —
Seen in DuckStation: A new option, "Use Software Renderer For Readbacks", makes `GPU_HW` own an `m_sw_renderer` (a `GPU_SW_Backend` on a forced worker thread). Each polygon, rectangle, fill and VRAM write is also pushed as a backend command (`NewDrawPolygonCommand`, …) with native-resolution vertices. VRAM reads by the emulated CPU are then served from the software copy and never touch the GPU. 5d877250e 2021-05 “GPU/HW: Add an option to use software renderer for readbacks”

### duckstation-119 — Read ahead into a ring buffer on a background thread
Lens: io · Source: DuckStation
Principle: For mostly sequential reads, a background thread can fetch the next N blocks into a ring buffer while the consumer handles the current one. Hits are served from memory with no lock and no wait. A miss drops the readahead and seeks. Also skip re-reading the block that was just returned.
Why it is faster: —
Applies elsewhere: the OS page cache's readahead (`posix_fadvise(SEQUENTIAL)`), video players buffering ahead, database prefetch on sequential scans, HTTP range-request prefetchers in streaming clients.
Watch out: —
Seen in DuckStation: `CDROMAsyncReader` went from a single pending sector to `m_buffers` of `readahead_count` slots with atomic `m_buffer_front/back/count`. `QueueReadSector(lba)` returns early if the front slot already holds that LBA. If the next slot holds it, it advances the front, decrements the count and notifies the thread ("Readahead buffer hit"). Otherwise it cancels and seeks. `WaitForReadToComplete()` doesn't take the lock when a buffer is ready. A follow-up made uncached reads (subchannel queries) wait for the thread to go idle and restore its seek position. a32ef4a96 2021-07 “CDROMAsyncReader: Support reading ahead more sectors” · c2c204c84 2021-07 “CDROM: Fix async readahead causing lockups in some games”

### duckstation-120 — Preload the whole working set into RAM when it fits
Lens: io · Source: DuckStation
Principle: If the data set is smaller than available RAM, decompressing and loading all of it up front removes disk latency and decompression cost from the runtime path entirely. Prefer the format's own precache facility, fall back to a generic copy into memory, and report progress so the load time is visible.
Why it is faster: —
Applies elsewhere: `vmtouch`/`mlock`-warming a database, loading SQLite with `:memory:`, game engines unpacking level archives into RAM, service workers precaching app shells.
Watch out: —
Seen in DuckStation: libchdr gained `chd_precache_progress()` with a progress callback, and `CDImageCHD::Precache()` uses it. `CDROM::PrecacheMedia()` then calls `CDROMAsyncReader::Precache()`, which falls back to `CDImage::CreateMemoryImage()` for formats without native support. An earlier fix had allowed preloading for M3U playlists with only one disc (`GetSubImageCount() > 1` check). ec9411c0a 2022-04 “dep/libchdr: Add chd_precache_progress()” · 5d097da16 2022-04 “CDImageCHD: Implement precaching” · a1edddc59 2022-07 “CDROM: Revamp/improve preload image procedure” · 717ee3103 2022-01 “M3U: Preloading to RAM for single-disc playlists.”

### duckstation-121 — Turn off integrity checks and decoding you don't use
Lens: io · Source: DuckStation
Principle: Libraries often verify checksums and decode every optional stream by default. If the input is trusted or the extra data is never read, compile those paths out. That saves CPU on every block, which matters most on low-end devices.
Why it is faster: —
Applies elsewhere: `PRAGMA cell_size_check=OFF`, skipping JPEG thumbnail decoding, disabling zlib's adler32 check for already-verified data, gzip `--no-name`, protobuf lazy fields.
Watch out: —
Seen in DuckStation: In `chdconfig.h`, `WANT_SUBCODE`, `NEED_CACHE_HUNK` and `VERIFY_BLOCK_CRC` changed from 1 to 0. The commit message says this gives a small speed boost on low-end devices. 0ff1402fc 2021-07 “dep/libchdr: Disable verification/subcode decompression”

### duckstation-122 — Reuse metadata from directory enumeration instead of stat-ing each file again
Lens: io · Source: DuckStation
Principle: Directory enumeration APIs (`FindFirstFile`, `readdir` with `d_type`, `getdents`) already return size and timestamps. Calling `stat()` on each entry afterwards doubles the syscalls, and on network or SAF storage each syscall is slow. Pass the enumeration record through and use its fields.
Why it is faster: —
Applies elsewhere: Python's `os.scandir` replacing `listdir`+`stat` (2–20× faster), `git status` using cached stat info, build tools like Ninja using directory mtime batches.
Watch out: —
Seen in DuckStation: `GameList::GetExeListEntry`, `GetPsfListEntry` and `GetGameListEntry` each called `FileSystem::StatFile(path, &ffd)` to get the modification time. The scan loop now fills `last_modified_time` from the `FILESYSTEM_FIND_DATA` it already has, and the per-entry stat calls and the extra `strrchr` extension checks were removed. 03f3f0369 2021-04 “GameList: Reduce number of system calls when scanning”

### duckstation-123 — Compress as a stream, not as a buffer-then-compress pass
Lens: io · Source: DuckStation
Principle: Serializing everything into a staging buffer, compressing that into a second buffer and then writing it means three full-size buffers and three passes over the data. A streaming compressor that wraps the output stream does one pass with small, bounded buffers. Fast modern codecs like zstd make compression nearly free compared to the I/O it saves.
Why it is faster: —
Applies elsewhere: `GZIPOutputStream`/`zstd.ZstdCompressor().stream_writer`, HTTP chunked transfer with on-the-fly gzip, `tar | zstd` pipelines, Node's `stream.pipeline(zlib.createGzip())`.
Watch out: —
Seen in DuckStation: Save states gained zstd compression (on by default). The first version serialized into a `GrowableMemoryByteStream`, then called `ZSTD_compress` into a `ZSTD_compressBound` buffer, and decompressing allocated buffers for both the compressed and uncompressed data. The follow-up wraps the output in `ByteStream::CreateZstdCompressStream`/`CreateZstdDecompressStream`, so `StateWrapper` writes through the compressor directly. 759938a5c 2022-08 “System: Support compressing save states” · 0f82dc916 2022-08 “System: Use streaming for compressed save”

### duckstation-124 — Replace a bundled reference implementation with an optimized library
Lens: algorithms · Source: DuckStation
Principle: Single-header libraries often ship a minimal but slow implementation of standard algorithms (deflate, float parsing). When the library allows it, plug in the platform's optimized implementation (zlib, `std::from_chars`, fast_float). That gets 2–10× on that step without changing the format.
Why it is faster: —
Applies elsewhere: swapping Python's `json` for `orjson`, linking zlib-ng or libdeflate instead of zlib, `simdjson`, using `Intl` over hand-rolled formatting.
Watch out: —
Seen in DuckStation: tb_image_write's built-in deflate was replaced via `#define STBIW_ZLIB_COMPRESS compress_for_stbiw`, which calls `compress2()` from zlib. This is used for screenshots and save-state previews. `StringUtil::FromChars` for floats used `std::istringstream` on non-MSVC toolchains and now uses `fast_float::from_chars`, and the integer versions use `std::from_chars`. 673f9927a 2021-07 “dep/stb: Use zlib for png compression” · 01270bac3 2023-01 “StringUtil: Add ToChars/fast_float FromChars”

### duckstation-125 — Leave large, rarely changing data out of frequent snapshots
Lens: laziness · Source: DuckStation
Principle: When snapshots are taken every frame, serializing big blobs that almost never change dominates the cost. Record when the blob last changed. Include it only if it changed within the rollback window, and take one backup when a change starts so rollbacks past that point can be restored. Data that can be recomputed or reloaded (such as firmware) can be replaced by a hash.
Why it is faster: —
Applies elsewhere: incremental and differential backups, Redis RDB vs AOF, game netcode rollback that leaves static level geometry out of snapshots, Docker layer caching.
Watch out: —
Seen in DuckStation: With runahead, writing 128 KB per memory card into every frame's memory state took up to 17% of CPU time. `Pad::DoState(sw, is_memory_state)` now stores `s_last_memory_card_transfer_frame`, includes card data only if `frames_since_transfer <= runahead_frames`, and calls `BackupMemoryCardState()` when a transfer begins outside that window. A reused `GetDummyMemcard()` replaced a `make_unique<MemoryCard>` per load, and the backup was then made conditional on `ShouldAvoidSavingToState()`. The 512 KB BIOS was also removed from save states and replaced by a hash check, which is skipped entirely for memory states. 367382736 2023-03 “PAD: Intelligently skip saving memcard state for runahead” · 3bbce19df 2023-03 “Pad: Don't backup card state unconditionally” · 33f5d9cb9 2023-03 “System: Remove BIOS from save state”

### duckstation-126 — Size auxiliary output to what's displayed, not to the source resolution
Lens: laziness · Source: DuckStation
Principle: Thumbnails and previews only need a fixed small size. Rendering or reading them back at the source's full resolution makes their cost grow with quality settings for no visible gain. Render at the target size on the GPU, then read back only those pixels.
Why it is faster: —
Applies elsewhere: responsive `srcset` images, video-player seek thumbnails from low-res sprite sheets, GPU-side downscale before `readPixels`, database previews using `LIMIT`.
Watch out: —
Seen in DuckStation: Save-state screenshots were read back at the full internal resolution, which caused visible stalls at high upscaling. `System::SaveState` now calls `RenderScreenshot` at a fixed 256-pixel size. `FlipTextureDataRGBA8` and `ConvertTextureDataToRGBA8` became shared `HostDisplay` helpers. c2916e071 2021-04 “System: Render save state screenshots at fixed resolution”

### duckstation-127 — Build asset tables only from what the code actually references
Lens: memory-layout · Source: DuckStation
Principle: Rasterizing an entire icon font or CJK range into a texture atlas costs startup time and GPU memory, and usually less than 5% of it is used. Scan the sources (or translations) at build time for the symbols that are actually used and generate a minimal range table.
Why it is faster: —
Applies elsewhere: font subsetting with `pyftsubset`/`glyphhanger` for web fonts, tree-shaking icon libraries, PurgeCSS, stripping unused locales from Moment or ICU bundles.
Watch out: —
Seen in DuckStation: `scripts/generate_update_fa_glyph_ranges.py` finds every `ICON_FA_*` token in `src/`, maps each one to its UTF-16 codepoint, and rewrites `range_fa[]` in `imgui_manager.cpp`, so only the icons in use get rasterized. A companion script builds the Japanese and Chinese glyph ranges from the characters that appear in the `.ts` translation files. ea8d77996 2022-08 “ImguiManager: Only rasterize font awesome glyphs which we need” · 0d03aa2b4 2022-08 “generate GlyphRanges for ja/zh-cn based on *.ts files”

### duckstation-128 — Convert sample formats in fixed-size SIMD chunks
Lens: simd · Source: DuckStation
Principle: Converting audio between int16 and float one sample at a time wastes the vector units. If the staging buffer is a power-of-two multiple of the vector width and aligned, the loop can widen, convert and scale 8 samples per iteration with no remainder handling. Provide a scalar fallback for other architectures.
Why it is faster: —
Applies elsewhere: PCM conversion in FFmpeg's `swresample`, Web Audio worklets using WASM SIMD, image pixel-format conversion (libyuv), ML int8 dequantization.
Watch out: —
Seen in DuckStation: The time-stretching audio stream (SoundTouch, float-based) buffers frames in `CHUNK_SIZE` pieces (a power of two). `S16ChunkToFloat` uses `_mm_unpacklo/hi_epi16`, then `_mm_srai_epi32(…,16)` for sign extension, then `_mm_cvtepi32_ps` and `_mm_mul_ps`. The NEON version uses `vld1q` and `vcvtq_f32_s32`. `FloatChunkToS16` does the reverse. A later fix keyed these paths on the project's `CPU_X64`/`CPU_AARCH64` macros so non-MSVC x86 builds also get the SSE path. 68b5dd869 2022-07 “SPU: Add time stretched audio output” · 4eb6b9082 2022-12 “AudioStream: Fix building without SSE on Linux”

### duckstation-129 — Never block the UI thread on a network round trip
Lens: concurrency · Source: DuckStation
Principle: Making the UI thread wait synchronously for a worker (even while pumping events) freezes the UI for the duration of a network call and risks re-entrancy bugs. Post the work, return immediately, and deliver the result back to the UI thread through a queued callback.
Why it is faster: —
Applies elsewhere: Android `NetworkOnMainThreadException`, `async`/`await` in UI frameworks, JS `fetch().then` instead of synchronous XHR, Qt `QueuedConnection`.
Watch out: —
Seen in DuckStation: The RetroAchievements login dialog ran `executeOnEmulationThread(..., true)` (blocking) inside `processEvents`. It now posts the login lambda without blocking, and the lambda calls `QMetaObject::invokeMethod(this, "processLoginResult", Qt::QueuedConnection, Q_ARG(bool, result))`. 2e86b65bf 2021-04 “Qt/AchievementLoginDialog: Make login fully asynchronous”

### duckstation-130 — Dispatch through a two-level table of code pointers and link blocks directly
Lens: jit-interpreters · Source: DuckStation
Principle: If every jump between compiled fragments goes through a hash-map lookup, the dispatcher becomes the bottleneck. Index a sparse two-level table by the guest address to get the code pointer. When the target of a direct branch is known, patch the branch to jump straight to it. Most transitions then cost no lookup at all.
Why it is faster: —
Applies elsewhere: threaded interpreters (computed goto into a handler table), inline caches in JS engines that patch call sites, page-table-style radix lookups for IP routing or sparse ID maps.
Watch out: —
Seen in DuckStation: The code cache was rewritten around NewRec's block management. `g_code_lut` is an array of `LUT_TABLE_COUNT = 0x10000` pointers to 16 K-entry tables, indexed by `pc >> 16` and then the low bits, giving one slot per PC. Unused tables share a single default table. Blocks keep up to `MAX_BLOCK_EXIT_LINKS = 2` exit links, and a `BlockLinkMap` lets `BacklinkBlocks()` and `UnlinkBlockExits()` repoint linked jumps when a block is compiled or invalidated. The mature form in `cpu_code_cache_private.h` still uses this layout. 79e1ae3e5 2023-10 “CPU/CodeCache: Rewrite using new-rec's block management”

### duckstation-131 — Adaptive invalidation: hardware write traps for stable code, manual checks for churning code
Lens: jit-interpreters · Source: DuckStation
Principle: Write-protecting the pages that hold cached code makes self-modification detection free until a write actually happens. A page that is rewritten constantly, though, takes a fault every time. Count invalidations per page over a time window. Hot pages switch to a cheap explicit check, and blocks that recompile too often fall back to the interpreter.
Why it is faster: —
Applies elsewhere: JS engines that deoptimize a function permanently after N bailouts, copy-on-write GCs falling back to card marking, caches that stop caching keys whose hit rate is too low.
Watch out: —
Seen in DuckStation: Each page has a `PageProtectionInfo` with `invalidate_count`/`invalidate_frame`. More than `INVALIDATE_COUNT_FOR_MANUAL_PROTECTION` (4) invalidations within the frame window switch the page to `PageProtectionMode::ManualCheck`, so blocks compare their source bytes instead of relying on a fault. More than 3 recompiles within 15 frames marks the block `FallbackToInterpreter`. The window was later widened from 20 to 60 frames to cut stutter. Separately, the fault handler now checks the cheap “RAM write” case before the backpatch-map lookup. 79e1ae3e5 2023-10 “CPU/CodeCache: Rewrite using new-rec's block management” · ff9dc11a3 2024-02 “CPU/CodeCache: Increase frame delta for manual protected blocks” · 454cceab8 2024-02 “CPU/CodeCache: Move backpatch lookup after RAM check”

### duckstation-132 — Map the same memory twice to drop software write checks
Lens: memory-layout · Source: DuckStation
Principle: Map one shared-memory object at two virtual addresses. Put hardware page protection on one view and keep the other fully writable. Hot paths write through the protected view and pay nothing until a fault fires. Trusted code such as DMA, debuggers and loaders writes through the alias without tripping the trap. The per-write “is this page special?” test disappears.
Why it is faster: —
Applies elsewhere: ring buffers mapped twice back to back so wraparound needs no branch, GC write barriers built on mprotect, a JIT that maps code RW in one view and RX in another (W^X).
Watch out: —
Seen in DuckStation: `Bus::AllocateMemory()` maps the RAM region twice: `g_ram`, which can be protected, and `g_unprotected_ram`. Code pages are made read-only with `MemMap::MemProtect` in `SetRAMPageWritable()`. Before this, every write checked `g_ram_code_bits`. Interpreter-side safe accesses and the achievements reader now use `g_unprotected_ram`. The commit message is “Get rid of write-time checks.” a062b00c1 2023-10 “Bus: Protect main RAM buffer”

### duckstation-133 — Route address-decoded accesses through a per-page handler table
Lens: lookup-tables · Source: DuckStation
Principle: Replace a cascade of range comparisons (`if addr < X … else if …`) with an array of function pointers indexed by `addr >> PAGE_SHIFT`. Each access then costs one shift, one load and one indirect call. Swapping the whole table changes behaviour for every page at once.
Why it is faster: —
Applies elsewhere: MMIO dispatch in device simulators, URL routers that index by the first path segment, syscall tables, opcode dispatch tables.
Watch out: —
Seen in DuckStation: The 2000-line hand-written decode tree in `bus.cpp` was replaced by `g_memory_handlers`, a LUT allocated in the same shared-memory block as RAM and BIOS. `CPU::GetMemoryReadHandler()` does `return base[address >> Bus::MEMORY_LUT_PAGE_SHIFT];`. A second table, `g_memory_handlers_isc`, serves the cache-isolated mode, and `UpdateMemoryPointers()` swaps between them when the status register changes. 56fc207af 2023-10 “CPU: Use lookup tables for memory access”

### duckstation-134 — Store pre-biased pointers in a translation table
Lens: memory-layout · Source: DuckStation
Principle: A page table normally stores the page's base pointer, so each lookup needs `base + (addr & mask)`. Store `base - page_start` instead, and the final address is just `entry + addr`. That saves an AND and a register on every access, and identical mirrors can share the same physical memory.
Why it is faster: —
Applies elsewhere: software TLBs in emulators and sanitizers, segment-relative addressing, relocation tables that store pre-adjusted deltas.
Watch out: —
Seen in DuckStation: `GetLUTFastmemPointer(address, ram_ptr)` returns `ram_ptr - address`. The AArch32/AArch64/x64 emitters dropped the `and_(RARG2, addr, PAGE_MASK)` and now load through `[entry + address_reg]`. A follow-up let the LUT cover all RAM mirrors (`RAM_MIRROR_END`) rather than only the first copy, so mirror accesses also take the fast path. 9d40164f8 2023-10 “CPU/Recompiler: Offset LUT fastmem pointers” · c179473c2 2023-10 “Bus: Allow LUT fastmem on mirrors”

### duckstation-135 — Liveness-driven register allocation with constant propagation in a block compiler
Lens: jit-interpreters · Source: DuckStation
Principle: A simple JIT loads guest registers from memory and stores them back on every instruction. A better one first analyses the block: which registers are read, which are dead, and which hold a value known at compile time. Values then stay in host registers, constants fold into immediates, and stores are emitted only for registers that are live at an exit or a call.
Why it is faster: —
Applies elsewhere: baseline JITs in V8 (Sparkplug to Maglev), eBPF JITs, WebAssembly single-pass compilers such as Liftoff, shader compilers.
Watch out: —
Seen in DuckStation: NewRec (`cpu_newrec_compiler.h`; today `cpu_recompiler.cpp`) has x64, AArch64 and RISC-V64 backends from day one, and AArch32 was added later. Each `InstructionInfo` carries per-register `RI_LIVE`/`RI_USED`/`RI_LASTUSE` flags. Host registers have `HR_MODE_READ`/`HR_MODE_WRITE` (valid/dirty) state, and instruction templates declare `TF_READS_S`, `TF_WRITES_D`, `TF_COMMUTATIVE`, `TF_RENAME_WITH_ZERO_IMM` and so on. The `FLUSH_*` flag sets flush only what each event needs, so a C call frees caller-saved registers while a branch flushes guest registers. Constant registers go through `m_constant_regs_valid`/`_dirty`. 9501439d6 2023-10 “CPU: Add new experimental recompiler”

### duckstation-136 — Prefer callee-saved registers for values that cross calls
Lens: jit-interpreters · Source: DuckStation · Also: ares
Principle: If a long-lived value sits in a caller-saved register, every out-of-line helper call forces a spill and a reload around it. Hand out callee-saved registers first, and use caller-saved ones as the fallback. The value then survives calls for free, and only the prologue and epilogue pay.
Why it is faster: —
Applies elsewhere: hand-written assembly loops that call `memcpy`, register allocators in LLVM and GCC (which weight cost by call density), interpreters that keep the dispatch pointer in a callee-saved register.
Watch out: —
Seen in DuckStation: The NewRec host-register search now returns the first free register with `HR_CALLEE_SAVED` and only falls back to the first caller-saved one. Load and store slow paths call C, so this removed spills around memory ops. On AArch32, `Mul` was likewise reworked to reuse argument registers rather than allocate two more. 8fe6ef1fa 2023-12 “CPU/NewRec: Allocate callee-saved registers before caller-saved” · cce1ec598 2023-10 “CPU/Recompiler/AArch32: Reduce register usage of Mul” • Also seen in ares (ares-119): A lazy scheme that kept `dmem.data` in scratch `reg(2)` and invalidated it after every possible call was replaced by `beginFunction(3, 4)` plus one `mov64(DmemReg /*sreg(3)*/, ...)` in the prologue. Supporting this took a nall change that lets a recompiler declare how many saved registers it uses. 716d41735 2026-04 “n64: use sreg(3) in RSP JIT to hold the DMEM pointer”

### duckstation-137 — Specialize compiled code on values observed at compile time
Lens: jit-interpreters · Source: DuckStation
Principle: When you compile at run time, you can see the program's actual state. Use the current register values as speculative constants to predict addresses and operands. Then emit the specialized path, or avoid a fast path that would certainly fail. Guard or recompile if the guess turns out wrong.
Why it is faster: —
Applies elsewhere: type feedback and shape-based inline caches in JS VMs, PGO, query planners that build plans from parameter sniffing, profile-guided devirtualization.
Watch out: —
Seen in DuckStation: NewRec keeps `m_speculative_constants` (regs, a small memory map, and `cop0_sr`), seeded from `g_state.regs` in `InitSpeculativeRegs()`. A load or store whose speculative address is not RAM skips fastmem, so no fault and backpatch is needed. When a backpatch does happen, the block is marked `NeedsRecompile` and recompiled with that knowledge. The same machinery detects stores into the block being compiled and truncates it (SMC inside a block). It also lets `mult`/`div` on speculative operands be precomputed. Later cleanup dropped a duplicate speculative register read. 6592cafad 2023-10 “CPU/NewRec: Add speculative constants” · e10ff550d 2023-11 “CPU/NewRec: Handle inside-block SMC” · 53c0dc8bb 2024-02 “CPU/NewRec: Speculative execution for mult/div” · 640ed4a2d 2024-07 “CPU/Recompiler: Eliminate redundant speculative reg read”

### duckstation-138 — Reorder instructions to remove pipeline-hazard bookkeeping
Lens: jit-interpreters · Source: DuckStation
Principle: Architectural quirks such as branch delay slots force a naive translator to emit extra state handling. If the instruction in the delay slot has no dependency on the branch, move it ahead of the branch. Ordinary code then comes out, with no special case. Widening the set of cases that are provably safe directly widens the gain.
Why it is faster: —
Applies elsewhere: compiler instruction scheduling, filling MIPS and SPARC delay slots, reordering independent awaits to overlap I/O.
Watch out: —
Seen in DuckStation: `SWAP_BRANCH_DELAY_SLOTS` was already on. Since the load delay is always updated, swapping no longer has to be refused when the slot reads a delayed register, so stores, `lwc2`/`swc2` and branches on delayed registers now qualify. ab4c6f2dd 2024-05 “CPU/NewRec: Enable delay slot swapping in more situations”

### duckstation-139 — Place generated code near the code it calls
Lens: jit-interpreters · Source: DuckStation
Principle: On x86-64 and ARM64, direct branches and PC-relative addressing reach only ±2–4 GB. Allocate the JIT buffer close to the executable, before other big reservations take the nearby space. Calls into the runtime and accesses to globals then become single PC-relative instructions, not trampolines or 64-bit immediates.
Why it is faster: —
Applies elsewhere: LuaJIT and V8 reserving code space near the binary, `-mcmodel=small`, hot/cold function splitting to keep branches short.
Watch out: —
Seen in DuckStation: `JitCodeBuffer` scans in steps for a region within `0x80000000 - NextPow2(256MB + size)` of the binary's base. Allocation order was later changed so the JIT reserves first and the fastmem arena second. The code buffer is now always allocated dynamically via `MemMap::AllocateJITMemoryAt`, rather than living in a huge `.bss`. On AArch64, `armIsInAdrpRange()` uses `adrp`+`add` in place of a trampoline for targets within ±4 GB. Far code (backpatch-only under NewRec) was halved. 5cf41a41f 2023-11 “JitCodeBuffer: Allocate within 32-bit range” · ebf50edb7 2024-06 “System: Allocate JIT space before fastmem area” · 44142ab4f 2024-06 “CPU/Recompiler: Prefer adrp over trampoline for +/-4GB branches” · be8fbafd7 2024-06 “CPU/CodeCache: Always dynamically allocate code buffer” · f3671d21f 2024-06 “CPU/CodeCache: Reduce far code size when using NewRec”

### duckstation-140 — Materialize a base register on first use, not up front
Lens: laziness · Source: DuckStation
Principle: Preloading a pointer into a reserved register for a whole function costs an instruction on entry, and on register-starved targets it permanently takes a register away. Track a “loaded” flag at compile time and emit the load only before the first instruction that needs it. Reset the flag when the value may change.
Why it is faster: —
Applies elsewhere: lazily computing the GOT pointer in PIC code, deferring `this` captures in closures, lazy TLS base loads.
Watch out: —
Seen in DuckStation: On AArch32, the fastmem base used to be loaded into `RMEMBASEPTR` at block entry whenever the block had `ContainsLoadStoreInstructions`. `EnsureMembaseLoaded()` now emits the `ldr` once, on the first memory op, and `EmitUpdateFastmemBase()` clears `m_membase_loaded`. This also frees the register for the rest of the block. 52e0d8d47 2023-10 “CPU/Recompiler/AArch32: Load membase on demand”

### duckstation-141 — Don't optimize the pathological case; punt it to the slow path
Lens: jit-interpreters · Source: DuckStation
Principle: Supporting a rare, complex pattern in the fast path adds bookkeeping and invalidation coupling that every common case then pays for. Detect it and decline. Let the interpreter or generic path handle it, and keep the fast path's invariants simple.
Why it is faster: —
Applies elsewhere: JITs refusing to compile functions that contain `eval` or `with`, databases falling back from vectorized to row-at-a-time execution on exotic types, SIMD loops with a scalar epilogue for awkward tails.
Watch out: —
Seen in DuckStation: A direct branch in a delay slot used to be compiled by fetching the second slot from the first branch's target, which is probably on another page and so entangles invalidation. Block analysis now returns `false` and logs “Direct branch in delay slot … skipping block” (commit note: “Invalidation nightmare”). a499e2145 2023-12 “CPU/CodeCache: Don't try to compile direct double branches”

### duckstation-142 — Bake an indirection, not a value, when the value can change
Lens: jit-interpreters · Source: DuckStation
Principle: If generated or cached code embeds a parameter that can change at run time, every change forces invalidation and recompilation. Embed a pointer to the parameter and read it at run time. The read costs a little, and the code never needs flushing because of that parameter.
Why it is faster: —
Applies elsewhere: referencing CSS custom properties rather than baking values into compiled styles, uniform buffers rather than shader constants, config objects read through an atomic pointer.
Watch out: —
Seen in DuckStation: Blocks running from BIOS ROM get `BlockFlags::NeedsDynamicFetchTicks`. At run time they add `block->size * *Bus::GetMemoryAccessTimePtr(pc, Word)` in place of a precomputed `uncached_fetch_ticks`, so games that reprogram memory timings no longer trigger recompiles. 2e96931c3 2024-07 “CPU/CodeCache: Dynamically compute BIOS memory access timing”

### duckstation-143 — Align hot global state to the real cache-line size
Lens: memory-layout · Source: DuckStation
Principle: The linker or LTO can place a hot struct so it straddles a cache line, which can cost double-digit percentages for no visible reason. Align hot state explicitly with `alignas(cache_line)`. Use the platform's real line size, which is 128 bytes on Apple Silicon, not 64. Separating atomics that different threads write into different lines also avoids false sharing.
Why it is faster: —
Applies elsewhere: `std::hardware_destructive_interference_size`, Java's `@Contended`, padding per-core counters in kernels and allocators.
Watch out: —
Seen in DuckStation: `alignas(HOST_CACHE_LINE_SIZE) State g_state;` and SPU voices got `ALIGN_TO_CACHE_LINE`. The commit notes that this fixed a ~12% regression caused by LTO placing the CPU struct mid-line. `HOST_CACHE_LINE_SIZE` is 128 on `__APPLE__ && __aarch64__`, and the GPU backend's FIFO read and write pointers are aligned to it. 77488db3d 2024-07 “Misc: Align CPU state and SPU voices to cache lines” · 17b973689 2024-04 “Misc: Use a 128b cache line size on Apple Silicon”

### duckstation-144 — Order struct fields so hot ones stay in short-offset range
Lens: memory-layout · Source: DuckStation
Principle: Load and store instructions encode only a limited immediate offset: 4 KB on ARM32, scaled 12-bit on ARM64. Generated or compiled code that addresses fields beyond that range needs extra address arithmetic. Put large, rarely used arrays at the end of the struct, shrink oversized fields, and gather scattered globals into one struct. The hot members then sit within cheap reach of a single base pointer.
Why it is faster: —
Applies elsewhere: kernel `task_struct` layout tuning, ordering game ECS components by access frequency, Go and Rust struct reordering to cut padding.
Watch out: —
Seen in DuckStation: `GPU_HW`'s big `m_batch_pipelines` array moved to the end of the class (“Helps keep the rest in range for ARM32's 4K displacement”). In `CPU::State`, the PGXP register arrays moved ahead of the 4 KB icache and scratchpad arrays. `m_resolution_scale`/`m_multisamples` became `u8`, and flags became one-bit bitfields. SPU state was gathered into a single `SPUState` struct. 99b0fb591 2024-04 “GPU/HW: Move pipelines to end of struct” · 35cdfc5ef 2024-02 “CPU: Move PGXP registers earlier in struct” · b34742cdd 2024-03 “GPU/HW: Slightly reduce struct size” · 536b48d70 2024-07 “SPU: Pack state in struct”

### duckstation-145 — Embed objects by value instead of behind heap pointers
Lens: memory-layout · Source: DuckStation
Principle: A `unique_ptr` member adds a dependent load and a likely cache miss on every access, plus allocation at startup. If the object's lifetime matches its owner's, make it a plain member or a static, constructed once. Access then becomes a fixed offset.
Why it is faster: —
Applies elsewhere: intrusive linked lists, small-object inlining in Rust (`Box` to inline field), avoiding pointer-to-pimpl in hot classes.
Watch out: —
Seen in DuckStation: Every subsystem's timing events changed from `std::unique_ptr<TimingEvent>` created by `TimingEvents::CreateTimingEvent()` to static objects such as `static TimingEvent s_command_event{"CDROM Command Event", 1, 1, &CDROM::ExecuteCommand, nullptr};`. All `->` calls became `.` calls. The scheduler walks these events constantly. 4f7ddfaae 2024-07 “TimingEvents: Remove pointer indirection”

### duckstation-146 — Schedule by absolute deadline, not by decrementing every timer
Lens: algorithms · Source: DuckStation
Principle: Storing “ticks remaining” in each event means time can only advance by subtracting from every active event, which is O(n) per step. Store an absolute 64-bit deadline instead and keep the list sorted. Advancing time is then a single counter update, and only events that are actually due get touched.
Why it is faster: —
Applies elsewhere: OS timer wheels and hrtimers, game-loop schedulers, `setTimeout` queues in JS runtimes, priority-queue event simulation.
Watch out: —
Seen in DuckStation: `TimingEvent::m_downcount` became `m_next_run_time` (64-bit `GlobalTicks`). The loop `for (event = head; event; event = event->next) event->m_downcount -= time;` is gone, and `CommitGlobalTicks()` just runs events while `global_tick_counter >= event->m_next_run_time`. Related changes: interrupt dispatch moved to the end of the loop (“Saves another iteration”), the CPU downcount is refreshed whenever the head is re-sorted, and the deferred frame-done flag was replaced by a direct call. 86d4d9275 2024-08 “TimingEvents: Switch to 64-bit counters” · 9b77a9aff 2024-07 “TimingEvents: Move interrupt dispatch to end of loop” · f82d08e22 2023-10 “TimingEvents: Reset downcount when head is resorted” · 41c860778 2024-08 “TimingEvents: Don't defer frame done callback”

### duckstation-147 — Turn a runtime selector into a template parameter
Lens: compiler-build · Source: DuckStation
Principle: When a function branches on an enum argument throughout its body, instantiate it once per enum value. Then dispatch once through a table of the instantiations. Every inner `switch(channel)` becomes a compile-time constant and folds away, and each copy is a straight line.
Why it is faster: —
Applies elsewhere: image codecs specialized per pixel format, serializers templated on endianness, kernels in C++ BLAS libraries templated on transpose flags.
Watch out: —
Seen in DuckStation: `DMA::TransferChannel(Channel)` became `template<Channel channel> TransferChannel()`, and so did `TransferMemoryToDevice`/`TransferDeviceToMemory`. Callers index `s_channel_transfer_functions[channel_index]()`. The commit reports “~20% speedup in FMV playback… CPUs hate branches.” f5ddd7ba3 2023-12 “DMA: Template transfer functions”

### duckstation-148 — Tell the compiler which branches are cold or impossible
Lens: branching-fast-paths · Source: DuckStation · Also: Seastar
Principle: Marking error and edge paths `[[unlikely]]` makes the compiler lay them out of line, so the hot path falls straight through and stays dense in the instruction cache. `__assume(false)` or `__builtin_unreachable()` in impossible switch defaults lets it drop range checks entirely.
Why it is faster: —
Applies elsewhere: Linux's `likely()`/`unlikely()`, Rust's `#[cold]` and `unreachable_unchecked`, V8's `V8_UNLIKELY` on error paths.
Watch out: —
Seen in DuckStation: In release builds `UnreachableCode()` expands to `ASSUME(false)`, and `Y_OnPanicReached` became `[[noreturn]]`. DMA address wraparound and transfer-buffer fallbacks are `[[unlikely]]`, while `g_gpu->BeginDMAWrite()` is `[[likely]]`. Log macros mark `if (level <= GetLogLevel())` as unlikely to “Move it out of the hot path.” c0af478df 2023-09 “Common: Add ASSUME() and use for UnreachableCode()” · bbd021bd5 2023-12 “DMA: Memory wraparound is unlikely” · 890f3fcf5 2024-06 “Common/Log: Make actually writing unlikely” • Also seen in Seastar (seastar-072): 0eaf3e6015 2018-07-01 `need_preempt()` returns `__builtin_expect(g_need_preempt, false)`. • 15ad5bc2f1 2018-07-19 `logger::is_enabled()` predicted false: "info+ levels will be mispredicted, but logging is so slow anyway that we don't care." • 2f089f63e7 2019-08-24 `future_state_base::failed()` marked unlikely, spreading to every inline caller of `future::failed()`; 00855e4bdc the local-free ownership check marked likely.

### duckstation-149 — Filter before you format
Lens: laziness · Source: DuckStation
Principle: Logging calls build their argument packs even when the message will be dropped. Check the level first, so a disabled log line costs one compare and the type-erasure and formatting work never runs.
Why it is faster: —
Applies elsewhere: SLF4J's `{}` placeholders and `isDebugEnabled()`, Python's `logger.debug("%s", x)` rather than f-strings, Rust's `log!` macros that check the level before evaluating arguments.
Watch out: —
Seen in DuckStation: The fmt logging template now wraps `WriteFmtArgs(..., fmt::make_format_args(args...))` in `if (level <= GetLogLevel())`. 03d650025 2024-01 “Log: Skip argument packing when below filter level”

### duckstation-150 — Write SIMD once through a thin portable vector class
Lens: simd · Source: DuckStation
Principle: Writing separate SSE and NEON intrinsics for each kernel doubles the code and leaves one path untested. Wrap 128-bit (and 64-bit) vectors in a header-only class of `ALWAYS_INLINE` operations with SSE4.1, NEON and scalar backends. Kernels are written once, compile to the same instructions, and new platforms get every kernel for free.
Why it is faster: —
Applies elsewhere: Highway, xsimd, `std::experimental::simd`, Rust's `std::simd`, WebAssembly SIMD as a portable target.
Watch out: —
Seen in DuckStation: `common/gsvector.h` (from PCSX2) has `gsvector_sse.h`, `gsvector_neon.h` and `gsvector_nosimd.h`, and the build now requires `-msse4.1`. The software scanout's per-arch `#if` blocks collapsed into one `GSVector4i` path: `(value & 0x3E0) | (value.srl16<10>() & mask) | (value & mask).sll16<10>()`. Half-width `GSVector2i` types were added for 2-component GPU math. Before the class existed, one-off NEON and SSE versions were hand-written for `GetPeakVolume()` (`vmaxq_s16`) and CHD `CopyAndSwap()`. 0ae6ddcf7 2024-06 “Common: Add GSVector” · f9bf2e823 2024-07 “GPU: Use GSVector for SW scanout” · 59a2309a8 2024-07 “GPU: Use half width vector types where appropriate” · ecb082b67 2023-10 “CDROM: Add NEON implementation of GetPeakVolume()” · e2efec12b 2023-10 “CDImageCHD: Add SSE/NEON implementation of CopyAndSwap()”

### duckstation-151 — Transpose a matrix so inner products read contiguously, then use multiply-add pairs
Lens: simd · Source: DuckStation
Principle: A matrix multiply whose inner loop walks a column strides through memory and can't load a vector. Store the matrix transposed so each dot product reads two contiguous rows. Then one `pmaddwd` (multiply 16-bit pairs and add into 32-bit) plus a horizontal add computes 8 products per instruction pair.
Why it is faster: —
Applies elsewhere: JPEG and video IDCTs, int8/int16 quantized neural-net inference (`vpmaddubsw`/`vpdpbusd`), audio FIR filters, CRC folding.
Watch out: —
Seen in DuckStation: MDEC first stored the scale/IDCT matrix transposed (with save-state migration) and introduced `IDCTRow(blk, &s_scale_table[y*8])`, which replaced a `z`-strided loop. The vectorized version is `GSVector4i::load(blk).madd_s16(GSVector4i::load(idct_matrix)).addp_s32()`, and the blocks became `alignas(VECTOR_ALIGNMENT)` `s16`. e1f90e6ba 2024-07 “MDEC: Store scale/IDCT matrix transposed” · b094c9cd8 2024-07 “MDEC: Vectorize IDCT”

### duckstation-152 — Vectorize per-pixel color conversion with fixed-point lanes
Lens: simd · Source: DuckStation
Principle: Color-space conversion applies the same integer formula to every pixel, which makes it ideal for SIMD. Widen to 32-bit only where products overflow 16 bits. Use shift pairs for sign extension and saturating packs for clamping, so eight pixels go through in a handful of instructions with no branches.
Why it is faster: —
Applies elsewhere: libyuv and libjpeg-turbo YUV/RGB paths, camera ISP pipelines, WebCodecs frame conversion, GPU-less video thumbnails.
Watch out: —
Seen in DuckStation: `YUVToRGB` computes `Cr.mul32l(359).add16(0x80).sra32<8>().ps32()` and similar, uses `sll16<7>().sra16<7>()` for 9-bit sign extension and `ps16()` for the clamp, then interleaves with `upl8`/`upl16`. This replaced an 8-iteration scalar loop with `std::clamp`. The commit reports “Almost 10x speedup on the function alone, 15% in practice ingame.” 9be7a3741 2024-07 “GSVector: SIMD-ify YUVToRGB”

### duckstation-153 — Repack pixel formats with byte shuffles
Lens: simd · Source: DuckStation
Principle: Converting 32-bit RGBX to packed 24-bit RGB or to 16-bit forms byte by byte is slow and branchy. With constant shuffle masks (`pshufb`/`tbl`), each output vector is built from two input vectors in one or two instructions.
Why it is faster: —
Applies elsewhere: image decoders that drop alpha channels, UTF-8 to UTF-16 transcoders (simdutf), network header byte-swapping, de-interleaving stereo audio.
Watch out: —
Seen in DuckStation: MDEC's 24-bit output takes four `GSVector4i` loads per 16 pixels and writes three stores via `rgbx0.shuffle8(mask00) | rgbx1.shuffle8(mask01)`, with masks such as `cxpr8(0,1,2,4,5,6,8,9,10,12,13,14,-1,-1,-1,-1)`. The 16-bit path was vectorized too. The commit reports a 15% (16-bit) and 6% (24-bit) FMV speedup. 6a3b1a272 2024-09 “MDEC: Vectorize 16 and 24-bit copy-out”

### duckstation-154 — Vector main loop plus scalar tail for fills
Lens: simd · Source: DuckStation
Principle: Fill the aligned bulk with full-width vector stores and finish the remainder with scalar stores. Hoist the wraparound check out of the loop so the common, non-wrapping case runs without a per-element modulo.
Why it is faster: —
Applies elsewhere: `memset` implementations, clearing framebuffers in software rasterizers, initializing pointer tables or bitmaps in allocators.
Watch out: —
Seen in DuckStation: VRAM fill builds `GSVector4i fill(color16 ×8)`, stores 8 pixels per iteration up to `AlignDownPow2(width, 8)`, then writes the tail. When `(x + width) <= VRAM_WIDTH` it skips the per-column `% VRAM_WIDTH`. `MemsetPtrs()` applies the same idea to fill pointer LUTs with 128-bit stores, and disables Clang's over-eager unrolling. 9a58327cf 2024-08 “GPU: Vectorize VRAM fills” · 0ee671249 2023-10 “Common: Add MemsetPtrs()”

### duckstation-155 — Drop known-zero taps and vectorize the FIR
Lens: simd · Source: DuckStation
Principle: Many filters have structurally zero coefficients, such as every other tap in a half-band resampler. Delete those taps so the work halves. Then lay the remaining coefficients out contiguously and run the multiply-accumulate four lanes at a time.
Why it is faster: —
Applies elsewhere: half-band decimators in SDR and audio DSP, image downscalers, convolution kernels that exploit symmetry.
Watch out: —
Seen in DuckStation: The 39-tap reverb resampler already skipped its zero taps. It now loads `resample_coeff` in groups of four and does `acc.add32(coeff.mul32l(srcs.s16to32()))` in five steps, which replaced the scalar `for (i < 20) out += coeff[i] * src[i*2]`. b84d3f14f 2024-08 “SPU: Vectorize reverb resampling”

### duckstation-156 — Compile hot kernels per ISA and pick one at startup
Lens: platform · Source: DuckStation
Principle: Build one source file several times with different `-m` flags, each into its own namespace, and export a table of function pointers from each. At startup, query the CPU and point a global at the best table. The binary keeps baseline compatibility, and newer CPUs still get AVX2.
Why it is faster: —
Applies elsewhere: glibc IFUNC, simdjson's runtime dispatch, zlib-ng and OpenSSL capability probing, Highway's `HWY_DYNAMIC_DISPATCH`.
Watch out: —
Seen in DuckStation: The software rasterizer was split out into `gpu_sw_rasterizer.inl`, which is included into the default, `SIMD` and `AVX2` (compiled with `-mavx2`) namespaces. `SelectImplementation()` sets `SelectedDrawTriangleFunctions` and its siblings from `cpuinfo_has_x86_avx2()`. A constexpr `g_dither_lut` is built at compile time. aa955b8ae 2023-12 “GPU/SW: Split out rasterizer, add dynamic selection”

### duckstation-157 — Strength-reduce and short-circuit identity operations
Lens: algorithms · Source: DuckStation
Principle: Replace division by a power of two with a shift where the rounding difference doesn't matter. Detect identity cases such as subtracting 0 and turn them into a plain copy, skipping the general arithmetic.
Why it is faster: —
Applies elsewhere: fixed-point DSP code, compilers' peephole passes, fast-pathing `x*1`/`x+0` in symbolic math or autodiff engines.
Watch out: —
Seen in DuckStation: `ZigZagInterpolate` (XA audio) changed `/ 0x8000` to `>> 15` and became `ALWAYS_INLINE_RELEASE`. In PGXP `SUB`, `rtVal == 0` now just copies the source's precise value instead of running the float overflow math. 68575d15a 2024-07 “CDROM: Shift instead of divide in ZigZagInterpolate” · 5672b0da9 2024-05 “CPU/PGXP: sub rd, rs, 0 => move rd, rs”

### duckstation-158 — Write results in place instead of via a temporary copy
Lens: memory-layout · Source: DuckStation
Principle: The pattern “copy the struct into a local, modify it, copy it back” moves the whole struct twice per call. Take a reference to the destination and write only the fields that change, reading inputs through references. For small trivially copyable types, pass by value so they travel in registers instead of through memory.
Why it is faster: —
Applies elsewhere: Rust's `&mut` out-parameters, avoiding spread copies in hot JS reducers, passing `std::string_view` or `span` by value.
Watch out: —
Seen in DuckStation: In PGXP shift, add/sub and SLT handlers, `PGXP_value ret = g_state.pgxp_gpr[rt]; … g_state.pgxp_gpr[rd] = ret;` became `PGXP_value& prdVal = …; prdVal.x = …`, which the commit calls a “small speedup”. Instruction query helpers now take `const Instruction` (a `u32`) by value, and string utilities take `std::string_view` by value. 37b1aa45d 2024-05 “CPU/PGXP: Minor optimization to shift instructions” · 11f7bfc46 2024-07 “CPU/PGXP: Refactor add/sub for small speedup” · a76495497 2024-08 “CPU/PGXP: Eliminate extra copy in SLT/SLTU” · 3b9c48978 2024-07 “CPU: Pass instruction query values by reference” · 4f84a9886 2023-12 “StringUtil: Pass string_views by value”

### duckstation-159 — Pack flags as bits and manipulate them with masks
Lens: memory-layout · Source: DuckStation
Principle: Flags stored one per byte in a union need byte-sized loads and stores and make combined tests awkward. Use one bit per flag in a single integer. Setting, testing and merging several flags then becomes one or two ALU operations, and the struct shrinks.
Why it is faster: —
Applies elsewhere: file-permission bits, ECS component masks, React's fiber flags, bitboards in chess engines.
Watch out: —
Seen in DuckStation: `PGXP_value`'s `union { u32 flags; u8 compFlags[4]; u16 halfFlags[2]; }` became plain `u32 flags` with `VALID_0 = 1<<0`, `VALID_1 = 1<<1` and so on, plus `SetValidComp`: `flags = (flags & ~(1u << comp)) | (valid << comp)`. That made merges like `prsVal.flags | (prtVal.flags & VALID_XY)` single operations. 29d4e04e3 2024-05 “CPU/PGXP: Use bit math for flags instead of union”

### duckstation-160 — Size lookup tables to the reachable domain and drop the bounds check
Lens: memory-layout · Source: DuckStation
Principle: If upstream code already clamps the key range, a table sized for a wider range wastes memory and cache, and a per-lookup range test re-checks a guarantee that already holds. Shrink the table to the true domain and index it directly.
Why it is faster: —
Applies elsewhere: glyph caches sized to the actual font range, histogram arrays sized to the clamped value range, spatial hashes sized to world bounds.
Watch out: —
Seen in DuckStation: The PGXP vertex cache went from `0x800*2` × `0x800*2` entries to 2048 × 2048, a quarter of the memory. GTE screen coordinates are always within ±1024, so `CacheVertex`/`GetCachedVertex` lost their `if (sx >= -0x800 && …)` checks, and a `DebugAssert` took their place. 22c76d43c 2024-05 “CPU/PGXP: Reduce vertex cache memory”

### duckstation-161 — Avoid per-call heap allocation with reusable and small-buffer storage
Lens: allocation · Source: DuckStation
Principle: Allocating a fresh buffer, vector, string or assembler object on every call adds allocator overhead and cache churn. For frequently repeated operations, keep a persistent buffer and resize it. Use small-buffer containers (SmallVector, SmallString) so typical sizes live on the stack, and reset heavy objects rather than reconstructing them.
Why it is faster: —
Applies elsewhere: `sync.Pool` in Go, `ArrayPool<T>` in .NET, `absl::InlinedVector`, reusing `StringBuilder`s, per-frame arenas in game engines.
Watch out: —
Seen in DuckStation: The AArch64 NewRec compiler keeps its `m_emitter`/`m_far_emitter` as members and calls `GetBuffer()->Reset(...)`, where it used to `make_unique<Assembler>` per block. `SortEvents()` uses `llvm::SmallVector<TimingEvent*, 16>`. Achievements state save/load reuses a static `DynamicHeapArray`. GPU state load deserializes straight into `g_vram` rather than a temporary 1 MB `FixedHeapArray`. Fullscreen UI per-frame settings strings became `TinyString`/`SmallString`. d00ed000f 2023-10 “CPU/NewRec/AArch64: Don't recreate assembler every time” · b4136d283 2024-07 “TimingEvents: Avoid heap allocation in SortEvents()” · e7e09023e 2024-06 “Achievements: Avoid malloc on state save/load” · 044d7a479 2024-06 “GPU: Eliminate heap allocation on state load” · de1a1af90 2024-04 “FullscreenUI: Avoid per-frame memory allocations with SmallString”

### duckstation-162 — Bulk-copy ring-buffer ranges with memcpy for trivial types
Lens: memory-layout · Source: DuckStation
Principle: Popping N elements from a circular buffer one at a time costs a modulo and a branch per element. For trivially copyable types, copy the contiguous run up to the wrap point with `memcpy`, then the remainder. That is at most two vectorized copies.
Why it is faster: —
Applies elsewhere: audio ring buffers, kernel pipe and socket buffers, LMAX-disruptor-style queues, `VecDeque::as_slices` in Rust.
Watch out: —
Seen in DuckStation: `FIFOQueue::PopRange()` got a `std::is_trivial` overload that loops on `contig_count = min(count, CAPACITY - m_head)` with `memcpy`. It is used by MDEC and GPU data FIFOs. da2dc5fa1 2024-07 “FIFOQueue: Optimize PopRange() for trivially-copyable types”

### duckstation-163 — Shadow API state and skip redundant driver calls
Lens: gpu-rendering · Source: DuckStation
Principle: Graphics APIs with global state (GL, D3D11) charge per call, so re-sending unchanged state each draw wastes CPU in the driver. Keep a copy of the last applied state, compare cheaply (often packed into one integer), and issue only the deltas.
Why it is faster: —
Applies elsewhere: WebGL wrappers such as three.js `WebGLState`, React's virtual-DOM diffing, canvas libraries caching `fillStyle`, avoiding redundant `setState` in UI frameworks.
Watch out: —
Seen in DuckStation: `OpenGLDevice` now keeps `m_last_rasterization_state`, `m_last_depth_state` and `m_last_blend_state`. `ApplyDepthState()` returns early on equality and calls `glDepthMask` only if `depth_write` changed. Blend factors and ops were packed into bitfields so the comparison is cheap. D3D11 got a fix so a null SRV no longer matches, which removed redundant SRV rebinds on render-target changes. ce7f6128e 2023-10 “OpenGLDevice: Cache intermediate state” · 067571616 2024-06 “D3D11Device: Fix redundant SRV set on RT change”

### duckstation-164 — Don't restart render passes for clears or unchanged targets
Lens: gpu-rendering · Source: DuckStation
Principle: On tiled GPUs especially, ending and restarting a render pass flushes tile memory to DRAM and reloads it. Clear attachments inside the active pass, and leave the pass open when the requested targets are the ones already bound.
Why it is faster: —
Applies elsewhere: Vulkan and Metal mobile games, WebGPU render-pass batching, Flutter/Skia merging draw ops into one pass.
Watch out: —
Seen in DuckStation: `VulkanDevice` clears and invalidates of a bound render target or depth buffer now use `vkCmdClearAttachments()` where they used to call `EndRenderPass()`. `D3D12Device`'s set-render-targets path compares each RT, the depth target and the flags, and ends the pass only if something `changed` or a clear is pending. 41cc60e87 2024-04 “VulkanDevice: Use vkCmdClearAttachments() to avoid render pass restart” · 5e8870ec6 2024-07 “D3D12: Avoid redundant render pass restarts”

### duckstation-165 — Track dirty regions finely to avoid unnecessary syncs and copies
Lens: gpu-rendering · Source: DuckStation
Principle: A single coarse “dirty” rectangle forces a flush or copy whenever any read overlaps it. Keep separate dirty regions per source of writes (rendered versus uploaded). Also track the actual range a batch reads (min/max texture coordinates), not the whole texture page. Only true overlaps trigger the expensive synchronization.
Why it is faster: —
Applies elsewhere: dirty-rect compositing in browsers and window managers, incremental build systems tracking which inputs a step really read, database page-level vs row-level locking.
Watch out: —
Seen in DuckStation: `m_vram_dirty_rect` split into `m_vram_dirty_draw_rect` and `m_vram_dirty_write_rect`, with per-type `m_texpage_dirty` bits and `UpdateVRAMReadTexture(drawn, written)` (commit note: “Significant performance improvement in Persona 2”). Copies then used better draw-versus-write heuristics. `ComputePolygonUVLimits()` grows `m_current_uv_range`, and a flush happens only if `m_current_uv_range.Intersects(m_vram_dirty_rect)`. 23d5b20da 2023-12 “GPU/HW: Split dirty rect into draw/write” · 87a7c0946 2023-12 “GPU/HW: Improve heuristics for draw/write when copying” · 8446856ec 2023-11 “GPU/HW: Trace min/max UVs for hazards”

### duckstation-166 — Read the framebuffer in the shader instead of splitting draws
Lens: gpu-rendering · Source: DuckStation
Principle: Blend modes that fixed-function hardware can't express usually mean multiple passes, a separate depth/mask buffer, or one draw per primitive. Where framebuffer fetch or feedback loops are available, read the destination pixel in the fragment shader and blend there. Batches stay intact, and extra attachments can be dropped.
Why it is faster: —
Applies elsewhere: programmable blending on mobile GPUs (`GL_EXT_shader_framebuffer_fetch`, Metal `[[color(0)]]`), deferred-shading G-buffer reads in a subpass, custom blend modes in 2D engines.
Watch out: —
Seen in DuckStation: A framebuffer-fetch blending path was added to `gpu_hw_shadergen`. With fbfetch, reverse-subtract blending no longer forces a per-polygon flush (“If we have fbfetch, we don't need to draw it in two passes”). Feedback-loop rendering lets `NeedsDepthBuffer()` return false when PGXP depth is off, so the VRAM depth texture isn't created at all. Shader-blend pipelines are created only when `m_allow_shader_blend` is set. ff1744407 2023-09 “GPU/HW: Add framebuffer fetch blending path” · 90b14d623 2023-12 “GPU/HW: Skip per-primitive flush for BG-FB with fbfetch” · 74f3c8d1a 2024-03 “GPU/HW: Enable feedback loops for rendering” · 1097eab60 2024-04 “GPU/HW: Don't create shader blend pipelines if disallowed”

### duckstation-167 — Copy within a resource instead of through a shadow copy
Lens: gpu-rendering · Source: DuckStation
Principle: If the API allows copies within the same texture, do the copy directly when source and destination don't overlap. That skips refreshing a shadow copy and the extra full-region copy and barrier it needs. Fall back to the shadow only for real overlaps or on APIs that forbid it.
Why it is faster: —
Applies elsewhere: `memmove` vs bounce buffers, canvas `drawImage` of a canvas onto itself, in-place compaction in GCs.
Watch out: —
Seen in DuckStation: A `texture_copy_to_self` feature bit was added, on for Vulkan, GL and Metal and off for D3D11/12. `GPU_HW::CopyVRAM` uses `m_vram_texture` as the source unless the regions overlap. Only then does it refresh `m_vram_read_texture` via `UpdateVRAMReadTexture()`. 2ca503639 2023-12 “GPU/HW: Avoid extra texture copy in Vulkan/GL”

### duckstation-168 — Emit quads as indexed geometry
Lens: gpu-rendering · Source: DuckStation
Principle: Drawing quads as raw triangle lists duplicates two of the four vertices. That is 50% more vertex data to write, upload and shade. Write unique vertices once and reference them with small 16-bit indices, so the post-transform cache reuses shared vertices.
Why it is faster: —
Applies elsewhere: sprite batchers in 2D engines, text renderers (6 indices, 4 vertices per glyph), mesh formats such as glTF.
Watch out: —
Seen in DuckStation: `GPU_HW` moved from `AddVertex` into a mapped vertex buffer to `MapGPUBuffer(required_vertices, required_indices)`, which maps both the vertex and index stream buffers (`m_batch_index_ptr`, with space capped at `u16` max). Draws are indexed. 250fb5683 2024-03 “GPU/HW: Switch to indexed draws”

### duckstation-169 — Pool and recycle GPU resources
Lens: allocation · Source: DuckStation
Principle: Creating and destroying textures goes through the driver allocator and can stall. Keep a pool keyed by size, format and type. `Fetch` returns a matching recycled texture, and `Recycle` puts it back. Resources can then be sized to what is actually needed and recreated freely, without paying creation cost.
Why it is faster: —
Applies elsewhere: render-graph transient resource pools (Frostbite, Unreal RDG), object pools in Unity, connection pools, slab allocators.
Watch out: —
Seen in DuckStation: `GPUDevice::FetchTexture()`, `RecycleTexture()` and `FetchAutoRecycleTexture()` replaced direct `CreateTexture()` across the HW renderer, post-processing and UI previews. The pool was later improved, with D3D11 textures constructed directly and a VRAM usage stat. With pooling cheap, the display texture is now sized per frame to `scaled_display_width × scaled_display_height`, where it used to be a fixed maximum size. a907e1f55 2023-12 “GPUDevice: Pool textures” · dc5e4120c 2023-12 “GPUDevice: Improve texture pooling” · 6b00bf016 2023-12 “GPU/HW: Dynamically size display texture”

### duckstation-170 — Compile shaders lazily and persist compiled pipelines
Lens: laziness · Source: DuckStation
Principle: Compiling every shader variant at startup front-loads seconds of work for variants that may never be used. Store the source and compile on first bind. Then persist the driver's compiled pipeline blobs to disk, so the next run skips compilation entirely.
Why it is faster: —
Applies elsewhere: V8 lazy function compilation, Vulkan `VkPipelineCache`, Unity and Unreal PSO caches, Python `.pyc` files.
Watch out: —
Seen in DuckStation: `OpenGLShader` now holds `m_source` and an `std::optional<GLuint> m_id`, and `Compile()` runs only when a pipeline first links it. Metal gained an on-disk pipeline cache (`MTLBinaryArchive`), and the cache read/write code was unified in `GPUDevice` alongside D3D12's pipeline library. a11c9faba 2023-08 “OpenGLDevice: Lazily compile shaders” · d981dc747 2024-09 “MetalDevice: Implement pipeline cache”

### duckstation-171 — Prune variant permutations that can never be used
Lens: gpu-rendering · Source: DuckStation
Principle: Shader permutation counts multiply across dimensions. Collapse dimensions that are equivalent, such as different raw-texture modes that produce the same code, and skip variants the current device can't use. Startup compile time and memory shrink in proportion.
Why it is faster: —
Applies elsewhere: Unity shader-variant stripping, feature-flag build matrices, CSS-in-JS dead-style elimination.
Watch out: —
Seen in DuckStation: Raw-texture modes were folded into a bit, so batch shaders iterate over `active_texture_modes = 4` where there used to be 9. The pipeline count fell from `5*5*9*2*2*2` to `5*active_texture_modes*…`. Earlier, with fbfetch, multipass render modes were skipped, and without it, shader-blend variants were skipped. c1792108f 2024-06 “GPU/HW: Eliminate raw texture pipeline permutations” · ff1744407 2023-09 “GPU/HW: Add framebuffer fetch blending path”

### duckstation-172 — Track last use to pick the cheap upload path
Lens: gpu-rendering · Source: DuckStation
Principle: Updating a resource that the GPU may still be reading needs a staged copy ordered in the command stream. If the resource hasn't been touched in the current command buffer, a direct, cheaper upload is safe. Stamp each resource with the fence counter of its last use and compare.
Why it is faster: —
Applies elsewhere: buffer orphaning (`MAP_DISCARD`) vs no-overwrite maps, generation counters in ECS handles, ETag-style “changed since” checks.
Watch out: —
Seen in DuckStation: `MetalTexture` got `m_use_fence_counter`, set on bind, copy and render-target use. `Update()` now computes `is_inline = (m_use_fence_counter == dev.GetCurrentFenceCounter())`, which replaced a hard-coded `const bool is_inline = true; // TODO: track this`. 8a0033dfb 2023-08 “MetalDevice: Avoid inline texture uploads when unnecessary”

### duckstation-173 — Use integer texel fetches for exact lookups
Lens: gpu-rendering · Source: DuckStation
Principle: When a shader wants an exact texel (palette lookups, unfiltered VRAM reads), converting to normalized float coordinates and going through the sampler costs a multiply and filtering setup, and risks rounding errors. `texelFetch`/`Load` with integer coordinates is direct and exact.
Why it is faster: —
Applies elsewhere: lookup-table textures in post-processing, data textures in GPGPU shaders, sprite atlases with pixel-exact sampling.
Watch out: —
Seen in DuckStation: In the HW renderer's fragment shader, `SAMPLE_TEXTURE(samp0, float2(icoord) * RCP_VRAM_SIZE)` became `LOAD_TEXTURE(samp0, int2(icoord), 0)` for direct and palette reads. The 4-bit path drops its X wrap mask because it can't wrap. e70f0e1bc 2024-03 “GPU/HW: Use texelFetch() instead of sampler”

### duckstation-174 — Cull primitives on the CPU before they reach the batch
Lens: gpu-rendering · Source: DuckStation
Principle: Clamp a primitive's bounds to the clip rectangle once. If the result is empty, drop it before generating vertices or rasterizing. Reuse the clamped rectangle for any later cost accounting rather than recomputing clamps.
Why it is faster: —
Applies elsewhere: frustum and viewport culling in game engines, skipping off-screen DOM paint (`content-visibility`), quadtree culling in map renderers.
Watch out: —
Seen in DuckStation: Sprites and lines are now culled against the drawing area like polygons. `AddDrawRectangleTicks`/`AddDrawLineTicks` take the precomputed `GSVector4i clamped_rect` and use `clamped_rect.width()`/`height()`, replacing four `std::clamp` calls each. ed5a99066 2024-07 “GPU: Cull off-screen sprites/lines”

### duckstation-175 — Import host memory so the GPU reads into it directly
Lens: gpu-rendering · Source: DuckStation
Principle: GPU-to-CPU readbacks normally land in a driver staging buffer and then get copied again into application memory. Where the API allows it, import the application's page-aligned buffer as GPU-visible memory and copy straight into it. That removes a full-size CPU copy.
Why it is faster: —
Applies elsewhere: `VK_EXT_external_memory_host`, CUDA pinned and mapped host memory, DMA-BUF zero-copy video pipelines, io_uring registered buffers.
Watch out: —
Seen in DuckStation: `VulkanDevice::TryImportHostMemory()` aligns the pointer down to `HOST_PAGE_SIZE` and queries `vkGetMemoryHostPointerPropertiesEXT`. It then allocates with `VkImportMemoryHostPointerInfoEXT` and returns a buffer plus an offset. The new `GPUDownloadTexture` abstraction can be “based in host/client memory” and is used for VRAM readbacks and screenshots. 4309d8ebf 2024-01 “VulkanDevice: Add `TryImportHostMemory`” · f936a36c8 2024-02 “GPUDevice: Add GPUDownloadTexture”

### duckstation-176 — Don't busy-wait on results that aren't ready
Lens: concurrency · Source: DuckStation
Principle: Polling for asynchronous results (GPU queries, futures) in a tight loop burns a CPU core and can starve the producer. If the result isn't available, stop and try again next frame.
Why it is faster: —
Applies elsewhere: non-blocking `getQueryParameter` in WebGL, `try_recv` with backoff, async readiness instead of spin-polling sockets.
Watch out: —
Seen in DuckStation: D3D11 GPU timing queries now `break` out of the readback loop on “Data not ready yet” instead of spinning on `GetData`. 547587af1 2024-05 “D3D11Device: Don't spin on CPU when GPU results aren't available”

### duckstation-177 — Event-driven I/O: epoll, poll-as-sleep and no Nagle for request/response
Lens: io · Source: DuckStation
Principle: Use the OS's scalable readiness API (epoll) rather than per-call select. When the loop has idle time, sleep inside the poll call so incoming messages wake it immediately rather than waiting for the next frame. For small interactive request/response traffic, disable Nagle's algorithm so replies aren't held back to coalesce. For many concurrent transfers, use one multiplexed async handle rather than a thread pool of blocking calls.
Why it is faster: —
Applies elsewhere: libuv and Node's event loop, Redis using `TCP_NODELAY`, game servers polling sockets during frame sleep, the curl multi interface in download managers.
Watch out: —
Seen in DuckStation: `SocketMultiplexer` uses `epoll_create1()` on Linux. When PINE clients are connected, the frame throttler sleeps in `PollEventsWithTimeout(sleep_ms)` so a query→response chain isn't limited to one message per frame (“Should significantly reduce PINE latency”). PINE sockets call `SetNagleBuffering(false)`. `HTTPDownloaderCurl` moved from a `cb::ThreadPool` of blocking requests to `curl_multi`. ad374ef5e 2024-07 “Sockets: Use epoll on Linux” · bc73dacea 2024-07 “System: Use socket multiplier for sleeping when connected” · 3172eac8d 2024-07 “PINE: Disable nagle buffering” · 5b00ffb6c 2023-11 “HTTPDownloaderCurl: Switch to multi/async API”

### duckstation-178 — Load optional components only when first used
Lens: laziness · Source: DuckStation
Principle: Linking heavy libraries for rarely used features costs load time and memory for every user. Resolve them dynamically on first use, and defer side effects like creating files until they are actually needed.
Why it is faster: —
Applies elsewhere: dynamic `import()` and code splitting on the web, `dlopen` plugins, Windows delay-load DLLs, Python lazy imports.
Watch out: —
Seen in DuckStation: Media Foundation imports (`MFStartup`, `MFCreateSinkWriterFromURL`, …) are now resolved via `wrap_##X` pointers on demand, where they used to come from `#pragma comment(lib, …)` (“Saves loading a few megabytes worth of DLLs”). discord-rpc is loaded through `DynamicLibrary`. Memory cards are no longer written to disk when missing, only on first save. 417c1e43d 2024-08 “MediaCapture: Load media foundation on-demand” · f6d3a7987 2024-07 “System: Load discord-rpc dynamically” · 1eb1b0314 2024-09 “MemoryCard: Defer creation of card until first save”

### duckstation-179 — Size and share buffers by actual need, not worst case
Lens: memory-layout · Source: DuckStation · Also: Flycast
Principle: Buffers sized for the largest possible configuration, and multiplied across N history slots, waste a lot of memory. Compute the bound from the active configuration. Don't duplicate large blobs you already own, and build caches only for the ranges you will actually use.
Why it is faster: —
Applies elsewhere: undo stacks sized per document, font atlases limited to the needed scripts, zero-copy parsing with `Bytes`/`string_view`.
Watch out: —
Seen in DuckStation: Rewind and runahead buffers use `GetMaxSaveStateSize()`: 5 MB with 2 MB RAM, 11 MB with 8 MB. They used to use a fixed 32 MB per slot. ImGui font atlases rasterize only ASCII for the debug and fixed fonts. A patch stops imgui copying font data on atlas build (“up to 20MB+”). e6a682f8d 2024-08 “System: Reduce rewind/runahead memory usage w/o 8MB enabled” · 4c51337ed 2024-09 “ImGuiManager: Don't rasterize full glyph range for debug font” · 44471a66f 2024-08 “dep/imgui: Don't copy the font data on building” • Also seen in Flycast (flycast-114): The GD-ROM read cache went from `u8 cache[2352 * 8192]` (about 19 MB, "up to 8192 sectors") to `2352 * 32`. Save states shrank by about 20 MB when the serializer stopped writing redundant data. 64249eb68 2020-04 “gdrom: shrink gdrom buffer to decent size” · 1b49bdd31 2020-01 “reduce save states size by ~20mb”

### duckstation-180 — Memoize expensive directory scans
Lens: io · Source: DuckStation
Principle: Repeatedly scanning a directory and opening every file to read headers is pure I/O cost. Cache the (filename → header) results process-wide behind a mutex, and look them up before touching the filesystem.
Why it is faster: —
Applies elsewhere: module-resolution caches in bundlers, `ccache`, IDE file indexes, DNS caches.
Watch out: —
Seen in DuckStation: Resolving a CHD's parent used to `FindFiles()` the directory and open every candidate. `s_chd_hash_cache` (a vector of `pair<filename, chd_header>` behind a `recursive_mutex`) now remembers which hash came from which file. 9112b6a85 2023-09 “CDImageCHD: Cache parent hashes”

### duckstation-181 — Pick a compression level for throughput, not maximum ratio
Lens: io · Source: DuckStation
Principle: Maximum compression levels can be 10–100× slower for a few percent smaller output. For data written interactively, the default level is usually the better tradeoff.
Why it is faster: —
Applies elsewhere: zstd -3 vs -19 for logs and backups, gzip level 6 for HTTP, LZ4 for real-time caches.
Watch out: —
Seen in DuckStation: Save-state compression changed from `Z_BEST_COMPRESSION` to `Z_DEFAULT_COMPRESSION`, and from zstd level 19 to 0 (default). 33f6cc246 2024-07 “System: Use default compression level”

### duckstation-182 — Tell the scheduler your deadline, and throttle before presenting
Lens: platform · Source: DuckStation
Principle: On OSes that schedule by load heuristics, a periodic real-time workload gets inconsistent timeslices. Declare the period and compute budget with real-time thread constraints. Separately, split frame submission from presentation, so work is submitted early, you sleep until the target time, and only then present.
Why it is faster: —
Applies elsewhere: audio threads using `THREAD_TIME_CONSTRAINT_POLICY` or MMCSS, game frame limiters that submit then wait on `Present`, VR compositors.
Watch out: —
Seen in DuckStation: On macOS, `s_cpu_thread_handle.SetTimeConstraints(..., s_frame_period, MAX_FRAME_TIME_NS = 7ms, ...)` made frame times a “flat line”. With the `explicit_present` feature, the frame loop calls `PresentDisplay(...)`, then `Throttle()`, then `g_gpu_device->SubmitPresent()`. e03deb492 2024-09 “System: Set realtime constraints on MacOS” · 734466180 2024-04 “GPUDevice: Split submission and presentation”

### duckstation-183 — Skip presenting frames that didn't change
Lens: other · Source: DuckStation
Principle: If the producer hasn't generated a new frame, presenting the old one again costs GPU and compositor time for no visible change. Compare frame identifiers and skip the duplicate, capping how many consecutive skips are allowed.
Why it is faster: —
Applies elsewhere: React skipping commits when nothing changed, compositor damage tracking, video players dropping duplicate frames.
Watch out: —
Seen in DuckStation: The “Skip Duplicate Frame Display” option compares `s_last_presented_internal_frame_number != s_internal_frame_number`, with `MAX_SKIPPED_FRAME_COUNT = 2`. 5b4f74122 2024-05 “System: Add "Skip Duplicate Frame Display" option”

### duckstation-184 — Turn off unused language runtime features
Lens: compiler-build · Source: DuckStation
Principle: RTTI and exception support add type-info tables, unwind data and code-size overhead, and can inhibit some optimizations. Code that doesn't use them should build with `-fno-rtti`/`-fno-exceptions`. That gives smaller binaries and better i-cache density.
Why it is faster: —
Applies elsewhere: LLVM and Chromium build without RTTI and exceptions, embedded C++ (`-fno-exceptions`), Rust's `panic = "abort"`.
Watch out: —
Seen in DuckStation: The build drops `/EHsc` and `/GR`, defines `_HAS_EXCEPTIONS=0` and keeps `-fno-exceptions`. Later, `RuntimeTypeInfo` was set to false for Windows builds (“It's not used”), with RTTI kept only for Qt debug builds. cee4f9309 2023-09 “Build: Use precompiled headers” · 128d5254c 2024-09 “MSBuild: Disable RTTI on Windows builds”

### duckstation-185 — Move a whole subsystem behind a single-producer/single-consumer ring buffer
Lens: concurrency · Source: DuckStation
Principle: When one thread both produces work and executes it, give the execution to a second thread and connect the two with a lock-free SPSC ring buffer of variable-sized commands. The producer only bumps a write index and the consumer only bumps a read index, so no locks are needed and the two halves run on separate cores in parallel.
Why it is faster: —
Applies elsewhere: a game engine's render thread fed by a command list; an audio callback fed from a lock-free queue; a logging thread that formats and writes records so request handlers don't block on disk; LMAX Disruptor-style pipelines.
Watch out: —
Seen in DuckStation: All rendering (hardware and software backends, presentation, ImGui) used to run on the thread that emulates the CPU. Now `GPUThread` (today `video_thread.cpp`) owns a 16 MB cache-line-aligned `command_fifo_data` with atomic `command_fifo_write_ptr`/`command_fifo_read_ptr`. The emulation thread calls `AllocateCommand()`, fills the command in place, and publishes it with a release store. Commands that would cross the end of the buffer are preceded by a dummy wrap command. Vertex culling moved to the GPU thread too, so the emulation thread forwards primitives with only a cheap oversize check. 2d659fc3e 2024-12 “GPU: Move backend work off CPU thread” · c99625e4c 2024-12 “GPU: Move vertex culling to GPU thread”

### duckstation-186 — Spin briefly, then sleep, and wake the consumer only once enough work has built up
Lens: concurrency · Source: DuckStation
Principle: Signalling a kernel semaphore for every item costs a syscall and a context switch. Let work accumulate, and wake a sleeping consumer only once a size threshold is crossed or at a sync point. A thread that has to wait should spin with a pause instruction for a short bounded time before blocking, because the result often arrives within microseconds.
Why it is faster: —
Applies elsewhere: adaptive mutexes (glibc `PTHREAD_MUTEX_ADAPTIVE_NP`, WebKit's WTF::Lock), Nagle-style batching in network stacks, io_uring's SQ polling, Go's scheduler spinning Ms before parking.
Watch out: —
Seen in DuckStation: `PushCommand()` only calls `WakeThread()` once `GetPendingCommandSize() >= THRESHOLD_TO_WAKE_GPU` (64 KB). The wake protocol is a single `fetch_add(2)` on `thread_wake_count`, and the semaphore is posted only when the previous value was negative (the consumer was asleep). `SyncThread(spin)` busy-waits with `PauseCPU()` for `THREAD_SPIN_TIME_US` (50 µs on x86, 200 µs on ARM because of power management) before falling back to a blocking wait. 2d659fc3e 2024-12 “GPU: Move backend work off CPU thread”

### duckstation-187 — Carry payloads inline in the queue instead of heap-allocated closures
Lens: allocation · Source: DuckStation
Principle: Sending a `std::function` or another boxed closure to another thread usually costs a heap allocation plus an indirect call. If the queue supports variable-sized records, copy or move the data straight into the queue slot and dispatch on a type tag. You then have one memcpy-sized write and no allocator traffic.
Why it is faster: —
Applies elsewhere: tagged-union message enums in Rust channels; Vulkan/D3D command buffers; actor mailboxes that store messages in arena slots rather than as individual objects.
Watch out: —
Seen in DuckStation: Updating GPU settings used to queue a lambda that captured the old settings, which allocated a `std::function`. It now allocates a `GPUThreadUpdateSettingsCommand` in the FIFO with a `settings` member, which the GPU thread moves out and destroys in place. `AllocateCommand<T>(size, type)` was generalised to support a trailing flexible array. This lets buffers such as the input-overlay state travel inside the command itself (`BeginASyncBufferCall()`). 89504b018 2025-01 “GPUThread: Push settings through FIFO” · 3476140ba 2025-01 “GPUThread: Add BeginASyncBufferCall()” · f6c7681ef 2025-01 “ImGuiOverlays: Double buffer input OSD…”

### duckstation-188 — Group state by the thread that owns it and pad between the groups
Lens: concurrency · Source: DuckStation
Principle: When two threads write to variables on the same cache line, every write bounces that line between cores (false sharing). Group fields by owning thread and start each group on a new cache line. Don't pad more than that, because padding wastes cache.
Why it is faster: —
Applies elsewhere: Java's `@Contended`; `std::hardware_destructive_interference_size`; per-CPU counters in the Linux kernel; head and tail indices of a ring buffer kept on separate lines.
Watch out: —
Seen in DuckStation: The GPU thread state puts `ALIGN_TO_CACHE_LINE` on `command_fifo_write_ptr`, which the producer writes, and keeps `command_fifo_read_ptr`, which the consumer writes, away from it. `ImGuiManager`'s state was reordered so fields written by the GPU thread start on their own line (`ALIGN_TO_CACHE_LINE Timer::Value last_render_time`). `GPUBackend` and `GPUPresenter` classes are cache-line aligned. Padding that turned out unnecessary (the presenter pointer) was removed later. 17e8930e9 2025-01 “ImGuiManager: Pack state based on thread access” · da1357935 2025-01 “GPU: Align presenter/backends to cache line” · 03db7ea8e 2025-05 “GPUThread: Remove unnecessary padding”

### duckstation-189 — Send slow I/O and compression to a shared worker pool
Lens: concurrency · Source: DuckStation
Principle: Compressing, encoding and writing files on the latency-critical thread causes visible hitches. Spawning a thread per job adds creation cost and makes shutdown messy. Capture the data, move it into a closure, and queue it on a small persistent task pool.
Why it is faster: —
Applies elsewhere: Node's libuv threadpool for fs and zlib; Android's rule against disk I/O on the UI thread; Unity's `Task.Run` for save games; web servers that hand image resizing to a job queue.
Watch out: —
Seen in DuckStation: A generic `TaskQueue` (a mutex, a condition variable and N `std::thread`s) replaced ad-hoc `s_screenshot_threads`. Save-state compression and writing, screenshots, GPU dumps, texture dumps (including the alpha fix-up loop) and fullscreen-UI cover loading all go through `System::QueueAsyncTask(...)`. Previously these ran inline or on one dedicated loader thread. bfadd608f 2024-10 “System: Add 'Task Threads'” · 5d7cb6c5d 2024-12 “System: Move state compression/writing to worker thread” · 547601559 2025-01 “Common: Add TaskQueue class” · 092e819a0 2025-01 “ImGuiFullscreen: Move texture loading to async tasks” · 6437a5db6 2025-03 “GPU/TextureCache: Dump textures asynchronously”

### duckstation-190 — Write SIMD kernels against a width-agnostic vector alias
Lens: simd · Source: DuckStation
Principle: Write the kernel once in terms of an alias such as `VecN` plus constants like `LANES`, and choose 128-bit or 256-bit lanes at compile time. The same source then processes 4 or 8 elements per iteration, so moving to wider hardware means recompiling rather than rewriting.
Why it is faster: —
Applies elsewhere: Highway's `ScalableTag`; `std::experimental::simd`; xsimd; Rust's `portable_simd` with const-generic lane counts; image codecs that build SSE and AVX2 variants from one template.
Watch out: —
Seen in DuckStation: A `GSVector8i` wrapper was added for AVX2. The software rasterizer uses `using GSVectorNi = GSVector8i` or `GSVector4i`, with `PIXELS_PER_VEC`, `SPAN_OFFSET_VEC = cxpr(0,1,…,7)`, and so on. Gathers from VRAM and the CLUT were rewritten to index `g_vram[y*1024+x]` directly instead of computing byte offsets and doing `memcpy`s. The SIMD path then became the default renderer. a7747c5be 2024-09 “Common: Add 256-bit integer vector wrapper” · 40670726b 2024-09 “GPU/SW: Fix SSE4 path and add 256-bit AVX2 path” · 54e5d9316 2024-09 “GPU/SW: Enable SIMD path and make default”

### duckstation-191 — Compile the hot kernel once per ISA and dispatch at startup
Lens: platform · Source: DuckStation
Principle: Baseline builds can't assume AVX2. Compile the hot module several times into separate namespaces with different target flags, detect the CPU once, and install function pointers to the best variant. The cost is one indirect call per batch instead of per element.
Why it is faster: —
Applies elsewhere: glibc's IFUNC-resolved `memcpy`; xxHash's `xxh_x86dispatch`; OpenSSL's `OPENSSL_ia32cap`; simdjson's runtime implementation selection.
Watch out: —
Seen in DuckStation: `GPU_SW_Rasterizer::SelectImplementation()` runs once and, if `cpuinfo_has_x86_avx2()` is true (overridable with `SW_USE_ISA`), points `DrawTriangleFunctions`, `FillVRAM`, `WriteVRAM` and `CopyVRAM` at the `AVX2::` namespace. Otherwise it uses the baseline SSE or NEON namespace. The commit that enabled the SIMD path left the AVX2 variant switched off over ODR concerns. The current file enables it. 54e5d9316 2024-09 “GPU/SW: Enable SIMD path and make default”

### duckstation-192 — Emulate missing SIMD instructions so one code path serves older CPUs
Lens: simd · Source: DuckStation
Principle: Don't keep a separate scalar path for CPUs that lack a newer instruction set. Implement the missing intrinsics (min/max on other widths, blends, unsigned compares) as short sequences of baseline instructions inside the vector wrapper. Callers keep one vectorized algorithm, and only the few slow emulated ops cost extra.
Why it is faster: —
Applies elsewhere: SIMDe (SIMD Everywhere) and sse2neon; compilers lowering `_mm_blendv_epi8` to and/andnot/or on SSE2; WebAssembly SIMD polyfills.
Watch out: —
Seen in DuckStation: `gsvector_sse.h` gained `#ifndef CPU_ARCH_SSE41` fallbacks such as `sse2_min_s8` (a compare plus a blend through and/andnot), credited to “LLVM for these”. Features with no cheap emulation are flagged instead: `GSVECTOR_HAS_FAST_INT_SHUFFLE8` is set only with SSE4.1, so callers can avoid the slow `shuffle8()`. The no-SIMD `runion()` was rewritten as plain `std::min`/`std::max`, because the emulated vector version produced poor code. c439de636 2024-09 “Common: SSE2 backsupport for vector classes”

### duckstation-193 — Vectorize pixel-format conversion loops
Lens: simd · Source: DuckStation
Principle: Bit-unpacking loops (5:5:5 to 8:8:8, 565 to RGBA) apply the same shifts and masks to every element, so they map directly onto SIMD. Process a full vector of pixels per iteration with lane-wise shifts and ORs, and finish the remainder with scalar code.
Why it is faster: —
Applies elsewhere: libyuv's colourspace converters; PNG and WebP decoders; Skia's raster pipeline; audio sample-format conversion (s16 to f32) in FFmpeg's swresample.
Watch out: —
Seen in DuckStation: `DecodeTexture4/8/16` in the texture cache were templated on the destination format and vectorized, about 100% faster for RGBA8 and 50% for RGB5A1. `Image::ConvertToRGBA8` got a SIMD path for RGB565, 8 pixels at a time with `srl16`/`sll16`. A `DONT_VECTORIZE_THIS_LOOP` macro stops the compiler from auto-vectorizing the scalar tail as well. c6e2235ee 2024-11 “GPU/HW: Vectorize texture conversion routines” · b68d7c8ad 2025-02 “Image: Vectorize RGBA8 conversion and add A1BGR5”

### duckstation-194 — Replace per-element conditionals with a computed bit-select mask
Lens: simd · Source: DuckStation
Principle: "Keep the old value if a flag bit is set, otherwise write the new one" is a per-element branch. Turn the flag into an all-ones or all-zeros mask with an arithmetic shift and combine with `(old & m) | (new & ~m)`. Then many elements are handled per instruction with no branches.
Why it is faster: —
Applies elsewhere: branchless `select` in shader code; masked stores in database column scans; alpha-key compositing; constant-time crypto comparisons.
Watch out: —
Seen in DuckStation: Software VRAM uploads honour the PS1 "check mask" bit. The scalar loop tested each pixel. The vector path does `mask = (dst & mask_and_vec).sra16<15>(); dst = (dst & mask) | src.andnot(mask) | mask_or_vec;` across 8 or 16 pixels, and when masking is off it is just a load, OR and store. d5b9b54a6 2024-11 “GPU/SW: Vectorize VRAM writes/copies”

### duckstation-195 — Skip the read in read-modify-write when nothing will be kept
Lens: branching-fast-paths · Source: DuckStation
Principle: If the new value fully overwrites the destination (no blending, no masking, no partial lanes), the load of the old value is wasted bandwidth. Test for that case once per span and store directly.
Why it is faster: —
Applies elsewhere: GPUs' `LOAD_OP_DONT_CARE`; write-combining full cache lines with non-temporal stores; `memset` instead of read-modify-write for opaque fills; databases skipping page reads when a write covers the whole page.
Watch out: —
Seen in DuckStation: The vector `ShadePixel` always ran `LoadVector(start_x, y)` for the background. It now checks `if (!mask_bit_test && !transparency_enable && preserve_mask.allfalse())` and in that case just ORs in the mask bit and stores. This gave about 12% on a Cortex-A35. 18c90718c 2025-01 “GPU/SW: Skip pixel load if none preserved”

### duckstation-196 — Pick the cheapest instruction for mask tests and blends on each ISA
Lens: simd · Source: DuckStation
Principle: The same operation can cost 1 or 5 instructions depending on how it is written. When lanes are known to be all-ones or all-zeros, "all true" is a single 64-bit compare and needs no horizontal min over bytes. A constant-mask blend is a compile-time shuffle and needs no table load plus bit-select.
Why it is faster: —
Applies elsewhere: `_mm_movemask_epi8 == 0xFFFF` versus per-lane extraction; `__builtin_shufflevector` in portable SIMD libraries; checking a 16-byte mask with one `ptest`.
Watch out: —
Seen in DuckStation: On NEON, 64-bit `alltrue()`/`allfalse()` became `vget_lane_u64(...) == ~0ULL` / `== 0`. The 128-bit forms use `vminvq_u32`/`vmaxvq_u32` instead of byte-wise reductions masked with `0x80`. `blend16<mask>`/`blend32<mask>` now use `__builtin_shufflevector(a, b, …)` with indices computed from the template mask, which drops the `vld1q_u16(_mask)` constant load and `vbslq`. 1320d9b45 2025-01 “Common: Use __builtin_shufflevector() for NEON blend()” · 57f3fee28 2025-01 “Common: Further optimize alltrue()/allfalse() on ARM”

### duckstation-197 — Stop specializing on a flag when code size costs more than the branch
Lens: compiler-build · Source: DuckStation
Principle: Each bool template parameter doubles the number of instantiations. If a flag is cheap to test at runtime, or can be folded into arithmetic, removing it halves the code. That reduces instruction-cache misses, which matters most on small-cache CPUs.
Why it is faster: —
Applies elsewhere: collapsing shader permutations into uniform branches; replacing `template<bool Debug>` with a runtime check in a library; reducing monomorphization bloat in Rust generics.
Watch out: —
Seen in DuckStation: The software rasterizer's draw tables were `[2][2][2][2][2]` with `dithering_enable` as a template parameter. Dithering is now read from `cmd->draw_mode.dither_enable` and the table is `[2][2][2][2]`. That roughly halves the rasterizer's code size, which the commit says “may be beneficial on icache-starved processors”. bb3d27f15 2024-09 “GPU/SW: Drop dithering function specialization”

### duckstation-198 — Make unused template members take zero bytes
Lens: memory-layout · Source: DuckStation · Also: Seastar
Principle: When a field exists only in some instantiations, an empty placeholder type still takes at least one byte and its alignment padding, which bloats every object. `[[no_unique_address]]` lets empty members overlap, so the disabled variant's struct really shrinks and stays in fewer registers or cache lines.
Why it is faster: —
Applies elsewhere: stateless allocators and comparators in `std::` containers (EBO); optional instrumentation fields in policy-based designs; Rust's zero-sized types.
Watch out: —
Seen in DuckStation: `PixelVectors<texture_enable>` holds six vector-sized texture-window fields typed `std::conditional_t<texture_enable, GSVectorNi, UnusedField>`. They gained `NO_UNIQUE_ADDRESS` (with `[[msvc::no_unique_address]]` on MSVC), so untextured draws no longer carry padding for them. c4a745668 2025-01 “GPU/SW: Use no_unique_address for potentially-unused varyings” • Also seen in Seastar (seastar-041): 0ee33a9f2f 2019-04-12 `future_state<>` uses the empty-base optimization to shrink, which also let 455817b5 remove a specialized copy. • 0ecf8222a0 2020-06-17 The continuation stores a stateless `Wrapper` as `[[no_unique_address]]` beside the user's `Func`. It also moves func only once, for -0.7% text. • 84e9889454 2024-12-23 `uninitialized_wrapper` switched from inheriting the payload to composition plus `[[no_unique_address]]`, verified to keep `sizeof(future<>)` unchanged without leaking the payload's members into scope.

### duckstation-199 — Guarantee alignment, then use aligned accesses
Lens: memory-layout · Source: DuckStation
Principle: If an allocator rounds every record to a known alignment, the consumer can use aligned vector loads and stores, which are cheaper and on some ISAs the only fast form. The record size must be rounded to that alignment too, or one odd record misaligns everything after it.
Why it is faster: —
Applies elsewhere: 16-byte aligned arena allocators for SIMD math; Vulkan's `minUniformBufferOffsetAlignment`; network packet buffers aligned for DMA; `std::assume_aligned`.
Watch out: —
Seen in DuckStation: Vertex commands from the GPU FIFO are now 8-byte aligned, so `GSVector2::load<false>` became `load<true>` throughout `DrawLine`, `DrawPolygon` and `BeginPolygonDraw`. A follow-up raised command alignment to 16 because some types were 16-byte aligned on Android x86-64. Using `alignas(16)` on the struct inflated every command, so the fix was changed to round the allocation size (`COMMAND_QUEUE_ALLOCATION_ALIGNMENT = 16`) instead. 799f5bdf9 2024-12 “GPU: Assume vertex commands are 8-byte aligned” · 1f10cac42 2025-04 “GPUThread: Align commands to 16 bytes” · 769e6f2ba 2025-04 “GPUThread: Fix command size blow-up”

### duckstation-200 — Store data in the format the consumer can use as-is
Lens: memory-layout · Source: DuckStation
Principle: If the destination can accept the source's bit layout directly, choose that format. Then no per-pixel conversion runs, and a 16-bit format instead of 32-bit halves the memory, upload bandwidth and cache footprint.
Why it is faster: —
Applies elsewhere: uploading BGRA to GPUs that prefer it; keeping audio as s16 when the device takes s16; using Arrow's in-memory format as the wire format; storing timestamps as epoch ints instead of strings.
Watch out: —
Seen in DuckStation: The texture-cache hash cache was allowed to use `RGB5A1` instead of RGBA8, which the commit puts at 50% less bandwidth and storage. Later the device format `A1BGR5`, the PS1's native 1:5:5:5 bit order, was added and preferred for both the hash cache and the software renderer's display texture. That removed a family of per-format `VRAM16ToOutput` converters from `gpu_sw.cpp`, so VRAM rows are copied out with no swizzle. 53008eb34 2024-11 “GPU/HW: Allow use of RGB5A1 for texture cache” · 5725a0360 2024-12 “GPU: Use A1BGR5 format for SW/HashCache if available”

### duckstation-201 — Cache derived resources by a hash of their source content
Lens: algorithms · Source: DuckStation
Principle: If an expensive derived object (decoded texture, compiled artifact) depends only on some input bytes, key it by a fast hash of those bytes. Track writes at page granularity so you rehash only pages that may have changed. When the same content comes back, even at a different address, the lookup hits and the rebuild is skipped.
Why it is faster: —
Applies elsewhere: ccache and Bazel's content-addressed action cache; browser HTTP caches keyed by ETag; webpack's content-hashed chunks; Git's object store.
Watch out: —
Seen in DuckStation: The hardware texture cache decodes PS1 VRAM pages (4/8/16-bit with CLUT) into host textures. Each 64×256 page has a `PageEntry` with an intrusive list of `Source`s and up to four dirty `draw_rects`. `LookupSource()` walks the page list and moves hits to the front. On a miss, `HashPage()` runs XXH3 over the page rows and the palette, and the result is looked up in an `unordered_map<HashCacheKey{texture_hash, palette_hash, mode}, HashCacheEntry>`. A VRAM write or copy invalidates only the sources on the pages it touches (`LoopRectPages`). The same structure also drives texture replacement. e06f1f100 2023-12 “GPU: Add hardware texture cache”

### duckstation-202 — Bound a cache by count and bytes, and evict by age with one sort
Lens: allocation · Source: DuckStation
Principle: An unbounded cache eventually exhausts memory. Evict entries that have gone unused for N epochs on every pass. When the cache is still over budget, collect the candidates, sort them once by last use, and drop from the oldest end until it fits. That is O(n log n) once, not one O(n) scan per eviction.
Why it is faster: —
Applies elsewhere: Redis `maxmemory` with approximate LRU; browser image caches limited by bytes; CDN edge caches; database buffer pools with clock-sweep eviction.
Watch out: —
Seen in DuckStation: `GPUTextureCache::Compact()` runs once per frame. It removes entries with `ref_count == 0 && last_used_frame < frame - 600`. If `hash_cache.size() > max_hash_cache_entries` or memory exceeds `max_hash_cache_vram_usage_mb`, it pushes `(iterator, last_used_frame)` pairs into a list, `std::sort`s them, and removes from the front. The source comments that this is “faster than re-iterating the map”. The limits moved from hard-coded values (500 entries, 1 GB) into config. eb390a9b5 2024-11 “GPU/TextureCache: Specify max hash cache size/memory in config”

### duckstation-203 — Evict cache entries as soon as they are known to be dead
Lens: allocation · Source: DuckStation
Principle: Age-based eviction keeps garbage around until it times out. If an event proves an entry can never hit again (its source region was overwritten wholesale), drop it the moment its refcount reaches zero. Memory is freed sooner and the cache doesn't fill with entries that will never be reused.
Why it is faster: —
Applies elsewhere: invalidating CDN objects on deploy instead of waiting for TTL; dropping query-plan caches when a table is altered; freeing GPU buffers on explicit discard.
Watch out: —
Seen in DuckStation: `CopyVRAM` now calls `AddWrittenRectangle(dst_bounds, …, remove_from_hash_cache=true)`. When `DestroySource()` drops the last reference to a `HashCacheEntry` in that case, it calls `RemoveFromHashCache()` right away instead of leaving it for the 600-frame sweep. 9838dd9df 2024-11 “GPU/TextureCache: Purge copy-target sources from hash cache”

### duckstation-204 — Keep uploaded results resident on the device, not just on the host
Lens: gpu-rendering · Source: DuckStation
Principle: Caching decoded data in RAM still costs a host-to-device copy every time it is used. Cache the device-side object (texture or buffer) under a budget, so repeated uses become a bind with no transfer.
Why it is faster: —
Applies elsewhere: WebGL apps keeping sprite atlases as textures; ML frameworks keeping weights on the GPU between batches; database GPU accelerators pinning hot columns in VRAM.
Watch out: —
Seen in DuckStation: VRAM-write replacements were stored as CPU `TextureReplacementImage`s and copied into a reusable `m_vram_replacement_texture` through `Update()` on every use. A `GPUReplacementImageCache` of `(unique_ptr<GPUTexture>, last_used)` now hands `BlitVRAMReplacementTexture()` a ready texture. `CompactTextureReplacementGPUImages()` enforces a maximum. This “should sort out the upload spam”. 57595c47a 2024-11 “GPU/TextureCache: Move replacement cache onto the GPU”

### duckstation-205 — Prune shader permutations that can never be used
Lens: gpu-rendering · Source: DuckStation
Principle: Pipeline caches are built from the cross product of feature switches, and many combinations are impossible under the current config. Skip combinations that are contradictory (dithering with true colour) or belong to disabled features. Cold-start compile time and cache size shrink in proportion.
Why it is faster: —
Applies elsewhere: Unity's shader variant stripping; Unreal's permutation reduction; trimming unused locales or polyfills from a JS bundle.
Watch out: —
Seen in DuckStation: `GPU_HW::CompilePipelines()` now has `if (dithering && true_color) continue;` and `if (interlacing && m_force_progressive_scan) continue;`, giving about 3x faster shader compilation on a cold cache. Page-texture modes and their vertex shaders are compiled only when the texture cache is enabled (`needs_page_texture`), which removes about a quarter of the default pipelines. 0538b95d9 2024-09 “GPU/HW: Reduce number of shaders/pipelines in default config” · e91ae5f1b 2024-12 “GPU/HW: Only compile page texture shaders if TC is enabled”

### duckstation-206 — Turn baked-in constants into uniforms to collapse variants
Lens: gpu-rendering · Source: DuckStation
Principle: Each value baked into generated code as a literal multiplies the number of distinct programs. Moving rarely-changing parameters into a uniform buffer costs one load per use, but every setting then shares a single compiled program and cached binary.
Why it is faster: —
Applies elsewhere: parameterized SQL instead of literal-embedded queries (one cached plan); a JIT avoiding constant specialization for values that vary often; CSS custom properties instead of generated classes.
Watch out: —
Seen in DuckStation: Batch shaders embedded `CONSTANT uint RESOLUTION_SCALE = N`, so every upscale factor needed its own set. They now read `u_resolution_scale`, `u_rcp_resolution_scale` and `u_resolution_scale_minus_one` from the batch UBO, and dithering divides by multiplying with the reciprocal. The compressed shader cache shrank almost 6x. 986e207cf 2024-09 “GPU/HW: Make batch shaders independent on resolution”

### duckstation-207 — Enable expensive optional features only in the programs that use them
Lens: gpu-rendering · Source: DuckStation
Principle: Some driver paths get slower when a program merely declares a feature (framebuffer fetch, dual-source blending), even if it never uses it. Make the declaration conditional on actual use, so other programs stay on the fast path.
Why it is faster: —
Applies elsewhere: requesting `preserveDrawingBuffer` in WebGL only when needed; leaving the GL debug context off in release; opening files without `O_SYNC` unless durability is required.
Watch out: —
Seen in DuckStation: `ShaderGen::WriteHeader()` used to emit `GL_EXT_shader_framebuffer_fetch` and `GL_*_blend_func_extended` for every shader. It now takes `enable_framebuffer_fetch`/`enable_dual_source_blend`, and batch shaders pass these only when they blend that way (“Thanks Adreno...”). Texture filters such as ScaleNx and MMPX don't produce fractional alpha, so they no longer request dual-source output (`is_blended_texture_filtering`). d24e8fda8 2024-10 “GPU/HW: Only enable fbfetch/DSB when needed” · a375d558f 2025-05 “GPU/HW: ScaleNx/MMPX do not require blending” · 94b88fad2 2025-06 “GPU/HW: Avoid texture filter dual-source output when unnecessary”

### duckstation-208 — Choose the copy method that keeps the current render pass alive
Lens: gpu-rendering · Source: DuckStation
Principle: On tile-based GPUs, ending a render pass flushes tile memory to DRAM, which costs a lot. For each copy, pick the method (shader draw or transfer command) that avoids a pass break. Use a draw when the source isn't being rendered to. Avoid a draw when source and destination overlap, because that would force a break immediately.
Why it is faster: —
Applies elsewhere: Vulkan subpass design on Mali and Adreno; Metal's advice to batch blits outside render encoders; canvas libraries batching `drawImage` to avoid flushes.
Watch out: —
Seen in DuckStation: `GPU_HW::CopyVRAM` computes `intersect_with_draw`/`intersect_with_write` up front. If the source is clean it uses the shader copy instead of `CopyTextureRegion`, which would end the pass. Later `overlaps_with_self` was added to the condition, because a self-overlapping shader copy dirties its own source and ends the pass anyway. This was costly on Mali in Dino Crisis 2's menu. 99e65282f 2024-10 “GPU/HW: Prefer shader over image copy if not dirty” · 958fd0049 2025-02 “GPU/HW: Don't use shader for self-overlapping copies”

### duckstation-209 — Draw and clear only the pixels that need it
Lens: gpu-rendering · Source: DuckStation
Principle: Fill rate and memory bandwidth scale with the pixels touched. Size quads to the destination rectangle instead of covering the whole viewport. Clear only the active region, inside the current pass. Fill borders as separate strips instead of drawing over the whole target. Skip fully transparent backgrounds.
Why it is faster: —
Applies elsewhere: scissored partial clears; browsers repainting only damaged rectangles; `glClear` with a scissor; skipping `fillRect` when alpha is 0.
Watch out: —
Seen in DuckStation: Fullscreen-triangle passes were replaced by `DrawScreenQuad(bounds, rt_size, uv)`, which emits a 4-vertex triangle strip in clip space for the exact rectangle. A new in-pass clear-depth pipeline wipes only the current drawing area instead of clearing the whole depth buffer, which saves bandwidth in games that clear often. `DrawOverlayBorders()` fills the four borders individually instead of blitting over the whole texture. Fullscreen UI windows set `ImGuiWindowFlags_NoBackground` when their alpha is 0. b9186139d 2025-01 “GPU/HW: Use sized tristrips instead of fullscreen quads” · 0129679bb 2025-01 “GPU: Reduce overdraw when using overlays” · 58dc7562a 2025-03 “GPU/HW: Add in-pass clear depth pipeline” · 11bc1935c 2025-06 “FullscreenUI: Skip drawing background if transparent”

### duckstation-210 — Compare batch-break keys against current state, not derived copies
Lens: gpu-rendering · Source: DuckStation
Principle: Draw batching ends a batch whenever the state key changes. If the comparison reads a value that is updated lazily (a uniform staged for upload, say), batches split spuriously and draw calls spike. Compare against the authoritative batch state.
Why it is faster: —
Applies elsewhere: sprite batchers keying on texture id; React reconciliation keyed on stable ids; deduplicating writes by comparing to the committed value rather than a pending one.
Watch out: —
Seen in DuckStation: `GPU_HW::PrepareDraw` compared `m_batch_ubo_data.u_set_mask_while_drawing` with the command's flag. That UBO field wasn't always in sync, so batches broke repeatedly. It now compares `m_batch.set_mask_while_drawing`. A related reshuffle moved `texture_cache_key` and the texture window bits out of the batch struct into dedicated members. 45b500bc2 2025-01 “GPU/HW: Fix draw call count spiking in some scenarios” · 670dc461c 2025-01 “GPU/HW: Slight re-shuffling of field offsets”

### duckstation-211 — Choose the buffer-upload strategy the driver is fast at
Lens: gpu-rendering · Source: DuckStation
Principle: Streaming vertex or uniform data has several API strategies (persistent mapping, map-unsynchronized with orphaning, sub-data), and their speed varies a lot between drivers. Implement several and pick by capability or driver, instead of forcing one onto drivers that handle it badly.
Why it is faster: —
Applies elsewhere: ANGLE choosing buffer paths per backend; WebGPU implementations switching staging strategies; database drivers choosing COPY or batched INSERT per server.
Watch out: —
Seen in DuckStation: The OpenGL stream buffer, which used `ARB_buffer_storage` persistent mapping or `glBufferSubData`, gained `MapAndOrphanStreamBuffer` and `MapAndSync` variants built on `glMapBufferRange`. MapAndSync is about 25% faster on old Mali drivers that lack buffer storage. 45afc8f7b 2025-01 “OpenGLDevice: Add map-based stream buffer paths”

### duckstation-212 — Reuse pooled GPU objects whenever the hazard doesn't apply
Lens: allocation · Source: DuckStation
Principle: Creating a GPU texture costs a driver call and memory. Pools recycle them, but a conservative rule ("never reuse an object from this frame") throws away hits. Narrow the rule to the real hazard: a same-frame reuse only matters if data is uploaded, which would break the pass.
Why it is faster: —
Applies elsewhere: connection pools that skip health checks for recently returned connections; object pools in game engines; reusing `ArrayBuffer`s in WebAudio worklets.
Watch out: —
Seen in DuckStation: `GPUDevice::FetchTexture()` avoided this-frame textures whenever `prefer_unused_textures` was set. It now does so only when `data` is non-null, meaning an upload would happen. Memory save states also flush the GPU and purge the pool before allocating, to improve reuse. f51dda3e6 2025-01 “GPUDevice: Allow this-frame pooled textures when not uploading data” · a08acdb93 2025-01 “System: Improve texture recycling when changing rewind/runahead settings”

### duckstation-213 — Emit conditional-select instead of a branch in generated code
Lens: jit-interpreters · Source: DuckStation
Principle: A data-dependent branch in hot generated code mispredicts whenever the outcome is irregular. Conditional instructions (`csel`, `cmov`, ARM predicated `add ne`) compute both outcomes and pick one, so there is nothing to mispredict. Hoist constants into registers so the select has operands ready.
Why it is faster: —
Applies elsewhere: compilers' if-conversion; branchless binary search; `Math.max`-style code in JS hot loops; branch-free sort networks.
Watch out: —
Seen in DuckStation: Each recompiled block checks the emulated instruction-cache tag of every cache line it covers and adds fill cycles on a miss. The ARM32, ARM64 and x64 backends emitted `cmp; b.eq hit; add ticks, fill; hit:`. They now load `fill_ticks` once into a register and emit `cmp; add ne ticks, fill_reg` (ARM32), `csel` (ARM64) or `cmov` (x64), for about a 4% speedup. 9d52e27e1 2024-12 “CPU/Recompiler: Use condition select for ICache updates”

### duckstation-214 — Keep tiny slow paths inline, and reload values instead of spilling them
Lens: jit-interpreters · Source: DuckStation
Principle: Moving cold code to a far section helps when it is large. For three instructions, the jump out and back costs more than it saves. Use a short forward branch over inline code. When a value is needed again after a call, reloading it from memory beats a push and pop that can misalign the stack.
Why it is faster: —
Applies elsewhere: compilers' `__builtin_expect` heuristics that don't outline tiny blocks; V8 deferred-code thresholds; hand-written assembly re-reading a field instead of saving it.
Watch out: —
Seen in DuckStation: In `Compile_mtc0`, the cache-isolation check used `SwitchToFarCodeIfBitSet(changed_bits, 16)`. It now uses `tbz changed_bits, #16, caches_unchanged` (or `BEQ`, or `jz`) around the few instructions, and reloads `RMEMBASE` only if fastmem is active. The surrounding `str`/`ldr` stack spill of the argument became `ldr RARG1, PTR(ptr)`, and the 224-byte stack reserve in the dispatcher went away. 10e2079ee 2024-12 “CPU/Recompiler: Don't use far code for mtc0 cache check” · fe1fa765f 2024-12 “CPU/Recompiler: Don't back up value to stack in mtc0”

### duckstation-215 — Use plain jumps, not calls, for tail transfers
Lens: jit-interpreters · Source: DuckStation
Principle: Branch-and-link pushes a return address onto the CPU's return-stack predictor. If the callee never returns there, that predictor entry is wrong and later returns can mispredict. An indirect jump that doesn't return should be a plain branch.
Why it is faster: —
Applies elsewhere: tail-call optimisation (`musttail` in Clang); interpreter dispatch with `jmp *table[op]`; coroutine switches using `jmp`.
Watch out: —
Seen in DuckStation: The dispatcher in `EmitASMFunctions` jumped into compiled blocks with `blx RARG1` (ARM32) and `blr RXARG1` (ARM64). The commit notes this is “not a subroutine call”, so these became `bx`/`br`. 2da692b34 2024-12 “CPU/Recompiler: Swap bl{x,r} for b{x,r} on ARM”

### duckstation-216 — Reset only the used prefix of a large buffer
Lens: jit-interpreters · Source: DuckStation
Principle: If a buffer is filled from the front, record a high-water mark and clear or flush only up to it on reset. Clearing a 48 MB code region that is mostly empty wastes bandwidth and instruction-cache flush time.
Why it is faster: —
Applies elsewhere: arena allocators that reset a bump pointer; zeroing only the used part of a reusable network buffer; `std::vector::clear()` keeping capacity.
Watch out: —
Seen in DuckStation: `CPU::CodeCache::ResetCodeBuffer()` did `memset(s_code_ptr, 0, RECOMPILER_CODE_CACHE_SIZE)` plus a full `FlushInstructionCache`. It now zeroes and flushes only `s_code_used` and `s_far_code_used`, and skips both when they are 0. The commit says this “saves writing 48MB every reset”. 3687697d0 2025-01 “CPU/CodeCache: Only reset used portion of buffer”

### duckstation-217 — Base JIT space reservations on real code expansion
Lens: jit-interpreters · Source: DuckStation
Principle: A JIT flushes its whole code cache when free space drops below the worst case for the next block. If that estimate is too pessimistic, flushes (and the recompilation after them) happen far too often. Update the per-instruction bound as codegen improves, and add a small fixed minimum.
Why it is faster: —
Applies elsewhere: sizing `realloc` growth from measured ratios; HotSpot's code-cache sweeper thresholds; preallocating output buffers from compression ratio statistics.
Watch out: —
Seen in DuckStation: `CompileOrRevalidateBlock` now checks near-code space against `block_size * MAX_NEAR_HOST_BYTES_PER_INSTRUCTION` plus `MIN_CODE_RESERVE_FOR_BLOCK`, with lower per-instruction maxima. Far code needs only the fixed minimum, because the newer recompiler rarely uses it. Code-size statistics were moved behind `DUMP_CODE_SIZE_STATS`. d3ceda0c5 2024-12 “CPU/CodeCache: Improve block host size heuristics”

### duckstation-218 — Map only the hot alias of a mirrored memory region
Lens: memory-layout · Source: DuckStation
Principle: Mapping every mirror of a region with the MMU makes all of them fast, but each protection change then needs one `mprotect` per mirror. Map only the mirror that hot code actually uses and send the rest through the slow path. Common accesses stay fast and protection updates stay cheap.
Why it is faster: —
Applies elsewhere: ring buffers double-mapped only where wrap-around matters; selective hugepage use; mapping only the hot shards of a database file with `mmap`.
Watch out: —
Seen in DuckStation: With 2 MB RAM, `Bus::MapFastmemViews()` maps the RAM view only once more, at `0x80600000` (the top KSEG0 mirror, where some games put their stack), not at all four mirrors. `CanUseFastmemForAddress` accepts that range. Stack accesses avoid backpatching overhead, and page-protection changes for code invalidation don't cost 4x the `mprotect` calls on slow ARM devices. 2bfc40824 2025-02 “Bus: Enable mmap fastmem on uppermost KSEG0 mirror”

### duckstation-219 — Size lookup tables to the reachable key space
Lens: lookup-tables · Source: DuckStation
Principle: A direct-indexed table that covers address ranges which can never hold valid keys wastes memory and page-table entries. Clamp each range to what can actually occur.
Why it is faster: —
Applies elsewhere: sparse page tables instead of flat ones; trimming Unicode property tables to assigned planes; bitmap indexes covering only the live id range.
Watch out: —
Seen in DuckStation: The recompiler's code LUT covered the expansion-1 region `0x1F000000–0x1F800000` in all three segments. No cartridge maps code above `0x1F060000`, so `GetLUTRanges()` now stops there, saving about 91 MB. 726aa67d1 2024-12 “CPU/CodeCache: Only cache EXP1 blocks up to 0x1F060000”

### duckstation-220 — Wait at the point of use, not the point of issue
Lens: laziness · Source: DuckStation
Principle: If an operation runs asynchronously, block only when something actually reads a result it is still producing. Waiting on every interaction (including writes that don't depend on it) serializes work that could overlap.
Why it is faster: —
Applies elsewhere: futures awaited only when the value is needed; CPU scoreboarding of register reads; GPU fences placed just before a CPU readback, not after every submit.
Watch out: —
Seen in DuckStation: The interpreter and recompiler stalled until the GTE coprocessor finished on any `mtc2`/`ctc2`/`lwc2`. The `TF_GTE_STALL` flag is now dropped from writes, and `StallUntilGTEComplete()` runs only on reads, which matches hardware. Heavily optimized game code that writes the next inputs while a command is still running now overlaps the two. 8772c336e 2025-02 “CPU: Don't stall until GTE completion on write”

### duckstation-221 — Defer expensive reads until someone asks for the result
Lens: laziness · Source: DuckStation
Principle: If a value is updated far more often than it is read, and producing it is costly (device I/O), set a "stale" flag on update and compute the value on demand.
Why it is faster: —
Applies elsewhere: lazy layout invalidation in browsers; stat caches marked dirty on write; computed properties in reactive frameworks.
Watch out: —
Seen in DuckStation: While the emulated drive head moved, `UpdateSubQPositionWhileSeeking()` called `ReadSectorUncached()` to fetch the subchannel-Q data for each intermediate position. That is a real disc read on physical drives. It now only sets `last_subq_needs_update`, and `EnsureLastSubQValid()` does the read when a GetlocP command actually requests the position. 378fd80e3 2024-11 “CDROM: Defer subq read until needed”

### duckstation-222 — Defer heavyweight callbacks out of the fast path to a safe point
Lens: laziness · Source: DuckStation
Principle: If an event fires in the middle of a tight loop, such as from a side effect inside JIT code, handling it there forces the loop to exit early. Schedule the handler to run at the next scheduled boundary instead, so the fast path keeps running.
Why it is faster: —
Applies elsewhere: Linux softirqs and bottom halves; `queueMicrotask` instead of synchronous re-entry; signal handlers that set a flag the main loop checks.
Watch out: —
Seen in DuckStation: A GPUSTAT read by the CPU can tick the CRTC past the end of a frame. `CRTCTickEvent` now checks `if (!TimingEvents::IsRunningEvents())` and, if so, schedules a zero-delay `s_frame_done_event` instead of calling `System::FrameDone()` in the middle of a block. This prevents “interruption/execution exiting”. f169b892c 2024-09 “GPU: Defer frame done if not running events”

### duckstation-223 — Track dirtiness so flushes and resets happen only after real changes
Lens: other · Source: DuckStation
Principle: Writing a settings file, resetting a descriptor pool or interrupting execution on every "apply" wastes I/O or driver work when nothing changed. Check whether the new value equals the old one before setting a dirty flag, and make the flush conditional on that flag.
Why it is faster: —
Applies elsewhere: React's bail-out when `setState` gets an equal value; ORMs that UPDATE only changed columns; `make` skipping up-to-date targets.
Watch out: —
Seen in DuckStation: `INISettingsInterface::Set*Value` returns early when the stored value already matches, and `QtHost::SaveSettings()` writes only `if (IsDirty())`. `VulkanDevice` calls `vkResetDescriptorPool` only when `needs_descriptor_pool_reset` was set by an allocation. `System::ApplySettings()` no longer calls `InterruptExecution()` and resets performance counters on every settings change. e7aa0495a 2024-09 “System: Not all setting changes need to interrupt” · 6bea16b6e 2025-06 “INISettingsInterface: Only set changed if the value actually changes” · 5f2355510 2025-06 “Qt: Only write settings ini if it is changed” · 889df1d8b 2025-06 “VulkanDevice: Only reset descriptor pool if used”

### duckstation-224 — Make disabled diagnostics nearly free
Lens: other · Source: DuckStation
Principle: Logging and debug markers sit on hot paths. Turn the filter check into a bit test on an enum index instead of a string search, short-circuit everything when no sink is active, and compile GPU debug labels out of release builds entirely. A disabled log statement then costs one predictable branch.
Why it is faster: —
Applies elsewhere: spdlog's compile-time level macros; Linux `pr_debug` with static keys; Android stripping `Log.v` through ProGuard; Rust's `tracing` level filters fixed at compile time.
Watch out: —
Seen in DuckStation: Log channels changed from strings matched against `s_log_filter` to an enum generated by `ENUMERATE_LOG_CHANNELS`, with a `std::bitset<Channel::MaxCount>` and a level packed with the channel into one byte. This removed about 20 KB of code. The log level is forced to `None` when no console, debug, window or file sink is enabled. GPU object names and debug scopes moved behind `ENABLE_GPU_OBJECT_NAMES` and runtime checks, which brought Devel builds close to Release. 23ec40d03 2024-09 “Settings: Set log level to none if no sinks enabled” · 655135821 2024-10 “Log: Replace channel string search with bitset” · 9df59713d 2024-12 “GPUDevice: Put debug messages/scopes behind conditions”

### duckstation-225 — Use overflow-checking builtins instead of hand-written sign tricks
Lens: compiler-build · Source: DuckStation
Principle: `__builtin_add_overflow` and `__builtin_sub_overflow` compile to the add plus a jump on the CPU's overflow flag. The XOR/AND sign-bit formula computes the same answer in several extra ALU instructions that the compiler often can't pattern-match.
Why it is faster: —
Applies elsewhere: Rust's `checked_add`; C23 `ckd_add`; safe integer libraries in browsers (Chromium's `base::CheckedNumeric`).
Watch out: —
Seen in DuckStation: The interpreter's `AddOverflow`/`SubOverflow` used `((new ^ old) & (new ^ add)) & 0x80000000`. On GCC and Clang they now use the builtins and return the sum through a pointer, and callers mark the trap path `[[unlikely]]`. The recompiler already emitted the flag-based form. 9939f1cd0 2025-03 “CPU/Interpreter: Use __builtin_{add,sub}_overflow for exceptions”

### duckstation-226 — Promise no aliasing with restrict
Lens: compiler-build · Source: DuckStation
Principle: If the compiler can't rule out two pointers overlapping, it has to reload memory after every store and can't vectorize freely. Marking a pointer `restrict` promises that nothing else accesses that memory during its lifetime, which lets values stay in registers.
Why it is faster: —
Applies elsewhere: `memcpy`'s restrict-qualified signature; Fortran's no-alias default; Rust's `&mut` exclusivity enabling `noalias`; BLAS kernels.
Watch out: —
Seen in DuckStation: A portable `RESTRICT` macro (`__restrict`/`__restrict__`) was added, with the comment “Gotta coerce the compiler into generating better code”. It is now used on DMA word buffers (`GPU::DMAWrite(const u32* RESTRICT words, …)`), draw command structs in the rasterizer, and the ARM64 assembler pointer. 7116a8043 2024-12 “Common: Add RESTRICT macro”

### duckstation-227 — Make hot singletons static objects, not heap pointers
Lens: memory-layout · Source: DuckStation
Principle: A global `unique_ptr<T>` costs an extra dependent load on every access and can't be resolved at link time. A statically allocated, `constinit`, cache-line-aligned object has a fixed address. Accesses become a single RIP-relative load, and no dynamic initializer is needed.
Why it is faster: —
Applies elsewhere: embedded firmware using static driver instances; Linux per-CPU static structs; replacing lazily-allocated Java singletons with static final fields.
Watch out: —
Seen in DuckStation: `std::unique_ptr<GPU> g_gpu` became `ALIGN_TO_CACHE_LINE GPU g_gpu`, and `g_gpu->` became `g_gpu.` throughout the bus, DMA, timers and rasterizer. The CPU's `State g_state` is now `constinit` with default member initializers, so it is constant-initialized in `.data`/`.bss`. 26db661a0 2024-12 “GPU: Remove global indirection” · e08dda0a0 2025-01 “CPU: Declare state with constinit”

### duckstation-228 — Order struct fields and narrow their types to remove padding
Lens: memory-layout · Source: DuckStation
Principle: The compiler pads after each field to satisfy the next field's alignment. Group fields by size (largest first) and use the smallest type that fits each range. The struct shrinks, so copying it is cheaper and more of it fits in each cache line.
Why it is faster: —
Applies elsewhere: `pahole`-driven struct packing in the kernel; Go's fieldalignment linter; column ordering in Postgres tables for tuple alignment.
Watch out: —
Seen in DuckStation: `Settings` and `GPUSettings`, which are copied whole and now pushed through the GPU FIFO, were reordered. Fields such as `cdrom_read_speedup` and `audio_output_volume` became `u8`, and texture-cache thresholds became `u16`, with clamping on load. 4449ae790 2025-01 “Settings: Rearrange fields to avoid padding”

### duckstation-229 — Validate once at the boundary, not again in every consumer
Lens: algorithms · Source: DuckStation
Principle: When a check is already enforced before data enters a pipeline, repeating it downstream adds branches and arithmetic per item for no benefit. Document the invariant once and remove the duplicates.
Why it is faster: —
Applies elsewhere: API gateways validating requests so internal services trust them; parsing into typed values once ("parse, don't validate"); bounds checks hoisted out of loops.
Watch out: —
Seen in DuckStation: `DrawTriangle` in the software rasterizer repeated six `abs()` comparisons against `MAX_PRIMITIVE_WIDTH/HEIGHT`. The front end already culls oversized primitives before handing them off, so only the degenerate `v0->y == v2->y` check remains. 3c5db39ee 2024-12 “GPU/SW: Remove polygon size checks from backend”

### duckstation-230 — Encode hex by arithmetic into a presized buffer
Lens: algorithms · Source: DuckStation
Principle: Stream or format machinery (locale, width and fill state, virtual calls) costs orders of magnitude more than the actual work of converting a nibble. Reserve the exact output size once and write two characters per byte with a small inline function.
Why it is faster: —
Applies elsewhere: Go's `hex.Encode`; hand-rolled UUID formatting in high-throughput loggers; avoiding `String.format` in Java hot paths.
Watch out: —
Seen in DuckStation: `StringUtil::EncodeHex` used `std::stringstream << std::hex << setw(2)`, and `SmallString::append_hex` called `append_format("{:02X}")` for each byte. Both now `reserve(len*2)` and push `hex_char(b >> 4)`, `hex_char(b & 0xF)`, where `hex_char` is a constexpr lambda. 369599202 2024-09 “StringUtil: Speed up EncodeHex()/append_hex()”

### duckstation-231 — Borrow views into retained data instead of copying strings
Lens: allocation · Source: DuckStation
Principle: If you parse a large document and keep the source buffer alive, every string field can be a `string_view` into it. That saves one heap allocation and copy per field. Likewise, case-insensitive matching with a custom comparator avoids building lowercase copies.
Why it is faster: —
Applies elsewhere: simdjson's on-demand `string_view`s; Rust's `serde` zero-copy `&str` fields; Cap'n Proto readers; avoiding `toLowerCase()` per comparison in JS filters.
Watch out: —
Seen in DuckStation: The game database keeps a copy of the YAML bytes in `s_db_data`, and `GameDatabase::Entry` fields such as `developer` and `genre` are views into it, filled by a new `GetStringFromObject(..., std::string_view*)`. The commit says this saves “a ton of heap allocations”. The Qt game-list filter uses `StringUtil::ContainsNoCase` (`std::search` with an ASCII `ToLower` predicate) instead of allocating, and context-menu lambdas capture a copy-on-write `QString` instead of a `std::string`. d8fef6f22 2024-10 “GameDatabase: Store strings as views” · 4bacbc895 2025-05 “Qt: Eliminate heap allocations in GameListSortModel” · 4616dd329 2025-06 “Qt: Avoid heap string copy for each context menu action”

### duckstation-232 — Open each file once, and remember negative results
Lens: io · Source: DuckStation
Principle: Each `stat`/`open`/`read` round trip costs a syscall, and much more on network or content-URI file systems. Read once and pass the buffer or handle along instead of reopening by path. Cache "this file is invalid" too, so scans don't repeat the failure on every launch.
Why it is faster: —
Applies elsewhere: negative DNS caching; webpack's resolver cache for missing modules; `openat`+`fstat` instead of `stat`+`open`; Git's untracked-cache.
Watch out: —
Seen in DuckStation: For EXE game-list entries, the file was opened for the header and then reopened by `GetGameHashFromFile`. Now it is read once with `ReadBinaryFile(fp)` and hashed from the buffer. Subchannel loaders take an already open `std::FILE*` instead of calling `FileExists()` and then opening. The game list stores invalid files as entries of type `EntryType::Count` (`MakeInvalidEntry`), so they aren't rescanned every time the application starts. 7d2216c28 2024-10 “GameList: Avoid double fopen/read of EXEs” · 160c34ef2 2025-02 “CDROM: Avoid additional stat() call loading subchannel” · dd885cfe2 2025-03 “GameList: Cache invalid entries”

### duckstation-233 — Vectorize horizontal reductions, then finish with a scalar tail
Lens: simd · Source: DuckStation
Principle: To reduce a long array to a sum, min or max, keep a 4-wide vector accumulator in the main loop and collapse it once at the end with a horizontal add, min or max. Elements left over after the last full vector go through a scalar loop. You do a quarter of the loop iterations and stop waiting on one serial dependency chain.
Why it is faster: —
Applies elsewhere: moving averages in telemetry, min/max of an audio peak meter, dot products in a search ranker, `Math.max(...arr)` replaced by WASM SIMD.
Watch out: —
Seen in DuckStation: The audio stream's averaging window (`AudioStream` buffer-fullness average) now sums with `GSVector4::load` plus `vsum.addv()`, with a scalar fallback at the ring-buffer wraparound and for the tail. To support this, new `addv()/minv()/maxv()` horizontal ops were added for SSE (`_mm_movehl_ps`/`_mm_shuffle_ps`) and NEON (`vaddvq_f32`, `vminvq_f32`), and the frame-time graph's min/max now uses them. d3f04dcda 2025-08 “AudioStream: SIMD averaging window calculation” · 6c9d33985 2025-08 “PerformanceCounters: Align to 4 elements”

### duckstation-234 — Guarantee alignment so vector loads can be aligned
Lens: memory-layout · Source: DuckStation
Principle: If a buffer's size is padded to a multiple of the vector width and its start is aligned, you can drop the unaligned-load variants, the prologue and epilogue loops, and any split-cache-line accesses. Put `alignas` on the member, or align the heap allocation, and then use aligned load and store instructions.
Why it is faster: —
Applies elsewhere: 64-byte-aligned tensors in ML runtimes, `posix_memalign` for DMA buffers, Arrow's padded column buffers.
Watch out: —
Seen in DuckStation: The frame-time history is padded to 4 elements, so the min/max scan uses `GSVector4::load<true>` instead of `load<false>`. In the hardware renderer, `m_batch_ubo_data` became `alignas(VECTOR_ALIGNMENT)`, so the texture-window write is a `GSVector4i::store<true>`. GPU backends now get their alignment from the allocation (`make_unique_aligned`) instead of an over-aligned class. 5c682d232 2025-08 “ImGuiOverlays: Frame time history is aligned” · 012c7de23 2025-11 “GPU/HW: Slightly rearrange data fields” · aecdfd5a1 2025-11 “GPU: Align the memory allocation rather than the class”

### duckstation-235 — Load adjacent scalar fields as one vector
Lens: simd · Source: DuckStation
Principle: When two small fields sit next to each other in memory, read them with one load and widen them in registers. Building the vector from two separate scalar loads costs more instructions. A static assertion on the field offsets keeps the trick safe if someone reorders the struct.
Why it is faster: —
Applies elsewhere: loading an (x, y) `u16` pair as a single `u32`, reading RGBA bytes with one 32-bit load, packed width/height in UI layout engines.
Watch out: —
Seen in DuckStation: `GPUTexture::GetSizeVec()` used to construct `GSVector2i(m_width, m_height)`. It is now `GSVector2i::load32(&m_width).u16to32()`, which compiles to `movd`+`pmovzxwd`. `GetRect()` is built from that result, and a `static_assert(OFFSETOF(m_height) == OFFSETOF(m_width) + 2)` guards the layout. 401a0ba4c 2026-03 “GPUTexture: Don't assemble vector in GetSizeVec()”

### duckstation-236 — Mark error paths as unlikely
Lens: branching-fast-paths · Source: DuckStation
Principle: Branch-probability hints such as `[[unlikely]]` or `__builtin_expect` tell the compiler to lay the common path out as straight fall-through code and move the error handling out of line. The hot loop then uses fewer instruction-cache lines and takes fewer jumps.
Why it is faster: —
Applies elsewhere: checks for failed syscalls in a server's hot loop, bounds-check failure paths, Linux's `unlikely()` on error returns.
Watch out: —
Seen in DuckStation: Every interpreter load (`lb/lh/lw/lbu/lhu/lwl/lwr`) now has `if (!ReadMemoryWord(addr, &value)) [[unlikely]]`, because bus errors almost never happen. The log macros wrap their level check the same way. 0de2aafdb 2025-09 “CPU/Interpreter: Memory read failures are unlikely”

### duckstation-237 — Drop packed structs on hot data
Lens: memory-layout · Source: DuckStation
Principle: `#pragma pack(1)` takes away natural alignment. The compiler then has to assume misaligned fields and emit byte-wise or unaligned accesses, which are slow and sometimes trap on strict-alignment CPUs. Keep packing for wire formats and lay hot in-memory structs out naturally.
Why it is faster: —
Applies elsewhere: network headers parsed into packed structs and then copied into a natural one, ARM Cortex-M structs where unaligned access faults.
Watch out: —
Seen in DuckStation: The GTE register struct in `gte_types.h` lost its `#pragma pack(push, 1)`. The shader-cache index entry was also changed from a packed struct to a naturally laid-out key whose layout is checked with `static_assert`s. 27d26794d 2025-10 “CPU/GTE: Remove unnecessary struct padding” · 525dba5c5 2026-01 “GPUDevice: Use sorted array for storing shader cache entries”

### duckstation-238 — Order fields to remove padding, and narrow types that are too wide
Lens: memory-layout · Source: DuckStation
Principle: The compiler pads each field up to its alignment. Grouping fields by size (vectors, then words, then bools) recovers the holes. Shrinking a field to the range it actually needs, such as `u16` for a scanline or a bitfield for flags, makes the object fit in fewer cache lines and makes copies and hashes cheaper.
Why it is faster: —
Applies elsewhere: Go's struct-alignment linters, Rust enum niche packing, shrinking ECS components so more fit per cache line.
Watch out: —
Seen in DuckStation: In `GPU`, the bools were moved ahead of `GSVector4i m_clamped_drawing_area`, `current_scanline` became `u16`, and the duplicated `m_force_frame_timings` was removed. `Settings` and `ProgressCallback` fields were regrouped. `GraphicsConfig` lost its separate `samples` and `per_sample_shading` fields, which now live in a `u16` rasterization bitfield, so the pipeline key is smaller and cheaper to hash. 242280d6e 2025-11 “GPU: Rearrange and reduce struct padding” · 82a44b19b 2025-11 “ProgressCallback: Avoid unnecessary struct padding” · b5316239b 2025-11 “Settings: Slightly reduce struct size” · 7a539ba69 2025-12 “GPUDevice: Reduce GraphicsConfig struct size”

### duckstation-239 — Put fields that are used together next to each other
Lens: memory-layout · Source: DuckStation
Principle: Arrange a struct so the fields touched on the hot path share cache lines, and push cold configuration to the end. Scattered file-scope statics can land anywhere in `.bss`. Collecting them into one cache-line-aligned struct gives them predictable locality and a single base address.
Why it is faster: —
Applies elsewhere: splitting hot and cold fields in a database row cache, the Linux `task_struct` reordering patches, game-object structs ordered by update-loop access.
Watch out: —
Seen in DuckStation: `VulkanDevice` now places `m_current_command_buffer`, `m_dirty_flags`, `m_current_pipeline` and the descriptor pool together, and moves extension flags and debug handles out of the way. It also caches `m_uniform_buffer_alignment` so it no longer reads the large device-properties struct. In `cpu_core.cpp`, about ten `s_` statics (breakpoints, execution mode, jump buffer, trace flags) were folded into `ALIGN_TO_CACHE_LINE static Locals s_locals`, which sits next to the cache-aligned `g_state`. c3415c489 2025-12 “VulkanDevice: Rearrange fields for locality” · 2e659d8cf 2026-01 “CPU: Pack local state in struct”

### duckstation-240 — Keep each thread's private data on its own cache line
Lens: concurrency · Source: DuckStation
Principle: When one thread writes a variable that shares a cache line with data another thread reads or writes, the line bounces between cores even though no data is logically shared. This is false sharing. Group fields by which thread owns them and start each group on a cache-line boundary.
Why it is faster: —
Applies elsewhere: per-core counters in metrics libraries, padded head and tail indices in a lock-free SPSC queue, Java's `@Contended`.
Watch out: —
Seen in DuckStation: In `ImGuiManager`'s state, the members were split into "shared", "read by both" and "owned by GPU thread" groups. The GPU-thread group begins at `ALIGN_TO_CACHE_LINE Timer::Value last_render_time`, so event writes from the input thread no longer invalidate the render thread's line. 593348945 2025-12 “ImGuiManager: Reduce possible false sharing when adding events”

### duckstation-241 — Use the weakest atomic ordering that is correct, and no extra fences
Lens: concurrency · Source: DuckStation
Principle: The default seq_cst atomics and standalone fences emit full barriers on ARM and often on x86. Release on publish, acquire on consume, and relaxed for a single writer's own index are enough for most handoffs. Read an atomic once into a local instead of re-reading it in every condition. If only one thread writes a counter, a relaxed load followed by a release store replaces a locked read-modify-write.
Why it is faster: —
Applies elsewhere: ring buffers in audio callbacks, reference counts (relaxed increment, acq_rel decrement), status flags in a Rust `AtomicU8` state machine.
Watch out: —
Seen in DuckStation: The HTTP request state moved to `load(std::memory_order_acquire)`, taken once per loop, plus `store(..., memory_order_release)`. Many stray `std::atomic_thread_fence` calls in the video thread, presenter and Qt host were removed, and a performance-counter reset that relied on a fence now runs on the owning thread. The single-producer command FIFO replaced `fetch_add` with `load(relaxed)` followed by `store(new % SIZE, release)`. d6f708430 2025-11 “HTTPDownloader: Use release-acquire ordering for request state” · 36788b859 2026-01 “Misc: Eliminate unnecessary atomic thread fences” · 92b888fb8 2026-01 “VideoThread: Fix overreserve in command allocator”

### duckstation-242 — Store many small strings in one bump-allocated pool and refer to them by offset
Lens: allocation · Source: DuckStation
Principle: Thousands of separate `std::string` objects mean thousands of heap allocations and pointer-chasing. Append them all, null-terminated, to one growable byte buffer and keep integer offsets. You get one allocation, contiguous storage and trivially copyable handles. An interning variant first looks the string up in a hash map so duplicates share one copy.
Why it is faster: —
Applies elsewhere: string tables in ELF and compilers, symbol interning in interpreters, the shared-strings table in a JSON or XLSX parser, Arrow's offsets-plus-data string columns.
Watch out: —
Seen in DuckStation: `common/string_pool.h` introduced `BumpStringPool` (`AddString()` returns an `Offset` into a `std::vector<char>`, and `GetString(off)` returns a `string_view`). It was followed by `BumpUniqueStringPool`, which deduplicates through a heterogeneous lookup map. These are used for large, mostly-read string sets such as databases and caches. ac4634c28 2025-10 “Add StringPool and BumpStringPool classes” · 1f9122cca 2026-02 “Common: Add BumpUniqueStringPool”

### duckstation-243 — Use a sorted flat array instead of a node-based hash map for read-mostly indexes
Lens: algorithms · Source: DuckStation
Principle: A node-based hash map allocates once per entry and scatters the entries across the heap. When an index is built once and then mostly read, a contiguous array that is sorted once and searched with `lower_bound` uses a single allocation and stays cache-friendly. It is also trivially serializable.
Why it is faster: —
Applies elsewhere: `boost::flat_map`, SSTable indexes, sorted symbol tables, and route tables loaded at startup in a web server.
Watch out: —
Seen in DuckStation: `GPUShaderCache` replaced `std::unordered_map<CacheIndexKey, CacheIndexData>` with a vector of trivially copyable entries. Because the file is append-only, the vector is `std::ranges::sort`ed after loading, and lookups use `std::lower_bound` with a `memcmp` over the key prefix. The commit says this “saves hundreds of heap allocations”. 525dba5c5 2026-01 “GPUDevice: Use sorted array for storing shader cache entries”

### duckstation-244 — Look up string keys by view so no temporary string is built
Lens: allocation · Source: DuckStation
Principle: If a map's key is `std::string`, a lookup with a `string_view` or a literal builds a temporary `std::string`, which may allocate, just to hash and compare it. Transparent hash and equality functors (heterogeneous lookup) let `find` accept the view directly. For the same reason, functions that only parse or compare should take `string_view`, and small compound values should be encoded as integers rather than strings.
Why it is faster: —
Applies elsewhere: HTTP header maps queried with literals, config lookups, Rust's `HashMap<String,_>::get(&str)` through `Borrow`.
Watch out: —
Seen in DuckStation: The `PreferUnorderedStringMap` fallback, which used a slower ordered map on GCC 10, was removed, so every cache and DB map (game list, texture replacements, translations, game database) is a transparent `UnorderedStringMap`. Played-time parsing now produces `std::string_view serial` instead of a `std::string` per line. The aspect-ratio combo box stores `num<<16|den` in a `QVariant` instead of a formatted name that had to be parsed back. 14625af98 2025-12 “Common: Remove fallback for missing heterogeneous lookup” · 3d2c3c919 2025-10 “GameList: Avoid a couple of heap allocations” · 38bb240a4 2025-09 “Qt: Avoid extra string storage”

### duckstation-245 — Decode straight into the final buffer
Lens: allocation · Source: DuckStation
Principle: Library convenience calls often want an array of row pointers or a scratch buffer, which costs an extra allocation and sometimes a copy. Calling the lower-level per-row API and passing the destination rows directly removes the temporary.
Why it is faster: —
Applies elsewhere: `read()` straight into a pre-sized `Vec`, decoding JPEG scanlines into a texture upload buffer, zero-copy protobuf parsing.
Watch out: —
Seen in DuckStation: The PNG loader built a `std::vector<png_bytep>` for `png_read_image()`. It now loops over `png_set_interlace_handling()` passes and calls `png_read_row(png_ptr, image->GetRowPixels(y), nullptr)` for each row. 8f78f8358 2025-09 “Image: Don't use png_read_image()”

### duckstation-246 — Check for a change before doing the expensive update
Lens: laziness · Source: DuckStation · Also: curl
Principle: Compare the new value with the old one and return early if they match, or collect change flags and notify once. Redundant resizes, redraws and device writes then cost a single compare instead of the full update.
Why it is faster: —
Applies elsewhere: React's `shouldComponentUpdate` and memo, skipping a GPIO write when the pin level is unchanged, setters that return early in observable models.
Watch out: —
Seen in DuckStation: `DynamicHeapArray::resize()` now returns immediately when `m_size == new_size`. `ProgressCallback::SetState()` builds a `StateChange` bitmask and calls one virtual `StateChanged()` only if the mask is non-zero, replacing several virtual setters. Controller vibration is collected and sent to the device once per frame instead of on every motor write. a8776a3b2 2026-01 “DynamicHeapArray: Don't resize if size is the same” · 7659a70c1 2026-04 “Common: Get rid of multiple virtuals in ProgressCallback” · 9d8d81d09 2025-10 “InputManager: Only update vibration once per frame” • Also seen in curl (curl-037): 1128029599 2006-10-07 `multistate()`: `if(oldstate == state) return;`. • 4d28a59fc2 2012-04-17 The progress bar redraws only `if(point != bar->prev)`. • 9da14a96ab 2015-03-07 `Curl_conncontrol` logs and assigns only when the close bit changes. • bf601472a0 2021-02-22 Once-per-transfer setup moved so re-entry from RATELIMITING no longer reruns it; it had "set the PRETRANSFER time much too often". • bc90308328 2023-03-12 SIGPIPE ignore and restore happen only when the next handle's `no_signal` differs ("For improved performance"); 9703dabd77 2026-01-15 carries a context down so `sigaction()` is called only where socket I/O happens and the setting differs.

### duckstation-247 — Run speculative re-execution only when its input actually changed
Lens: laziness · Source: DuckStation
Principle: Speculative schemes such as rollback, runahead and optimistic UI pay a large cost each time they are invalidated. Trigger the invalidation only when the observed input differs from the previous value. Let callers say which inputs are worth speculating on, so noisy or low-value inputs don't trigger expensive replays.
Why it is faster: —
Applies elsewhere: rollback netcode that resimulates only on a misprediction, incremental builds that rebuild only when content hashes change, React concurrent rendering that discards stale work.
Watch out: —
Seen in DuckStation: Analog controllers, the analog joystick and JogCon save their previous axis state and call `System::SetRunaheadReplayFlag()` only if `memcmp` shows a difference, which reduces idle CPU use. A new “Runahead for Analog Input” option passes `SetRunaheadReplayFlag(is_analog)`, so analog-only changes can skip runahead. This avoids the cost of serializing PGXP state. 10f9039de 2025-10 “Controller: Skip runahead replay when analog is unchanged” · e6788b313 2025-10 “System: Add 'Runahead for Analog Input' option”

### duckstation-248 — Size buffers to the active configuration, not the maximum
Lens: memory-layout · Source: DuckStation
Principle: Allocating for the worst case wastes memory, and any per-frame copy, snapshot or clear of that buffer then touches all of it. Sizing it to the current configuration shrinks every memcpy proportionally.
Why it is faster: —
Applies elsewhere: sizing a frame buffer to the real resolution instead of 4K, sizing a ring buffer to the negotiated MTU, snapshotting only the used part of an arena.
Watch out: —
Seen in DuckStation: PGXP shadow memory was always `(RAM_8MB + scratchpad)/4` values. It is now `scratch + g_ram_size/4`, and the save-state size depends on `enable_8mb_memory`. Runahead copies this array every frame, and the commit reports “roughly a 3x performance improvement” with PGXP and runahead together. e62de2322 2026-03 “CPU/PGXP: Allocate storage based on active RAM size”

### duckstation-249 — Skip render passes that would have no effect
Lens: gpu-rendering · Source: DuckStation
Principle: An intermediate pass costs a full-screen read and write of bandwidth. If its output would equal its input, or it would blur a blank background, skip it. Base the check on the dimensions that actually matter, not on proxies that differ even when the pass is an identity.
Why it is faster: —
Applies elsewhere: skipping a CSS filter layer when the blur radius is 0, bypassing a colour-grading LUT when it is identity, not resampling images that are already the target size.
Watch out: —
Seen in DuckStation: The post-processing source preprocess was triggered whenever the texture view size differed from the VRAM size, which is always true when upscaling. It now compares `m_display_vram_width` with `m_display_width`. The loading screen blurs its background only when a display texture exists. 4a246a74a 2025-11 “GPU: Avoid unnecessary postfx source preprocess when upscaling” · 0cc6bde71 2026-04 “FullscreenUI: Don't blur empty background in loading screen”

### duckstation-250 — Don't build frames that will never be shown
Lens: gpu-rendering · Source: DuckStation
Principle: When output runs faster than the display, for example during fast-forward, decide first whether this frame will be presented. Skip the UI build, draw submission and context restore for frames that will be dropped. Making the decision on refresh-rate timing, and only when vsync would otherwise block, keeps pacing smooth.
Why it is faster: —
Applies elsewhere: `requestAnimationFrame` instead of `setInterval` rendering, skipping chart redraws while a tab is hidden, frame-skipping in video players.
Watch out: —
Seen in DuckStation: ImGui `NewFrame`/`EndFrame` no longer runs on skipped presents. The rewritten present-skip logic, `VideoPresenter::ShouldPresentFrame(present_time)`, adds a `WhenVSyncBlocks` mode and throttles to the surface refresh rate (at most 50 skips in a row). `RestoreDeviceContext()` runs only if something was drawn. The commit says this “significantly improves frame pacing for fast forward speeds”. 628e41f84 2025-08 “ImGuiManager: Don't render when skipping present” · ff0ad1b17 2026-01 “System: Rewrite present skipping” · 25211c6fd 2026-01 “VideoPresenter: Move present throttling from VideoThread”

### duckstation-251 — Don't break a render pass unless you must
Lens: gpu-rendering · Source: DuckStation
Principle: On tiled GPUs and modern APIs, ending and restarting a render pass flushes tile memory and can force load and store operations. Rebinding the same target should be a no-op. Order state changes so that the operations that may restart the encoder (such as uniform pushes) come after the cheap ones.
Why it is faster: —
Applies elsewhere: merging Vulkan subpasses on mobile, batching Metal encoder state, WebGPU code that reuses one render pass for several draws.
Watch out: —
Seen in DuckStation: `D3D12Device` no longer calls `EndRenderPass()` when the render target being set is the current one. On Metal, the hardware renderer now calls `SetScissor` and `SetViewportAndScissor` before `PushUniformBuffer`, which avoids redundant calls. 4b3014735 2026-01 “D3D12Device: Avoid render pass end when setting same target” · eefcd9071 2025-10 “GPU/HW: Avoid some redundant calls in Metal”

### duckstation-252 — Keep a small dedicated buffer for per-draw constants
Lens: gpu-rendering · Source: DuckStation
Principle: Streaming a handful of per-draw constants through a big shared uniform ring brings alignment padding (often 256 bytes), map and unmap bookkeeping, and descriptor rebinding with every draw. A fixed small buffer that is updated with discard semantics, or real push constants, sits permanently in its own slot. Uploading the constants in the same call as the draw ensures a flush cannot separate them.
Why it is faster: —
Applies elsewhere: Vulkan push constants, D3D12 root constants, WebGPU's small per-object uniform buffers with dynamic offsets.
Watch out: —
Seen in DuckStation: `D3D11Device` now creates a 128-byte `m_push_constant_buffer`, bound once to slot 1 for the VS, PS and CS stages and updated with `D3D11_MAP_WRITE_DISCARD`, replacing aligned slices of the shared stream buffer. Metal and GL got matching layouts, and D3D12 moved root constants to slot 1. Calls are now `DrawWithPushConstants(3, 0, &uniforms, sizeof(uniforms))`. 916b23f85 2025-10 “GPUDevice: Use separate buffer for push constants” · 6848f07c5 2025-10 “GPUDevice: Combine draw and push constant functions”

### duckstation-253 — Do the CPU work before blocking on a fence, when it's safe
Lens: gpu-rendering · Source: DuckStation
Principle: When the CPU must wait for submitted GPU work, check whether the next per-frame resource is already free. If it is, reset and begin it first, then wait, so the setup work overlaps the GPU. If it isn't, wait first, because waiting afterwards would mean two fence waits.
Why it is faster: —
Applies elsewhere: double-buffered command lists in any graphics engine, issuing the next DMA descriptor before polling the previous one, preparing the next HTTP request while the previous response streams in.
Watch out: —
Seen in DuckStation: In Vulkan's `EndAndSubmitCommandBuffer(wait_for_completion, ...)` and D3D12's readback path: `if (m_completed_fence_counter >= m_frame_resources[next].fence_counter) { BeginCommandBuffer(next); WaitForCommandBufferCompletion(cur); } else { Wait(cur); Begin(next); }`. 5e6a18a58 2026-01 “VulkanDevice: Wait before starting next command buffer” · af305d381 2026-01 “D3D12Device: Avoid second fence wait for infrequent readback”

### duckstation-254 — Choose the fastest equivalent API per driver
Lens: platform · Source: DuckStation
Principle: Two API calls that do the same thing can differ several times in speed on a particular vendor's driver. Detect the vendor once at startup and route to the faster path, keeping the standard path as the default.
Why it is faster: —
Applies elsewhere: picking `memcpy` variants by CPUID, browser GPU blocklists and workarounds, database drivers that switch to bulk-copy protocols on servers that support them.
Watch out: —
Seen in DuckStation: `OpenGLDevice` sets `m_use_get_texture_sub_image = false` on `NVIDIAProprietary`, so texture readback uses `glReadPixels()`, which the commit says is “about 2x faster”. d67b9b9d2 2026-01 “OpenGLDevice: Don't use glGetTextureSubImage() on NVIDIA”

### duckstation-255 — Reuse an intermediate result you already have instead of copying it
Lens: gpu-rendering · Source: DuckStation
Principle: If an earlier step has already produced the image or buffer a later step needs, pass that object along. Don't render or copy it into another target. Each avoided full-screen copy saves bandwidth and a resource allocation.
Why it is faster: —
Applies elsewhere: sharing a decoded frame between preview and encoder, passing `ArrayBuffer`s by transfer instead of copy, render-graph aliasing.
Watch out: —
Seen in DuckStation: UI transitions now receive the existing blur target, `RenderTransitionBlend(swap_chain, transition_texture)`, rather than drawing into a separate `transition_current_texture` and copying out of it. cbc6a162e 2026-03 “VideoPresenter: Reuse blur target for transitions”

### duckstation-256 — Rebuild compiled artifacts only when an input they depend on changes
Lens: laziness · Source: DuckStation
Principle: Pipelines, shaders and query plans are expensive to build. Key the cache on what the artifact actually depends on (for a shader, its output format, not the target size) and keep the artifact across unrelated changes. Likewise, bind only the resources the compiled program actually uses.
Why it is faster: —
Applies elsewhere: prepared statements reused across parameter changes, webpack's persistent cache keyed on content, regexes compiled once.
Watch out: —
Seen in DuckStation: The ReShade GLSL post-processing shader recompiled its pipeline on every resize. It now returns early `if (m_output_format == format)`. The slang loader asks SPIRV-Cross for the active interface variables and remaps unused textures out of range, so they are never bound. a49d0b665 2025-10 “PostProcessing/GLSL: Skip recompile on target size change” · fbc99916f 2025-11 “PostProcessing/Slang: Remap and skip binding unused textures”

### duckstation-257 — Keep expensive system handles alive, and update them in place
Lens: platform · Source: DuckStation
Principle: Driver instances, audio contexts and similar handles are slow to create and sometimes leak when destroyed. Create them once and share them by reference count. When settings change, update the parameters on the live object instead of tearing it down and recreating it.
Why it is faster: —
Applies elsewhere: database connection pools, a reused `HttpClient`, a long-lived `AudioContext` in web apps, keeping a GL context across scene loads.
Watch out: —
Seen in DuckStation: `VulkanDevice` now keeps its `VkInstance` for the whole process. The commit notes that recreating it leaked 20–30 MB and that keeping it “reduces startup time too”. Cubeb streams share one ref-counted `CubebContextHolder`. `CoreAudioStream::UpdateParameters()` applies stretch and buffer changes to the live stream, and a restart is needed only for latency changes. 69458bd90 2025-12 “VulkanDevice: Persist instance throughout launches” · 132ca44f0 2025-12 “CubebAudioStream: Use a single shared context” · f2adc3970 2026-01 “CoreAudioStream: Handle parameter changes without restarting”

### duckstation-258 — Run network I/O on a worker that blocks on readiness
Lens: io · Source: DuckStation
Principle: Driving transfers by polling from a client thread ties throughput to how often that thread happens to poll. A dedicated worker blocking in the multiplexer (`poll`/`epoll`, `curl_multi_poll`) reacts as soon as data arrives, and a wakeup handle lets other threads interrupt it. Waiters that do have to poll should sleep for a reasonable interval rather than spinning at 1 ms.
Why it is faster: —
Applies elsewhere: Node's libuv thread, a dedicated Netty event loop, background download managers on mobile.
Watch out: —
Seen in DuckStation: `HTTPDownloaderCurl` gained `WorkerThreadEntryPoint()`. Requests are queued under a mutex, and `curl_multi_wakeup` is used for shutdown. The commit says it “should significantly speed up update downloads”. The `InternalPollRequests()` hook was then removed. `WaitForAllRequests` now sleeps 16 ms instead of 1 ms. e2b3f6513 2025-11 “HTTPDownloaderCurl: Use worker thread for processing” · a1162731a 2025-11 “HTTPDownloader: Remove InternalPollRequests()” · 574be46fe 2026-04 “HTTPDownloader: Increase poll interval to 16ms”

### duckstation-259 — Keep slow enumeration off latency-critical threads
Lens: concurrency · Source: DuckStation
Principle: Enumerating devices, adapters and files can block for tens of milliseconds. Run it on a background or owning thread and post the result back, so the UI or render thread never stalls. Conversely, if you are already on the right thread, call directly instead of queueing a hop to yourself.
Why it is faster: —
Applies elsewhere: listing Bluetooth devices asynchronously in a mobile app, populating a file picker from a worker, avoiding `Dispatcher.Invoke` when already on the UI thread.
Watch out: —
Seen in DuckStation: Qt audio settings populate output devices through `QtAsyncTask`, which the commit says saves 40–60 ms when opening the window. Fullscreen UI auto-mapping calls `InputManager::EnumerateDevices()` on the core thread and posts the list back to the video thread. Controller-disconnect handling no longer wraps `PauseSystem` in a redundant `RunOnCoreThread`. fbdd2689b 2025-12 “Qt: Populate audio devices asynchronously” · bf292ddf7 2026-03 “FullscreenUI: Avoid calling EnumerateDevices() on video thread” · 249f9d03f 2026-01 “InputManager: Avoid redundant core thread invocation”

### duckstation-260 — Defer startup work until first use, and make it thread-safe with call_once
Lens: laziness · Source: DuckStation
Principle: Work done at construction is paid even when the feature is never used. Load fonts, lay out views and compile shaders the first time they are needed. When several threads might trigger that first load, `std::call_once` (or a double-checked flag) does it exactly once, and the fast path after that costs almost nothing.
Why it is faster: —
Applies elsewhere: code-split routes in SPAs, lazily loaded web fonts, Python's lazy imports, `static` local initialization in C++.
Watch out: —
Seen in DuckStation: The Qt interface settings page defers loading CJK fonts, which the commit says saves about 170 ms. The game grid sets `m_layout_update_deferred = true` until it is first shown, which saves about 200 ms at startup. The fullscreen UI no longer loads the app icon or background shader just to show the pause menu. `GameDatabase` loads via `std::call_once(s_state.load_once_flag, &Load)`. 3d6627c7a 2025-11 “Qt: Speed up interface page creation” · f03bb5b25 2025-12 “Qt: Defer initial layout of game grid” · f4f8aa587 2025-10 “FullscreenUI: Defer more resource loads” · dec06fa67 2025-08 “GameDatabase: Use once_flag for thread-safe initialization”

### duckstation-261 — Prefetch what the user is about to need
Lens: io · Source: DuckStation
Principle: Fetch likely-needed assets just ahead of the viewport or event, such as the next rows of a grid or the image for the next unlock, so they are ready before they are shown. The latency moves to idle time that nobody perceives.
Why it is faster: —
Applies elsewhere: `<link rel=prefetch>`, infinite-scroll lists that load a page ahead, OS read-ahead, map tiles fetched around the viewport.
Watch out: —
Seen in DuckStation: The fullscreen game grid requests covers within `2 × ClipRect.height` below the scroll position. The achievements list prefetches three scrolls' worth of badges. On unlock or start, `PrefetchNextAchievementBadge()` downloads the most likely next badge, and an optional mode pre-downloads all locked badges. c9a7bf742 2026-04 “FullscreenUI: Prefetch two rows of covers in grid view” · 92b7cb98a 2026-02 “Achievements: Prefetch next achievement badge on unlock/start” · 2bccb2929 2026-02 “Achievements: Add option to prefetch badges”

### duckstation-262 — Persistent download cache with request coalescing
Lens: io · Source: DuckStation
Principle: Put remote assets in a single local archive (index plus blob) instead of thousands of loose files. Collapse concurrent requests for the same URL into one in-flight fetch whose callbacks all fire when it completes. After a miss, check again under the lock so a racing completion isn't fetched twice.
Why it is faster: —
Applies elsewhere: Go's `singleflight`, CDN request collapsing, a browser HTTP cache, a service-worker cache-first strategy.
Watch out: —
Seen in DuckStation: `util/http_cache` provides `Lookup`, `LookupOrFetch` and `Prefetch` on top of the new `ObjectArchive` (index and blob files). Pending callbacks are grouped per URL so a request is never queued twice. Achievements, game-list covers and Qt downloads moved to it, replacing the `achievement_images` directory of loose files. 1690c6949 2026-04 “Util: Add HTTPCache module” · 331571b38 2026-04 “Util: Add ObjectArchive class” · 32ab1b70f 2026-04 “Achievements: Use HTTP cache instead of loose files”

### duckstation-263 — Cache decoded assets in memory with an LRU, and parse from memory
Lens: io · Source: DuckStation
Principle: Re-reading and re-parsing a small asset from disk every time it is used costs a syscall and a decode each time. Keep the decoded form in a bounded LRU and play or draw from it. Parse from an in-memory span so the cached bytes can be decoded without going through file I/O again.
Why it is faster: —
Applies elsewhere: UI sound-effect pools in games, an image-decode cache in browsers, template caches in web frameworks.
Watch out: —
Seen in DuckStation: `SoundEffectManager` keeps an `LRUCache<std::string, CachedEffectPtr>` of 32 decoded sample arrays and plays a `PlayingCachedEffect` instead of reopening a `WAVReader`. Reads happen asynchronously. `WAVReader` gained in-memory chunk parsing over a `std::span<const u8>`. a49447722 2025-12 “SoundEffectManager: Add caching and async reading” · c7c29d414 2025-12 “WAVReader: Add in-memory parsing”

### duckstation-264 — Cache values that are expensive to derive and rarely change
Lens: laziness · Source: DuckStation
Principle: If a value such as a locale object, a native window handle or a file path is costly to compute but changes only on a known event, compute it once, store it, and invalidate it only on that event.
Why it is faster: —
Applies elsewhere: caching `Intl.DateTimeFormat` instances, caching DOM measurements until resize, memoizing a resolved config path.
Watch out: —
Seen in DuckStation: Qt previously called `QLocale::system()` for every date and number it formatted. It now uses `s_app_locale`, which is updated only when the language changes. `DisplayWidget` caches `m_window_info` rather than calling `winId()` on every resize, and the commit notes this is also faster. Achievements stores `user_badge_path` at login instead of rebuilding it each time the badge is drawn. ef320c62a 2025-07 “Qt: Cache app locale object” · fa23d16c3 2025-12 “Qt: Cache window handle for size updates” · ba28a3fb3 2025-10 “Achievements: Cache user badge path”

### duckstation-265 — Reuse the slot from your lookup instead of searching twice
Lens: algorithms · Source: DuckStation
Principle: The pattern "if not found, insert" probes the map twice. Keep the pointer or iterator the first lookup returned and write through it, or use `try_emplace`, so one hash and probe does both jobs.
Why it is faster: —
Applies elsewhere: Python's `dict.setdefault`, Rust's `entry()` API, `Map.get` followed by `Map.set` collapsed into one call.
Watch out: —
Seen in DuckStation: In the Qt game-list cover loader, `!m_cover_pixmap_cache.Lookup(path)` followed by `Insert(path, ...)` became `QPixmap* pm; ... !(pm = Lookup(path))` followed by `*pm = QPixmap::fromImage(image)`. 393724abd 2025-09 “Qt: Save one lookup when loading covers”

### duckstation-266 — Give the renderer data in its native format
Lens: other · Source: DuckStation
Principle: 2D rasterizers have fast paths for particular pixel formats, such as opaque `RGB32` instead of `ARGB32`. Convert images once when you load them so they don't go through a slow blend or convert step on every paint. Hand the library the source (a path) and let it decode lazily rather than eagerly converting between types.
Why it is faster: —
Applies elsewhere: premultiplied-alpha textures, `CGImage` formats matched to the display, calling `convert()` on Pygame surfaces once at load.
Watch out: —
Seen in DuckStation: Cover and background images are converted to `QImage::Format_RGB32` before padding and drawing, following Qt's QPainter performance notes. Game icons are created with `QIcon(path)` instead of loading a `QPixmap`, resizing it and caching it for every row. dfc650410 2025-09 “Qt: Use RGB32 format for covers/background” · e89590be4 2025-08 “Qt: Avoid QPixmap -> QIcon conversion in getIconForGame()”

### duckstation-267 — Invalidate only the parts that changed
Lens: laziness · Source: DuckStation
Principle: A full model reset forces every view to rebuild its layout and caches. Signal a change to just the affected rows, columns or roles, and views can update only those items.
Why it is faster: —
Applies elsewhere: React keyed list updates, `notifyItemRangeChanged` on Android RecyclerView, partial cache purges by tag.
Watch out: —
Seen in DuckStation: Toggling cover titles or localized titles called `m_model->refresh()`. It now emits `dataChanged(index(0, Column_Cover), index(rowCount()-1, Column_Cover), {Qt::DisplayRole})`, which covers just the affected column and role. d499d8e2d 2025-08 “Qt: Avoid a full reset when toggling show cover titles / localized titles”

### duckstation-268 — Poll inputs only when something consumes them
Lens: platform · Source: DuckStation
Principle: High-rate sources such as gyros, accelerometers and sensors cost CPU and bus bandwidth even when nothing reads them. Enable polling only when there is a binding or a listener. Drop unwanted events at the source instead of forwarding them to another thread to be discarded there.
Why it is faster: —
Applies elsewhere: registering Android sensor listeners only while visible, unsubscribing websocket topics when no widget needs them, gating GPS on mobile.
Watch out: —
Seen in DuckStation: `InputManager` gained `HasAnyBindingsForSubclass()` and `UpdateInputSubclassPolling()`. The SDL source now enables sensor reports only for devices with bindings. The binding dialog's hook returns `StopProcessingEvent` for sensor events when sensor mapping is off, so they are never forwarded to the UI thread. 682c9c643 2026-02 “InputManager: Only enable sensor polling if there are bindings”

### duckstation-269 — Use a plain function pointer and inline payload for queued commands
Lens: allocation · Source: DuckStation
Principle: A `std::function` holding a capturing lambda can allocate, needs a destructor call, and calls through two levels of indirection. For commands in a ring buffer, store a raw `void(*)(void*)` and place the argument bytes directly after the header. The command becomes trivially destructible and is consumed by a single call.
Why it is faster: —
Applies elsewhere: job systems in game engines, the Linux `work_struct`, message queues on embedded RTOSes, C callback-plus-context APIs.
Watch out: —
Seen in DuckStation: A new `VideoThreadAsyncBufferCallCommand { void (*func)(void*); }` was added, with `static_assert(std::is_trivially_destructible_v<...>)`. The consumer runs `acmd->func(acmd + 1)`. The old path wrapped the buffer pointer in a `std::function` lambda and then had to call `~VideoThreadAsyncCallCommand()`. 05d470be1 2026-04 “VideoThread: Add VideoThreadAsyncBufferCallCommand”

### duckstation-270 — Evaluate log arguments only when the level is enabled
Lens: compiler-build · Source: DuckStation
Principle: If a log call is a function, its arguments (formatting, `__func__` strings, temporaries) are built before the level check. A macro that tests the level first and only then evaluates the arguments makes disabled logs close to free. Keeping optional metadata only on the levels that need it shrinks the payload further.
Why it is faster: —
Applies elsewhere: SLF4J's `isDebugEnabled()` guards, Rust's `log!` macros, `if (VLOG_IS_ON(2))` in glog.
Watch out: —
Seen in DuckStation: `Log::FastWrite()` was replaced by a macro: `do { if ((level) <= Log::GetLogLevel()) [[unlikely]] Log::Write(PackCategory(...), __VA_ARGS__); } while (0)`, so the format arguments are packed inside the branch. The INFO, DEV, DEBUG and TRACE macros also stopped passing function names, which are now kept for warnings and errors only. e3a044a8a 2025-11 “Log: Move FastWrite() into macro” · 480d00232 2025-11 “Log: Only include function names for warn/error”

### duckstation-271 — Write header functions so the linker can deduplicate them
Lens: compiler-build · Source: DuckStation
Principle: A `static` function or constant in a header gives every translation unit its own private copy. `inline` gives them external linkage with one-definition semantics, so the linker folds the copies into one. The binary gets smaller and less instruction-cache space is wasted.
Why it is faster: —
Applies elsewhere: `inline constexpr` constants in C++17 headers, C99 `inline` plus a single `extern` definition, avoiding `static` helpers in widely included headers.
Watch out: —
Seen in DuckStation: Many `ALWAYS_INLINE static` helpers in `log.h`, `string_util.h`, `bus.h`, `cpu_core.h` and `ryml_helpers.h` became plain `inline`. The commit reports “a couple percent” off the debug binary size. 52d9f73f9 2025-07 “Misc: Replace a bunch of static qualifiers with inline”

### duckstation-272 — Return one named object so the compiler can elide the copy
Lens: compiler-build · Source: DuckStation
Principle: Named return value optimization (NRVO) only applies when every return path returns the same local. If different branches return different objects or literals, the result gets constructed and then moved or copied. Declare the result once, assign it on each branch, and return it.
Why it is faster: —
Applies elsewhere: builders in C++ that return a `std::vector`, factory functions for heavy structs, Swift and Rust struct returns.
Watch out: —
Seen in DuckStation: `GetExecutableNameForImage()` used to `return FALLBACK_EXE_NAME;` on some paths and `std::string code = iter->second; return code;` on others. It now declares `std::string code;` up front, assigns it on every branch and always returns `code`. d8912cb3c 2025-09 “System: Allow RVO in GetExecutableNameForImage()”

### duckstation-273 — Format numbers with locale-free primitives, not streams
Lens: algorithms · Source: DuckStation
Principle: `std::ostringstream` with an imbued locale allocates, takes locale locks and dispatches through virtual facets. `std::to_chars` writes into a caller-provided buffer with no allocation and gives the shortest round-trip output.
Why it is faster: —
Applies elsewhere: JSON serializers using Ryu or `to_chars`, metrics exporters, CSV writers.
Watch out: —
Seen in DuckStation: `StringUtil` removed its non-MSVC fallback (`ss.imbue(std::locale::classic()); ss << value;`) now that every supported toolchain has floating-point `to_chars()`. 0a9bf5aa0 2025-12 “StringUtil: Use to_chars() for floating point types”

### duckstation-274 — Don't compress data that is already compressed
Lens: io · Source: DuckStation
Principle: Running a general-purpose compressor over archives, images or media burns CPU for little or no size reduction. Store such data as is (compression level 0).
Why it is faster: —
Applies elsewhere: nginx `gzip_types` that excludes images, zip `-0` for JPEGs, Docker layers containing `.tar.gz` files.
Watch out: —
Seen in DuckStation: GitHub Actions `upload-artifact` steps for the AppImage, macOS and Windows builds now set `compression-level: 0`, because the artifacts are already compressed. 3072e65c4 2026-02 “CI: Don't compress already-compressed artifacts”

### duckstation-275 — Port the JIT backend from a similar ISA
Lens: jit-interpreters · Source: DuckStation
Principle: A JIT is only fast on architectures that have a backend. On new targets, everything else falls back to an interpreter that can be an order of magnitude slower. Translating an existing backend for a structurally similar ISA (another load/store RISC with a similar register file) is the cheapest way to bring the fast path to a new target.
Why it is faster: —
Applies elsewhere: V8 and SpiderMonkey adding RISC-V and LoongArch backends, LuaJIT ports, the same issue for WASM runtimes on new CPUs.
Watch out: —
Seen in DuckStation: A new `cpu_recompiler_loongarch64.cpp` of about 2,650 lines was added, “mostly a direct translation from the RISC-V backend”. c1ae04a7b 2026-03 “CPU/LoongArch64: Add recompiler”

### duckstation-276 — Keep snapshots in the cheaper memory tier
Lens: memory-layout · Source: DuckStation
Principle: When frequent snapshots of state would use up a scarce resource (VRAM, device memory), keep an equivalent snapshot in a cheaper tier. Use a CPU-side representation, accepting some extra compute or lower fidelity, so the scarce tier is left for live work.
Why it is faster: —
Applies elsewhere: spilling GPU tensors to host memory, undo history stored as diffs on disk, swapping Redis values out to SSD.
Watch out: —
Seen in DuckStation: A “Low VRAM” rewind mode (`UseSoftwareRendererForMemoryStates`) uses the software renderer's VRAM for rewind states. This removes the cost of upscaled texture copies per slot, and `CalculateRewindMemoryUsage` reports zero VRAM in this mode. 654587ea3 2025-11 “System: Add 'Low VRAM' rewind mode”

### duckstation-277 — Make a dense switch provably exhaustive so it drops its bounds check
Lens: branching-fast-paths · Source: DuckStation
Principle: A switch over a small integer usually compiles to a jump table guarded by a range check plus a branch to `default`. Enumerate every remaining value as an explicit (cold) case and mark the default unreachable. The compiler can then index the table directly, which removes a compare and branch from every dispatch.
Why it is faster: —
Applies elsewhere: bytecode interpreters (Lua, WASM), protocol parsers that switch on an opcode byte, and state machines in embedded firmware where every state value is known.
Watch out: —
Seen in DuckStation: The interpreter now switches on `static_cast<u8>(inst.op.GetValue())` instead of on the enum, and its fetch and trace paths gained `[[unlikely]]` hints. Together these give up to 20% in interpreter-bound games. The GTE dispatcher replaced `default: Panic()` with every unused opcode listed as `[[unlikely]] case N:`, followed by `DefaultCaseIsUnreachable()`. The I-cache fill and the MDEC command decoder got the same treatment. 667e1147a 2026-06 “CPU: Add branch hints to interpreter dispatch” · 2b5baa1b8 2026-06 “CPU: Add more branch hints to interpreter” · 0b700d1b8 2026-06 “GTE: Eliminate bounds check in dispatcher” · 91273f2b3 2026-06 “CPU: Eliminate default branch in icache fill” · 4ac22708f 2026-10 “MDEC: Turn state dispatcher into a jump table”

### duckstation-278 — Replace a register-select switch with a table of (pointer, mask) pairs
Lens: lookup-tables · Source: DuckStation
Principle: When each case of a switch just reads or writes a different variable, and maybe applies a mask, put that variable's address and its mask in a constexpr table indexed by the selector. One indexed load replaces a multi-way branch, and a null pointer marks the rare invalid entries.
Why it is faster: —
Applies elsewhere: memory-mapped register files in device drivers, CSR access in RISC-V simulators, and generic property get/set by numeric ID in game engines.
Watch out: —
Seen in DuckStation: The interpreter's `mfc0` handling was a long `switch (Cop0Reg)`. It now uses `s_cop0_table`, a `std::array<std::pair<u32*, u32>, 32>` of register pointer and write mask: `if (const u32* value = GetCop0RegPtr(reg)) [[likely]] WriteRegDelayed(rt, *value);`. c660f8532 2026-06 “CPU: Use table lookup for mfc0”

### duckstation-279 — Put per-opcode metadata next to the handler pointer in a constexpr table
Lens: lookup-tables · Source: DuckStation
Principle: If every handler starts by checking "do I have enough input yet?" against a per-opcode constant, store that constant in the dispatch table beside the function pointer. The caller checks it once, generically, before the call. Each handler loses its prologue, and a failed check never pays for the indirect call.
Why it is faster: —
Applies elsewhere: length-prefixed protocol decoders (minimum bytes per message type), x86 decoders with operand-length tables, and command buffers in GPU drivers.
Watch out: —
Seen in DuckStation: The GP0 command table changed from `std::array<GP0CommandHandler, 256>` to `std::array<GP0Command, 256>`, where `struct GP0Command { GP0CommandHandler handler; u8 num_words; }`. A constexpr lambda builds it, computing word counts from the primitive bits. It replaces the `CHECK_COMMAND_SIZE` checks that were inside each handler. f7fc96137 2026-11 “GPU: Look up GP0 command word counts in the command table”

### duckstation-280 — Turn a caller-known enum into a template parameter
Lens: compiler-build · Source: DuckStation
Principle: When every call site passes a compile-time-constant "kind" and the callee branches on it, make the kind a template parameter and use `if constexpr`. Each instantiation keeps only its own logic, the checks disappear, and the smaller bodies inline more easily.
Why it is faster: —
Applies elsewhere: serializers templated on format, image filters templated on pixel format, and Rust generics or C++ templates over a strategy type instead of a runtime flag.
Watch out: —
Seen in DuckStation: `GPU::FillDrawCommand` became `template<GPUPrimitive primitive>`. Dithering is now `if constexpr (primitive == Line) ... else if constexpr (primitive == Rectangle) cmd->dither_enable = false;` instead of calling `rc.IsDitheringEnabled()` and `rc.IsTexturingEnabled()` at runtime. Likewise, PGXP's `CPU_BITWISE` takes the funct as a template parameter, so the AND-mask precision special case is only compiled into the AND instantiation. 4144c1e09 2026-06 “GPU: Eliminate redundant type comparions in FillDrawCommand()” · 9ec8c968b 2026-11 “CPU/PGXP: Make CPU_BITWISE funct a template parameter”

### duckstation-281 — Bake decode-time flags into specialized handlers when the caller resolves them once
Lens: compiler-build · Source: DuckStation
Principle: If a handler is looked up once (by a JIT, or when a pipeline is built) and then called many times, return a variant with the instruction's mode bits already fixed. Shift amounts and clamp limits become constants, so nothing is decoded or branched on in the hot call.
Why it is faster: —
Applies elsewhere: query engines that pick a comparator specialized per column type at plan time, regex engines that compile flags into the matcher, and shader permutation systems.
Watch out: —
Seen in DuckStation: The recompilers already resolved the GTE handler once per instruction. `GTE::GetInstructionImpl` now returns a version specialized on the `sf` (shift) and `lm` (clamp) bits, and the `Execute_*` bodies are `ALWAYS_INLINE` so the constants propagate into them. d6d1373c5 2026-10 “GTE: Specialize instruction handlers on sf/lm”

### duckstation-282 — Force-inline small helpers so values stay in registers, and push the rare path out of line
Lens: compiler-build · Source: DuckStation
Principle: When a hot routine calls several small helpers that share state, every non-inlined call forces that state through memory. Force-inline the helpers (constructors included, since compilers outline those at -Os). Mark the cold optional work `NEVER_INLINE` so it doesn't bloat the fast body.
Why it is faster: —
Applies elsewhere: math-vector wrappers in game engines, iterator adaptors in hot loops, and the "fast path inline, slow path in a separate function" split used in allocators like mimalloc.
Watch out: —
Seen in DuckStation: GTE's triple variants (RTPT, NCT and others) made up to nine calls per instruction, with `FLAG` and IR1–3 round-tripping through memory each time. `MulMatVec`, `RTPS`, `NCS` and the others are now `ALWAYS_INLINE`, and the PGXP part of RTPS moved to a `NEVER_INLINE RTPS_PGXP()`. In the software rasterizer, the `PixelVectors` and `TriangleVectors` constructors got `ALWAYS_INLINE_RELEASE`, because clang outlined the textured ones and spilled every vector field. 769f4330b 2026-10 “GTE: Force inline matrix/lighting/projection helpers” · c94a799f5 2026-10 “GPU/SW: Force inline vector constructors”

### duckstation-283 — Store a fixed-capacity container inline instead of behind a heap pointer
Lens: memory-layout · Source: DuckStation
Principle: A container whose capacity is a small compile-time constant can live inside its owning struct. Every access then saves a pointer load, and the data sits in the same cache lines as the code's other state. Shrinking the capacity to the real worst case keeps the owner compact.
Why it is faster: —
Applies elsewhere: `SmallVector`/`absl::InlinedVector`, fixed ring buffers in RTOS message queues, and inline small-string buffers.
Watch out: —
Seen in DuckStation: The GPU command FIFO went from `HeapFIFOQueue<u64, 4096>` to `InlineFIFOQueue<u64, 256>`, moved to the end of the `Locals` struct. The commit message: “Avoid the indirection.” 3dd7497f6 2026-05 “GPU: Use InlineFIFOQueue”

### duckstation-284 — Reorder fields and align instances, not types, to remove padding
Lens: memory-layout · Source: DuckStation
Principle: Group fields by size, largest first and bools together, so the compiler inserts no alignment holes. If something needs cache-line isolation, put `alignas` on the single global instance, not on the struct type, so that copies and arrays of the type don't inherit 64-byte padding. Smaller hot state takes fewer cache lines.
Why it is faster: —
Applies elsewhere: Go and Rust struct field ordering for smaller allocations, packed ECS components, and `pahole`-driven cleanups in the Linux kernel.
Watch out: —
Seen in DuckStation: `struct ALIGN_TO_CACHE_LINE SPUState` became `ALIGN_TO_CACHE_LINE static SPUState s_state`, and `CoreThreadState` got the same change. The SPU `Voice` struct (24 instances, walked every sample) was reordered so its bools and phase come first, and the large sample arrays moved to the end. `Settings` packs all its bitfields together. `TimingEvent::m_name` went from `std::string_view` (16 bytes) to `const char*`. 9fe36d9c0 2026-06 “Misc: Reduce unnecessary struct padding” · de332d542 2026-10 “SPU: Eliminate padding in Voice structure” · 0ff4c50bf 2026-10 “Settings: Pack all bitfields together” · 15ad0024d 2026-06 “TimingEvent: Use const char* for name instead of string_view” · 77fe65995 2026-06 “GPU: Inline draw mode into state struct”

### duckstation-285 — Keep optional-value flags in a shared bitmask
Lens: memory-layout · Source: DuckStation
Principle: `std::optional<T>` adds a bool that usually rounds up to `alignof(T)`, so a struct with many optional fields doubles in size. Store all the "has value" bits in one integer and keep each payload bare. Many fields then cost one bit apiece.
Why it is faster: —
Applies elsewhere: protobuf's hasbits, null bitmaps in columnar formats (Arrow, Parquet), and sparse config/override records.
Watch out: —
Seen in DuckStation: `common/packed_optional.h` adds `PackedOptional<T, Bit>`, whose accessors take the owner's mask: `a.set(mask, 123); if (a.has_value(mask)) ...`. The game database's per-game overrides moved to it, and a `PackedOptionalArray` followed. 507e24952 2026-10 “Common: Add PackedOptional class”

### duckstation-286 — Deserialize and convert in place instead of through a temporary heap buffer
Lens: allocation · Source: DuckStation
Principle: When the source bytes are already in memory, decode straight from them into the destination. Don't allocate a scratch array, copy into it, and then copy again. Doing so removes an allocation and a full pass over the data.
Why it is faster: —
Applies elsewhere: zero-copy parsers (FlatBuffers, Cap'n Proto, simdjson's on-demand API), and `string_view`-based text decoders.
Watch out: —
Seen in DuckStation: `StateWrapper`'s FIFOQueue reader did `T* temp = new T[size]; DoArray(temp, size); ... delete[] temp;`, which was left over from when save states were stream-backed. It now bounds-checks and calls `data->PushRange(reinterpret_cast<const T*>(&m_data[m_pos]), size)` directly. Memory-card title decoding stopped copying into a NUL-padded stack buffer and `malloc`ing the UTF-8 result. It now converts from a `string_view`. c42082c4b 2026-08 “StateWrapper: Avoid heap allocation in FIFOQueue read” · 12a4a2648 2026-08 “Util: Avoid intermediate memory allocation in Shift-JIS conversion”

### duckstation-287 — Copy a ring buffer as at most two contiguous memcpys
Lens: memory-layout · Source: DuckStation
Principle: A wrapped ring or circular buffer is just two contiguous spans. Copying each span with `memcpy` replaces a per-element loop that masks the index on every step, and turns the copy into a vectorized, bandwidth-bound operation.
Why it is faster: —
Applies elsewhere: audio ring buffers, network receive rings (copying a packet that wraps), and log buffers being flushed to disk.
Watch out: —
Seen in DuckStation: `StateWrapper` now saves a trivially copyable FIFO by doing a `memcpy` of `GetContiguousSize()` elements and then handling the remainder. DMA's forward-wrapping transfers to and from RAM went from a per-word `(address + increment) & mask` loop to two bulk copies: `[address, end)` and `[0, rest)`. 865fd9e57 2026-08 “StateWrapper: Save contiguous FIFOQueue bytes in one memcpy()” · c37f3901e 2026-10 “DMA: Copy forward wrapping transfers in contiguous spans”

### duckstation-288 — Hand a consumer the whole buffer instead of calling it once per word
Lens: algorithms · Source: DuckStation
Principle: If a producer loops over N items and makes a cross-module call for each, the per-call overhead and lost optimization dominate. Normalize the input into one contiguous span, using a temp buffer only in the rare wrap case, and pass it in a single call. The consumer can then run a tight loop.
Why it is faster: —
Applies elsewhere: `writev` instead of many `write`s, batched inserts into a database, and bulk `postMessage` of a typed array instead of per-element messages.
Watch out: —
Seen in DuckStation: For the GPU channel, DMA used to loop `GPU::DMAWrite(address, value)` once per word, with separate GPU-dump handling inline. Now the GPU channel shares the generic wrap-to-temp-buffer path, and the DMA code makes one call: `GPU::DMAWrite(src_pointer, address, increment, word_count)`. 4e0374ef8 2026-05 “GPU: Slightly faster path for DMA writes”

### duckstation-289 — Generate arithmetic sequences with vector adds instead of a dependent scalar chain
Lens: simd · Source: DuckStation
Principle: Filling memory with values where each depends on the previous one (`next = prev - 4`) forms a serial chain. Compute the values from the index instead (`base + 4*i`). Each vector then holds several consecutive values, and the loop steps by a constant vector add, with no masking.
Why it is faster: —
Applies elsewhere: initializing a free-list or linked-list arena, building index buffers for a grid mesh, and `std::iota`-style fills.
Watch out: —
Seen in DuckStation: The ordering-table-clear DMA writes a reversed linked list into RAM. When it doesn't wrap, it now fills in ascending address order, storing `indices.sll32<2>().add32(base_address)` eight words per iteration, with `indices += 8`. The masked per-link loop is kept for the wrapping case. d7120b22a 2026-10 “DMA: Optimize contiguous ordering table clears”

### duckstation-290 — Guard expensive calls with the cheapest condition that makes them pointless
Lens: branching-fast-paths · Source: DuckStation
Principle: Before calling an eligibility check or bookkeeping hook, test the trivial condition under which its result can't matter. Examples: the request is being cleared, the destination is a write-ignored register, or the feature is off. One compare saves a call on the common path.
Why it is faster: —
Applies elsewhere: `if (log.isDebugEnabled())` before formatting a log message, skipping change notifications when the value is unchanged, and short-circuiting validators on null input.
Watch out: —
Seen in DuckStation: `DMA::SetRequest` now does `if (request && CanTransferChannel(...))`, since a deasserted request can't start a transfer. The interpreter only tests for the BIOS A0/B0 syscall trampolines `if (g_settings.bios_tty_logging)`. PGXP hooks are skipped when `inst.r.rd == Reg::zero`. 859d7e875 2026-10 “DMA: Skip transfer eligibility checks when clearing requests” · fc48940aa 2026-10 “CPU: Only intercept TTY syscalls in interpreter with BIOS TTY logging” · ca0dfa540 2026-11 “CPU/PGXP: Don't call PGXP for instructions that write $zero”

### duckstation-291 — Invalidate only the affected cache entries instead of flushing everything
Lens: laziness · Source: DuckStation · Also: ares
Principle: When a small memory range changes, invalidate only the cache lines whose tags match that range. A full flush makes everything miss and refill, while the targeted version costs a handful of tag compares.
Why it is faster: —
Applies elsewhere: CDN purge by URL instead of a full purge, per-key cache invalidation in Redis-backed apps, and `__builtin___clear_cache` on just the patched range.
Watch out: —
Seen in DuckStation: `CPU::InvalidateICacheAt` and `InvalidICacheRange` walk the words of a range and reset `icache_tags[line]` only when `(tag & ICACHE_TAG_ADDRESS_MASK) == GetICacheTagForAddress(addr)`. Cheats that write assembly use them in place of `ClearICache()`. b4834b982 2026-06 “CPU: Add targeted icache invalidation” • Also seen in ares (ares-079): On every RAM write the PS1 recompiler's `invalidate()` used to do `pools[address >> 8] = nullptr`, which discarded all 64 blocks in the 256-byte page. It now clears only `pool->blocks[address >> 2 & 0x3f]`. 5bcef4741 2024-12 “ps1: invalidate single blocks rather than entire pools (recompiler)”

### duckstation-292 — Use 16-bit multiply-add when value bounds prove it cannot overflow
Lens: simd · Source: DuckStation
Principle: A dot product of 16-bit samples and 16-bit coefficients maps onto `pmaddwd`/`vmlal`, which does two multiplies and an add per 32-bit lane. Prove the accumulator stays within s32 range, for example from the sum of absolute coefficients. You can then pack the coefficients as s16 and do half the multiplies on twice the data per instruction, with bit-exact results.
Why it is faster: —
Applies elsewhere: FIR/IIR audio filters, int8/int16 neural-network inference kernels, and image convolution in libjpeg-turbo or libyuv.
Watch out: —
Seen in DuckStation: SPU Gaussian interpolation used to do four scattered coefficient loads and four scalar multiplies. With a phase-major coefficient table, it is now `loadl(samples).madd_s16(loadl(coeffs)).xy().addv_s32() >> 15` (about 2.5% when SPU-bound). The reverb upsampling FIR went from five `mul32l` passes on s32 coefficients to three `madd_s16` ops; the coefficients' absolute sum is 31142. XA-ADPCM resampling was vectorized the same way, replacing per-tap scalar zig-zag table sums. 690ead7ee 2026-09 “SPU: Vectorize Gaussian interpolation” · fd8574a84 2026-10 “SPU: Pack reverb FIR coefficients into 16-bit lanes” · cf71052c4 2026-10 “CDROM: Vectorize XA resampling”

### duckstation-293 — Walk arrays by pointer and consume bitmasks by shifting
Lens: algorithms · Source: DuckStation
Principle: Indexing `array[i]` into large structs needs a multiply by the stride, and testing `(mask >> i) & 1` needs a variable shift every iteration. Advance a pointer and shift each mask right by one per iteration instead. Values the next iteration needs (like the previous element's output) should be passed in a register, not re-read through the index.
Why it is faster: —
Applies elsewhere: iterating set bits in bitboards, mixers that walk channel structs, and interrupt-pending scanning in firmware.
Watch out: —
Seen in DuckStation: The SPU voice loop now walks `Voice*` directly and consumes the noise, pitch-modulation and reverb masks one bit at a time. It passes the previous voice's volume into `SampleVoice` explicitly, which avoids the per-voice `* 148` address multiply. The commit reports about 2% when SPU-bound. 2781502e0 2026-09 “SPU: Streamline per-voice flag processing”

### duckstation-294 — Hoist invariant loads and flag checks out of short loops
Lens: algorithms · Source: DuckStation · Also: Flycast
Principle: If a loop re-reads the same state, or re-tests the same enable flag, on every iteration, the compiler often can't hoist it because of possible aliasing. Load it into locals once, or move the whole loop under the flag test. Fewer memory ops and branches per iteration, and nothing runs when the work is disabled.
Why it is faster: —
Applies elsewhere: render loops that check `if (debugEnabled)` per pixel, and any loop body that calls through `this->config.x`.
Watch out: —
Seen in DuckStation: The four SPU capture-buffer writes now go through one helper that loads the position, IRQ address and "IRQ triggerable" once. The reverb downsampling FIR, which used to run every sample before `if (SPUCNT.reverb_master_enable)` was checked per channel, now runs only inside that check. 5a7a301a7 2026-09 “SPU: Consolidate capture buffer writes” · 5a5bb3db4 2026-10 “SPU: Extract reverb-master-enable out of loop” • Also seen in Flycast (flycast-068): `glHint(GL_GENERATE_MIPMAP_HINT, ...)` moved from each texture upload in `gltex.cpp` to `gles_init`, and from NICEST to FASTEST. `ReadRTTBuffer` reads `FB_W_CTRL.fb_packmode`, `(fb_kval & 0x80) << 8` and `fb_alpha_threshold` into locals once per frame, not once per pixel. In `tryfit`, the fog curve fit computes `log(x[i])` once instead of four times. e73d11d0a 2018-05 “glHint nicest->fastest, and just once” · 43c125c20 2018-05 “Better texture cache: avoid duplicate...” · 35d3bd98d 2018-04 “Disable fog table logging. Small optimization”

### duckstation-295 — Process packed sub-word items in pairs to halve loop overhead
Lens: algorithms · Source: DuckStation
Principle: When each input byte holds two 4-bit items, decode both per iteration. That halves the loop-control and byte-load work, and gives the CPU two independent instruction streams to overlap.
Why it is faster: —
Applies elsewhere: IMA-ADPCM and other nibble codecs, hex decoding, and 4-bit quantized weights in LLM inference.
Watch out: —
Seen in DuckStation: `SPU::Voice::DecodeBlock` used to compute `(data[i/2] >> ((i%2)*4)) & 0xF` for each of the 28 samples. It now loops over the 14 bytes and emits the low and high nibble in sequence, with the filter coefficients hoisted to locals. a24350c47 2026-09 “SPU: Decode ADPCM nibbles in pairs”

### duckstation-296 — Special-case inputs where most of the work collapses to zero
Lens: algorithms · Source: DuckStation
Principle: Real data is often skewed toward degenerate cases: zero filter coefficients, or only the low-frequency corner of a transform being non-zero. Detect that cheaply up front and run a path that skips the terms known to be zero, but measure the hit rate first.
Why it is faster: —
Applies elsewhere: sparse-matrix fast paths, JPEG decoders' DC-only IDCT, and skipping the alpha blend for fully opaque pixels.
Watch out: —
Seen in DuckStation: When both ADPCM predictor coefficients are zero, the block decodes as `(nibble << 12) >> shift` with no feedback. The MDEC IDCT detects blocks whose coefficients all lie in the top-left 4×4. For those, the last four rows of the first pass and the upper half of every row are skipped. Such blocks are about half of all blocks, and over 80% in some FMVs, and are about 1.9× faster. 5b37f3d91 2026-10 “SPU: Fast-path ADPCM blocks without predictors” · 16dc60f97 2026-10 “MDEC: Add IDCT fast path for low frequency blocks”

### duckstation-297 — Skip computing an output that will be multiplied by zero
Lens: laziness · Source: DuckStation
Principle: If a stage's result only feeds a gain or weight that is currently zero, skip the stage. Keep updating its internal history so it resumes exactly when the gain comes back.
Why it is faster: —
Applies elsewhere: audio graphs that bypass muted buses, renderers that skip passes whose blend weight is zero, and ML inference that skips masked branches.
Watch out: —
Seen in DuckStation: `SPU::ProcessReverb` now returns 0 for a channel without running the 20-tap upsampling FIR when `vLOUT`/`vROUT` is zero. The feedback and resample history are still updated above that point. 16fa96940 2026-10 “SPU: Skip reverb output filtering at zero volume”

### duckstation-298 — Check a two-sided range with one unsigned compare, and move rare fixups into that branch
Lens: branching-fast-paths · Source: DuckStation
Principle: `(u64)(v - MIN) > (u64)(MAX - MIN)` tests both bounds at once, because values below MIN wrap to huge unsigned numbers. If a normalization (for example, sign-extension) is a no-op for in-range values, do it only inside the out-of-range branch. This shortens the dependency chain on the common path.
Why it is faster: —
Applies elsewhere: array bounds checks in JITs (V8, HotSpot), `isdigit` as `(unsigned)(c - '0') < 10`, and saturating fixed-point DSP code.
Watch out: —
Seen in DuckStation: `GTE::CheckMACOverflow` had separate `value < MIN` and `value > MAX` branches, each setting a flag through a constexpr if-chain. It became `IsMACOutOfRange`, which uses the biased unsigned compare and computes the flag bit position. `SignExtendN<44>` now runs only when out of range. 953285bf5 2026-10 “GTE: Only sign extend MAC results on overflow”

### duckstation-299 — Factor algebra to cut multiplies
Lens: algorithms · Source: DuckStation
Principle: Expressions like the 2D cross product `x0*y1 + x1*y2 + x2*y0 - x0*y2 - x1*y0 - x2*y1` regroup into `x0*(y1-y2) + x1*(y2-y0) + x2*(y0-y1)`. That is half the multiplies. Check that the intermediates can't overflow and that any range checks only apply to the final sum.
Why it is faster: —
Applies elsewhere: triangle orientation and area tests in geometry libraries, polynomial evaluation by Horner's rule, and Karatsuba multiplication.
Watch out: —
Seen in DuckStation: GTE `NCLIP` (the back-face cull test) now computes the factorized form, using three 64-bit multiplies instead of six. MAC0 and its flags are unchanged. 9a5bb3d85 2026-10 “GTE: Factorize NCLIP to use three multiplies”

### duckstation-300 — Use invariants to pick a cheaper equivalent operation
Lens: other · Source: DuckStation
Principle: Signed division by a power of two needs a rounding fixup for negatives, but if negatives get clamped to zero afterwards, an arithmetic shift gives the same result. Likewise, if a bit is known to be clear at that point, OR it in instead of clearing and re-inserting it.
Why it is faster: —
Applies elsewhere: color quantization (`>> 3` for 8→5-bit), flag-register emulation, and compilers' strength-reduction passes once value ranges are known.
Watch out: —
Seen in DuckStation: GTE `ORGB` now uses `std::clamp(REGS.IR1 >> 7, 0, 0x1F)` instead of `/ 0x80`. `FLAGS::UpdateError()` became `bits |= (bits & 0x7F87E000) ? 0x80000000 : 0`, because the error bit is always clear when it is computed. 79161b097 2026-10 “GTE: Use shift for IRGB/ORGB conversion” · 5b3bcf139 2026-10 “GTE: Simplify FLAG error bit update”

### duckstation-301 — Go branchless where the outcome is data-dependent and unpredictable
Lens: branching-fast-paths · Source: DuckStation
Principle: A saturation branch costs nothing when it's almost always taken one way, and about 15 cycles each time it mispredicts. Where the data makes it a coin flip, clamp unconditionally (min/max or cmov) and derive the flag from `clamped != original`. Keep the branchy version where the branch is predictable, since the branchless one costs more there.
Why it is faster: —
Applies elsewhere: branchless binary search, image clamping in shaders and codecs, and sorting networks.
Watch out: —
Seen in DuckStation: For the light-matrix multiply with `lm` set, every normal that faces away from a light clamps to zero, so mispredictions are frequent. `TruncateAndSetMACAndIRBranchless` does `clamped = std::clamp(v, lm ? 0 : MIN, MAX); FLAG.bits |= (clamped != v) << FLAG_SHIFT;`, and it is used only there. Elsewhere, `WriteRegDelayed` was rewritten as a ternary to invite a cmov. e43d2f81a 2026-10 “GTE: Use branchless saturation for light matrix multiply” · 18f581ef9 2026-06 “CPU: Use ternary in CPU::WriteRegDelayed()”

### duckstation-302 — Vectorize a separable 2D transform across rows and transpose between passes
Lens: simd · Source: DuckStation
Principle: Computing a matrix transform one output at a time, as a dot product per element, wastes lanes on horizontal reductions. Broadcast each input pair and multiply-add it against whole matrix columns so that all 8 outputs of a row come out at once. Then transpose in registers for the second pass instead of scattering scalars.
Why it is faster: —
Applies elsewhere: JPEG/MPEG IDCT and DCT, 2D FFTs, and separable Gaussian blur.
Watch out: —
Seen in DuckStation: `MDEC::IDCTRow` used to return one `s16` per call from a `madd_s16().addp_s32()` dot product. It now returns a whole `GSVector4i` row, built from `row.xxxx()/yyyy()/zzzz()/wwww()` times eight matrix vectors. The high and low halves are combined with an overflow-free average trick that keeps the rounding exact. The commit reports about 2.4× faster on x64. 8fb990710 2026-10 “MDEC: Vectorize IDCT across matrix rows”

### duckstation-303 — Compute shared terms once per larger unit
Lens: algorithms · Source: DuckStation
Principle: When several outputs share an input at lower resolution, as with chroma subsampled 2:1 under four luma blocks, process the whole unit together and compute the shared terms once. Values already known to be narrow can also use narrower (16-bit) multiplies.
Why it is faster: —
Applies elsewhere: YUV420→RGB converters, mip-chain generation, and any per-group precompute in a batch job.
Watch out: —
Seen in DuckStation: `YUVToRGB_New` used to be called four times per macroblock (one per luma block), recomputing the Cr/Cb terms each time. It now converts the whole 16×16 macroblock: chroma terms once per chroma row, multiplies split to fit 16 bits, and the 9-bit sign extension folded into one shift. About 1.9× faster. d5bfad7d7 2026-10 “MDEC: Convert whole macroblock in YUVToRGB_New()”

### duckstation-304 — Index with the natural element width instead of gathering words and extracting fields
Lens: simd · Source: DuckStation
Principle: If each index is exactly one byte, read the bytes directly and use them as table indices. Gathering the containing 16-bit word into a vector, then extracting and shifting each lane, is slow, especially without variable per-lane shifts.
Why it is faster: —
Applies elsewhere: palette and LUT image formats, byte-coded dictionary decoding in columnar stores, and font glyph-index lookup.
Watch out: —
Seen in DuckStation: `GatherCLUT8Vector` computes byte offsets `y*2048 + x` and does `g_gpu_clut[vram[off]]` per lane. Before, the code gathered VRAM words, then did 12 extracts and 4 scalar shifts per 4 pixels on SSE4. The commit reports 13–25% faster small 8-bit textured triangles, and 40–65% for large triangles and sprites. 65c554c44 2026-10 “GPU/SW: Read 8-bit palette indices as bytes”

### duckstation-305 — Hold a 16-entry table in registers and look it up with a byte shuffle
Lens: simd · Source: DuckStation
Principle: A 16-entry table of 16-bit values fits in two registers once split into low and high bytes. `pshufb`/`tbl` then looks up every lane in parallel in one instruction, with no trips through scalar registers or memory.
Why it is faster: —
Applies elsewhere: hex encoding and decoding, base64 SIMD codecs, popcount by nibble LUT, and 4-bit quantization dequantize kernels.
Watch out: —
Seen in DuckStation: For 4-bit textures, `PixelVectors` now preloads `clut_lo`/`clut_hi` from `g_gpu_clut[0..15]`, and `LookupCLUT4Vector` uses `shuffle8` on both and interleaves the results. Without `srlv`, the index is moved to the top of the word by a multiply. 5–20% faster for 4-bit primitives. 5542670f4 2026-10 “GPU/SW: Use shuffle for 4-bit palette lookups”

### duckstation-306 — Replace a gather with one contiguous load when the access pattern is sequential
Lens: simd · Source: DuckStation
Principle: Gathers cost roughly one load per lane. If you can prove the addresses for a vector are consecutive, for example when coordinates step by one and no wrap or remap applies, issue a single vector load. Check per vector, and fall back to the gather otherwise.
Why it is faster: —
Applies elsewhere: sprite blitters, texture sampling in software renderers for axis-aligned quads, and database scans that switch from index lookups to sequential reads.
Watch out: —
Seen in DuckStation: For sprites, texture coordinates increase by 1 per pixel. Unless the texture window rearranges them, a 16-bit texture becomes one load and a 4-bit texture becomes one load plus the palette shuffle. 8-bit textures still gather. Sampling was split out of `ShadePixel()` so it can be bypassed. Reported as 25% faster 16-bit sprites, 20–45% faster 4-bit sprites, and up to 10% slower small 8-bit sprites. 96ecf13a0 2026-10 “GPU/SW: Load sequential texels for sprites”

### duckstation-307 — Drop per-iteration tests that a pre-pass already guarantees
Lens: algorithms · Source: DuckStation
Principle: If inputs are clipped or validated before a loop, re-testing the same condition per element can never change the result. Remove it, and leave a comment saying why it's safe.
Why it is faster: —
Applies elsewhere: bounds checks after an explicit length check (which compilers can't always prove), and re-validating already-sanitized request fields deep in a handler.
Watch out: —
Seen in DuckStation: `DrawSpan` masked each vector against `clip_left`/`clip_right` and skipped all-masked vectors. Spans are already clipped to the drawing area and never empty, so only the span-width mask remains. 4–17% faster triangles. 84dcb202d 2026-10 “GPU/SW: Remove redundant clip test for triangle spans”

### duckstation-308 — Take the slow overlap-safe path only when the regions really overlap
Lens: algorithms · Source: DuckStation
Principle: Like `memmove` versus `memcpy`, the reverse-copy path is only needed when source and destination overlap in the direction of travel. A loose condition such as "destination is to the right" pushes many safe copies onto the slow, non-vectorized path.
Why it is faster: —
Applies elsewhere: `memmove` implementations, 2D blitters and canvas `drawImage` onto itself, and in-place array shifts.
Watch out: —
Seen in DuckStation: `CopyVRAMImpl` went reverse whenever `src_x < dst_x`. Rows are always copied top to bottom, so the condition is now `same row && src_x < dst_x && dst_x < src_x + width`. A 320×240 copy to the right went from 40 µs to 2.4 µs. 3054fb0f0 2026-10 “GPU/SW: Only copy VRAM in reverse when necessary”

### duckstation-309 — Compile one hot file at a higher ISA level and select it at runtime
Lens: platform · Source: DuckStation
Principle: Keep the baseline binary portable, but build the hot kernel a second time with `-mavx2` (or `/arch:AVX2`) in its own namespace. Exclude that file from precompiled headers so the flags can't leak into shared inline functions (ODR). Pick the implementation once at startup from a CPUID check.
Why it is faster: —
Applies elsewhere: simdjson and zlib-ng dispatch, glibc IFUNC string routines, and FFmpeg's per-CPU function pointers.
Watch out: —
Seen in DuckStation: On x64, `gpu_sw_rasterizer_avx2.cpp` is compiled with AVX2 and `SKIP_PRECOMPILE_HEADERS`. `SelectImplementation()`, previously disabled with `&& 0`, now picks `AVX2` when `cpuinfo_has_x86_avx2()` is true, and it switches the Fill/Write/Copy VRAM functions too. 22c78a107 2026-10 “GPU/SW: Enable AVX2 rasterizer on x64”

### duckstation-310 — Don't reorder a priority queue for an update that ends where it started
Lens: algorithms · Source: DuckStation
Principle: Some "run now" operations move an event to the front of a sorted queue and then reschedule it back to roughly the same spot, which costs two linked-list walks for nothing. Add a variant that runs the callback without touching the event's queue position, and let the callback reschedule only if needed.
Why it is faster: —
Applies elsewhere: timer wheels and heap-based schedulers where "touch" operations shouldn't re-heapify, and LRU caches that skip move-to-front for already-hot entries.
Watch out: —
Seen in DuckStation: Every timer register access called `sysclk_event.InvokeEarly()`. Because that event's interval is one tick, it moved to the head of the queue each time, only for the callback to put it back. `TimingEvent::InvokeEarlyNoReschedule()` runs the pending ticks and leaves the event's position alone. It is now used for the timers, and the commit series also moved the GPU CRTC/command events and the SPU transfer event to it. 5b536b769 2026-10 “TimingEvents: Add InvokeEarlyNoReschedule()”

### duckstation-311 — Answer reads by computing a projected value instead of advancing the simulation
Lens: laziness · Source: DuckStation
Principle: If a counter grows linearly between events, a read can return `counter + elapsed * rate` without mutating state. Fall back to a real update only when the projection crosses a threshold that would cause side effects, such as an IRQ.
Why it is faster: —
Applies elsewhere: token-bucket rate limiters that compute tokens lazily on request, lazily evaluated game timers, and progress bars that interpolate between updates.
Watch out: —
Seen in DuckStation: `Timers::UpdateForRead()` works out `cs.counter + ticks_since_last_execution` (with the sysclk/8 carry) and returns it if it stays below both the target and overflow. Only otherwise does it call `InvokeEarlyNoReschedule()`. Reads of GPU-clocked or paused timers no longer touch the sysclk timers at all. 436c1e61e 2026-10 “Timers: Avoid updating counters when reading registers”

### duckstation-312 — Early-out when an increment doesn't cross any threshold
Lens: branching-fast-paths · Source: DuckStation
Principle: The slow path in an accumulate-then-check routine is only needed when the new value crosses a boundary. A single combined compare lets the common case store the sum and return immediately.
Why it is faster: —
Applies elsewhere: buffered writers that only flush when crossing the buffer size, and metrics counters that only emit at bucket boundaries.
Watch out: —
Seen in DuckStation: `Timers::AddTicks` now does `if (new_counter < TIMER_OVERFLOW && (new_counter < cs.target || cs.counter >= cs.target)) { cs.counter = new_counter; return; }`. Before, `CheckForIRQ()` ran for all three timers on every update. b9d4fc917 2026-10 “Timers: Skip IRQ check when not reaching target or overflowing”

### duckstation-313 — Predict when the next interesting event can happen instead of stepping one unit at a time
Lens: laziness · Source: DuckStation
Principle: Fine-grained stepping "in case something happens" wastes most iterations. Compute the earliest step at which the event could possibly fire (a conservative bound is enough), and schedule the next wakeup there. Recompute whenever an input that affects the prediction changes.
Why it is faster: —
Applies elsewhere: tickless kernels (NO_HZ), discrete-event simulators, and game servers that schedule the next AI think time instead of ticking every frame.
Watch out: —
Seen in DuckStation: With the SPU RAM IRQ enabled, the SPU used to run one sample per event. It now stops doing that while the IRQ flag is already set. It also predicts the next sample that could touch the IRQ address, given voice addresses, rates and capture position, and calls `UpdateEventInterval()` from every relevant register write. The commit reports up to about 15% in SPU-bound games. a15efdc57 2026-10 “SPU: Don't run every sample while the IRQ flag is set” · c893d47ba 2026-10 “SPU: Predict the next sample that can trigger a RAM IRQ”

### duckstation-314 — Pick instruction forms that encode the operation directly
Lens: jit-interpreters · Source: DuckStation
Principle: In a code generator, many patterns have a shorter encoding. Test-bit-and-branch can check a sign bit, rotated or inverted immediates avoid materializing constants, reverse-subtract takes the constant on the left, and the zero register stands in for 0. Where the host's native semantics already match the guest's edge cases, drop the explicit checks. Each saved instruction is paid back on every execution.
Why it is faster: —
Applies elsewhere: peephole passes in LLVM/V8/LuaJIT backends, WASM baseline compilers, and hand-written ARM assembly in codecs.
Watch out: —
Seen in DuckStation: ARM64: `bltz`/`bgez` became `tbnz`/`tbz rs, #31` instead of `cmp` + `b.cond`, and `negu` uses `wzr`. ARM32: constants that fit a rotated 8-bit immediate use one `mov`/`mvn` instead of `movw`+`movt`, and `const - reg` uses `rsb`. Since `sdiv`/`udiv` never trap and return 0 for x/0, MIPS `div` only fixes up `lo` when dividing by zero, which removes two compares and branches. 6de56e0ef 2026-10 “CPU/Recompiler: Use tbz/tbnz for bltz/bgez on ARM64” · 383fa0235 2026-10 “CPU/Recompiler: Use rotated/inverted immediates for ARM32 constants” · f7a237e49 2026-10 “CPU/Recompiler: Avoid constant load for const - reg on ARM” · 10ecadbfd 2026-10 “CPU/Recompiler: Use native divide semantics on ARM”

### duckstation-315 — Place co-accessed fields adjacently and load them as a pair
Lens: memory-layout · Source: DuckStation
Principle: If two fields are always read together, put them next to each other and enforce it with `static_assert(offsetof(...))`. Then a single paired load (`ldp`, or a 64-bit load) fetches both.
Why it is faster: —
Applies elsewhere: (head, tail) indices in lock-free queues, (ptr, len) slices, and interpreter state (pc, sp) loaded on entry.
Watch out: —
Seen in DuckStation: The event test at the end of every recompiled block (`if (pending_ticks >= downcount)`) now emits `ldp RWARG2, RWARG1, [&g_state.downcount]` on ARM64 instead of two `ldr`s. 9f171d3c6 2026-10 “CPU/Recompiler: Load downcount and pending ticks as a pair on ARM64”

### duckstation-316 — Specialize generated validation code on the data's size
Lens: jit-interpreters · Source: DuckStation · Also: Flycast
Principle: When generated code compares a region against a known snapshot, small regions can compare against immediates baked into the code, which needs no second pointer or memory read. For larger ones, overlap the last vector compare instead of finishing with a scalar tail loop.
Why it is faster: —
Applies elsewhere: `memcmp` implementations' overlapping-tail trick, and JIT-ed guard checks on hidden classes or shapes.
Watch out: —
Seen in DuckStation: The x64 self-modifying-code check for a block (`GenerateBlockProtectCheck`) now emits `mov rarg2, imm64; cmp qword [ram], rarg2` plus a `cmp dword [ram+8], imm32` for blocks under 16 bytes. Larger blocks use `pcmpeqd` with an overlapped final 16 bytes. On ARM64, the per-chunk differences are accumulated so the check ends in a single branch. 6a45a1cc0 2026-10 “CPU/Recompiler: Improve x64 block protection check codegen” · 49169d694 2026-10 “CPU/Recompiler: Improve ARM64 block protection check codegen” • Also seen in Flycast (flycast-060): The block check in `rec_cpp.cpp` started as a naive `memcmp(ptr, &code[0], size)`. It became `opcode_check_block<sz>` with specializations for 4, 6 and 8 bytes that compare `u32` and `u16` words directly. The ARM64 `CheckBlock` emits `Ldr x10,[x9]; Mov x11, imm64; Cmp` per 8 bytes. x64 gained an emitted check (`cmp dword[rax], edx; jne ngen_blockcheckfail`), and ARM32 now checks the whole block instead of only its first 4 bytes. 2489a7258 2018-08 “rec-cpp block checks naive implementation” · 2cc55dfce 2018-08 “rec-cpp: optimize small block checks” · c430a7906 2018-08 “Block checks for rec-cpp” · 3d8b01c51 2019-01 “... Block check optimization” · ad05c35a7 2018-06 “Implement CheckBlock for x64” · 87a8fcb6f 2018-11 “ARM dynarec: do a block check on the entire block”

### duckstation-317 — Size lookup tables to the active configuration, in one allocation
Lens: allocation · Source: DuckStation
Principle: Tables sized for the largest possible configuration waste memory and cache. Allocate them for what is actually installed or enabled, with related tables in a single block, and reallocate only when the configuration changes.
Why it is faster: —
Applies elsewhere: page tables sized to physical RAM, hash tables sized from expected cardinality, and per-feature buffers allocated only when the feature is on.
Watch out: —
Seen in DuckStation: The code-cache LUTs (`lut_code_pointers`, `lut_block_pointers`) were two `unique_ptr` arrays. They are now one allocation covering at least 8 MB of RAM, with EXP1 included only when a PIO device is enabled. `ReallocateLUTs()` runs when a save state changes the RAM size. 042261b59 2026-11 “CPU/CodeCache: Size LUTs to installed RAM and PIO device”

### duckstation-318 — Compile independent shaders and pipelines on a worker pool
Lens: concurrency · Source: DuckStation
Principle: When startup work is a large set of independent compile jobs, submit them to a thread pool and track completion with an atomic counter, with an atomic abort flag on the first error. Fall back to serial when the backend isn't thread-safe. Grow the pool on demand only when every worker is busy.
Why it is faster: —
Applies elsewhere: parallel PSO creation in game engines, `make -j`, and asset pipelines that transcode textures in parallel.
Watch out: —
Seen in DuckStation: `ShaderCompileProgressTracker::CompileShader`/`CompilePipeline` now call `Host::QueueAsyncTask` when `features.thread_safe_shader_compile` is set. Workers check `m_tasks_abort` and bump `m_tasks_completed`. The `TaskQueue` spawns another thread when `m_threads_busy == m_threads.size() && m_threads.size() < m_max_threads`. 94e5f08f1 2026-09 “GPU/HW: Use multi-threaded shader compilation” · 74a743a5e 2026-09 “GPU/HW: Use multi-threaded pipeline compilation” · 0d2e66235 2026-09 “Core: Make TaskQueue thread count dynamic”

### duckstation-319 — Defer expensive setup until it is actually needed
Lens: laziness · Source: DuckStation
Principle: Don't do work in a constructor or at startup for a resource that may never be used. Look it up on first use. For optional libraries, load the shared object the first time a feature needs it, guarded by a `std::once_flag` rather than a global mutex.
Why it is faster: —
Applies elsewhere: lazy imports in Python and JS (`import()`), lazy-initialized singletons, and loading optional codecs only when a matching file is opened.
Watch out: —
Seen in DuckStation: The shader-compile progress tracker fetched the game's cover image in its constructor. It now does so on the first screen update, because fast compiles finish before the screen ever appears. libpng, libjpeg, libwebp and libzip are now loaded dynamically on demand through `g_dyn_libpng` and similar wrappers. MediaCapture replaced its global `s_load_mutex` with per-backend `std::once_flag`s. a2f8455b6 2026-10 “GPU/HW: Defer cover lookup until needed” · f39feb9d8 2026-09 “Image: Load libpng/libjpeg/libwebp dynamically” · 17a721c8f 2026-09 “ZipHelpers: Load libzip on-demand” · 59a3e09fa 2026-05 “MediaCapture: Eliminate global load mutex”

### duckstation-320 — Wrap native locks directly instead of using the standard library's heavier types
Lens: concurrency · Source: DuckStation
Principle: Some standard library implementations make `std::mutex`/`std::shared_mutex` large, or add indirection (MSVC's is around 80 bytes). A thin wrapper over the OS primitive (SRWLOCK, or a pthread mutex stored inline) gives a smaller object and a cheaper uncontended lock. Keep the `lock()`/`unlock()` interface so `std::lock_guard` still works.
Why it is faster: —
Applies elsewhere: WebKit's `WTF::Lock`, `parking_lot` in Rust, Abseil's `Mutex`, and futex-based locks in game engines.
Watch out: —
Seen in DuckStation: `Threading::Mutex`, `ConditionVariable` and `SharedMutex` in `common/threading.h` store an SRWLOCK pointer on Windows, or native storage sized per platform. Recursive and standard mutexes across the codebase (settings lock, translation map, Vulkan device, achievements, FullscreenUI) were replaced with them. 7137ce2be 2026-09 “Threading: Add lightweight Mutex/ConditionVariable wrappers” · 8a46ed8b7 2026-09 “Misc: Replace std::shared_mutex with Threading::SharedMutex”

### duckstation-321 — Replace a global lock in a hot handler with an ownership test and an atomic flag
Lens: concurrency · Source: DuckStation
Principle: A signal or exception handler that takes a process-wide recursive mutex serializes every fault on every thread. If only one thread can legitimately be handled there, filter by thread first and guard against reentrancy with an atomic flag. Avoid thread-locals inside vectored exception handlers, because TLS may not be initialized yet.
Why it is faster: —
Applies elsewhere: SIGSEGV-based garbage-collector write barriers, userfaultfd handlers, and crash reporters that must not deadlock.
Watch out: —
Seen in DuckStation: `PageFaultHandler` dropped its `std::recursive_mutex s_exception_handler_mutex`. `CodeCache::HandlePageFault` now returns early unless `Host::IsOnCoreThread()`, and uses an `std::atomic_flag in_page_fault_handler` as its recursion guard. An interim thread-local guard was dropped after crashes under Windows-on-ARM x86 emulation. b0c0a8ab7 2026-08 “PageFaultHandler: Remove the exception handler mutex” · cbe7951be 2026-09 “CPU/CodeCache: Remove thread_local recursive guard”

### duckstation-322 — Don't take or retake locks you don't need
Lens: concurrency · Source: DuckStation
Principle: Pass lock ownership into a callee (`std::unique_lock` by value) instead of unlocking and having the callee lock again. When reacting to settings, act only on a transition (old != new), so the unchanged case skips the lock entirely.
Why it is faster: —
Applies elsewhere: lock hand-off in connection pools, and React-style "only rerender if props changed" applied to locked state.
Watch out: —
Seen in DuckStation: `UpdateGameSettingsLayer` and `UpdateInputSettingsLayer` now take `std::unique_lock<Threading::Mutex>` by value, so `ReloadSettingsForPath` no longer does `lock.unlock()` just before a relock. `Achievements::UpdateSettings` only clears trackers or indicators (which takes a lock) when the setting has changed since `old_config`. a22292bf8 2026-10 “System: Avoid unnecessary mutex unlock->relock” · 7326f90c9 2026-09 “Achievements: Avoid lock when settings unchanged”

### duckstation-323 — Spin-wait with CPU pause hints, spaced by elapsed time
Lens: concurrency · Source: DuckStation
Principle: Busy-wait loops should issue `pause` (x86) or `isb`/`yield` (ARM) between polls. That cuts power, frees resources for the sibling hyperthread, and reduces cache-line ping-pong. Because pause latency varies widely across microarchitectures, space the polls by measured time rather than by a fixed count of pauses.
Why it is faster: —
Applies elsewhere: spinlocks in the Linux kernel, the adaptive spinning in .NET's `SpinWait`, and audio threads waiting on a lock-free queue.
Watch out: —
Seen in DuckStation: `Timer::SleepUntil(exact)` now calls `MultiPause()` in its final spin. `VideoThread::SyncThread` on ARM spins `PauseCPU()` until 500 ns of timer ticks have passed between polls of the shared state, because ISB timing differs between cores and clusters. `PauseCPU()` on ARM became a single `isb`. 73b571c4b 2026-09 “Timer: Use pause intrinsics when spin-sleeping” · 4c721f375 2026-09 “VideoThread: Use dynamic spin loop on ARM” · 608a0c3c2 2026-09 “Common: Use single isb for ARM PauseCPU()”

### duckstation-324 — Keep a persistent worker thread instead of starting and stopping it
Lens: concurrency · Source: DuckStation
Principle: Creating and joining threads whenever a feature toggles or media changes adds latency and failure points. Start one long-lived thread and reconfigure it with a parameter such as the readahead depth.
Why it is faster: —
Applies elsewhere: thread pools versus thread-per-request, and long-lived I/O workers in game engines.
Watch out: —
Seen in DuckStation: The CD-ROM async reader used to call `StartThread()`/`StopThread()` on init, shutdown and readahead changes. It now keeps a persistent thread and calls `CDROMAsyncReader::SetReadaheadSectors(n)`. 742d2d11b 2026-09 “CDROMAsyncReader: Use persistent thread”

### duckstation-325 — Move blocking file writes off the latency-critical thread
Lens: io · Source: DuckStation
Principle: Synchronous file I/O on a real-time thread causes stutter when storage is slow. Snapshot the data, write it on a worker, and track "save pending" with an atomic that is waited on only where ordering matters, such as before reopening the file or on destruction.
Why it is faster: —
Applies elsewhere: autosave in editors and games, write-behind logging, and SQLite WAL checkpoints on a background thread.
Watch out: —
Seen in DuckStation: `MemoryCard::SaveIfChanged` queues the write as an async task and sets `s_save_pending[index]`. The destructor and `Flush()` call `s_save_pending[m_index].wait(true)`. The target was stutter from slow scoped storage on Android. 34c37444a 2026-10 “MemoryCard: Save card to file asynchronously”

### duckstation-326 — Produce directly into the shared ring buffer, and move heavy transforms to the consumer thread
Lens: concurrency · Source: DuckStation
Principle: Have the producer write into the consumer's ring buffer through a begin/end-write span. Writing into a staging buffer and copying it in adds a pass. If the ring is full, write into a throwaway buffer rather than blocking. Expensive processing that only the consumer needs, such as time-stretching, can run on the consumer thread, keeping the producer's hot loop lean.
Why it is faster: —
Applies elsewhere: JACK and PortAudio callbacks, the LMAX Disruptor, and video decoders writing straight into mapped GPU buffers.
Watch out: —
Seen in DuckStation: `SPU::Execute` calls `audio_stream.BeginWrite(&ptr, &space)`, and when that fails it writes into `s_muted_output_buffer`. `InternalWriteFrames` and its copy are gone. The SoundTouch stretching moved from the core thread into the audio callback, so the core only copies raw samples. 2a2b7d086 2026-11 “CoreAudioStream: Write directly to the ring buffer” · 85efae5df 2026-11 “CoreAudioStream: Run stretching on the audio thread”

### duckstation-327 — Track a dirty rectangle so clears and copies cover only what changed
Lens: gpu-rendering · Source: DuckStation
Principle: Keep the union of the areas actually written since the last reset. Full-surface clears, merges and copies can then be clipped to that rectangle, or skipped when it is empty, which saves fill rate and bandwidth.
Why it is faster: —
Applies elsewhere: damage regions in compositors (Wayland, browsers), partial e-ink refresh, and dirty-region tracking in VNC/RDP.
Watch out: —
Seen in DuckStation: `GPU_HW` keeps `m_depth_dirty_rect`. `CopyDepthBuffer(merge)` returns early when it is empty and clips to it otherwise, and `ClearDepthBuffer` intersects the drawing area with it. A full clear resets it to `INVALID_RECT`. dd4a0eb77 2026-10 “GPU/HW: Track dirty depth rect to shrink depth merges/clears”

### duckstation-328 — Skip redundant graphics-API state updates, and batch the ones you do make
Lens: gpu-rendering · Source: DuckStation
Principle: Re-binding descriptors or setting state that hasn't changed, or isn't used, costs CPU in the driver. Gate updates on per-category dirty flags, set dynamic state only when the pipeline consumes it, and submit barriers as one array rather than one call each.
Why it is faster: —
Applies elsewhere: state caches in bgfx, sokol and WebGL wrappers, and barrier batching in Vulkan/D3D12 render graphs.
Watch out: —
Seen in DuckStation: In `VulkanDevice::UpdateDescriptorSetsForLayout`, texture descriptors are pushed or rewritten only when `dirty & DIRTY_FLAG_TEXTURES_OR_SAMPLERS`, so a UBO-offset change no longer rebinds textures. Blend constants are set only when a blend factor is `ConstantColor`/`InvConstantColor`. D3D12 `PreDispatchCheck` collects the UAV barriers into an array and calls `ResourceBarrier(n, barriers)` once. 571201311 2026-10 “VulkanDevice: Only update multi-texture descriptors when dirty” · 6bff12e6c 2026-10 “VulkanDevice: Don't set blend constants unless used” · 3d7b934ca 2026-10 “D3D12Device: Batch image RT UAV barriers before dispatch”

### duckstation-329 — Write GPU readback results with a compute shader directly into the destination buffer
Lens: gpu-rendering · Source: DuckStation
Principle: The render-to-texture, copy-to-staging, read-back path adds a full intermediate surface and copy. A compute shader can encode the data and write it straight into a host-visible download buffer, or into imported host memory. The readback is then bandwidth-bound instead of pipeline-bound.
Why it is faster: —
Applies elsewhere: GPU screenshot or video capture with compute-based format conversion, and ML frameworks that read results from mapped buffers.
Watch out: —
Seen in DuckStation: `GPU_HW::CreatePersistentBuffers` creates a shader-writable download texture. It tries to import `g_vram` itself when `features.memory_import` is available, and the readback encoding runs as a compute dispatch. Reading back all of VRAM went from 150–200 ms to 3 ms in RenderDoc on an RTX 5080. 32a9f32dc 2026-11 “GPU/HW: Use compute shader for VRAM readback encoding”

### duckstation-330 — Change presentation mode per present instead of recreating the swap chain
Lens: gpu-rendering · Source: DuckStation
Principle: Recreating a swap chain to toggle vsync is expensive and causes hitches, so with blocking vsync people skip presents instead. If the API can switch present mode on each present call (VK_KHR_swapchain_maintenance1), allocate enough images for both modes once and flip per frame. Fast-forward can then simply turn vsync off.
Why it is faster: —
Applies elsewhere: games that toggle frame limiting or VRR at runtime, and browsers that switch compositing modes.
Watch out: —
Seen in DuckStation: `VulkanSwapChain` creates a FIFO/IMMEDIATE-compatible swap chain and chains `VkSwapchainPresentModeInfoKHR` into `vkQueuePresentKHR`. `System::GetEffectiveVSyncMode` returns `Disabled` during fast-forward when `VideoThread::HasPerPresentVSync()` is true. D3D11 and D3D12 report the same capability. 637ff514c 2026-10 “VulkanSwapChain: Switch between FIFO and immediate without recreating” · 5d936496a 2026-10 “System: Disable vsync while fast forwarding if it can be switched cheaply”

### duckstation-331 — Coalesce many small scattered writes into a few block writes at load time
Lens: algorithms · Source: DuckStation
Principle: If a script or patch list applies many individual writes to nearby addresses every frame, merge runs of them into masked block writes when it is parsed. Each frame then applies a few vector-wide masked stores instead of dozens of scalar ones.
Why it is faster: —
Applies elsewhere: write-combining in database WALs, merging small DOM mutations into one batch, and the coalescing of scattered register writes in GPU command builders.
Watch out: —
Seen in DuckStation: `GamesharkCheatCode::MergeInstructions()` scans the parsed codes for runs of at least 4 constant or match writes within 64 bytes, in RAM and not crossing a 2 MB mirror. It replaces each run with an internal masked-block instruction that is applied with GSVector ops. Lines with options, or following a conditional, are never merged. 25ff3364c 2026-10 “Cheats: Merge runs of constant/match writes into vectorised block writes”

### duckstation-332 — Enable allocator and aliasing build options for release binaries
Lens: compiler-build · Source: DuckStation
Principle: Some defaults leave speed on the table. Opting into the Windows segment heap gives better small-allocation behavior and lower fragmentation. Strict aliasing (`-fstrict-aliasing`, which clang-cl does not enable by default) lets the optimizer keep values in registers across stores through unrelated pointer types. Pair it with `-Wstrict-aliasing` to catch violations.
Why it is faster: —
Applies elsewhere: the `heapType` manifest setting for Windows apps, mimalloc or jemalloc swaps in servers, and `-fno-strict-aliasing` in the Linux kernel as the counterexample.
Watch out: —
Seen in DuckStation: `Win32Application.props` sets `<EnableSegmentHeap>true</EnableSegmentHeap>` in the manifest. `Base.props` adds `-fstrict-aliasing` to non-debug clang configs, and `-Wstrict-aliasing` to all of them. 77ecf9939 2026-05 “MSBuild: Enable segment heap via vsprops” · 2f8e9a851 2026-06 “MSBuild: Enable strict aliasing for clang”

### duckstation-333 — Map guest memory into a reserved host address range so loads become one host instruction
Lens: jit-interpreters · Source: DuckStation
Principle: Reserve a large virtual region and map the guest's RAM into it, mirrors included, as views of one shared-memory object. Guest address a is then simply `base + a`, and generated code does a single host load or store with no bounds check or table walk. Where mapping isn't possible, a page-granular pointer LUT is the fallback.
Why it is faster: —
Applies elsewhere: WebAssembly runtimes' guard-page linear memory (V8, Wasmtime), database buffer pools mapped with `mmap`, and ring buffers mapped twice back-to-back.
Watch out: —
Seen in DuckStation: `Bus` creates a `SharedMemoryMappingArea fastmem_arena`. On Windows it uses `VirtualAlloc2(MEM_RESERVE_PLACEHOLDER)` plus `MapViewOfFile3`; on POSIX, `memfd_create`/`shm_open` plus `mmap(MAP_SHARED|MAP_FIXED)`. RAM is mapped at each mirror base. In `CPUFastmemMode::LUT`, a `fastmem_lut` of per-page host pointers is indexed by `addr >> FASTMEM_LUT_PAGE_SHIFT` instead. The ARM64 JIT emits `lsr w3, addr, #shift; ldr x3, [membase, x3, lsl #3]` for it. current code: src/core/bus.cpp, src/common/memmap.cpp, src/core/cpu_recompiler_arm64.cpp

### duckstation-334 — Speculate the fast path and backpatch it to a slow call on the first fault
Lens: jit-interpreters · Source: DuckStation · Also: Flycast
Principle: Emit every memory access as the direct fast-path instruction, and record its host PC and register metadata in a side table. If it faults (MMIO, unmapped or protected memory), the signal handler looks up the PC and rewrites that site into a call to the slow handler. The common case pays nothing, and only the sites that actually hit I/O pay the slow path.
Why it is faster: —
Applies elsewhere: JVM implicit null checks via SIGSEGV, Wasm bounds checks via guard pages, and inline caches that are patched on a miss.
Watch out: —
Seen in DuckStation: `CodeCache::HandleFastmemException` checks that the fault lies inside the arena, then looks up `fastmem_backpatch_info` (a `std::map<const void*, LoadstoreBackpatchInfo>` holding the address and data registers, size, signedness and live GPR bitmask). It calls `BackpatchLoadStore(exception_pc, info)` and queues the block for recompilation without fastmem. A write fault on code-protected RAM instead invalidates that page's blocks and resumes. current code: src/core/cpu_code_cache.cpp, src/util/page_fault_handler.cpp • Also seen in Flycast (flycast-052): `GenReadMemoryFast()` in `rec_arm64.cpp` emits `Add` and `Bfc(w1, 29, 3)` (mask out the mirror bits) followed by `Ldr reg, [x28, x1]`. It records the host PC in `block->memory_accesses`. In `core/linux/common.cpp`, `fault_handler` calls `ngen_Rewrite(ctx.pc, ...)`, which overwrites that site with the `GenReadMemorySlow` call to `ReadMem32`. `EnsureCodeSize` pads every fast path to the slow path's size so the rewrite fits in place. cb8e81d47 2019-01 “arm64: direct memory access and jit rewrite — trap sigsegv and rewrite using slow path” • Also seen in Flycast (flycast-101): The x64 rec was reworked to call generated `MemHandlers[Fast|Slow][size][R|W]` stubs, with the fast path also used when the MMU is on. `ngen_Rewrite` swaps a fast call for the slow one at the faulting `host_pc`. The x86 rec got the same mechanism. On ARM64 the rewrite size was cut to the fast-path length (3–4 instructions), measured at about 3% on A57. On rewrite, register allocation now runs only for `readm`. 594241ffa 2021-01 “x64 rec: generate main loop. use mem stubs. fast mem access for !mmu” · 958775ffb 2021-01 “x86 rec: add fast mem access and rewrite” · 5ba56627a 2019-05 “Optimize rec-ARM64 by emitting less instructions on loads.” · b8a00b9c7 2019-05 “rec-arm64/x64: only do regalloc on rewrite when needed”

### duckstation-335 — Detect self-modifying code with page protection, and fall back to checksums for hot pages
Lens: jit-interpreters · Source: DuckStation
Principle: Write-protect host pages that contain translated code. The first write faults, invalidates that page's blocks, and unprotects it, so code that doesn't change costs nothing. Pages that get rewritten constantly would fault-storm, so after a few invalidations within a short window, switch them to a per-block check that compares the source against a saved copy on entry.
Why it is faster: —
Applies elsewhere: QEMU's TB invalidation, browser JITs watching for patched code, and incremental-build tools that switch from mtime to content hashing.
Watch out: —
Seen in DuckStation: `Bus::SetRAMCodePage` calls `MemMap::MemProtect(..., ReadOnly)`. `InvalidateBlocksWithPageIndex` bumps `s_page_protection[page].invalidate_count`. More than `INVALIDATE_COUNT_FOR_MANUAL_PROTECTION` (4) within `INVALIDATE_FRAMES_FOR_MANUAL_PROTECTION` (60) switches the page to `PageProtectionMode::ManualCheck`. There, `IsBlockCodeCurrent` does a straight `memcmp`, and the JIT emits the inline compare described above. current code: src/core/cpu_code_cache.cpp, src/core/bus.cpp

### duckstation-336 — Chain translated blocks with direct jumps, and keep a reverse map to unpatch them
Lens: jit-interpreters · Source: DuckStation
Principle: Ending each block with a return to the dispatcher costs a lookup and an indirect jump. Patch block exits to jump straight to their successor, or to a "compile me" stub when the successor doesn't exist yet. Keep a multimap from target address to patch sites, so invalidating a block can redirect every incoming link.
Why it is faster: —
Applies elsewhere: QEMU TB chaining, Dolphin and PCSX2 block linking, and threaded-code interpreters.
Watch out: —
Seen in DuckStation: `CodeCache::CreateBlockLink` resolves `next_block->host_code`, or the `compile_or_revalidate_block`/`interpret_block` stubs, and records the site in `block_links`. `BacklinkBlocks(pc, dst)` walks `equal_range(pc)` and calls `EmitJump(site, dst)`. A self-link is used for tight loops. current code: src/core/cpu_code_cache.cpp

### duckstation-337 — Allocate registers in a single pass, with constant propagation and LRU eviction
Lens: jit-interpreters · Source: DuckStation
Principle: A baseline JIT can stay fast to compile and still produce decent code. Keep guest registers cached in host registers with read and dirty flags, propagate known constants so immediates fold in, evict the least recently allocated register under pressure, and write back only dirty values at block exits and calls.
Why it is faster: —
Applies elsewhere: V8 Sparkplug and Liftoff, LuaJIT's register allocator, and the baseline tiers of JVM and .NET.
Watch out: —
Seen in DuckStation: `Recompiler::m_host_regs[]` carries `HR_ALLOCATED`/`HR_NEEDED`/`HR_MODE_READ`/`HR_MODE_WRITE` flags and a type (CPU reg, load-delay value, membase, and so on), plus a `counter` from `m_register_alloc_counter++` for LRU. `m_constant_regs_valid` and `m_constant_regs_dirty` bitsets with `m_constant_reg_values` track constants. `TrySwapDelaySlot` moves branch delay-slot instructions ahead of the branch when no dependency forbids it. current code: src/core/cpu_recompiler.h, src/core/cpu_recompiler.cpp

### duckstation-338 — Place the JIT code buffer near the executable so calls stay short
Lens: platform · Source: DuckStation · Also: ares, Flycast
Principle: Direct branches and calls have limited range (±32 MB on ARM32, ±128 MB on ARM64, ±2 GB rel32 on x64). If generated code sits close to the host functions it calls, it can use short direct calls instead of loading 64-bit addresses. Reserving the buffer as a BSS section in the binary guarantees that placement.
Why it is faster: —
Applies elsewhere: V8 and LuaJIT allocating code near the binary, and the `-mcmodel` choices in compilers.
Watch out: —
Seen in DuckStation: On Linux ARM32 and ARM64, `s_code_buffer_storage[RECOMPILER_CODE_CACHE_SIZE]` lives in a `.jitstorage` section (`#pragma clang section bss`), page-aligned and mprotected RWX at startup. Elsewhere it comes from `MemMap::AllocateJITMemory`. ARM32 uses a smaller 16 MB buffer to stay in range. A separate far-code buffer keeps cold slow paths out of the hot instruction stream. current code: src/core/cpu_code_cache.cpp • Also seen in ares (ares-098): The macOS code cache, which is dynamic for security reasons, was allocated early in `nall::main`. That was then done on all platforms, with the static `fixedBuffer` aligned to 64 KiB and `memory::protect` checked, falling back to dynamic allocation. 32X resets stopped releasing the fixed allocator, which had pushed the SH-2s onto distant fallback allocations. Finally, the cache shrank from 128 to 64 MiB (CPU 63 MiB, RSP 1 MiB), and the `Pool* pools[1 << 21]`/`[1 << 24]` arrays became heap `vector`s. f396a9cf6 2023-08 “recompiler: allocate code cache early in main” · 4b3b78b77 2023-09 “recompiler: more robust code cache allocation (#1242)” · 07c320e55 2023-09 “32x: maintain code cache allocation across resets (#1243)” · e808d2c5a 2025-01 “recompiler: code cache compaction (#1755)” • Also seen in Flycast (flycast-026): On Win64, `recSh4_Init` now tries `VirtualAlloc((u8*)recSh4_Init - i*1MB, CODE_SIZE, ...)` for i=10..1300 until one succeeds, rather than using a static array. That made room for registering unwind tables for the cache, and `CODE_SIZE` grew to 6 MB. e3845465f 2015-08 “win64/jit: Generate unwind table for the jit code cache”

### duckstation-339 — Two-level page tables of function pointers for address-keyed dispatch
Lens: lookup-tables · Source: DuckStation
Principle: Look up "what code runs at address X" or "which handler serves address X" with a two-level table: top bits select a table, low bits select a slot. Unpopulated regions share one default table, so memory use stays small while lookup is two loads with no hashing. Pre-filling slots with a stub pointer removes the null check.
Why it is faster: —
Applies elsewhere: CPU page tables, sparse arrays in Unicode property lookup, and the routing tables behind fast HTTP routers.
Watch out: —
Seen in DuckStation: `g_code_lut[pc >> LUT_TABLE_SHIFT][idx]` holds host code pointers, pre-filled with `compile_or_revalidate_block` via `MemsetPtrs`, and unreachable regions point at a shared table. Bus I/O uses `g_bus.memory_handlers`, page-indexed read and write handler tables mapped from shared memory, with a second table `memory_handlers_isc` for the cache-isolated mode, so switching modes just swaps the base pointer. current code: src/core/cpu_code_cache.cpp, src/core/bus.cpp

### duckstation-340 — A thin, header-only SIMD wrapper with one backend per ISA and feature macros
Lens: simd · Source: DuckStation
Principle: Write vector kernels once against a small value-type API (`add32`, `madd_s16`, `shuffle8`), and pick the SSE, NEON or scalar backend at compile time. Expose capability macros so kernels can choose the best formulation on each target. Everything is `ALWAYS_INLINE`, so the abstraction costs nothing.
Why it is faster: —
Applies elsewhere: Google Highway, xsimd, `std::experimental::simd`, and Rust `std::simd`.
Watch out: —
Seen in DuckStation: `common/gsvector.h` includes `gsvector_sse.h`, `gsvector_neon.h` or `gsvector_nosimd.h`. The SSE backend defines `GSVECTOR_HAS_FAST_INT_SHUFFLE8`, plus `GSVECTOR_HAS_SRLV` and `GSVECTOR_HAS_256` under AVX2. The same `GSVector4i`/`GSVectorNi` code backs the SW rasterizer, SPU, MDEC, DMA, texture-cache rect math and cheat block writes. current code: src/common/gsvector.h, src/common/gsvector_sse.h, src/common/gsvector_neon.h

### duckstation-341 — Pre-instantiate every combination of hot flags into a function-pointer table
Lens: compiler-build · Source: DuckStation · Also: Flycast
Principle: Template the inner loop on each boolean or enum that changes per draw, such as shading, texture mode or transparency, and instantiate all the combinations. Store them in a multi-dimensional array and pick one per primitive. The per-pixel loop then has no mode branches. This works when the combinations number in the tens, not thousands.
Why it is faster: —
Applies elsewhere: shader permutations, blitter variants in pixman and Skia, and codec kernels specialized per bit depth.
Watch out: —
Seen in DuckStation: `DrawTriangleFunctionTable` is `const DrawTriangleFunction[2][4][2]`, indexed by shading, modulation mode and transparency. `DrawLineFunctionTable` is `[2][2]`. `GetDrawTriangleFunction()` indexes it. Each ISA namespace (Scalar, SIMD, AVX2) provides a full table, and `SelectImplementation()` swaps the table pointers once. The interpreter is instantiated in the same spirit as `ExecuteImpl<PGXPMode, bool debug>`, so PGXP and debug hooks compile out entirely. current code: src/core/gpu_sw_rasterizer.h, src/core/gpu_sw_rasterizer.inl, src/core/cpu_core.cpp • Also seen in Flycast (flycast-029): `Rendtriangle`/`PixelFlush` take `TPL_DECL_triangle` parameters: `alpha_mode, pp_UseAlpha, pp_Texture, pp_IgnoreTexA, pp_ShadInstr, pp_Offset`. `RenderParamList` looks up `RendtriangleFns[alpha][UseAlpha][Texture][IgnoreTexA][ShadInstr][Offset]` once for each poly param. e6a9d3e66 2015-08 “softrend: Resembling a pixel pipeline, textures, sort, cull” · 3e9431b2b 2015-08 “softrend: billinear filtering, perspective uv, alpha test, ARGB1555”

### duckstation-342 — A sorted event list feeding a single downcount
Lens: algorithms · Source: DuckStation · Also: Flycast
Principle: Instead of polling every subsystem each cycle, keep scheduled events in a list sorted by due time and expose only "cycles until the head event" to the CPU loop. The hot loop compares one counter. Re-sorting moves an event locally (forward or back) from its current position instead of re-inserting it from the head.
Why it is faster: —
Applies elsewhere: timer queues in event loops (libuv, Node), discrete-event simulators, and kernel hrtimers.
Watch out: —
Seen in DuckStation: In `TimingEvents`, a doubly linked list (`active_events_head`/`tail`) is ordered by `m_next_run_time`. `SortEvent()` walks backward or forward from the event's current position. `UpdateCPUDowncount()` sets `CPU::g_state.downcount = head->m_next_run_time - global_tick_counter`, or 0 when an interrupt is pending. Recompiled blocks test `pending_ticks >= downcount`. current code: src/core/timing_event.cpp • Also seen in Flycast (flycast-223): `sh4_sched_ffts()` scans `sch_list` for the minimum `end - now`, using `u32` wraparound with `-1` meaning "never". It writes `Sh4cntx.sh4_sched_next` and adjusts `sh4_sched_ffb`, so `sh4_sched_now()` is `sh4_sched_ffb - Sh4cntx.sh4_sched_next`. The scheduler file was already present in the 2013 import (1c16baeb0). current code: core/hw/sh4/sh4_sched.cpp

### duckstation-343 — Accumulate draws into persistently mapped buffers and flush only on state change
Lens: gpu-rendering · Source: DuckStation
Principle: Write vertices straight into a mapped stream buffer through a cursor pointer. Treat pipeline, texture, depth and UBO state as part of a batch key, and issue one draw call when the key changes or space runs out. Pre-build every pipeline permutation so a key change is a pointer swap, not a compile.
Why it is faster: —
Applies elsewhere: sprite batchers (SpriteBatch, Dear ImGui's draw lists), WebGL instancing layers, and 2D UI renderers.
Watch out: —
Seen in DuckStation: `GPU_HW` keeps `m_batch_vertex_ptr`/`m_batch_index_ptr` and remaining-space counters, a `BatchConfig m_batch`, and a dirty `m_batch_ubo_data`. Setters such as `SetBatchDepthBuffer` call `FlushRender()` only when the value actually changes and indices are pending. The pipelines sit in `DimensionalArray<std::unique_ptr<GPUPipeline>, 2, 2, 2, NUM_TEXTURE_MODES, 5, 5, 2> m_batch_pipelines`. current code: src/core/gpu_hw.h, src/core/gpu_hw.cpp

### duckstation-344 — Page-indexed texture cache with content-hash reuse and intrusive lists
Lens: gpu-rendering · Source: DuckStation
Principle: Index cached textures by the fixed-size pages of source memory they cover, so a write invalidates only the sources on the touched pages. Key reusable uploads by a fast content hash (xxHash) of the texels plus palette, so identical data is never uploaded twice. Intrusive linked lists let a source sit on many page lists without extra allocations.
Why it is faster: —
Applies elsewhere: texture caches in Dolphin and PCSX2, browser image caches keyed by content hash, and CDN deduplication.
Watch out: —
Seen in DuckStation: In `GPUTextureCache`, each `PageEntry` holds `TList<Source> sources` and `TList<VRAMWrite> writes`. `HashCacheKey{texture_hash, palette_hash, mode}` maps to a `HashCacheEntry` with an age for eviction. `ListPrepend`, `ListMoveToFront` and `ListUnlink` are `ALWAYS_INLINE_RELEASE` intrusive-list helpers. VRAM write tracking uses dirty rectangles (`m_vram_dirty_draw_rect`, `m_vram_dirty_write_rect`) to flag texture-page overlap. current code: src/core/gpu_hw_texture_cache.cpp, src/core/gpu_hw.cpp

### duckstation-345 — Single-producer/single-consumer command ring with cache-line-separated indices
Lens: concurrency · Source: DuckStation
Principle: Pass work from the emulation thread to the render thread through a large byte ring of variable-sized commands, using atomic read and write offsets. Put the producer's and consumer's indices on separate cache lines so they don't false-share. The consumer spins briefly before sleeping, which keeps wake-up latency low.
Why it is faster: —
Applies elsewhere: render-thread command buffers in Unreal and Godot, the LMAX Disruptor, and kernel io_uring submission and completion rings.
Watch out: —
Seen in DuckStation: `VideoThread` has a 16 MB `command_fifo_data` (`COMMAND_QUEUE_SIZE`), an `ALIGN_TO_CACHE_LINE std::atomic<u32> command_fifo_write_ptr`, and a separate `command_fifo_read_ptr`. `WakeThreadIfSleeping()` avoids a syscall when the consumer is still spinning, and `SyncThread(spin)` handles the producer-side wait. current code: src/core/video_thread.cpp

### duckstation-346 — Raise the ISA baseline and use whole-program optimization in release builds
Lens: compiler-build · Source: DuckStation
Principle: Set a realistic minimum ISA (SSE4.1 on x86-64, NEON-VFPv4 on ARMv7) so every translation unit can use those instructions without dispatch. Enable LTO, or MSVC's whole-program optimization and LTCG, so inlining crosses source files. Drop the frame pointer where it costs a lot in leaf functions (RISC-V).
Why it is faster: —
Applies elsewhere: distro x86-64-v2/v3 builds, Chrome's ThinLTO and PGO builds, and `-C target-cpu` in Rust release profiles.
Watch out: —
Seen in DuckStation: `detect_architecture()` adds `-msse4.1` on x86-64 (opt-out via `DISABLE_SSE4`), `-marm -march=armv7-a -mfpu=neon-vfpv4` on ARM32, and `-fomit-frame-pointer` plus `-finline-atomics` on RISC-V. `Base.props` enables `WholeProgramOptimization` and `LinkTimeCodeGeneration`, `-flto=thin` for ReleaseLTCG clang builds, and `IntrinsicFunctions` and `AnySuitable` inlining for release. current code: CMakeModules/DuckStationUtils.cmake, dep/vsprops/Base.props

### duckstation-347 — Stack-buffered strings and fixed-capacity containers for hot temporaries
Lens: allocation · Source: DuckStation
Principle: Most formatted strings and queues in a hot path are small and short-lived. A string type with an inline buffer (spilling to the heap only when it overflows), and FIFOs with a compile-time capacity, remove malloc and free from per-frame code paths entirely.
Why it is faster: —
Applies elsewhere: `fmt::memory_buffer`, LLVM `SmallString`, and `heapless` collections in embedded Rust.
Watch out: —
Seen in DuckStation: `SmallStackString<L>` with aliases `TinyString` (64), `SmallString` (256) and `LargeString` (512) is used for OSD and log formatting. `FIFOQueue<T, CAPACITY>` has `InlineFIFOQueue` and `HeapFIFOQueue` variants, and `FixedHeapArray` covers fixed-size aligned heap arrays. current code: src/common/small_string.h, src/common/fifo_queue.h, src/common/heap_array.h
