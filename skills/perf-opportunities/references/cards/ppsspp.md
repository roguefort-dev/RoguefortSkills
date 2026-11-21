# PPSSPP — technique cards (293)

Each card is exactly 7 lines. Fetch one with: grep -A 6 "^### ppsspp-NNN " ppsspp.md

### ppsspp-001 — Shadow driver state and drop redundant sets
Lens: gpu-rendering · Source: PPSSPP
Principle: Calls into a graphics driver (or any stateful API) cost a lot even when they change nothing. Keep a CPU-side copy of each state value and only make the call when the new value differs. The comparison costs one register compare; the redundant call can cost thousands of cycles of driver validation.
Why it is faster: —
Applies elsewhere: skip `element.style.x = v` when the value is unchanged, avoid re-issuing identical `ioctl`/register writes in a device driver, keep a "current bound texture/program" variable in any OpenGL/WebGL renderer.
Watch out: —
Seen in PPSSPP: A new `OpenGLState glstate` wraps blend, cull, depth and later viewport in `BoolState`/`set()` templates that compare before calling `glEnable`/`glBlendFunc`/`glViewport`. The texture cache stores `minFilt`, `magFilt`, `sClamp`, `tClamp` in each `TexCacheEntry` so `UpdateSamplingParams` skips `glTexParameteri` when unchanged, and tracks `lastBoundTexture` to skip redundant binds. 8900d99779 2012-11 “Add OpenGL state cache to save some redundant state change.” · 26a5bfcd5d 2013-01 “Avoid setting texture filter over and over again.” · 9363a867e2 2013-01 “Add glViewport to cached gl state.” · 174b9749d7 2013-02 “Minor speedup to 16-bit texture color conversion.”

### ppsspp-002 — Invalidate only when the value actually changed
Lens: laziness · Source: PPSSPP
Principle: A write to a configuration field doesn't mean the configuration changed. XOR the new word against the old one and raise dirty flags (or flush pending work) only if the result is non-zero. Programs that re-send the same settings every frame then trigger no downstream work.
Why it is faster: —
Applies elsewhere: React's shallow-equal before re-render, a setter that only fires a change event when `old !== new`, a config reloader that diffs before restarting workers.
Watch out: —
Seen in PPSSPP: The GE command loop computes `diff = op ^ gstate.cmdmem[cmd]`. Many commands moved from "always flush the batched draws" to "flush only if `diff`" in `flushBeforeCommand_`. Texture-scale/offset commands only call `DirtyUniform(DIRTY_UVSCALEOFFSET)` when `diff` is set. Framebuffer and viewport commands set `gstate_c.framebufChanged` only on a diff, so `SetRenderFrameBuffer()` returns at once for repeated prims; the commit says this cut its time from 1% to 0.13%. 1b1a275dcc 2013-01 “GPU "dirty" optimization, wip vertexcache” · f3c227041d 2013-06 “Only set uniform variables as dirty when there's a difference in value.” · 85a23dac63 2013-04 “Avoid framebuffer lookups for repeated prims.”

### ppsspp-003 — Defer and merge many small submissions into one batch
Lens: gpu-rendering · Source: PPSSPP
Principle: Fixed per-call overhead dominates when each call carries little data. Record requests in a queue, keep merging them while they are compatible, and submit once when an incompatible state change or the queue limit forces a flush. Rewriting strips and fans as one indexed triangle list lets different primitive shapes share a single submission. Track whether the merged pieces were already sequential, and skip the index buffer entirely when it would map i to i.
Why it is faster: —
Applies elsewhere: batching SQL inserts into one transaction, Nagle-style coalescing of small network writes, sprite batching in 2D game engines, grouping DOM writes into one animation frame.
Watch out: —
Seen in PPSSPP: `IndexGenerator` converts points, lists, strips, fans and rectangles into a shared index buffer so several PSP draw calls become one `glDrawElements`. `SubmitPrim` later only records a `DeferredDrawCall` (pointers, prim, index bounds). `DecodeVerts()` runs at `Flush()`, which happens when `!indexGen.PrimCompatible(prim)` or `numDrawCalls >= MAX_DEFERRED_DRAW_CALLS`. `IndexGenerator` records `seenPrims_` plus `SEEN_INDEX8`/`SEEN_INDEX16` bits. When only plain lists were seen, the draw engine uses `glDrawArrays` without an element buffer. A later change adds `pureCount_` so a single triangle strip is drawn natively with no generated indices. 4eeab83389 2012-12 “Add IndexGenerator.cpp/h which will later be used to combine small draw calls into large indexed draw calls” · 8ff96bbcd8 2013-01 “Defer vertex decoding until flush. Track draw calls so that they can be cached later.” · 1b35964445 2012-12 “Avoid using indexed draws when unnecessary.” · e15b580df6 2013-01 “Avoid using an element buffer in more cases.” · b52d21bd4a 2013-04 “Don't use an index buffer for single triangle strips.”

### ppsspp-004 — Process unique elements once, then expand by index
Lens: algorithms · Source: PPSSPP
Principle: When a list refers to shared items by index, transforming items while walking the index list repeats work for every reference. Find the index range, transform each referenced item once into a table, then gather through the indices. Expensive work then scales with unique items, not references.
Why it is faster: —
Applies elsewhere: resolve each distinct foreign key once before joining, decode each distinct dictionary entry once in a columnar reader, render each unique glyph once and blit by reference.
Watch out: —
Seen in PPSSPP: `TransformAndDrawPrim` used to light and transform once per index inside a `for (i < vertexCount)` loop. `DecodeVerts` now reports `indexLowerBound`/`indexUpperBound`. The T&L loop runs `for (index = lower; index <= upper; ...)` into `transformed[]`, and a second pass expands through the index buffer into `transformedExpanded[]`. Non-indexed, non-rectangle draws skip the expansion and use the unexpanded buffer. 7c9107030d 2012-11 “Speed up T&L by only transforming/lighting each vertex once, for indexed draws”

### ppsspp-005 — Hoist batch-invariant math out of the per-item loop
Lens: algorithms · Source: PPSSPP
Principle: Terms that depend only on per-batch parameters (material colors, scale factors, a volume setting, a light direction) should be computed once per batch and reused by every item. Divisions and square roots are the most profitable to move. Where possible, also move them from per-vertex shader code to a single CPU computation per uniform update. Fold chains of constant factors such as `/ 31.0f` then `* 255.0f` into one precomputed multiplier.
Why it is faster: —
Applies elsewhere: compute `1/width` once before a pixel loop, normalize a query vector once before scoring many documents, read a config value into a local before an audio mixing loop.
Watch out: —
Seen in PPSSPP: A `Lighter` class precomputes global ambient, material emissive and ambient terms in its constructor so `Light()` only does per-vertex work. `SoftwareTransformAndDraw` computes `widthFactor`/`heightFactor` before the vertex loop and multiplies, where it used to divide per vertex. Directional lights are normalized in `LinkedShader::updateUniforms()`, so the generated shader drops a per-vertex `normalize`. `sceAtracLowLevelDecode` computes `volumeShift` once outside the sample loop. The morph colour steps in `VertexDecoder` (`Step_Color565Morph`, `Step_Color5551Morph`, …) accumulate `w * (cdata & 0x1f) * (255.0f / 31.0f)` and store `(u8)col[i]`. They used to divide by 31 per channel and then multiply by 255 at the end. e5c6cf965b 2012-11 “Fixes and optimizations to vertex decoding and lighting.” · 78f85db1bb 2013-07 “Use multiplies instead of divide” · 916184d2f4 2013-06 “Optimize the most common light type, directional, by doing the normalization beforehand.” · 17792ac623 2013-07 “Audio optimization to sceAtracLowLevelDecode()” · 0b9de18f8c 2013-04 “Small optimization to morph vertex step funcs.”

### ppsspp-006 — Early-out when the operation is an identity
Lens: branching-fast-paths · Source: PPSSPP
Principle: Generic transforms often run with neutral parameters. Test for the identity configuration up front and return before the loop. One compare replaces a full pass that would produce the input unchanged.
Why it is faster: —
Applies elsewhere: skip resampling when source and target rates match, skip a matrix multiply when the transform is identity, bypass an image filter whose kernel is a delta.
Watch out: —
Seen in PPSSPP: `ApplyPrefixST` returns immediately when the swizzle prefix is `0xe4` (neutral), and `ApplyPrefixD` returns when the destination prefix is 0. The two swizzle helpers were also made `inline`. `TextureScaler::Scale` calls `IsEmptyOrFlat()` first and skips the xBR/bilinear upscale for single-colour textures, which the commit says are common in some games. 423bba5d8d 2012-11 “Some optimization and minor fixes” · fa8250ba12 2013-05 “Avoid scaling empty and flat textures”

### ppsspp-007 — Hand over the source buffer when no conversion is needed
Lens: memory-layout · Source: PPSSPP
Principle: A staging copy is only needed when the layout changes. When source and destination formats and strides already match, pass a pointer to the original data. Likewise, don't write back a buffer that wasn't modified.
Why it is faster: —
Applies elsewhere: zero-copy `sendfile`/`mmap` paths, passing a typed-array view instead of copying to WebGL, using `JNI_ABORT` or read-only buffer modes to skip copy-back.
Watch out: —
Seen in PPSSPP: `DecodeTextureLevel` for unswizzled 8888 textures now uses `finalBuf = Memory::GetPointer(texaddr)` when `w == bufw`, where it used to memcpy into `tmpTexBuf32`. `DrawPixels` similarly avoids its conversion copy in the common case. On Android, `audioRender` releases the Java array with `JNI_ABORT` when no samples were mixed, so the JVM skips copying it back. df65bb40ac 2013-05 “Avoid copying a tex if the PSP format is perfect.” · 443e9eca8e 2013-06 “DrawPixels: Avoid copying in the common case.” · a17a16eda2 2013-05 “Avoid a memory copy when audio is ahead.”

### ppsspp-008 — Compile a format descriptor into a step-function list, and cache it
Lens: compiler-build · Source: PPSSPP
Principle: Interpreting a format description per element means re-testing the same flags millions of times. Translate the descriptor once into an array of specialized function pointers, one per field, then run the list per element. Cache the compiled list by descriptor key so a format change costs a lookup, not a rebuild.
Why it is faster: —
Applies elsewhere: precompiled serializers in protobuf/Cap'n Proto, per-schema CSV parsers, pixel-format converters chosen once per image, prepared statements in databases.
Watch out: —
Seen in PPSSPP: The vertex decoder was rewritten around tables of `StepFunction`s (`wtstep`, `tcstep`, `colstep`, `nrmstep`, `posstep` and their morph and through-mode variants) selected in `SetVertexType`. The engine first re-ran setup only when `gstate.vertType != lastVType`, then kept a map of ready decoders via `GetVertexDecoder(vertType)`. 1b5407f505 2012-12 “Rewrite the vertex decoder a bit. Turn on SSE2 compiler flag etc.” · 7c64f9c61e 2012-12 “Tiny optimization - only init the vertex decoder on vertex type change” · d96c588288 2013-03 “Cache vertex decoders.”

### ppsspp-009 — Key caches by content hash, not by location
Lens: laziness · Source: PPSSPP
Principle: When data is often rewritten in place with identical contents, or the same contents show up at several addresses, a location-keyed cache misses all the time. Put a hash of the contents in the key instead, so identical payloads share one entry wherever they live. A secondary cache keyed purely by full hash catches contents that flip back and forth.
Why it is faster: —
Applies elsewhere: content-addressed build caches (Bazel, ccache), deduplicating blob stores, HTTP ETags based on body hashes, asset pipelines that key on file digest instead of path.
Watch out: —
Seen in PPSSPP: `TextureCache::SetTexture` built its key from `clutaddr << 32`; it now uses `QuickClutHash(clutaddr)`, which the commit says makes Final Fantasy Tactics "immensely faster". A `secondCache` keyed by `fullhash | cluthash << 32` is consulted only for entries invalidated more than 2 and fewer than 128 times, so textures that keep alternating are found again without being decoded. 0cd65102af 2013-04 “Cache cluts by hash not address.” · 02c88b9517 2013-05 “Add a secondary texture cache again.”

### ppsspp-010 — Back off revalidation for items that keep proving stable
Lens: laziness · Source: PPSSPP
Principle: Checking a cached item against its source on every use can cost as much as recomputing it. Each time a check passes, lengthen the interval before the next full check, up to a cap. Items that do change are caught by the next check and dropped to an always-recompute state. The same per-item stability count can gate optional expensive processing, so only items that never changed receive it.
Why it is faster: —
Applies elsewhere: TCP retransmit backoff, conditional HTTP polling with growing intervals, adaptive health-check periods, file-watcher fallbacks that stat stable files less often.
Watch out: —
Seen in PPSSPP: The vertex cache (`VertexArrayInfo`) gained `drawsUntilNextFullHash`. After a matching hash it is set to `std::min(16, vai->numDraws)` (later 24 frames), so CityHash runs on only a fraction of draws. A mismatch marks the array `VAI_UNRELIABLE` and frees its VBO. A follow-up stops caching arrays of 100 vertices or fewer, which "seem much more likely to change". `TexCacheEntry` got a `numInvalidated` counter, incremented on parameter mismatch or hash failure. `LoadTextureLevel` calls `scaler.Scale()` (xBR upscaling) only when `entry.numInvalidated == 0`, so animated textures are not upscaled again on every change. 0ca279e81d 2013-02 “Don't hash vertex buffers every frame (TCP-style backoff up until every 16th)” · 1e1979b786 2013-02 “Simply don't cache < 100 verts for now.” · 1f7a3776b3 2013-05 “Only scale textures which were never invalidated”

### ppsspp-011 — Pick the fastest hash that is good enough, and vectorize it
Lens: algorithms · Source: PPSSPP
Principle: When a hash only detects changes, throughput matters more than cryptographic strength. Measure the candidates (additive checksum, CityHash, xxHash, multiply-add) and keep the fastest one with acceptable collisions. A plain sum vectorizes easily: accumulate four lanes with 128-bit adds and fold them at the end.
Why it is faster: —
Applies elsewhere: xxHash/CityHash for dedup or cache keys instead of SHA, SIMD checksums in storage engines, rolling hashes for rsync-style diffs.
Watch out: —
Seen in PPSSPP: `QuickTexHash` went from a scalar `check += *checkp++` loop to CityHash32, and also gained an SSE2 path using `_mm_add_epi32` over aligned 16-byte chunks with a final `_mm_srli_si128` fold. `QuickClutHash` is an SSE multiply-add hash for CLUTs. Finally CityHash was replaced by `XXH32` for the CLUT and vertex hashes because "it's faster", at least on Windows. 6a7722283c 2013-04 “Hash textures using cityhash rather than adding.” · 5b970d76e3 2013-05 “Use SSE2 in the texture hashing function.” · d2a52f48e5 2013-05 “Switch to a weaker, but faster clut hash.” · 52f1de8f1d 2013-08 “Use xxHash instead of CityHash, it's faster.”

### ppsspp-012 — Compute derived values lazily, and only once
Lens: laziness · Source: PPSSPP
Principle: Data is often loaded far more often than it is consumed. Mark derived values (hashes, converted forms) dirty when the source loads, compute them the first time someone reads them, and pass along a value you already computed instead of recomputing it moments later. Clamp loops to the parts that actually exist, and create helper resources on first use rather than at startup.
Why it is faster: —
Applies elsewhere: lazy getters with memoization, computing a content hash only when a cache lookup actually happens, threading an already-parsed object through a call chain instead of re-parsing.
Watch out: —
Seen in PPSSPP: CLUT handling moved from "load every time a texture is set" to "load on `GE_CMD_LOADCLUT`, set `clutChanged`". `LoadClut()` then stopped hashing; `clutHash_` is computed in `SetTexture()` only when `clutDirty_` is set and a CLUT texture actually uses it. When a texture's `fullhash` check fails, that value is reused for the secondary-cache key and the new entry, where it used to be recomputed right away. `SetTexture` lowers `maxLevel` to the last mip level whose address passes `Memory::IsValidAddress`, which fixed a slowdown from decoding levels pointing at nothing. `FramebufferManager` no longer creates its 480×272 DrawPixels texture in the constructor; it is created when `DrawPixels` first needs it. 7dd0d242e2 2013-05 “Load the clut only using LOADCLUT, not every tex.” · c53386244d 2013-05 “Don't hash the clut when it's not even used.” · 342ae6c322 2013-05 “Reuse the fullhash when it triggers a reload.” · 2fcea2a7e6 2013-02 “Fix mipmap-related slowdown caused by excessive texture decoding” · 2221f221e3 2013-06 “Only create backing texture for DrawPixels when needed.”

### ppsspp-013 — Walk raw pointers instead of calling checked accessors per element
Lens: memory-layout · Source: PPSSPP
Principle: An accessor like `ReadU32(addr)` may mask, bounds-check or translate the address on every call. In a bulk loop, translate the base address once, then walk a typed pointer. The loop body becomes a load and an add, which the compiler can unroll or vectorize.
Why it is faster: —
Applies elsewhere: taking a `Span<T>`/slice once and iterating it instead of `list.get(i)`, using `DataView` once and then typed arrays, avoiding `vector::at()` in hot loops.
Watch out: —
Seen in PPSSPP: The texture hash went from `check += Memory::ReadUnchecked_U32(texaddr + i)` to `u32 *checkp = Memory::GetPointer(texaddr); check += *checkp++`. `UnswizzleFromMem` was changed the same way to use `src`/`dest` pointers. Later, `GetPointerUnchecked` and both `Memory::Memcpy` overloads were made inline in `MemMap.h` so per-call overhead disappears. 63ef98a994 2013-01 “Use pointer walking and sizeInRAM for texture hash.” · 8d55445b28 2013-04 “Optimize texture unswizzling by walking pointers.” · 9109882c04 2013-08 “More microoptimization”

### ppsspp-014 — Replace element-wise copy loops with one bulk copy
Lens: memory-layout · Source: PPSSPP
Principle: Copying arrays field by field in a loop leaves speed on the table compared with `memcpy`, which uses the widest moves and prefetch-friendly patterns. Lay out related fields contiguously so a whole group moves in one call.
Why it is faster: —
Applies elsewhere: `System.arraycopy`/`TypedArray.set` instead of loops, struct-of-registers save/restore in RTOS context switches, snapshotting game state with one copy.
Watch out: —
Seen in PPSSPP: `__KernelSaveContext`/`__KernelLoadContext` replaced per-register `for` loops over `r`, `f`, `v`, `vfpuCtrl` with `memcpy`. The commit reports these functions dropping from 1.6% to 0.5% of time. Later, `r` and `f` were placed back-to-back and the scalar registers (pc, hi, lo, fcr…) grouped into an `other` array, so each is one copy ("~2.5% improvement" in one game). Texture decoding swapped `ReadUnchecked_U16` loops for `Memory::Memcpy`. 7f4e322266 2013-01 “Cut thread switching time down.” · 846178a588 2013-08 “Optimize thread switching a bit.” · 89ad3fb41a 2013-04 “Swap a couple Unchecked loops for Memcpy calls.”

### ppsspp-015 — SWAR: process several small values in one machine word
Lens: simd · Source: PPSSPP
Principle: When data elements are 8 or 16 bits wide, load two or four of them into a 32-bit register and apply shifts and masks that act on every lane at once. Packing a multi-byte field into one word also turns several byte stores into one store.
Why it is faster: —
Applies elsewhere: RGB565↔RGBA conversions in embedded displays, byte-wise `strlen`/`memchr` tricks, packing RGBA as one `uint32` in canvas `ImageData` manipulation.
Watch out: —
Seen in PPSSPP: The 16-bit texture converters (4444, 5551, 565) process `(numPixels + 1) / 2` words with doubled masks such as `((c >> 12) & 0x000F000F) | ((c << 4) & 0x0F000F00) …`. The alpha-only 4-bit CLUT path writes two pixels per `u32` as `color32 | ((index & 0x00f0) << 12) | (index & 0x000f)`. In the UI `DrawBuffer`, `Vertex` stores `uint32_t rgba` where it used to store four bytes unpacked with shifts. 174b9749d7 2013-02 “Minor speedup to 16-bit texture color conversion.” · 995ab510e9 2013-05 “Optimize alpha-only CLUTs, common for fonts.” · 950a6b65e8 2012-05 “Minor optimization”

### ppsspp-016 — Specialize the loop for the common configuration
Lens: branching-fast-paths · Source: PPSSPP
Principle: A general loop that applies offset, mask and shift per element pays for flexibility it rarely uses. Detect the common "plain" configuration once, outside the loop, and run a tight dedicated loop for it. Keep the general loop for the rest.
Why it is faster: —
Applies elsewhere: fast paths for ASCII in UTF-8 decoders, a dedicated memcpy path for stride-1 tensor copies, special-casing unit-stride in BLAS routines.
Watch out: —
Seen in PPSSPP: `DeIndexTexture` checks `nakedIndex = (gstate.clutformat & ~3) == 0xC500FF00` once and then does plain `dest[i] = clut[index]` with no `GetClutIndex` call. `DeIndexTexture4` unpacks two 4-bit indices per byte. `DeIndexTexture4Optimal` handles CLUTs whose entries differ only in alpha, common for fonts, without reading the CLUT at all. 4e4861a104 2013-04 “Optimize common cases for unswizzled indexed tex.” · 394a8eaf9e 2013-04 “Small optimization to typical 4-bit textures.” · 995ab510e9 2013-05 “Optimize alpha-only CLUTs, common for fonts.”

### ppsspp-017 — Ping-pong between scratch buffers to avoid in-place shuffling
Lens: memory-layout · Source: PPSSPP
Principle: Expanding data in place forces backwards iteration and temporary blocks to avoid overwriting unread input. With two scratch buffers you can read from one and write forward into the other. The loop gets simpler and streams through memory.
Why it is faster: —
Applies elsewhere: double-buffered image filter passes, GPU ping-pong render targets, radix sort alternating between two arrays.
Watch out: —
Seen in PPSSPP: `readIndexedTex` for swizzled textures used to de-index backwards inside `tmpTexBuf32` through a 1024-byte `block` copy. After `UnswizzleFromMem` fills `tmpTexBuf32`, it now writes the de-indexed output forward into `tmpTexBuf16`. 7b6f7abf42 2013-04 “Swap buffers after unswizzling, it's faster.”

### ppsspp-018 — Put the range field in the high bits of an ordered key
Lens: algorithms · Source: PPSSPP
Principle: In an ordered map, a composite key with the address in the most significant bits keeps entries sorted by address. "Everything overlapping this range" then becomes a `lower_bound`/`upper_bound` walk instead of a full scan. Keep the scanned window as narrow as correctness allows.
Why it is faster: —
Applies elsewhere: time-series keys like `(timestamp, id)` in LSM stores, interval lookup in memory allocators, geo-hash prefixes for bounding-box queries.
Watch out: —
Seen in PPSSPP: The texture cache key changed from `texaddr` in the low bits to `(u64)texaddr << 32`, with the CLUT hash in the low bits. `TextureCache::Invalidate` iterates `cache.lower_bound(addr - LARGEST_TEXTURE_SIZE)` to `upper_bound(addr + size + …)`. `NotifyFramebuffer` then narrowed its window from the full framebuffer height to `MAX_SUBAREA_Y_OFFSET` rows. `SymbolMap` gained `entryRanges`, a map keyed by symbol end address, so `GetSymbolInfo` finds the containing symbol with one ordered lookup. 4bc7752eb5 2013-06 “Flip around the texture key to invalid faster.” · 0a9315d0e2 2013-07 “Cut down the scan range to avoid perf hit.” · 0ca8f5c655 2013-07 “Improve performance of ScanForFunctions().”

### ppsspp-019 — Binary-search an array that is already sorted
Lens: algorithms · Source: PPSSPP
Principle: If items are appended in increasing order of some field, the array is already sorted on it. Replace a reverse linear scan with a lower-bound binary search, which turns O(n) lookups into O(log n) as the collection grows.
Why it is faster: —
Applies elsewhere: finding a log segment by offset, mapping a code address to a function in a symbolizer, locating a row in an append-only time-ordered file.
Watch out: —
Seen in PPSSPP: JIT blocks are emitted at increasing code addresses, so `JitBlockCache::GetBlockNumberFromEmuHackOp` swapped its `for (i = num_blocks - 1; i >= 0; i--)` scan for a `binary_search(blocks, baseoff, 0, num_blocks - 1)` on `normalEntry`, after block linking "got too slow". 338bc0049d 2013-04 “Seems block linking got too slow, speed it up again.”

### ppsspp-020 — Bucket by the first key so lookups skip unrelated entries
Lens: algorithms · Source: PPSSPP
Principle: A flat list searched with a compound predicate (name AND id) does string compares across every entry. Group entries into a map keyed by the first component, holding vectors of the rest, so each lookup only scans the matching group.
Why it is faster: —
Applies elsewhere: per-module symbol tables in dynamic linkers, routing tables keyed by HTTP method then path, event listeners grouped by event type.
Watch out: —
Seen in PPSSPP: `HLE.cpp` kept `unresolvedSyscalls` and `exportedCalls` as flat `std::vector<Syscall>`, checked with `strncmp(moduleName) && nid == nib`. They are now `std::map<std::string, SyscallVector>`. Export resolution does one `find(moduleName)`, then compares NIDs only; the commit says the old code was noticeably slow in one game. b9c2e21105 2013-07 “Improve performance of syscall resolving.”

### ppsspp-021 — For small collections, a linear vector beats a tree map
Lens: algorithms · Source: PPSSPP
Principle: For a few dozen entries, a contiguous vector with a linear scan beats a node-based map: no pointer chasing, no allocation per node, and branch-predictable compares. This is especially true when the code already iterates the whole collection for other reasons.
Why it is faster: —
Applies elsewhere: small-vector/flat-map types (Boost `flat_map`, Rust `SmallVec`), HTTP header lists, V8's inline caches for few shapes.
Watch out: —
Seen in PPSSPP: `ShaderManager` replaced `std::map<std::pair<Shader*,Shader*>, LinkedShader*>` with a `std::vector<LinkedShaderCacheEntry{vs, fs, ls}>` scanned in `ApplyShader()`. The commit reports time in a flush-heavy game dropping from 9.84% to 1.85%. 386d51081c 2013-06 “Optimize linked shader lookup in ApplyShader().”

### ppsspp-022 — Keep a pending-work counter to skip empty scans
Lens: laziness · Source: PPSSPP
Principle: If a check runs constantly but rarely finds anything (every scheduler tick scanning all threads for callbacks), keep a global count of pending items, updated on enqueue and dequeue. Return immediately when it is zero.
Why it is faster: —
Applies elsewhere: a "has dirty items" counter before walking a scene graph, an outstanding-request count before polling sockets, a pending-timer count in an event loop.
Watch out: —
Seen in PPSSPP: A global `readyCallbacksCount` is maintained as callbacks are queued, run, or their thread deleted. `__KernelForceCallbacks()` starts with `if (readyCallbacksCount == 0) return false;`. The commit notes this is checked on every reschedule and "could save as much as 1.6% of time"; a later commit applies the same guard to `sceKernelCheckCallback()`. 54e6a39e74 2013-01 “Optimize callback checking.” · f9fc1af5c8 2013-03 “Improve perf of sceKernelCheckCallback().”

### ppsspp-023 — Per-priority ready queues instead of scanning every task
Lens: algorithms · Source: PPSSPP
Principle: A scheduler that loops over all tasks to find the best ready one costs O(tasks) per switch. Keep one FIFO of ready tasks per priority level, so picking the next task is a pop from the first non-empty queue. Tweak the queue for the hot operations: front padding for O(1) push-front, and a "pop only if strictly better than the current task" call that avoids requeuing the running task.
Why it is faster: —
Applies elsewhere: OS run queues (O(1) Linux scheduler, FreeRTOS ready lists), multilevel job queues in worker pools, priority lanes in message brokers.
Watch out: —
Seen in PPSSPP: `__KernelNextThread()` replaced its round-robin scan over `threadqueue` with `threadReadyQueue[prio]` lists holding only ready threads, kept in sync by `__KernelChangeReadyState`. `ThreadList` keeps `START_PAD` free slots at the front so `push_front` rarely moves memory. `pop_first_better(priority)` lets a running thread stay put unless a higher-priority thread is ready. ce2e62317b 2013-02 “Make the new queue only have ready threads.” · 996eb7a648 2013-02 “Use the new ready queue for thread switching.” · 543dfcd4d8 2013-03 “Optimize thread ready queue pop_front().” · c7020d4872 2013-04 “Optimize resched with a running thread.”

### ppsspp-024 — Skip checked lookups for handles you already trust
Lens: other · Source: PPSSPP
Principle: Generic handle lookups validate the ID and do a type check (`dynamic_cast`/RTTI) every time. On hot paths where the handle is known to be valid, use an unchecked `static_cast` accessor. Better still, pass the already-resolved object pointer down to callees, or cache it in a local. The same goes for a second search right after a first one: keep the iterator or pointer it returned.
Why it is faster: —
Applies elsewhere: passing a resolved entity instead of re-fetching by ID in ORMs, unchecked indexing after an explicit bounds check, caching `document.getElementById` results.
Watch out: —
Seen in PPSSPP: `KernelObjectPool::GetFast<T>` does a range and occupancy check, then `static_cast`; `__GetCurrentThread()` uses it in place of `Get<Thread>` with its `dynamic_cast`. The commit notes these "eat a few percentage points". `__KernelExecutePendingMipsCalls(Thread *thread, …)` now takes the current thread from its caller; the old lookup cost "~3% in many games with fast forward". `__KernelNextThread` stores `__GetCurrentThread()` in a local rather than calling it twice. `TransformDrawEngine::Flush()` computed `iter = vai_.find(id)` but then tested `vai_.find(id) != vai_.end()` again; it now tests `iter`. `BlockAllocator::FreeExact` already has the block from `GetBlockFromAddress` and now frees it in place with `MergeFreeBlocks(b)`, where it used to call `Free(position)` and search again. 14e065cbb2 2013-03 “Avoid dynamic_cast() for threads where possible.” · 6b7ff47617 2013-03 “Improve perf of __KernelExecutePendingMipsCalls().” · 537fbe4d7d 2013-02 “Minor perf gain in __KernelNextThread.” · 6e403be006 2013-01 “Skip a double map lookup.” · 90715c2ec2 2013-05 “Avoid a double lookup in FreeExact().”

### ppsspp-025 — Call the hottest handler directly, bypassing generic dispatch
Lens: branching-fast-paths · Source: PPSSPP
Principle: A generic dispatcher (decode the ID, look up a table, run common prologue logic) adds overhead to every call. When profiling shows one target dominates, resolve it at setup or compile time and call it directly. Do the same for event handlers that go through a generic "find all waiters of this type" path when the target is already known.
Why it is faster: —
Applies elsewhere: devirtualizing a hot interface call, special-casing the most common opcode in an interpreter, binding a direct function pointer for a hot RPC method.
Watch out: —
Seen in PPSSPP: In both JITs' `Comp_Syscall`, if `op == GetSyscallOp("FakeSysCalls", NID_IDLE)`, the emitted code calls `__KernelIdle`'s function directly, skipping `CallSyscall` ("~1.8% improvement" in one game). `hleScheduledWakeup` calls `__KernelResumeThreadFromWait` after a single `__KernelGetWaitID` check, where it used to go through `__KernelTriggerWait` (~1% in fast-forward). `__KernelIdle` calls `__KernelReSchedule` directly instead of the deferred `hleReSchedule`. 2758634a0f 2013-08 “Avoid overhead calling __KernelIdle().” · f552cb3370 2013-02 “Wake delayed threads directly, rather than looping.” · 955247b939 2012-12 “Minor opt: skip the deferred resched in idle.”

### ppsspp-026 — Gate diagnostics behind a cheap check
Lens: other · Source: PPSSPP
Principle: Timers, name lookups and log formatting in hot paths cost real time even when their output is discarded. Wrap them in a compile-time log-level test or a single runtime flag, so release builds skip the work entirely, not just its output. Output that must happen can go to an in-memory buffer drained by a separate writer thread.
Why it is faster: —
Applies elsewhere: `if (logger.isDebugEnabled())` around expensive message building, sampling-based tracing, stripping `console.debug` in production bundles.
Watch out: —
Seen in PPSSPP: `__KernelSwitchContext` only fetches `cur->GetName()` when `DEBUG_LEVEL <= MAX_LOGLEVEL` ("a couple percentage points" in a profile). `GPUCommon::InterpretList` only calls `time_update()`/`time_now_d()` when `g_Config.bShowDebugStats` is on, because games hammer `sceGeListUpdateStallAddr`. `GenericLog` returns at once when `bEnableLogging` is off. On Windows, `ConsoleListener` gained `hThread`, `hTriggerEvent` and a `logPending` buffer guarded by a critical section. The writer thread flushes after `LOG_LATENCY_DELAY_MS`. A later fix made it a single shared writer thread with static read/write positions. 0eb13142aa 2012-12 “Avoid calling thread->getName() in release builds.” · 9208f6389d 2013-04 “Avoid calling time_update in GE w/o debug stats.” · 528d81151a 2013-07 “Initial support for disabling logging” · 7fa89777fc 2012-12 “Attempt to put the console in a thread for speed?” · d6c94eb9a4 2013-06 “Keep only one actual console writer thread.”

### ppsspp-027 — Hold locks only around the shared data, or use an atomic access instead
Lens: concurrency · Source: PPSSPP
Principle: Lock scope should cover only the shared structure being touched, not I/O or unrelated work. For a single word that the hardware reads and writes atomically when aligned, an aligned load or store does the job without any mutex.
Why it is faster: —
Applies elsewhere: `std::atomic<uint64_t>` for counters instead of a mutex, narrowing `synchronized` blocks in Java, reading a file outside the lock and swapping the result in.
Watch out: —
Seen in PPSSPP: `__AudioDoState` now locks `section` only around `outAudioQueue.DoState(p)`. `GameInfoCache`'s `ReadFileToString` takes the mutex only for the resize and read into the shared string. `GPUCommon::GetTickEstimate`/`UpdateTickEstimate` dropped the `lock_guard` on x64, then used a 16-byte-aligned `curTickEst_` accessed via an `__m64` move on 32-bit SSE builds. 996bfdc1dd 2013-05 “Reduce locking in sceAudio.” · 09e58eadd3 2013-04 “Shorten the amount of time the lock is held when reading the ISO.” · 5c96782f29 2013-08 “Improve performance on x64 at least.” · 24203b1a5d 2013-08 “Improve performance on x86 as well using movq.”

### ppsspp-028 — Write the first contributor instead of clearing and then accumulating
Lens: memory-layout · Source: PPSSPP
Principle: Zeroing a buffer and then adding every source into it costs one extra full pass. Copy the first source directly and add only the others. Clear only when there was no source at all. Producers can also write straight into the consumer's ring buffer through returned pointers, skipping a temporary.
Why it is faster: —
Applies elsewhere: audio mixers, summing gradient buffers, compositing layers onto a canvas, reducing partial results from worker threads.
Watch out: —
Seen in PPSSPP: `__AudioUpdate` uses a `firstChannel` flag: the first active channel is assigned into `mixBuffer` and later channels are added; `memset` runs only if no channel played. `FixedSizeQueue::pushPointers()` returns up to two contiguous regions so `__AudioEnqueue` writes volume-adjusted samples straight into the channel queue. 7dac11922d 2013-06 “Some minor optimization of audio channel queues and mixing”

### ppsspp-029 — Throttle periodic housekeeping
Lens: laziness · Source: PPSSPP
Principle: Scanning a cache for stale entries every frame costs more than the memory it frees. Run eviction every Nth tick instead, ideally with an interval coprime to other periodic jobs so the scans don't all land on the same frame. Run it eagerly only at moments of pressure, such as right after creating a new large resource. When an invalidation hint arrives, mark entries suspect for revalidation instead of destroying them, and ignore over-broad "flush everything" hints.
Why it is faster: —
Applies elsewhere: TTL sweeps in Redis-like caches, incremental GC slices, cron-style compaction staggered across shards.
Watch out: —
Seen in PPSSPP: `TextureCache::Decimate()` and the vertex-array decimation return early until `decimationCounter_` reaches zero (`TEXCACHE_DECIMATION_INTERVAL 13`, "prime to other decimation intervals"). `FramebufferManager::DecimateFBOs(bool force)` keeps FBOs for `FBO_OLD_AGE = 10` frames normally, but is forced right after a new FBO is created and stays aggressive for `FBO_OLD_FORCE_RETAIN_FRAMES`. `TextureCache::Invalidate` with `force` used to `glDeleteTextures` and erase the entry; it now sets `STATUS_UNRELIABLE` so the next `SetTexture` rehashes it. `sceKernelDcacheWritebackAll` no longer calls `gpu->InvalidateCache(0, -1)`, since "Some games seem to use this a lot". 61d419b2ec 2013-08 “Don't scan for something to decimate every frame. Actually costs perf.” · 79e3c66514 2013-04 “Adjust FBO decimation to optimize for creation.” · 6ab6045baf 2013-01 “Throw away unused FBOs.” · ca943f070c 2013-02 “Don't delete cache invalidated textures.” · d657fc9fa2 2013-01 “Disable clearing all the GPU's cache on sceKernelCacheWritebackAll, the performance penalty was too big.”

### ppsspp-030 — Skip presenting a frame that hasn't changed
Lens: gpu-rendering · Source: PPSSPP
Principle: Copying the back buffer to the screen costs bandwidth and may block on vsync. Track a dirty flag on the displayed buffer and skip the copy/flip when nothing was drawn since the last present. This roughly halves the work for content that updates at half the display rate. When a frame does present late, adaptive vsync shows it immediately instead of waiting a whole refresh period.
Why it is faster: —
Applies elsewhere: damage tracking in compositors, skipping `requestAnimationFrame` work when state is unchanged, e-ink displays refreshing only on change.
Watch out: —
Seen in PPSSPP: `hleEnterVblank` calls `gpu->CopyDisplayToOutput()` only inside `if (gpu->FramebufferDirty())`. `GLES_GPU::FramebufferDirty()` now checks `vfb->dirtyAfterDisplay` in buffered mode too, which the commit says speeds up 30fps games. When `gl_extensions.EXT_swap_control_tear` is available, `GLES_GPU` calls `glstate.SetVSyncInterval(-interval)`; the negative interval enables tearing on late frames. Both the constructor and `BeginFrame()` do this. a41cd7bdba 2013-07 “Save some work when the displayed buffer isn't dirty.” · b5d5291a58 2013-07 “Avoid unnecessary buffer flips even when buffered rendering is on.” · 1256d5528d 2013-07 “Use EXT_swap_control_tear when available to ensure that VSync never slows things down”

### ppsspp-031 — Tell tiled GPUs the old contents are garbage
Lens: gpu-rendering · Source: PPSSPP
Principle: Mobile tile-based GPUs must load the previous framebuffer contents into tile memory unless told they are irrelevant. Clearing (or discarding) right after binding a render target lets the driver skip that full read of the old contents.
Why it is faster: —
Applies elsewhere: `loadOp: 'clear'` in WebGPU/Vulkan render passes, `glInvalidateFramebuffer` on Android, Metal's `MTLLoadActionDontCare`.
Watch out: —
Seen in PPSSPP: `GLES_GPU::SetRenderFrameBuffer` issues a clear right after `fbo_bind_as_render_target(vfb->fbo)` because "Tiled renderers are much faster if we clear FBOs before rendering". d9c9a9102a 2013-01 “Tiled renderers are much faster if we clear FBOs before rendering.”

### ppsspp-032 — Rotate through several buffers to avoid producer–consumer stalls
Lens: gpu-rendering · Source: PPSSPP
Principle: Overwriting a buffer the GPU (or any async consumer) may still be reading forces the driver to stall or copy. Cycle through a small ring of buffers so each write targets one that was last used several submissions ago.
Why it is faster: —
Applies elsewhere: triple buffering, ring-buffered uniform/staging buffers in Vulkan and WebGPU, double-buffered DMA in embedded audio.
Watch out: —
Seen in PPSSPP: `TransformDrawEngine` replaced its single `vbo_`/`ebo_` with arrays of `NUM_VBOS` created via `glGenBuffers(NUM_VBOS, …)` and a rotating `curVbo_` index. 5a2fc722f6 2013-01 “VBO buffer rotation. Helps a little on my Nexus S but perf is still bad.”

### ppsspp-033 — Coalesce many small uploads into one call
Lens: gpu-rendering · Source: PPSSPP
Principle: When all elements of an array-valued parameter changed, upload them in one call over a contiguous staging array instead of one call per element. Fall back to per-element updates only when just a few are dirty.
Why it is faster: —
Applies elsewhere: `bufferSubData` over one contiguous range, batching Redis `MSET`, sending one diff patch instead of many small ones.
Watch out: —
Seen in PPSSPP: `LinkedShader::updateUniforms()` converts every bone matrix into `float allBones[8 * 16]`. If every bone's `DIRTY_BONEMATRIX` bit is set it calls `glUniformMatrix4fv(u_bone, numBones, …)` once; otherwise it uploads only the dirty ones. 9af5499b21 2013-05 “Optimize bone matrix updates.”

### ppsspp-034 — Accumulate invalidations and apply them on switch, not by broadcast
Lens: laziness · Source: PPSSPP
Principle: Propagating every change to all consumers right away costs O(consumers) per change. Apply the change only to the active consumer, and keep a pending mask that is ORed into another consumer only when it becomes active.
Why it is faster: —
Applies elsewhere: lazy propagation in segment trees, per-subscriber version numbers instead of push notifications, dirty-on-activate for inactive tabs.
Watch out: —
Seen in PPSSPP: `ShaderManager::ApplyShader` used to loop over all of `linkedShaderCache` doing `dirtyUniforms |= globalDirty`. It now marks only `lastShader` and accumulates the bits in `shaderSwitchDirty` for the next shader that becomes active. The commit reports `ApplyShader` dropping from 2% to 0.5% of the profile. bf51291527 2013-02 “Don't translate bone weights to floats unnecessarily. Minor optimization to ApplyShader”

### ppsspp-035 — Normalize cache keys so irrelevant bits don't create variants
Lens: compiler-build · Source: PPSSPP
Principle: If a cache key includes fields that have no effect in the current mode, you get duplicate entries and repeated expensive builds. Include a sub-field in the key only when the feature that uses it is enabled.
Why it is faster: —
Applies elsewhere: leaving unused options out of build-cache keys, normalizing query strings before HTTP caching, canonicalizing ASTs before memoizing compilation.
Watch out: —
Seen in PPSSPP: `ComputeFragmentShaderID` now adds `texfunc` bits only `if (gstate.textureMapEnable & 1)`, adds the alpha-test function only when alpha test is on, and folds lighting mode and fog into derived booleans (`lmode`, `enableFog`). Equivalent states then share one compiled shader ("fewer shaders"). e04f5156ee 2013-01 “Fix a few graphics issues, optimize shader ID generation (fewer shaders)”

### ppsspp-036 — Constant-fold through known register values at code-generation time
Lens: jit-interpreters · Source: PPSSPP
Principle: A code generator that tracks which registers hold compile-time constants can evaluate operations on them during compilation, emitting nothing or a single immediate. It can also simplify identities such as `x + 0` and `x & 0`, and turn a load from a constant address into a direct memory reference.
Why it is faster: —
Applies elsewhere: constant propagation in any compiler, partial evaluation of templates, query planners folding constant predicates, shader compilers specializing on uniforms.
Watch out: —
Seen in PPSSPP: The x86 JIT `CompTriArith` calls `gpr.SetImmediate32(rd, doImm(a, b))` when both operands are immediate, treats an immediate 0 operand as `$zero`, sets `and` with 0 to the constant 0 and `or`/`add` with 0 to a `MOV`. `slti`/`sltiu` and `bitrev` on immediates are folded. In `CompLoadStore`, when `gpr.R(rs).IsImm()`, the access is emitted as `M(Memory::GetPointer(imm + offset))` with no address computation; ARM got the same, including in slowmem mode. ab9bea068c 2013-01 “Jit reg+reg compile time, and avoid flushing EDX.” · a7c6f46829 2013-01 “Optimize and/or 0 to just a mov in x86 jit.” · abde404c00 2013-02 “Optimize out some addu/etc. calls against imms.” · dede852c03 2013-02 “Optimize out slti in the x86 jit.” · 90e6f0b7df 2013-01 “Optimize static memory read/writes in jit as well.” · 8f3904d32d 2013-03 “armjit: Speed up imm addresses in slowmem mode.”

### ppsspp-037 — Choose richer target instructions and fuse instruction pairs
Lens: jit-interpreters · Source: PPSSPP
Principle: A one-to-one translation leaves throughput behind when the target ISA has more expressive instructions. Use three-operand address arithmetic (x86 `LEA`) for add-immediate, replace bit-by-bit loops with branch-free parallel bit tricks, and recognize adjacent source instruction pairs that map onto one target instruction.
Why it is faster: —
Applies elsewhere: peephole optimizers in compilers, fused multiply-add selection, recognizing `(x << n) | (x >> (32-n))` as a rotate, merging adjacent narrow loads into one wide load.
Watch out: —
Seen in PPSSPP: `addiu` becomes `LEA(32, rt, MDisp(rs, simm))` when `rs` is in a register. `bitrev` dropped a 32-iteration shift/and/or sequence for the five-step swap of bits, pairs, nibbles, bytes and halves ("Much less"). The ARM JIT spots `lwl`+`lwr` (or `swl`+`swr`) pairs with matching offsets, eats the second instruction and emits one unaligned `lw`/`sw`. It also fuses `mul.s`+`neg.s` into `VNMUL`. 9e479b4391 2013-02 “Optimize addi/addiu to just LEA when possible.” · 958d95a029 2013-02 “Make bitrev use less instructions in the x86 jit.” · 9152d2f2bb 2013-03 “Armjit: Optimise swl+swr and lwl+lwr cases that can be combined to a single sw or lw.” · f21218c3f9 2013-06 “Armjit: Combine mul.s + neg.s to VNMUL.”

### ppsspp-038 — Inline the common case, branch out to the slow path
Lens: jit-interpreters · Source: PPSSPP
Principle: Calling a safe helper for every memory access costs a call and register saves each time. Emit an inline range check that does the access directly when the address falls in the common region, and jump to the helper call only otherwise. Add a second cheap range for the next most common region.
Why it is faster: —
Applies elsewhere: inline caches in JavaScript engines, bounds-check-then-fast-path in WebAssembly runtimes, small-string optimization with heap fallback.
Watch out: —
Seen in PPSSPP: In the x86 JIT's non-fastmem mode, `CompITypeMemRead/Write` emit `CMP EAX, 0x08000000 / J_CC(CC_L)` and `CMP EAX, 0x0A000000 / J_CC(CC_GE)` guards around a direct `MOV` into `Memory::base`. Only out-of-range addresses fall through to `ABI_CallFunctionA(thunks.ProtectFunction(func))`. The scratchpad range `0x00010000–0x00014000` jumps back to the same fast `safe` label, and stores use `rs`'s host register directly as the address when possible. d5ae85201c 2013-01 “Optimize sw/lw even under safe memory.” · 37fb64ac83 2013-01 “Fast path scratchpad too, shouldn't be expensive.” · f5963df0dc 2013-01 “Optimize write to a single x64 op too.”

### ppsspp-039 — Keep hot globals next to a pointer already held in a register
Lens: jit-interpreters · Source: PPSSPP
Principle: On load/store architectures, reaching a global costs extra instructions to build its address. Move frequently touched counters into the context struct that generated code already addresses through a pinned base register, so access becomes one load or store at a fixed offset. Likewise, load loop-invariant constants into registers once, before the dispatch loop.
Why it is faster: —
Applies elsewhere: thread-local "current context" structs in interpreters (Lua `L`, CPython frame), passing a context pointer instead of using globals on ARM/RISC-V firmware.
Watch out: —
Seen in PPSSPP: `CoreTiming`'s global `downcount` became `currentMIPS->downcount` "to save a couple of clocks in every ARM JIT block", since R10 already holds the `MIPSState` pointer. That also allowed block linking. In `ArmAsmRoutineManager::Generate`, the `MOVI2R` loads of `Memory::base`, `mips` and the block-pointer table moved out of the inner dispatcher loop. The address mask became a single `BIC`, and the found/not-found branch became conditional execution (`SetCC(CC_EQ)`). 674911ddba 2013-01 “Move downcount into MIPSState for efficiency, enable block linking.” · 81c6c4805d 2013-01 “Small dispatcher optimizations, cleanup.”

### ppsspp-040 — Spill fewer registers: compute in the destination, flush only caller-saved
Lens: jit-interpreters · Source: PPSSPP
Principle: Every temporary and every spill around a call costs memory traffic. If the destination doesn't alias the sources, accumulate directly into it. Before calling a helper, flush only the registers the ABI says the callee may clobber, not the whole register file. Expose every register the CPU actually has to the allocator.
Why it is faster: —
Applies elsewhere: register allocators in any compiler backend, hand-written assembly kernels, writing hot C so locals stay in registers across calls.
Watch out: —
Seen in PPSSPP: The x86 VFPU `Comp_VDot`/`Comp_VecDo3` check `DestRegOverlaps()` and, if there is no overlap, map the destination as `tempxreg`, avoiding XMM0 and the final move. `ArmRegCache::FlushBeforeCall()` flushes only R2, R3 and R12 ("R4-R11 are preserved"), where it used to call `FlushAll()`. The ARM FPU allocator unlocks the other 16 VFP registers when NEON is present. 1c4c5e718b 2013-02 “Optimize VDot and VecDo3 to avoid temporaries.” · 3b9e6243eb 2013-07 “Only flush the required registers on function calls (only implemented for real on ARM)” · 637d75f47d 2013-06 “Unlock the other 16 regs that are available for NEON.”

### ppsspp-041 — Translate prefix-modified ops natively instead of falling back
Lens: jit-interpreters · Source: PPSSPP
Principle: If a fast path bails out to a slow generic handler whenever a modifier is present, even trivial modifiers turn hot code slow. Resolve modifiers known at compile time (operand swizzles, writemasks) into the operand selection itself, and fall back only when the modifier is truly unknown.
Why it is faster: —
Applies elsewhere: SQL engines compiling simple `CAST`s inline instead of to a generic path, regex engines handling common flags in the DFA, shader compilers folding swizzles into register selection.
Watch out: —
Seen in PPSSPP: `Comp_VDot` and `Comp_VecDo3` used to call `Comp_Generic(op)` whenever `js.MayHavePrefix()`. They now bail only on `js.HasUnknownPrefix()` and fetch operands through `GetVectorRegsPrefixS/T/D`, which apply known prefixes at compile time. 6855398add 2013-02 “Support known prefixes in the vfpu jit.”

### ppsspp-042 — Deduplicate constants and flush instruction caches incrementally
Lens: jit-interpreters · Source: PPSSPP
Principle: In generated code, identical constants should share one literal-pool slot, which shrinks code and cache footprint. After emitting new code, invalidate only the newly written range of the instruction cache, not the whole code region, and don't yield the thread while doing it.
Why it is faster: —
Applies elsewhere: string interning in compilers, constant-pool dedup in JVM class files, incremental icache maintenance in self-modifying embedded firmware.
Watch out: —
Seen in PPSSPP: `ARMXEmitter::FlushLitPool()` searches earlier entries for an equal `val`, reuses its `loc`, and backpatches the LDR offset to it. `FlushIcache()` now calls `FlushIcacheSection(lastCacheFlushEnd, code)` and tracks `lastCacheFlushEnd`; the trailing `SLEEP(0)` was removed. 9f0bb8cf52 2013-03 “armv6: Reuse constants in literal pool.” · e34288070c 2013-01 “Don't flush the icache from the start, also don't sleep. May help JIT slowdowns.”

### ppsspp-043 — Update edge functions incrementally while rasterizing
Lens: algorithms · Source: PPSSPP
Principle: Evaluating a linear function from scratch at every grid point repeats multiplies. Because it is linear in x and y, compute it once at the start of a row and add a constant step per pixel (another constant per row). Reject degenerate or back-facing inputs before entering the loop.
Why it is faster: —
Applies elsewhere: scanline polygon fill in canvas/SVG renderers, DDA line drawing, incremental interpolation in audio resamplers, forward differencing for curves.
Watch out: —
Seen in PPSSPP: The software rasterizer's `DrawTriangle` called `orient2d()` three times per pixel. It now computes `w0_base`/`w1_base`/`w2_base` once, adds per-row deltas, and subtracts per-pixel deltas (`orient2dIncX`/`orient2dIncY`). Triangles not in counter-clockwise order are dropped up front. 0c6a9dcf50 2013-07 “softgpu: Optimization, part 1.” · b22a82d350 2013-07 “softgpu: Optimization, part 2.” · bacadefde4 2013-07 “softgpu: Optimization, part 3.”

### ppsspp-044 — One system call instead of several for the same answer
Lens: io · Source: PPSSPP
Principle: Filesystem helpers often chain calls: `exists()` then `stat()`, or open, sniff and close, then reopen. Each one is a syscall plus a path lookup. Make one `stat` and derive everything from its result, and hand an already-open file handle to the code that will read it.
Why it is faster: —
Applies elsewhere: using `fstat` on an open fd, `os.scandir` instead of `listdir` plus per-file `stat`, reusing an HTTP keep-alive connection after a HEAD probe.
Watch out: —
Seen in PPSSPP: `File::GetSize()` calls `stat64` once and checks `IsDirectory(file_info)` on the result, where it used to call `Exists()` (its own stat) first. `constructBlockDevice` reads the 4-byte magic, `fseek`s back to 0, and passes the `FILE*` to `CISOFileBlockDevice`/`FileBlockDevice`, which no longer reopen by filename. a9e7a9ce3d 2013-07 “Improve performance of File::GetSize().” · 61b510b753 2013-06 “Open ISOs very slightly faster by not reopening”

### ppsspp-045 — Suspend redraw and preallocate during bulk UI inserts
Lens: platform · Source: PPSSPP
Principle: UI widgets that repaint and grow storage on every insert become quadratic when filled with thousands of rows. Turn off redraw, reserve capacity up front, insert everything, then re-enable redraw and repaint once.
Why it is faster: —
Applies elsewhere: building a DOM fragment off-document before attaching, `beginUpdate/endUpdate` on grid controls, `reserve()` before bulk vector pushes.
Watch out: —
Seen in PPSSPP: `SymbolMap::FillSymbolListBox`/`FillSymbolComboBox` send `WM_SETREDRAW FALSE` and `LB_INITSTORAGE`/`CB_INITSTORAGE` sized to `entries.size()` before the loop, then `WM_SETREDRAW TRUE` plus one `RedrawWindow`. 82dd2780f5 2013-05 “Improve perf when loading a large debug map file.”

### ppsspp-046 — Make sure the optimizer and fast paths are actually on
Lens: compiler-build · Source: PPSSPP
Principle: Build settings fail silently: an `-O2` that never reaches the compiler, or a library's fast path gated on GCC-only macros that MSVC never defines. Check the effective flags. Always optimize third-party hot code you never debug, even in debug builds.
Why it is faster: —
Applies elsewhere: verifying `CMAKE_BUILD_TYPE` in CI, building dependencies in release while debugging your own code, checking `__SSE2__`/`_M_X64` equivalents across compilers.
Watch out: —
Seen in PPSSPP: For BlackBerry, `-O2` "wasn't taking effect" in `CMakeLists.txt`, and fixing it improved performance "significantly". The project files force optimization for CityHash even in debug, taking it "from 10% to 0.34% in a debug build". For savestates, `snappy-stubs-internal.h` now recognizes `_M_X64`/`_M_IX86`, so MSVC builds use the 64-bit and unaligned-load paths. 42bf90728f 2013-05 “-O2 wasn't taking effect. Improves performance significantly for Blackberry.” · 3fdbc2ea21 2013-04 “Mark cityhash to always be optimized, even in debug.” · abc9b84bac 2013-07 “Improve performance of savestates on x86.”

### ppsspp-047 — Reset state by copying a prebuilt template
Lens: memory-layout · Source: PPSSPP
Principle: If an object goes back to the same initial state over and over, build that state once and restore it with a single `memcpy`. A field-by-field init loop costs branches, address arithmetic and scattered stores every time. A block copy is one streaming operation.
Why it is faster: —
Applies elsewhere: resetting per-request context structs in a web server, re-arming parser state machines, zeroing game-entity pools between levels, restoring device register shadows in firmware.
Watch out: —
Seen in PPSSPP: The JIT resets its register caches at the start of every compiled block. `FPURegCache::Start()` used to loop over every MIPS/x86 register assigning location, away and locked flags one at a time. It now fills `regsInitial`/`xregsInitial` once in `SetupInitialRegs()` and then just does `memcpy(regs, regsInitial, sizeof(regs))`. The commit says this cut the function by 97% in tests, where it had been 8% of total time. The ARM FPU cache got the same treatment. An earlier step had already swapped per-register `GetDefaultLocation()` calls for `memset` plus an incrementing offset. e8091dce44 2013-10 “Speed up FPURegCache::Start() on x86.” · 2751da1cec 2013-09 “Cut down on work in regcache init on x86.” · 3001866d18 2014-03 “Skip flushing FPU/VFPU regs if none were allocated.”

### ppsspp-048 — Keep a “nothing to do” flag so cleanup can return early
Lens: laziness · Source: PPSSPP
Principle: When a cleanup pass (flush, write-back, sync) has to scan a whole structure but is usually a no-op, set a boolean whenever something is actually used and check it before scanning. In the common case the full walk becomes a single test.
Why it is faster: —
Applies elsewhere: a dirty flag on a write-back cache, a “pending” bit before walking a timer wheel, skipping a DOM diff when no state changed, skipping fsync when nothing was written.
Watch out: —
Seen in PPSSPP: FPU and VFPU registers are much rarer than integer registers in game code. Both `ArmRegCacheFPU` and x86 `FPURegCache` now set `pendingFlush = true` on any mapping, and `FlushAll()` returns immediately when it is false. The commit reports about 1% in tests. 3001866d18 2014-03 “Skip flushing FPU/VFPU regs if none were allocated.”

### ppsspp-049 — Do the cheap rejection test before taking the lock
Lens: concurrency · Source: PPSSPP
Principle: If most calls to a function end with “nothing to do”, run that check before acquiring a mutex, using an unsynchronized counter or flag that is safe to read slightly stale. The common path then never pays for the lock or the shared-cache-line traffic.
Why it is faster: —
Applies elsewhere: a logger that checks the level before locking the sink, a metrics library that skips disabled counters, a breakpoint or feature-flag check in a hot loop, a thread pool that runs tiny jobs inline.
Watch out: —
Seen in PPSSPP: `GenericLog()` took `log_lock_` before testing the level. It now tests `level > log->GetLevel() || !IsEnabled() || !HasListeners()` first, and `HasListeners()` reads a cached bool rather than calling `m_listeners.empty()`. The commit says this took GenericLog from 13.8% to 1.5% in one game. The GE debugger breakpoint check reads a `breakPCsCount` mirror before locking `breaksLock`. `ThreadPool::ParallelLoop` now locks only when the range is large enough to split, so small loops run inline with no lock. ca938bf353 2014-03 “Cut down on locking in DEBUG_LOG.” · 4b8ace8a10 2013-10 “Small optimization to make breakpoint checks fast.” · 46bba78feb 2013-11 “Avoid a mutex lock on a small parallel loop.”

### ppsspp-050 — Turn impossible-case checks on hot accessors into debug asserts
Lens: branching-fast-paths · Source: PPSSPP
Principle: If an accessor is called constantly and its arguments are valid by construction, move the validation into a debug-only assert and inline the function. Release builds lose the compare, the branch and the cold error path, which keeps the inlined body small.
Why it is faster: —
Applies elsewhere: bounds checks inside a container's internal iterator, handle-table lookups in an OS kernel, ECS component getters, `get_unchecked` in Rust once the indices are proven.
Watch out: —
Seen in PPSSPP: `KernelObjectPool::GetFast()` logged and returned 0 on a bad handle. It now uses `_dbg_assert_(realHandle >= 0 && realHandle < maxCount && occupied[realHandle])`, and `__GetCurrentThread()`, which is called very often, became `inline`. In the same spirit, replaced functions use `GetPointerUnchecked` because their addresses are already known to be valid. 738dfc84e4 2013-11 “Speed up __GetCurrentThread(), it's called a lot.”

### ppsspp-051 — Split a dispatcher into specialized variants chosen once
Lens: branching-fast-paths · Source: PPSSPP
Principle: If a generic entry point tests flags and options on every call, but those flags are fixed per callee, pick the right specialized variant when the call is set up. Each call then runs only the code it needs and skips the repeated table lookups.
Why it is faster: —
Applies elsewhere: RPC routers that bind a handler with its middleware pre-resolved, syscall tables with per-entry fast paths, an interpreter that resolves a method once at link time.
Watch out: —
Seen in PPSSPP: `CallSyscall(op)` looked up `moduleDB[modulenum].funcTable[funcnum]` on every call, read timing state for debug stats and tested the dispatch and interrupt flags. JIT-compiled code now resolves the `HLEFunction*` at compile time through `GetSyscallInfo()` and calls `CallSyscallWithoutFlags` or `CallSyscallWithFlags` directly, using a new one-argument `ABI_CallFunctionP` emitter helper. 732ae13ebb 2013-11 “Fast path CallSyscall where possible.”

### ppsspp-052 — Keep hot state, and values derived from it, in registers
Lens: jit-interpreters · Source: PPSSPP
Principle: A flag written by one instruction and read by the next costs a store and a load if it lives in memory. Let the register allocator manage it like any other variable. Likewise, if a value is repeatedly turned into the same derived form, such as a guest address into a host pointer, keep that form in a register and remember its state, converting back only when the original is needed.
Why it is faster: —
Applies elsewhere: a VM that holds its flags or accumulator in register variables, caching `base + offset` for a repeatedly indexed struct, memoized resolved handles.
Watch out: —
Seen in PPSSPP: The ARM JIT made `fpcond` a pseudo-register, `MIPS_REG_FPCOND`. `c.cond.s` writes it through `gpr.MapReg(..., MAP_DIRTY|MAP_NOINIT)` and `bc1t`/`bc1f` `TST` it directly, which removes an `STR`/`LDR` pair from every compare-and-branch. The cycle `downcount` got a dedicated `DOWNCOUNTREG`. On x86-32, EBX, which had been left out, was added to the allocation order. `ArmRegCache::MapRegAsPointer()` converts a MIPS register in place with `BIC` to clear the top bits and `ADD R11` for the memory base, then marks it `ML_ARMREG_AS_PTR`. Consecutive `lv.s`/`sv.s` off the same base register then become a single `VLDR/VSTR [ptr, #offset]`. If the value is needed again as an integer, the register is reloaded from the context. 17074f5a7f 2013-11 “Cache fpcond in a register to avoid store/load between compare and branch” · dca457e6df 2013-11 “Optimize multiple sv.s and lv.s calls on ARM.” · a528921f3c 2014-11 “x86 JIT: EBX was free in 32-bit mode, let's use it in the regcache.” · 58c39a38ee 2013-11 “ARM regcache: Add mechanism to keep registers converted to pointers around”

### ppsspp-053 — Strength-reduce division
Lens: algorithms · Source: PPSSPP
Principle: Division takes tens of cycles. With a known power-of-two divisor, the quotient is a right shift and the remainder is an AND with `d-1`, and a known non-zero divisor removes the zero guard. For a repeated but arbitrary divisor, compute `1/d` once when it changes and multiply in the hot path, merging constant scale factors into one constant.
Why it is faster: —
Applies elsewhere: ring-buffer wrap with `& (size-1)`, hash bucket selection, vector normalization, animation easing with fixed durations, MCUs without a hardware divider.
Watch out: —
Seen in PPSSPP: The ARM JIT's `divu` checks `gpr.IsImm(rt) && (imm & (imm-1)) == 0` and, when it holds, emits `ANDI2R(HI, rs, d-1)` and `LSR(LO, rs, log2 d)`. The commit notes “These really happen.” When the divisor is any known non-zero immediate, the `CMP rt,0` guard sequences are skipped too. The SAS ADSR envelope stores `invDuration_` when it is set, and `computeExpCurveAt(index, invDuration)` multiplies rather than divides for every step. `PSP_SAS_ENVELOPE_HEIGHT_MAX / expCurveReference` became the single float constant `expCurveToEnvelopeHeight`. a3a061a69f 2013-11 “armjit: Optimize a division by a power of two.” · b30495c8e8 2013-11 “Reapply the "duration" SasAudio optimization.”

### ppsspp-054 — Build constants from values you already have
Lens: jit-interpreters · Source: PPSSPP
Principle: Loading an arbitrary 32-bit constant can take two instructions or a literal-pool read. If the constant fits an immediate encoding, or is close to a value already in a register, derive it with one `MOV`/`MVN`, one add from that register, or a pair of short adds.
Why it is faster: —
Applies elsewhere: compiler constant rematerialization, embedded assembly for RISC cores with short immediates, size-optimized shellcode or boot code.
Watch out: —
Seen in PPSSPP: `ArmRegCache::SetRegImm` first tries `TryMakeOperand2_AllowInverse`. If that fails, it scans the registers it knows hold immediates for one within 255 and emits a single `ADD` or `SUB` from it, which replaces a MOVW/MOVT pair. `ADDI2R` splits a 16-bit addend into two rotated-immediate `ADD`s (or `SUB`s) and so needs no scratch register. c63560c0dd 2013-11 “armjit: Try to find imms to optimize a reg load.” · 1bf83efe9e 2013-11 “ARM optimization in ADDI2R: Dual adds instead of MOVI2R, ADD when possible”

### ppsspp-055 — Use semantics to emit less code
Lens: jit-interpreters · Source: PPSSPP
Principle: Generated code only has to preserve observable behavior. Calling conventions say which registers are dead at a return, so their cached values can be discarded without writing them back. An operation that has no effect in your model can compile to zero instructions instead of falling back to a generic handler.
Why it is faster: —
Applies elsewhere: compilers clobbering caller-saved registers, binary translators trimming spill code, stripping disabled tracing at build time, dropping no-op middleware.
Watch out: —
Seen in PPSSPP: On `jr ra`, both JITs can call `DiscardR` on A0–T7, T8 and T9, which marks them non-dirty and unmapped without storing them. Some games do not follow the MIPS ABI, so the commit puts this behind an ini-only option, `DiscardRegsOnJRRA`, that is off by default. The MIPS `cache` instruction went to `Comp_Generic`, which calls the interpreter's `Int_Cache`, and that function did nothing. It showed up in profiles. A new `Comp_Cache` on x86 and ARM emits nothing. 5a95e267fb 2013-11 “Add an optimization to discard registers at the end of functions when possible.” · 455a7e090d 2013-11 “Compile the cache instruction to nothing.”

### ppsspp-056 — Pick operand order and instruction form to avoid extra loads
Lens: jit-interpreters · Source: PPSSPP
Principle: Many operations can be rewritten by swapping operands and flipping the condition (a<b is b>a), or replaced by an equivalent instruction with more flexible operands. Choose the form that uses values already in registers. That avoids loads, flushes and copies.
Why it is faster: —
Applies elsewhere: instruction selection in compilers, hand-written SIMD that avoids a `mov` by putting the destructive operand in the right place, peephole optimizers.
Watch out: —
Seen in PPSSPP: For `slt`/`sltu`, the x86 JIT now swaps `lhs`/`rhs` and uses `SwapCCFlag(cc)` when the left operand is an immediate, or when `MIPSAnalyst::IsRegisterUsed` shows the other register is the better one to load. `addu rd, rs, rt`, with all three distinct and in registers, becomes `LEA rd, [rs+rt]`, which saves a `MOV`. d0a2ced2f9 2014-11 “x86jit: Flip cc in stl* to avoid reg loads.” · 28ca8d4818 2014-11 “x86 jit: Use LEA to emulate addu but only when it can save a few bytes”

### ppsspp-057 — Keep short vectors in SIMD registers and move them whole
Lens: simd · Source: PPSSPP
Principle: Programs that work on 2–4-wide vectors should hold each vector in one SIMD register and load or store it with one wide or multi-register transfer. Scales, compares and write-backs become single packed instructions instead of four scalar ones, with no gathering or scattering of lanes and fewer address computations.
Why it is faster: —
Applies elsewhere: vec4 and quaternion math in `__m128`, ARM `LDM`/`STM` for struct copies, color blending in image processing, CPU rasterizers.
Watch out: —
Seen in PPSSPP: The ARM JIT's `lv.q`/`sv.q` check whether the four mapped VFP registers are consecutive and, if so, emit one `VLDMIA`/`VSTMIA` rather than four `VLDR`s. New emitter ops support this. On x86 with fast memory, `lv.q`/`sv.q` map the whole vector to one XMM register and use a single `MOVAPS`. The x86 JIT added SIMD mapping to the VFPU register cache (`TryMapRegsVS`, `TryMapDirtyInInVS`). Ops like `vscl` become `SHUFPS` to broadcast the scalar plus one `MULPS`. `vcmp` compares all lanes at once and builds the CC bits from an aligned mask table. Register-file write-backs use packed stores where possible. The software rasterizer moved alpha-blend factors, modulation and color conversion onto `__m128i` lanes in the same way, for example broadcasting alpha with `_mm_shuffle_epi32(src, _MM_SHUFFLE(3,3,3,3))`. 30a6a5d10f 2014-03 “ARMJIT: Implement VLDM/VSTM load/store combinations and use in armjit.” · b5ee47a80c 2014-11 “x86 jit: SIMD-ify lv.q and sv.q” · 804de50711 2014-11 “x86 jit: SIMD-ify VFPU register file writebacks where possible” · 2758e8fa3c 2014-11 “x86jit: Optimize vcmp for single and simd.” · 07ca96e226 2014-03 “softgpu: Use SSE in alpha blending.”

### ppsspp-058 — Fuse neighbouring operations that share an expensive subresult
Lens: algorithms · Source: PPSSPP
Principle: When two consecutive operations both compute the same costly function of the same input, handle them together and compute it once. The cost of the shared work drops by half or more.
Why it is faster: —
Applies elsewhere: using `sincos()` instead of separate `sin`/`cos`, query planners merging scans of the same table, GPU shader CSE across draw passes.
Watch out: —
Seen in PPSSPP: The x86 JIT's `Comp_VRot` checks whether the next instruction is also a `vrot` with the same source angle. If it is, it calls the sin/cos helper once and runs `CompVrotShuffle` for both destinations, then skips the second instruction. ac772f25ff 2014-11 “x86 JIT: Join adjacent vrot calls together to avoid redundant sin/cos calls.”

### ppsspp-059 — Hold rarely used heavy members behind a pointer
Lens: memory-layout · Source: PPSSPP
Principle: An embedded container in every element of a large array costs construction, destruction and memory for every element, even when it is almost always empty. Store a nullable pointer and allocate only on first use. The common element becomes trivially constructible and smaller.
Why it is faster: —
Applies elsewhere: optional metadata on AST nodes, per-row overflow lists in a database page, “extra” blocks on hot ECS components, `Box<Vec<T>>` for cold fields in Rust.
Watch out: —
Seen in PPSSPP: `JitBlock` had a `std::vector<u32> proxyFor`, used only by proxy blocks. It became `std::vector<u32> *proxyFor`, allocated in `ProxyBlock()` and deleted in `DestroyBlock()`. The commit comment says this avoids a generated constructor and destructor and “dog slow performance in debug”. 58237d976f 2014-03 “Fix performance issue in BlockCache due to an instance of std::vector in every block”

### ppsspp-060 — Index the lookups you do repeatedly
Lens: algorithms · Source: PPSSPP
Principle: A linear scan is fine while the collection is small, but it becomes the bottleneck as the collection grows. Keep a map keyed by what you search on: a hash map for point lookups, an ordered map with `lower_bound` to jump to the next candidate. Lookups go from O(n) to O(1) or O(log n).
Why it is faster: —
Applies elsewhere: adding a DB index on a foreign key, keeping a `Map` beside an array in a UI store, interval lookup in a symbolizer, switching `std::map` to `unordered_map` when ordering is unused.
Watch out: —
Seen in PPSSPP: Looking up a proxy block by address scanned `proxyBlockIndices_`. It now uses a `std::multimap<u32,int> proxyBlockMap_` with `equal_range`, and entries are removed on destroy. The function-hash lookup `LookupHash` changed from a loop to `hashMap.find`. The function scanner calls `FindPossibleFunctionAtAfter()` (`lower_bound`) to skip ahead, so it no longer runs a locked `GetSymbolInfo` at every address. `decoderMap_` and similar maps became `unordered_map`. 4853a1b7a0 2014-10 “jit: Optimize proxy block lookup from address.” · 72eb15f282 2014-05 “Speed up debug build hashfunc lookup.” · 8948990871 2014-02 “Optimize searching for existing symbols.” · e22fed8b9f 2014-10 “Use unordered_map where it makes sense.”

### ppsspp-061 — Don't throw away caches when they are still valid
Lens: laziness · Source: PPSSPP
Principle: Derived caches (compiled code, uploaded textures, shaders) are expensive to rebuild. Clear them only when their source really changes, not as a side effect of operations that leave the source intact. Saving a snapshot reads state and does not need to invalidate anything.
Why it is faster: —
Applies elsewhere: not flushing a CDN or ORM cache on read-only admin actions, keeping a build cache across a `git stash`, preserving a JIT code cache across a heap snapshot.
Watch out: —
Seen in PPSSPP: Savestates used to clear the JIT. Now `SaveAndClearEmuHackOps()` temporarily restores the original opcodes in emulated RAM for serialization, and `RestoreSavedEmuHackOps()` puts the block-entry markers back, so compiled blocks survive. GPU `DoState` clears the texture cache, vertex arrays and FBOs only when `p.mode == MODE_READ`, that is, when loading. 394d91371f 2013-09 “Avoid clearing the jit cache on savestate.” · 0aa263511d 2013-09 “Don't clear the GPU caches on savestate, just load.” · 040a6d1745 2014-10 “jit: Improve performance of clearing jit.”

### ppsspp-062 — Precompute shift/mask pairs for table-driven decoding
Lens: lookup-tables · Source: PPSSPP
Principle: A decoder that walks nested tables should store each level's field extraction ready to use: a shift and a ready-made mask, not a bit count to turn into a mask each step. Drop sentinel cases that never occur. The inner step becomes `(op >> shift) & mask` and an index.
Why it is faster: —
Applies elsewhere: instruction decoders in disassemblers, protocol parsers with nested opcode spaces, trie-based routers.
Watch out: —
Seen in PPSSPP: `MIPSGetInstruction()` used `encodingBits[][2]` = {shift, bits} and checked “Rese” and other cases that never occur in the tables. It now uses an `EncodingBitsInfo{shift, mask}` array, with mask computed in the constructor, and typed `Instruc`/`Inval` markers. The commit reports 35% faster lookups, which matters for the interpreter and module loading. f300f46b9c 2014-03 “Speed up MIPSGetInstruction() by 35%.”

### ppsspp-063 — Inline the fast check, keep the slow body out of line
Lens: branching-fast-paths · Source: PPSSPP
Principle: For a function called on every item whose usual answer is “nothing changed”, put the quick test in an inline header wrapper and the rest in an out-of-line function. Callers then pay a compare instead of a call in the common case, and code size stays small.
Why it is faster: —
Applies elsewhere: `if (likely(cached)) return v; return slow_path();` in kernels, the fast path of a refcount release, the cache-hit path of a memoizer.
Watch out: —
Seen in PPSSPP: Every primitive called `FramebufferManager::SetRenderFrameBuffer()`. The inline version now just updates `last_frame_render` and the dirty flags when `!gstate_c.framebufChanged && currentRenderVfb_`, and calls `DoSetRenderFrameBuffer()` otherwise. `EstimatePerVertexCost()` and `SetupVertexDecoder()` moved into headers the same way. e7639f666c 2014-03 “Ensure stuff run on every prim are inlined.”

### ppsspp-064 — Stay in the fast loop for common control-flow events
Lens: other · Source: PPSSPP
Principle: If an inner loop exits to an outer scheduler to “recalculate” on an event you assumed was rare, measure whether it really is. If it is common, recompute the loop budget inline and keep going, so you don't pay the exit and re-entry cost each time.
Why it is faster: —
Applies elsewhere: event loops that handle cheap events inline rather than yielding, bytecode interpreters that handle a common jump without leaving the dispatch loop, packet-processing fast paths.
Watch out: —
Seen in PPSSPP: GE display-list `JUMP` set `downcount = 0`, which forced `FastRunLoop` to exit. Games use jumps heavily to move around vertex data. The handler now recomputes `downcount` from `currentList->stall` and continues. The commit reports a few percent in FF4. 4fbb245382 2014-03 “Avoid leaving the fast runloop on jumps.”

### ppsspp-065 — Write code the compiler can keep in registers
Lens: compiler-build · Source: PPSSPP
Principle: Writing through a member pointer or into struct fields inside a loop makes the compiler assume aliasing and reload or store every iteration. Calling an out-of-line function with a SIMD value can force it through memory. Copy members into locals and write them back once, use simple strides, and define tiny SIMD helpers in headers so values stay in registers.
Why it is faster: —
Applies elsewhere: serializers that write through a local cursor, IIR filters with state in locals, JS loops caching `this.x`, header-only vector math libraries (GLM, DirectXMath).
Watch out: —
Seen in PPSSPP: `IndexGenerator` loops now use `u16 *outInds = inds_; … inds_ = outInds;`. The commit notes that walking arrays with member variables “throws off the MSVC optimizer”. Loops step `i += 3` over `numVerts`. The VAG ADPCM `DecodeBlock` holds `s_1`/`s_2` and the predictor coefficients in locals, and inlines `DecodeSample` (the comment says “to avoid bouncing to memory”). `ComputeFragmentShaderID` builds `int id0` locally, which drops a `memset` and repeated `id->d[0] |=` stores, and it folds the enable bit into the existing `if`. The commit reports 12.5% less time for that function. `Vec3/Vec4::FromRGB/ToRGB/FromRGBA/ToRGBA`, which are SSE unpack/convert sequences, moved from `Math3D.cpp` into `Math3D.h`. The commit message says this is so they don't have to store the value. cd57e8a09d 2013-10 “never walk arrays with member variables” · af46d17b94 2013-10 “Optimize index generator” · 0de71f27cb 2014-01 “Optimize the VAGADPCM decoder.” · 1209bc7ba4 2014-03 “Optimize ComputeFragmentShaderID() a bit.” · 416df17088 2014-03 “Inline From/ToRGB(A) to avoid losing SSE.”

### ppsspp-066 — Filter redundant state changes before they reach the driver
Lens: gpu-rendering · Source: PPSSPP
Principle: Graphics and device APIs often do validation or flush work on every call, even when the value is unchanged. Shadow the current state on the CPU and issue only real changes. When switching configurations, apply just the difference between the old and new sets.
Why it is faster: —
Applies elsewhere: WebGL state wrappers, React's diffing of DOM attributes, a GPIO driver that writes a register only when the pin value changes, avoiding redundant CSS style writes.
Watch out: —
Seen in PPSSPP: When a shader changes, `LinkedShader::use(vertType, previous)` computes `enable = attrMask & ~previous->attrMask` and `disable = ~attrMask & previous->attrMask` and calls `glEnable/DisableVertexAttribArray` only for those bits. The D3D9 `dx_state.h` wrappers now compare before calling `SetRenderState`/`SetSamplerState` (about 3%). `fbo.cpp` tracks `currentDrawHandle_`/`currentReadHandle_` and skips duplicate `glBindFramebuffer` calls. 0a4750909e 2013-10 “Try to avoid unnecessary calls to glEnable/DisableVertexAttribArray” · d987cfea37 2014-08 “d3d: Cache sampler/render state like in GL.” · f1ec201dbc 2014-08 “Cache bound fbos, allow unbinding only read.”

### ppsspp-067 — Intersect dirty bits with an “applicable” mask computed once
Lens: laziness · Source: PPSSPP
Principle: When an update routine tests many dirty flags and then checks whether each target exists, precompute a bitmask of the targets that exist when the object is created. At update time, AND it with the dirty mask and exit early if the result is zero.
Why it is faster: —
Applies elsewhere: render-pass uniform uploads, ECS systems that skip entities without a given component (archetype masks), incremental build steps limited to the affected outputs.
Watch out: —
Seen in PPSSPP: `LinkedShader` computes `availableUniforms` at link time from which `u_*` locations are not -1, looping bones only up to `numBones`. `UpdateUniforms()` then works only on `dirtyUniforms & availableUniforms`. 54217deb16 2013-11 “Speed up UpdateUniforms a little”

### ppsspp-068 — Vectorize per-pixel bit shuffling and bulk hashing
Lens: simd · Source: PPSSPP
Principle: Pixel-format conversion, swizzle copies and checksum loops apply the same shift/mask/add to every element. SIMD processes 8–16 elements per instruction. When the compiler won't vectorize the loop, write it with intrinsics or assembly.
Why it is faster: —
Applies elsewhere: image decoders (RGB565→RGBA8888), base64 and UTF-8 transcoders, checksumming in storage engines, tile copies for GPU upload.
Watch out: —
Seen in PPSSPP: `ConvertColors` handles 4444/5551 in eight-pixel `__m128i` chunks with masked shifts and finishes the rest in a scalar loop. `Unswizzle16` copies 16-byte texture block rows with aligned SSE loads and stores. The texture-cache quick hash multiplies and adds two vectors per iteration with `_mm_mullo_epi16`. On ARM, `QuickTexHashNEON` was rewritten in inline asm because “gcc couldn't optimize it”, and a NEON `XXH32` (`ReliableHashNEON`) takes at least 40% less time. f42cd11ddb 2013-10 “Speed up color conversion using SSE.” · 8bf015fe16 2014-03 “texcache: SSE optimized version of the most common case of Unswizzle” · 82761de992 2013-10 “Improve the texture cache hash on x86.” · 2dfa2379f4 2013-11 “Rewrite QuickTexHashNEON directly in asm.” · b800762ceb 2014-03 “Add a NEON-optimized version of XXH32.”

### ppsspp-069 — Let the consumer read with a stride instead of repacking
Lens: memory-layout · Source: PPSSPP
Principle: If data sits inside a wider buffer, many APIs (GL unpack row length, `memcpy2D`, image views) can read it in place with a pitch. Passing the stride avoids an extra repacking copy and a second pass over the data. Copying directly between overlapping regions, when the API allows it, avoids a temporary too.
Why it is faster: —
Applies elsewhere: uploading a sub-rectangle of an image without cropping it on the CPU, NumPy strided views, `io_uring`/`readv` scatter buffers.
Watch out: —
Seen in PPSSPP: With `GL_EXT_unpack_subimage`, `TextureCache` sets `GL_UNPACK_ROW_LENGTH = bufw` and uploads straight from the decoded buffer, skipping the `w != bufw` repack copy. A follow-up removed a redundant second `ConvertColors` call. Intra-buffer blits stopped copying through `GetTempFBO` and now blit source to destination directly. 5de7bb2e2d 2013-10 “Use GL_EXT_unpack_subimage to speed up non-pow-2 texture loads when available” · 1fb7cdfcd2 2013-11 “Remove redundant call to ConvertColors, skip a copy when possible” · 5a3a972d95 2014-08 “Perform intra-buffer blits directly.”

### ppsspp-070 — Push per-element fixups into the shader
Lens: gpu-rendering · Source: PPSSPP
Principle: A transformation applied to every element on the CPU before upload, such as a channel reorder or a sign flip, can usually be one swizzle or multiply in the shader, where it costs nothing. The CPU loop and its extra copy go away.
Why it is faster: —
Applies elsewhere: BGRA vs RGBA handling in video players, applying a normal-map sign convention in the shader, doing gamma in the fragment stage rather than on texture load.
Watch out: —
Seen in PPSSPP: `TextureCacheDX9` no longer rewrites textures with `ClutConvertColors`/`ABGR2RGBA`. A shader-ID bit `bgraTexture` makes the pixel shader sample with `.bgra`, which the commit puts at 1–2% in FF2. Normal reversal moved from the vertex decoder (xor or negate per vertex) into the generated vertex shader. 1f51fe7843 2014-08 “d3d: Avoid rewriting textures, just swizzle.” · 4d3e57d6eb 2013-11 “Move normal reversion into the vertex shader instead of the decoder.”

### ppsspp-071 — Recognize an idiom and handle it in bulk
Lens: other · Source: PPSSPP
Principle: If inputs often contain the same recognizable sequence, such as a subroutine that just loads a matrix, check for that exact shape cheaply and run a specialized bulk routine. Processing the items one by one costs per-item dispatch and checks.
Why it is faster: —
Applies elsewhere: compilers recognizing memcpy/memset loops, SQL engines spotting `COUNT(*)` without filters, interpreters with superinstructions for common opcode pairs.
Watch out: —
Seen in PPSSPP: Many games `CALL` a display-list subroutine of 12 `BONEMATRIXDATA` commands and a `RET`. The `CALL` handler checks for that shape (opcode at +0, +11*4, and RET at +12*4) and runs `gstate.FastLoadBoneMatrix()`, which shifts and stores three SSE rows in one go. Later commits extended this to inline, texgen and view/model matrix loads. 8c562a615f 2013-11 “Bone matrix loading speedup. Bit of a hack but seems reliable enough.” · a8f9635e28 2014-03 “Optimize loading of texgen matrices.”

### ppsspp-072 — Remove false dependencies that force batches to be split
Lens: gpu-rendering · Source: PPSSPP
Principle: A batching pipeline often flushes on any state change, including changes that don't affect pending work. Find the state the batch really depends on and flush only for that. If some state can be applied per item on the CPU, it no longer splits batches at all.
Why it is faster: —
Applies elsewhere: sprite batchers that put the texture in an atlas so it stops forcing flushes, DB write batching that ignores irrelevant session settings, coalescing React state updates.
Watch out: —
Seen in PPSSPP: With software skinning, bones are applied in the vertex decoder for each draw. So `GE_CMD_VERTEXTYPE` lost `FLAG_FLUSHBEFOREONCHANGE`, bone-matrix writes no longer flush, and `DIRTY_UVSCALEOFFSET` is set only when the texcoord or through-mode bits actually differ. The commit enables this so consecutive skinned draws merge. Bone values are also compared as raw `u32` (`data << 8`), which avoids a float conversion. 6976d6a3a0 2013-11 “Enable the softskinning optimizations that let us merge drawcalls” · 6221dbaf5d 2013-11 “Optimize software skinning for x86.”

### ppsspp-073 — Pass work to pooled threads by reference, without per-task closures
Lens: concurrency · Source: PPSSPP
Principle: Wrapping each task in a new `std::bind` or `std::function` means allocating and copying the closure for every chunk. Give workers a persistent “run range” interface and pass the callable by const reference along with its bounds.
Why it is faster: —
Applies elsewhere: parallel-for implementations (TBB, rayon), job systems in game engines, request handlers that reuse per-worker contexts.
Watch out: —
Seen in PPSSPP: `ThreadPool::ParallelLoop` took the loop by value and dispatched `std::bind(loop, s, s+chunk)` to each worker. A new `LoopWorkerThread::Process(const std::function<void(int,int)>&, start, end)` stores the reference and bounds and signals the worker, and `ParallelLoop` takes a const reference. 9b6dc64387 2013-12 “Cut down on overhead from std::bind copies.”

### ppsspp-074 — Prefetch the next chunk of a streaming read
Lens: memory-layout · Source: PPSSPP
Principle: In a loop that walks memory linearly with enough work per element, an explicit prefetch a cache line or two ahead overlaps memory latency with computation. This matters most on in-order or weak-prefetcher cores.
Why it is faster: —
Applies elsewhere: `__builtin_prefetch` in hash and checksum loops, packet-processing rings (DPDK), streaming decoders on mobile SoCs.
Watch out: —
Seen in PPSSPP: The ARM vertex-decoder JIT emits `PLD [srcReg, #64]` at the top of each vertex iteration, using a new `ARMXEmitter::PLD`. `xxhash.c` adds `__builtin_prefetch(p + 0xc0, 0, 0)` on ARM GCC, which the commit says hashes about 18% faster on Android. dfea160491 2013-11 “ARM: Use PLD (cache preload) in vertex decoder loop.” · 3ec61274fa 2014-03 “Add prefetch to xxhash func on gcc at least.”

### ppsspp-075 — Sidestep target-specific pipeline penalties
Lens: platform · Source: PPSSPP
Principle: Some cores punish particular patterns: unaligned loads that trap or are slow, or moving data between register files such as ARM VFP and NEON. Use naturally aligned narrow loads combined with shifts, and keep a hot sequence inside one execution domain, even if that costs a few more instructions.
Why it is faster: —
Applies elsewhere: parsing packed headers on microcontrollers, portable `memcpy`-based loads in C, avoiding `movd` round-trips in SSE kernels, keeping NEON DSP loops free of scalar ops.
Watch out: —
Seen in PPSSPP: The ARM vertex decoder replaced possibly unaligned `LDR`s of texcoords, normals and positions with `LDRH`/`LDRB`/`LDRSB` pairs or triples merged by `ORR … LSL`. The commit cites “contradictory information on the support and performance” of unaligned access. The ARM vertex JIT's weight and skinning loops replaced scalar `VLDR S12` plus `VMUL_scalar` with `VLD1_all_lanes` broadcasts and full-vector `VMUL`/`VMLA`, and use `VFMA` on VFPv4. A run of follow-up commits removed the remaining VFP instructions. b3fdfc01c8 2013-11 “ARM vtx dec: Avoid all unaligned accesses entirely.” · 58fe022ecd 2014-03 “vertexjit: Avoid switching to VFP from NEON.”

### ppsspp-076 — Cap expensive optional work per frame and defer the rest
Lens: other · Source: PPSSPP
Principle: In a real-time loop, spikes hurt more than throughput lost on average. Put a per-frame budget on costly optional work such as upscaling or mass invalidation. Once it is used up, take a cheap fallback now and mark the item to finish on a later frame.
Why it is faster: —
Applies elsewhere: incremental GC slices, progressive image decoding, game engines streaming N assets per frame, `requestIdleCallback` work queues.
Watch out: —
Seen in PPSSPP: `TextureCache` counts `texelsScaledThisFrame_`. Above `TEXCACHE_MAX_TEXELS_SCALED` (256×256) it uploads at 1× and sets `STATUS_TO_SCALE`, and a later frame reloads and scales the texture. `InvalidateAll` is ignored after 5 calls per frame. Both commits target stutter. c0c058937f 2014-06 “Limit the amount of texture scaling done per frame, to reduce stutters” · ab61255018 2014-06 “Allow only a small number of "InvalidateAll" operations per frame on the texture cache.”

### ppsspp-077 — Detect change with sampled hashes and staggered rechecks
Lens: algorithms · Source: PPSSPP
Principle: Fully hashing large buffers on every use to detect changes is costly. Hash a few spread-out samples often and do the full hash rarely, with exponential backoff for data that never changes. Add per-item jitter so the full rehashes don't all land on the same frame.
Why it is faster: —
Applies elsewhere: file watchers that check size and mtime before content, rsync's rolling checksums, cache revalidation with backoff, health checks with jittered intervals.
Watch out: —
Seen in PPSSPP: The vertex cache added `ComputeMiniHash()`, which samples at most about 8 draw calls and, for ranges over 100 words, hashes four strided 100-word windows. Otherwise it uses `p[0] + p[sz-1]`. The texture cache's rehash backoff now measures frames elapsed (`numFlips - lastFrame`) and adds `(entry->texture & 15)` jitter so that several textures don't rehash on the same frame. c8e6a809c3 2014-09 “Use a more frequent mini hash in the vertex cache.” · f667d26814 2013-09 “Fix texcache rehash backoff.”

### ppsspp-078 — Pool expensive handles and reuse them by key
Lens: allocation · Source: PPSSPP
Principle: Creating a driver object (buffer name, surface, context) costs a round trip or an allocation inside the driver. Allocate them in batches and hand them out from a free list, or cache them by their creation parameters and reuse them.
Why it is faster: —
Applies elsewhere: DB connection pools, preallocated GPU buffer rings, caching `Intl.NumberFormat` instances by locale, reusing ffmpeg `SwsContext`s.
Watch out: —
Seen in PPSSPP: `TransformDrawEngine::AllocateBuffer()` pops from `bufferNameCache_`, which is refilled 64 at a time by one `glGenBuffers`, so it no longer calls `glGenBuffers(1, …)` for each VBO or EBO. The D3D9 framebuffer manager caches system-memory offscreen surfaces by a `format<<32 | w<<16 | h` key. `SimpleAudio` creates its `SwrContext` once rather than on every audio frame. 4971206e92 2014-06 “Allocate vertexcache buffer names in chunks.” · 8c4c9e554e 2014-09 “d3d: Cache offscreen surfaces.” · dc66e18c00 2014-06 “Lazily create resampling contexts once instead of every audio frame”

### ppsspp-079 — Skip a transfer when the destination is already current
Lens: laziness · Source: PPSSPP
Principle: Track whether a mirrored copy is already in sync with its source and skip the expensive readback or copy when it is. The expensive operation then runs only after a real change.
Why it is faster: —
Applies elsewhere: HTTP ETags and 304 responses, skipping a GPU readback when the frame hasn't changed, incremental backup that skips unchanged blocks.
Watch out: —
Seen in PPSSPP: Block-transfer handling read the framebuffer back to RAM on every copy. It now checks `!srcBuffer->memoryUpdated` first, which fixed lag in Final Fantasy Tactics. 6ebdf2be29 2014-06 “Skip memcpy fb download when already downloaded.”

### ppsspp-080 — Store raw inputs and derive values only when consumed
Lens: laziness · Source: PPSSPP
Principle: If a frequently written parameter is converted eagerly on every write but read much less often, store the raw value and set a dirty bit. Convert at the point of use. Similarly, compute a derived value once and pass it to the helpers that need it, rather than letting each one recompute it.
Why it is faster: —
Applies elsewhere: lazy layout in UI toolkits, deferring JSON parsing until a field is accessed, passing a parsed config object rather than re-reading env vars in each helper.
Watch out: —
Seen in PPSSPP: GE light-parameter commands (`LX0`…`LKC3`) used to convert to float into `gstate_c.lightpos/lightdir/…` on every write. They now only dirty `DIRTY_LIGHTn`, and the shader manager and the shared `Lighter` read the raw `gstate` when needed. `ReplaceAlphaWithStencil(replaceBlend)` takes the already computed blend mode, so `ReplaceBlendWithShader()` is no longer called again inside it. 16f4622ac7 2014-04 “Remove cached light variables to reduce work when parsing DLs.” · 24a0dd12ef 2014-08 “Avoid recalculating replaceBlend so many times.”

### ppsspp-081 — Hoist per-call setup out of per-element functions
Lens: laziness · Source: PPSSPP
Principle: If a function called per pixel or per element recomputes values that are fixed for the whole operation (base pointers, strides, format info), compute them once in the caller and pass them in. The per-element body shrinks to the actual work.
Why it is faster: —
Applies elsewhere: image filters that compute row pointers once per row, SQL expression evaluators that resolve column offsets at plan time, shaders that move uniform math to the CPU.
Watch out: —
Seen in PPSSPP: The software rasterizer's `SampleNearest()` called `Memory::GetPointer(texaddr)` and `GetTextureBufw()` on every texel. It now takes `srcptr` and `texbufwidthbits` as parameters, computed once per primitive. ad4bd8ed74 2013-09 “softgpu: small optimization, precompute tex info.”

### ppsspp-082 — Bind SIMD variants at compile time where you can, at runtime where you must
Lens: platform · Source: PPSSPP
Principle: When the instruction set is known at build time (SSE2 on x86-64), call the optimized routine directly through a macro so it can be inlined, with no indirect call. Where support varies by device (NEON on ARMv7), set a function pointer once at startup. Use newer ISA extensions behind a CPU-feature check.
Why it is faster: —
Applies elsewhere: glibc ifuncs, simdjson's runtime dispatch, compiling with `-march` for known server fleets, Rust's `is_x86_feature_detected!`.
Watch out: —
Seen in PPSSPP: On `_M_SSE`, `DoQuickTexHash`, `DoUnswizzleTex16` and `DoReliableHash` became `#define`s for the SSE2 or XXH32 functions rather than function pointers. ARM keeps pointers selected at init (`DoReliableHash = &ReliableHashNEON`). The new `fast_math` library links SSE at compile time and picks NEON in `InitFastMath(enableNEON)`. The x86 vertex JIT uses `PMOVSX*` when `cpu_info.bSSE4_1` is set. 006ed53e73 2014-03 “Use defines on x86/SSE to avoid func pointers.” · 9a49f564ce 2014-03 “fast_math: SIMD function library that can select to use NEON at runtime.” · 632eec38e8 2014-03 “vertexjit: Use SSE4.1 where available on x86.”

### ppsspp-083 — Prefer branch-free arithmetic to a large lookup table
Lens: lookup-tables · Source: PPSSPP
Principle: A lookup table looks free, but a big one evicts useful data from cache and gathers poorly into SIMD lanes. When a few vector instructions can compute the value, such as half-float to float with a magic-number multiply, arithmetic often wins.
Why it is faster: —
Applies elsewhere: sRGB or gamma conversion with polynomial approximations, popcount via bit tricks instead of a 64 KB table, CRC with carry-less multiply instead of tables.
Watch out: —
Seen in PPSSPP: For the VFPU `vh2f` op, the x86 JIT prototyped both a lookup-table decoder and an SSE2 translation of a known `half_to_float` routine (a `magic = (254-15)<<23` multiply). The table version was deleted the same day. The comment had questioned its cache impact. 7ca6d73857 2013-09 “Two approaches to vh2f (half-float to float): lookuptable and fast SSE” · 20174d9410 2013-09 “Delete the lookup table version of vh2f”

### ppsspp-084 — Extract bit fields a word at a time
Lens: algorithms · Source: PPSSPP
Principle: Reading an N-bit field one bit per iteration costs N loop trips. Load the containing 32-bit word, shift and mask, and when the field crosses a boundary OR in the low bits of the next word. Any field up to 32 bits then takes at most two loads.
Why it is faster: —
Applies elsewhere: Huffman and entropy decoders, compressed bitmap indexes, parsing packed sensor telemetry, varint or bit-packed columnar formats.
Watch out: —
Seen in PPSSPP: `PGF::getBits()` looped bit by bit over `buf[pos>>3]`. It now reads `wordbuf[pos>>5] >> (pos&31)` and, when the field crosses into the next word, ORs in `(wordbuf[wordpos+1] & mask) << done`. d446659b2a 2013-11 “Improve the performance of getBits().”

### ppsspp-085 — Avoid the heap for transient or borrowed data
Lens: allocation · Source: PPSSPP
Principle: Data that only needs to live while a source buffer or a call is alive should not get its own heap block. Point into the existing buffer, or carve scratch space from the stack, which costs a pointer bump. Allocation, copying, freeing and lifetime bookkeeping all go away.
Why it is faster: —
Applies elsewhere: `string_view`/`&[u8]` slices in parsers, FlatBuffers zero-copy reads, `Buffer.subarray` in Node, `SmallVector` or stack buffers for per-call temporaries.
Watch out: —
Seen in PPSSPP: PGF font loading allocated `new u8[]` for `shadowCharMap`, `charMap` and `charPointerTable` and copied them byte by byte. They are now `const u8 *` into the file data, with `uptr += size`. VTimer handlers had their two 64-bit arguments in a 16-byte block from `kernelMemory.Alloc`, kept per timer and freed on delete. Now the code lowers the guest `SP` by `HANDLER_STACK_SPACE` (48), writes the arguments at `argArea-16`/`-8`, and raises `SP` again after the handler runs. d955d9b303 2013-11 “Cut down on unnecessary allocs in font loading.” · 7edebd1c2f 2013-10 “Put vtimer args on the stack, skip allocation.”

### ppsspp-086 — Detect identity or degenerate cases and take the trivial path
Lens: branching-fast-paths · Source: PPSSPP
Principle: General transforms often get parameters that make them trivial: unity gain, contiguous rows, blending that copies the source, a full-screen fill. Check for these cheaply up front and use `memcpy`, one bulk call or a hardware clear.
Why it is faster: —
Applies elsewhere: image resizers that copy when the scale is 1:1, audio mixers that skip gain at 0 dB, `memcpy` when the pitch equals the width, compositors that skip blending opaque layers.
Watch out: —
Seen in PPSSPP: When both audio channel volumes are `1<<15`, the most common case, `__AudioEnqueue` copies the samples directly into the ring buffer through `pushPointers`. Block transfers with `srcStride == dstStride == width` use one `memcpy(width*height*bpp)`, which the commit puts at 3.5% in God of War. Blending with `fixA == 0xFFFFFF && fixB == 0` turns blending off. A clear-mode rectangle that covers the target becomes a `glClear()` when `IsReallyAClear()` holds. b71255e7b0 2013-10 “Optimize the case where volume is 0x8000.” · 9982d04f9d 2014-04 “Optimize the case of a direct byte copy.” · 422db25277 2014-08 “Optimize fixed color blending that isn't blending.” · 097ab53952 2014-05 “Enable the glClear() optimization.”

### ppsspp-087 — Fold a summary into the append instead of a second pass
Lens: algorithms · Source: PPSSPP
Principle: If you build a list and then compute a hash or summary over all of it, update the summary as each item is added. You drop a full pass and the re-reading of memory that has already left cache.
Why it is faster: —
Applies elsewhere: running checksums while writing a file, maintaining aggregates in a DB on insert, keeping `size` and `sum` fields on a collection.
Watch out: —
Seen in PPSSPP: `ComputeFastDCID()` looped over all deferred draw calls at flush time. `SubmitPrim` now folds each call's verts, inds, vertType, count and prim into `dcid_` with xor/rotl when it is queued. 8b92dcea47 2014-03 “Transform: Compute the "DCID" (draw call ID) incrementally instead of an extra pass.”

### ppsspp-088 — Rebuild derived indexes once after a bulk load
Lens: algorithms · Source: PPSSPP
Principle: If every insert triggers a rebuild of a derived structure, a bulk load costs O(n²). Suspend the rebuild during the batch and run it once at the end. Similarly, cache a costly check (such as a filesystem probe) instead of repeating it on every request.
Why it is faster: —
Applies elsewhere: dropping indexes during a DB bulk import, batching DOM updates, deferring `Array.sort` until all items are pushed.
Watch out: —
Seen in PPSSPP: `SymbolMap::SetLabelName` called `UpdateActiveSymbols()` for every function that hash matching identified, which slowed startup badly when a signature file was used. `MIPSAnalyst` now calls it once after the pass. Reporting caches `IsSupported()`, which probed `flash0:/`, in `currentSupported` rather than checking on each request. b1d6eefb8a 2014-02 “Call UpdateActiveSymbols only once from MIPSAnalyst instead of for every recognized function.” · 3dea6fdd86 2014-02 “Cache reporting support to avoid slowdown.”

### ppsspp-089 — Read contiguous blocks with one large request
Lens: io · Source: PPSSPP
Principle: Each read call has a fixed cost: syscall, seek, decompression setup. When a caller wants a run of consecutive blocks, issue one sized read for the whole run and use a buffer big enough to hold it.
Why it is faster: —
Applies elsewhere: `readv` or large `pread`s in databases, coalescing HTTP range requests, bulk SPI flash reads in firmware.
Watch out: —
Seen in PPSSPP: `FileBlockDevice::ReadBlocks(minBlock, count, out)` does one `fseeko` and one `fread(out, 2048, count)`, or `read()` on the fd path, instead of a loop of 2 KB reads. The CSO device grew a 256 KB read buffer for multi-block reads. The commit says loading got faster even on desktop. 28b3b841d5 2014-11 “Implement optimized multi-block reads for iso/cso.”

### ppsspp-090 — Hide slow storage behind a block cache with read-ahead
Lens: io · Source: PPSSPP
Principle: For high-latency sources (network, optical, slow flash), keep recently used fixed-size blocks in RAM and fetch the next few blocks on a background thread after each read. Sequential access then mostly hits cache, and stalls move off the critical path. If memory allows, the whole image can be loaded up front.
Why it is faster: —
Applies elsewhere: OS page-cache read-ahead, video players buffering ahead, HTTP range-request caches for remote archives, SQLite on network filesystems.
Watch out: —
Seen in PPSSPP: `CachingFileLoader` wraps the HTTP loader with an LRU block cache (`ReadFromCache`/`SaveIntoCache`/`MakeCacheSpaceFor`). A follow-up adds `StartReadAhead(pos)`, which fetches `BLOCK_READAHEAD` blocks on a worker thread, with separate block and backend mutexes. `RAMBlockDevice` optionally reads the entire ISO into memory (`bCacheFullIsoInRam`) to avoid I/O stutter. 4e11f7c182 2014-11 “Implement a simple recently used cache for HTTP.” · 824e8cf17c 2014-11 “Add a readahead cache for HTTP.” · 291e5e53c0 2014-10 “Add feature to cache ISOs in RAM to avoid I/O stutters.”

### ppsspp-091 — Don't let anonymous shared memory write through to its file
Lens: io · Source: PPSSPP
Principle: A shared memory mapping backed by a file can make the OS flush dirty pages to disk, which wastes I/O when the file is only a vehicle for the mapping. Use the platform's no-sync mmap flag so the pages stay in RAM.
Why it is faster: —
Applies elsewhere: `MAP_NOSYNC` on FreeBSD for shared-memory IPC, tmpfs-backed shm, `memfd_create` on Linux.
Watch out: —
Seen in PPSSPP: `MemArena` adds `MAP_NOSYNCFILE` on BlackBerry and `MAP_NOSYNC` on FreeBSD to its `mmap` of emulated RAM. The commit calls the default “terrible on I/O performance” and notes that Linux already behaves this way. 6b8d4cfe7b 2013-12 “Blackberry/BSDs: Don't sync memory back to the underlying file.”

### ppsspp-092 — Run latency-sensitive output on its own thread
Lens: concurrency · Source: PPSSPP
Principle: If a periodic real-time task, such as feeding audio, runs on the same thread as variable-length work like rendering, it starves whenever a frame runs long. Move it to its own thread and it keeps its deadline regardless of frame time.
Why it is faster: —
Applies elsewhere: audio callbacks on dedicated threads in DAWs, a UI thread kept separate from network I/O, a high-priority RTOS task for sensor sampling.
Watch out: —
Seen in PPSSPP: The Qt frontend's audio object is moved to a `QThread` with `moveToThread`, and the mixer writes only as many frames as `NativeMix` returns. The commit says audio now runs “flawlessly at low frame rates or frameskipping”. 30edf6b894 2013-10 “Simply stick the Qt audio in a thread.”

### ppsspp-093 — Shadow driver state and drop redundant state calls
Lens: gpu-rendering · Source: PPSSPP
Principle: Calls into a graphics driver or other stateful API cost far more than comparing two integers. Keep a CPU-side copy of the last value set and skip the call when the new value is the same. Most redundant binds, enables and layout transitions then cost nothing.
Why it is faster: —
Applies elsewhere: a canvas or WebGL wrapper that remembers the current fillStyle or bound texture, a UART driver that skips rewriting an unchanged baud-rate register, a DOM layer that only writes `style.x` when the value differs.
Watch out: —
Seen in PPSSPP: `glBindBuffer(GL_ARRAY_BUFFER, …)` calls in `TransformDrawEngine::DoFlush` and the framebuffer and texture code became `glstate.arrayBuffer.bind(vbo)` and `glstate.elementArrayBuffer.unbind()`. These are cached `SavedState` objects that only call GL on a change. The Vulkan draw engine tracks `lastPipeline_` per command buffer and skips `vkCmdBindPipeline` when the pipeline is unchanged. `GLQueueRunner` keeps a local `stencilEnabled` flag to filter repeated `glEnable(GL_STENCIL_TEST)`. The thin3d Vulkan backend skips an image barrier when the image is already in `COLOR_ATTACHMENT_OPTIMAL`. 7a5e182fd1 2015-01 “Cache the bound VBO/index buffer in glstate.” · 40ea889b6a 2015-01 “Cache the VBO/index buffer to avoid redundancy.” · a1fd2d1ed1 2017-08 “Dirty-track the current pipeline to avoid many vkBindPipeline calls.” · bc339c0377 2018-02 “GL: Filter away redundant glEnable/Disable(GL_STENCIL_TEST)” · f762285b9c 2017-05 “Vulkan: Some barrier optimization”

### ppsspp-094 — Dirty bits gate expensive state derivation
Lens: laziness · Source: PPSSPP
Principle: Give each group of inputs a dirty bit. Rebuild derived objects (keys, pipelines, uniforms) only when a relevant bit is set, and clear the bits afterwards. Most frames change little, so hash lookups and conversions for unchanged state disappear.
Why it is faster: —
Applies elsewhere: a spreadsheet recalculating only cells downstream of an edit, a game UI rebuilding layout only when its text or size changes, a React-style memo keyed on changed props.
Watch out: —
Seen in PPSSPP: In `DrawEngineVulkan::DoFlush`, `ConvertStateToVulkanKey` and `GetOrCreatePipeline` now run only if `!lastPipeline_ || gstate_c.IsDirty(DIRTY_BLEND_STATE | DIRTY_RASTER_STATE | …) || prim != lastPrim_`. Afterwards the code calls `gstate_c.Clean(…)`. 50d771961b 2017-08 “Vulkan: More minor optimization (avoid pipeline cache lookups when possible)”

### ppsspp-095 — Table-driven side effects instead of per-entry callbacks
Lens: lookup-tables · Source: PPSSPP
Principle: When most handlers in a dispatch table only mark something dirty, store the dirty mask as data in the table. A generic loop then ORs it in, and only the few entries that really need code keep a function pointer. That removes an indirect call per entry and a lot of near-duplicate code. Adding a fast path for the no-change case skips even the table lookup's side effects.
Why it is faster: —
Applies elsewhere: a register-write table in a device driver that maps each register to the subsystems it invalidates, or an event table in a UI toolkit that maps events to repaint regions.
Watch out: —
Seen in PPSSPP: The GE command table gained a 64-bit dirty field. Entries like `{GE_CMD_FOGCOLOR, FLAG_FLUSHBEFOREONCHANGE, DIRTY_FOGCOLOR}` replaced `Execute_FogColor` callbacks. In `FastRunLoop` the `diff == 0` case now only runs `FLAG_EXECUTE` commands. It skips the flush check and the `cmdmem` write. The changed case ORs in `flags >> 8`, and the downcount is cached in a local `dc`. c70265d7f2 2017-01 “Use table-driven dirtying instead of callbacks in GPU_GLES” · 2c4e5e2303 2017-08 “Fastpath in fastrunloop when diff=0. Remove need for Execute for UV scale/offset.”

### ppsspp-096 — Inner loop for runs of the same command
Lens: branching-fast-paths · Source: PPSSPP
Principle: If a command is usually followed by more of the same kind, the handler can peek ahead and consume the whole run itself. It goes back to the general dispatcher only when something different appears. Per-command dispatch, state checks and bookkeeping are then paid once per run instead of once per item.
Why it is faster: —
Applies elsewhere: a bytecode VM that handles consecutive PUSH ops in one handler, a log parser that appends consecutive lines of the same type in one tight loop, a DB writer that coalesces consecutive INSERTs.
Watch out: —
Seen in PPSSPP: `GPUCommon::Execute_Prim` now scans forward from `currentList->pc + 4` to the stall address. It submits further `GE_CMD_PRIM` commands of the same primitive type directly and handles a few harmless commands inline. Anything else hits `goto bail`. The commit reports a 1–4% gain in PRIM-heavy games. 292f116821 2018-02 “Execute_Prim: Add a smaller "inner interpreter" to avoid going back to the runloop during long sequences of PRIM commands.” · 4130bcd416 2018-03 “inner interpreter: support skipping regular nops”

### ppsspp-097 — Check a cheap inline condition before a virtual call
Lens: branching-fast-paths · Source: PPSSPP
Principle: If a virtual method usually has nothing to do, test the condition through a non-virtual inline accessor at the call site. The common empty case then avoids an indirect call the compiler cannot inline.
Why it is faster: —
Applies elsewhere: `if (logger.enabled) logger.log(…)`, checking `queue.isEmpty()` before calling a polymorphic `flush()`, or skipping an observer notification when the listener list is empty.
Watch out: —
Seen in PPSSPP: The GE run loop called `drawEngineCommon_->DispatchFlush()` on every flush-before-change command. Now it calls it only `if (drawEngineCommon_->GetNumDrawCalls())`, using a new inline getter in `DrawEngineCommon.h`. 0b4b2bfe9c 2018-02 “Avoid the most common virtual call when possible.”

### ppsspp-098 — Specialize generated code for common constant values
Lens: compiler-build · Source: PPSSPP
Principle: When generating code (shaders, queries, JIT output) from runtime parameters, detect common special values and emit a cheaper variant for them. The check runs once at generation time, and every execution of the generated code saves the work.
Why it is faster: —
Applies elsewhere: a SQL planner that emits an index lookup for `col = const`, a template engine that inlines static fragments, a regex compiler that turns a literal pattern into `memchr`.
Watch out: —
Seen in PPSSPP: The fragment shader generators add an `IsColorTestAgainstZero()` bit to the shader ID. When it is set, they emit `if (v.r < 0.002 && v.g < 0.002 && v.b < 0.002) clip(-1);` instead of rounding and scaling the color to 0–255 and comparing it with a uniform. The vertex shader ID drops texture projection when `MatrixNeedsProjection(gstate.tgenMatrix)` shows that z will always be 1. f0676b0c85 2015-03 “Optimize color tests against zero.” · 0e4fb11a2d 2017-01 “If we can predict that the texcoord.z will always be 1.0 after the matrix, skip texture projection.”

### ppsspp-099 — Canonicalize cache keys to exclude don't-care bits
Lens: algorithms · Source: PPSSPP
Principle: A cache key should contain only the inputs that change the output. Bits for disabled features, or uninitialized padding bytes, split one logical entry into many. That multiplies compiles, misses and memory use.
Why it is faster: —
Applies elsewhere: an HTTP cache that normalizes query-parameter order and strips tracking params, a memo function keyed by a struct that must be zero-initialized before hashing with `memcmp`/`memhash`.
Watch out: —
Seen in PPSSPP: The alpha-test-against-zero bit moved inside the `if (alphaTest enabled)` block of the fragment shader ID. Light parameters now feed the vertex shader ID only `if (gstate.isLightChanEnabled(i))`, which cuts down shader counts in Tekken 6. `VulkanPipelineKey key{}` is zero-initialized and its fields reordered, so padding bytes no longer create duplicate pipelines. cf7a4ab2cf 2015-03 “Avoid shader id data that doesn't change shader.” · f4d50feacb 2015-10 “Reduce the number of shaders generated by games that juggle a lot of lights, like Tekken 6” · 5c7f7c1df6 2017-11 “Vulkan: Make sure to clear the pipeline key to avoid padding bytes causing us to allocate duplicate pipelines.”

### ppsspp-100 — Hoist inner-loop invariants into a precomputed row
Lens: algorithms · Source: PPSSPP
Principle: In a nested loop, any sub-expression that depends only on the outer or inner index can be computed once per index value and stored in a small array. That turns O(n·m) evaluations into O(n + m).
Why it is faster: —
Applies elsewhere: image filters that precompute per-column weights, separable convolutions, and 2-D interpolation that precomputes the per-x basis functions.
Watch out: —
Seen in PPSSPP: `_BezierPatchHighQuality` called `Bernstein3DDerivative` four times per vertex. It now fills `derivU1..4[tess_u + 1]` once in the horizontal pre-pass and indexes them by `tile_u` in the inner loop. 9a83c99e19 2014-12 “[spline/bezier]Precompute bernstein derivative functions.”

### ppsspp-101 — Turn divisions into multiplies and constant divisors
Lens: compiler-build · Source: PPSSPP
Principle: Division is many times slower than multiplication. Compute a reciprocal once and multiply by it, or restructure an expression so the divisor is a compile-time constant (often a power of two), which the compiler lowers to a shift.
Why it is faster: —
Applies elsewhere: perspective divide in a software renderer (`inv = 1/w`, then `x*inv, y*inv`), normalizing a vector, and choosing a `bytesPerPixel` branch outside so `len / 4` becomes `len >> 2`.
Watch out: —
Seen in PPSSPP: `SoftwareTransformCommon` replaced four divisions by `tl.w`/`br.w` with two reciprocals and four multiplies. In `TextureCache`, `clutMaxBytes_ / (fmt == 32BIT ? sizeof(u32) : sizeof(u16))` became `fmt == 32BIT ? clutMaxBytes_ / sizeof(u32) : clutMaxBytes_ / sizeof(u16)`, so each branch divides by a constant. b782165b6a 2015-05 “Avoid some division.” · ea33f50a8a 2015-07 “Minor optimization: Help stupid compilers avoid a division (and a multiplication).”

### ppsspp-102 — Template on loop-invariant flags
Lens: compiler-build · Source: PPSSPP
Principle: If a hot loop tests the same configuration flags on every iteration, make them template (compile-time) parameters and dispatch once to the right instantiation. Each copy of the loop then has no flag branches and is easier to vectorize.
Why it is faster: —
Applies elsewhere: pixel blitters specialized per format pair, serializers specialized on "has optional field", and DSP kernels templated on channel count.
Watch out: —
Seen in PPSSPP: `_SplinePatchFullQuality` checked `origVertType & GE_VTYPE_{NRM,COL,TC}_MASK` inside the weight loop. It became `template <bool origNrm, bool origCol, bool origTc> SplinePatchFullQuality`, with an SSE `AccumulateWeighted` that does `_mm_add_ps(out, _mm_mul_ps(in, w))` into local accumulators. 8e19f568d7 2015-02 “Optimize spline weighting using SSE and templates.”

### ppsspp-103 — SIMD for bulk sample and pixel format conversion
Lens: simd · Source: PPSSPP
Principle: Format conversion loops (integer↔float, bit-field unpacking, scaling) apply the same few operations to every element. With SSE or NEON they process 4–16 elements per instruction, plus a scalar tail for leftovers.
Why it is faster: —
Applies elsewhere: PCM s16→f32 in audio callbacks, RGB565→RGBA8888 in display drivers, and uint8→float normalization in image ML preprocessing.
Watch out: —
Seen in PPSSPP: `ConvertS16ToF32` sign-extends with `_mm_cmpgt_epi16` and `_mm_unpacklo/hi_epi16`, then runs `_mm_cvtepi32_ps` and a multiply, 16 samples per loop. High-volume audio channels use `_mm_mulhi_epi16` plus a shift. `ConvertRGBA5551ToRGBA8888` gained an SSE path that the commit says takes 85% less time. Bone matrix loads use NEON `vshlq_n_u32`. Uniform color conversion goes through shared `Uint8x3ToFloat4` helpers with SSE and NEON versions. 4705a2568e 2015-01 “Convert s16->float using SSE2.” · f5d5f7d3fa 2014-12 “Use SSE shifts for high-vol channels.” · 1cb2302f5c 2015-05 “Use SSE in ConvertRGBA5551ToRGBA8888().” · 2b8e81f01e 2017-08 “NEON-optimize bone matrix loads” · 91783a3281 2017-08 “SIMD-optimize some data conv routines used in uniform updates.” · c2bd739b49 2017-08 “Optimize uniform uploads for D3D9 and GL as well.”

### ppsspp-104 — Vector scan with OR-accumulators and per-row early exit
Lens: simd · Source: PPSSPP
Principle: To answer "does any element satisfy X?" over a big buffer, OR the results of SIMD compares into an accumulator. Do a horizontal reduction only once per row, and return as soon as the answer is known.
Why it is faster: —
Applies elsewhere: checking whether a buffer is all zero, whether a string has non-ASCII bytes, or whether an image has any transparent pixel before choosing JPEG or PNG.
Watch out: —
Seen in PPSSPP: `CheckAlphaRGBA8888SSE2` shifts alpha into place with `_mm_srli_epi32(…, 24)` and ORs `_mm_cmpeq_epi32(a, zero)` and `a & (a < 0xFF)` into cursors. After each row it calls `CombineSSEBits` and returns `CHECKALPHA_ANY` early. NEON versions followed. c75010931c 2015-05 “Use SSE in CheckAlpha scanning.” · 65e71f57c7 2017-11 “TexCache: Add NEON alpha checks for Vulkan.”

### ppsspp-105 — Hash index beside an ordered list
Lens: algorithms · Source: PPSSPP
Principle: If you need both a meaningful order (for enumeration) and fast lookup by key, keep the vector for order and add a hash map for lookup. A linear scan becomes O(1). Where ordering is never needed, swap a tree map or set for its unordered version.
Why it is faster: —
Applies elsewhere: a filesystem directory cache with entries in on-disk order plus a name→entry map, or an LRU built from a list plus a hash map.
Watch out: —
Seen in PPSSPP: `ISOFileSystem::TreeEntry` gained `std::unordered_map<std::string, TreeEntry *> fastChildren` next to the PSP-ordered `children` vector. `GetFromPath` now calls `fastChildren.find` instead of looping. `replacementNameLookup` and the MIPSAnalyst `hashMap` moved from `std::map`/`std::set` to unordered containers with a custom `std::hash<HashMapFunc>`, which cut savestate time. 3f36b476ad 2015-09 “Add an unordered_map to TreeEntry for faster lookup” · 3fe1a08cf5 2016-09 “Use unordered lookups for better speed.”

### ppsspp-106 — Open-addressing hash map with a separate state array
Lens: memory-layout · Source: PPSSPP
Principle: Node-based hash maps chase pointers and allocate on every insert. An open-addressing table with linear probing keeps keys and small values inline, which suits the cache. Keeping bucket states in their own byte array makes clearing a fast memset-like pass over one byte per bucket, and probing a dense scan.
Why it is faster: —
Applies elsewhere: per-frame lookup tables in game engines, symbol tables in compilers, and connection tables in packet processors that are cleared every tick.
Watch out: —
Seen in PPSSPP: `Common/Hashmaps.h` added `DenseHashMap` (XXH32 key hash, `memcmp` equality, linear probing, tombstones) and `PrehashMap`. These replaced `unordered_map` in the draw engines' vertex caches and pipeline and shader lookups. Later, `BucketState` became a `uint8_t` enum in a separate `std::vector<BucketState> state`, so `Clear()` only resets that array. e0e13e191f 2017-08 “Replace the most critical unordered_maps and maps with two new maps: FastHashMap and PrehashMap.” · 94b16bf097 2017-11 “DenseHashMap: Speed up clearing.”

### ppsspp-107 — Memoize expensive external queries
Lens: laziness · Source: PPSSPP
Principle: Font measurement, filesystem probes and name lookups often return the same answer for the same input. Cache the result under a cheap hash of the inputs so later calls skip the slow path entirely.
Why it is faster: —
Applies elsewhere: caching `measureText` results in a canvas UI, caching `stat()` results in a build tool, or memoizing DNS lookups.
Watch out: —
Seen in PPSSPP: `TextDrawer::MeasureString` keeps a `sizeCache_` keyed by `Fletcher(str) ^ fontHash_`, so `GetTextExtentPoint32` no longer runs every frame. `TextureReplacer::FindReplacement` returns a reference into a `cache_` keyed by `(cachekey, hash)` instead of probing the disk again. `CallSyscall` compared every op against `GetSyscallOp("FakeSysCalls", NID_IDLE)`, a by-name lookup. That value is now computed once into a static `idleOp`. 57e68be754 2016-09 “UI: Cache measured text sizes.” · f536182b37 2016-04 “Cache texture replacement lookup info.” · 38b7d89dfb 2016-05 “Fix a performance issue in CallSyscall” · 81197f18a0 2014-12 “Avoid calling GetPointer() multiple times in a row”

### ppsspp-108 — Know metadata at generation time instead of querying the driver
Lens: gpu-rendering · Source: PPSSPP
Principle: If you generate code yourself, you already know which inputs it uses. Record that as you emit the code, or query once and store it with the cached object. Repeated calls to the driver's reflection API then go away.
Why it is faster: —
Applies elsewhere: an ORM that records column indices when it builds a statement instead of calling `getColumnIndex` per row, or a code generator that emits a manifest of exported symbols.
Watch out: —
Seen in PPSSPP: `GenerateFragmentShader` and `GenerateVertexShader` now return a `uniformMask`/`attrMask` built while writing the GLSL, replacing a long chain of `if (u_x != -1) availableUniforms |= …` after `glGetUniformLocation`. The depal shader cache stores `a_position` and `a_texcoord0` in a `DepalShader` struct instead of calling `glGetAttribLocation` on every use. 33d3d2f178 2017-11 “GL: Compute "availableUniform" when generating shaders instead of by querying.” · d00f6adf11 2015-03 “Avoid querying attrib locations in depal runtime.”

### ppsspp-109 — Parse and split without temporary allocations
Lens: allocation · Source: PPSSPP
Principle: Stream abstractions and temporary copies allocate and copy for every token. Scan the buffer directly, compare in place with length-bounded case-insensitive functions, and take substrings only for the output.
Why it is faster: —
Applies elsewhere: HTTP header parsing in an embedded web server, CSV splitting, and command-line tokenizers.
Watch out: —
Seen in PPSSPP: `SplitString` used `std::istringstream` + `getline`. It is now a single `for` loop over `str` that pushes `substr`, and the commit says it takes about 10% of the time. That helps UI text wrapping. The HTTP header parser stopped doing `new char[]` + `StringUpper` on every key and uses `strncasecmp(key, "User-Agent", key_len)` on the original buffer. 740365235b 2016-09 “Make SplitString significantly faster.” · 85506e98fe 2016-05 “http: Parse headers with folding, not alloc.” · b55015992c 2015-09 “std::string comparison can be faster than strcmp”

### ppsspp-110 — Allocate large buffers on first use, free when idle
Lens: allocation · Source: PPSSPP
Principle: Static or eager allocation of big tables uses memory, and often startup time, even when the feature is off. Allocate on first use and release when the owning feature shuts down. Reserve address space and commit pages only for the ranges you use.
Why it is faster: —
Applies elsewhere: lazily creating a 64 MB lookup table only when a filter is enabled, `mmap` with `PROT_NONE` reservation followed by `mprotect` on demand, and freeing debug symbol tables when no session is active.
Watch out: —
Seen in PPSSPP: The xBRZ `DistYCbCrBuffer` (256³ floats) became a pointer created in `scale()` and deleted by `xbrz::shutdown()`. The global `symbolMap` became a heap `g_symbolMap` that is freed when no game runs. On UWP, `MemoryMap_Setup` now does `VirtualAllocFromApp(…, MEM_RESERVE)` and commits each view in `CreateView`, instead of committing all 256 MB. 10049e62a8 2015-02 “Allocate the xBRZ buffer only when actually used.” · b998131581 2015-10 “Move the symbol map to the heap, deallocate it when no game is running.” · 4bad28046b 2017-03 “UWP: Save memory by only committing the ranges we need, instead of doing a giant allocation.”

### ppsspp-111 — Load tree nodes on demand
Lens: laziness · Source: PPSSPP
Principle: Walking a whole hierarchical structure at open time costs I/O proportional to its total size. Mark each node "not yet read" and fill in its children the first time someone asks. Callers that only touch a few paths pay only for those.
Why it is faster: —
Applies elsewhere: file-browser trees, lazy JSON or XML DOMs, and ORM relations loaded on first access.
Watch out: —
Seen in PPSSPP: `ISOFileSystem` no longer calls `ReadDirectory(treeroot)` recursively in its constructor. Each `TreeEntry` has a `valid` flag, and `GetFromPath` calls `ReadDirectory(entry)` only `if (!entry->valid)`. Reading game icons from the game list got faster as a result. 6f9755d498 2016-02 “ISOFileSystem: Lazily load directory information as needed. Should speed up game icon loading a bit.”

### ppsspp-112 — Amortize cache eviction: run it rarely and only under pressure
Lens: algorithms · Source: PPSSPP
Principle: Scanning a whole cache for stale entries every frame is wasted work when little has aged out. Run eviction every N ticks, only once an estimated size crosses a threshold, and faster for entry kinds known to pile up.
Why it is faster: —
Applies elsewhere: a CDN edge evicting only above a high-water mark, a JS LRU that sweeps every 100 inserts, and a GC that collects young objects more aggressively.
Watch out: —
Seen in PPSSPP: `FragmentTestCache::Decimate` only walks the map when `--decimationCounter_ <= 0` (an interval of 113). The texture caches keep `cacheSizeEstimate_` from `EstimateTexMemoryUsage` and skip killing textures below `TEXCACHE_MIN_PRESSURE` (16 MB). CLUT texture variants get a shorter kill age, applied once there are at least `TEXTURE_CLUT_VARIANTS_MIN` of them. a82bb31c6f 2015-03 “Decimate the fragtest cache less often.” · d097999fad 2014-12 “Avoid killing textures unless we have several mb.” · e97046da4e 2018-02 “TexCache: Decimate aggressively with many variants.”

### ppsspp-113 — Hysteresis: stop optimizing data that keeps changing
Lens: other · Source: PPSSPP
Principle: An expensive enhancement step (scaling, compressing, indexing) pays off only if the result is reused. Flag items that change often and skip the step for them. Otherwise you get a loop of redoing work that is thrown away immediately.
Why it is faster: —
Applies elsewhere: a JIT that stops recompiling a function after repeated deoptimizations, or an image CDN that skips re-encoding assets updated every few seconds.
Watch out: —
Seen in PPSSPP: The texture caches used to set `match = false` to reload a texture for the scaling they had skipped. That now happens only `if ((entry->status & STATUS_CHANGE_FREQUENT) == 0)`. The alpha scan is also skipped for frequently changing textures (see e3b3828b15). 7ee9112425 2015-03 “Avoid constant reload of frequently changed tex.”

### ppsspp-114 — Limit work to the region actually used
Lens: algorithms · Source: PPSSPP
Principle: Hashing, copying or scanning a whole buffer when only part of it matters wastes bandwidth in proportion to the unused part. Track the range consumers actually touch, such as the max coordinate, the needed width or the code section, and process only that.
Why it is faster: —
Applies elsewhere: dirty-rectangle screen updates, partial-file ETags, and incremental backups over modified byte ranges.
Watch out: —
Seen in PPSSPP: `QuickTexHash` clamps `h` to `max(272, entry->maxSeenV)` for 512-tall textures, using the UV bounds seen by the vertex decoder. `BindFramebufferColor(…, BINDFBCOLOR_MAY_COPY_WITH_UV)` blits only `vertMinU..vertMaxU × vertMinV..vertMaxV`. CLUT readback uses `packWidth` instead of the full `fb_stride` when `h == 1`. Module loading skips the import and export stub tables when scanning for functions. f4df7f076e 2015-03 “Hash less of 512 tall textures when possible.” · 62de281e35 2015-09 “Use texture UV range to optimize framebuf copies.” · 095d8cb52a 2016-01 “Avoid download/conv of full rows.” · 29ed48c32a 2018-01 “Module: Avoid scanning stubs if possible.” · 0e44c3b9fe 2017-05 “Add reduceHash option, to hash 50% of texture when it's otherwise unreliable.”

### ppsspp-115 — Track which copy is current to skip redundant transfers
Lens: io · Source: PPSSPP
Principle: When data lives in two places (GPU and CPU, cache and origin), keep a flag or timestamp saying which side holds the newest version. Copy only when the destination is actually stale. A readback that duplicates the last one, or a clear that is already in memory, then costs nothing.
Why it is faster: —
Applies elsewhere: MESI-style dirty bits in a write-back cache, `If-Modified-Since` requests, and a sync client that compares a version number before uploading.
Watch out: —
Seen in PPSSPP: Each `VirtualFramebuffer` gained `clutUpdatedBytes`, so a CLUT that is reloaded but not re-rendered is not downloaded again. The commit calls it a "HUGE win" in Brave Story. `memoryUpdated` now also skips re-zeroing VRAM and skips downloading a just-cleared buffer. Depth blits between framebuffers compare `last_frame_depth_render` with `last_frame_depth_updated` and run only when the source depth is newer. 19877144ba 2016-01 “Avoid redownloading CLUT when reloaded.” · 70d17d1bc7 2017-04 “Track flags to reduce unnecessary VRAM zeroing.” · 4a8c661c00 2017-04 “Skip downloads of cleared VRAM.” · 0956fa3ff6 2017-12 “GPU: Reduce depth blits when not updated.”

### ppsspp-116 — Defer a decision until the information that settles it exists
Lens: laziness · Source: PPSSPP
Principle: Doing an expensive step early, before you know whether it is needed, means sometimes doing it for nothing. Set a "needs X" flag and run X at the latest point, after the inputs that decide it, or that it depends on, are known.
Why it is faster: —
Applies elsewhere: a lazy-evaluated query builder that only hits the DB on iteration, or a compositor that binds a texture only once it knows the layer is visible.
Watch out: —
Seen in PPSSPP: The shader-blend framebuffer copy now just sets `fboTexNeedBind_ = true`. The bind happens in a new `ApplyDrawStateLate()` after vertex decode, once full-alpha information is known. Texture hashing and rebuild moved from `SetTexture` to the end of texture apply, after vertex decode, in GLES, D3D9 and Vulkan. 5360422310 2015-03 “Defer bind of shader blend copy until after verts.” · 6bd86f462d 2016-05 “Move texture hashing to end of texture apply.” · f1dda6a53d 2016-05 “d3d9: Move texture hashing after vertex decode.”

### ppsspp-117 — Analyze data before it is expanded
Lens: algorithms · Source: PPSSPP
Principle: If a property survives a transformation, such as upscaling or decompression, compute it on the small input instead of the large output. The cost shrinks by the expansion factor.
Why it is faster: —
Applies elsewhere: checking an image for transparency before resizing, computing a checksum on the compressed stream, or validating a template before rendering.
Watch out: —
Seen in PPSSPP: In every texture cache backend, `CheckAlpha` moved to before `scaler.Scale`. Scaling cannot invent alpha from a fully opaque texture, so the scan now covers 1/scaleFactor² as many pixels. e3b3828b15 2017-11 “TexCache: Check alpha before scaling.”

### ppsspp-118 — Skip synchronization when running single-threaded
Lens: concurrency · Source: PPSSPP
Principle: A queue that might be shared across threads pays for locks and condition-variable signals on every operation. When a runtime flag says only one thread is involved, take a lock-free path with the same logic and none of the atomic or kernel overhead.
Why it is faster: —
Applies elsewhere: allocators that skip the mutex before the first thread is spawned, and logging libraries with a single-thread mode.
Watch out: —
Seen in PPSSPP: `ThreadEventQueue`'s `ScheduleEvent`, `HasEvents`, `GetNextEvent` and `RunEventsUntil` now branch on `threadEnabled_`. When it is false they touch `events_` directly, with no `lock_guard` or `notify_one`. bc3fb79fa2 2015-01 “If multithreading isn't enabled, don't bother synchronizing the ThreadEventQueue.”

### ppsspp-119 — Remove locks from a single-producer/single-consumer audio path
Lens: concurrency · Source: PPSSPP
Principle: An audio callback that blocks on a mutex risks glitches and adds latency. A ring buffer with one writer and one reader needs only atomic indices. Dropping the mixing lock also lets the platform choose a native low-latency "fast path" sample rate.
Why it is faster: —
Applies elsewhere: JACK and CoreAudio callbacks, and lock-free sensor sample rings in embedded firmware.
Watch out: —
Seen in PPSSPP: `StereoResampler::Mix` lost its `lock_guard lk(m_csMixing)` and `recursive_mutex`, and the "Atomic Audio locks" option went away. Additive mixing with clamping became direct writes through `clamp_s16`. Native audio no longer forces 44.1 kHz, so the Nexus 4 and Nexus 9 hit their fast path. 0c6ceee210 2015-01 “Optimize the resampler a little.”

### ppsspp-120 — Move self-contained heavy work to a background thread
Lens: concurrency · Source: PPSSPP
Principle: Work whose result isn't needed right away, or that can run alongside the main loop, can go to another core. Examples are compression, audio mixing, prefetching and API submission. The main thread only hands off inputs and syncs before it next needs the result.
Why it is faster: —
Applies elsewhere: web workers for image decoding, a render thread separate from game logic, or compressing log files on a worker.
Watch out: —
Seen in PPSSPP: SAS audio mixing can run on a `__SasThread` woken by a condition variable. Rewind snapshots are compressed in a detached `ScheduleCompress` thread. OpenGL commands are recorded and run on a separate render thread through `GLRenderManager`. FFmpeg video decoding is opened with `av_dict_set(&opt, "threads", "0", 0)`. 6290ac6cbd 2013-07 “Sas: Process SAS on a separate thread.” · 5faa65d9a9 2016-09 “Compress rewind snapshots off thread.” · af6431986d 2018-01 “OpenGL: Now run GL on a secondary thread.” · e5d571c25a 2018-01 “gl-render-manager: Now threaded mode seems to work fine (and fast!).” · 7a7e4ed23f 2017-03 “Video: Enable threads for video decoding.”

### ppsspp-121 — Split parallel work along the longer axis
Lens: concurrency · Source: PPSSPP
Principle: Splitting a 2-D job only by rows gives too few or too unbalanced slices for wide, short shapes. Pick the axis with more work units so every thread gets a useful share.
Why it is faster: —
Applies elsewhere: tiling image filters, partitioning matrix multiplications by the larger dimension, and sharding by whichever key has more cardinality.
Watch out: —
Seen in PPSSPP: The software rasterizer's `DrawTriangle` computes `rangeX` and `rangeY`. If `rangeY >= 12 && rangeX >= rangeY * 4`, it calls `GlobalThreadPool::Loop` over X slices, with `DrawTriangleSlice(…, byY=false, a, b)`, instead of Y slices. 176283c3f1 2017-11 “SoftGPU: Thread by width for wide polygons.”

### ppsspp-122 — Dedicated fast path for the dominant special case
Lens: branching-fast-paths · Source: PPSSPP
Principle: A general routine pays for generality on every call. When one input shape dominates, such as a full clear, a large send into an empty buffer or a stride that allows a direct upload, detect it up front and use code that does only what that case needs.
Why it is faster: —
Applies elsewhere: `memset` for zero-fill instead of a generic pattern fill, `writev` directly when the userspace buffer is empty, or zero-copy uploads when the row pitch matches.
Watch out: —
Seen in PPSSPP: For clear-mode rectangles the software clipper now calls `Rasterizer::ClearRectangle`, a scissor-clamped fill, instead of rasterizing four triangles. The HTTP `OutputSink` calls `send()` directly when nothing is buffered and the write is larger than `PRESSURE`. GLES `DrawPixels` widens the texture to `srcStride` and uses `GL_UNPACK_ROW_LENGTH`, which avoids a repacking copy for video frames. ed34cf06b8 2017-11 “SoftGPU: Add a clear mode fast path.” · 11c7c25b03 2016-05 “http: Add a fast-path for large chunked sends.” · 1b02734e56 2017-03 “GL: Avoid common-case shuffling video/raw draws.”

### ppsspp-123 — Early-out in a shader when the effect would not be visible
Lens: gpu-rendering · Source: PPSSPP
Principle: A costly per-pixel filter is pointless when its output can't differ visibly from a plain sample. Compute a uniform-based condition and branch around the filter. GPUs handle uniform branches cheaply.
Why it is faster: —
Applies elsewhere: skipping SSAO or blur at tiny render scales, or bypassing a sharpening filter when the display scale is 1:1.
Watch out: —
Seen in PPSSPP: The 5xBR post-shaders compute `bool upscale = u_texelDelta.x > (1.6 * u_pixelDelta.x)` and wrap the whole edge-detection body in `if (upscale)`. Otherwise they return the plain texture sample. f71759020d 2017-03 “Skip 5xBR based upscaling if output < (1.6 * input)”

### ppsspp-124 — Put a faster storage tier in front of slow I/O
Lens: io · Source: PPSSPP
Principle: Wrap a slow reader (network, compressed or optical image) in a loader that caches blocks in RAM or on local disk. A background thread can fill the cache ahead of demand. Later reads hit the fast tier, and startup isn't blocked by a full preload.
Why it is faster: —
Applies elsewhere: a browser HTTP cache, a read-ahead block cache in front of S3, or a local SQLite cache of API responses.
Watch out: —
Seen in PPSSPP: `DiskCachingFileLoader` stores fixed-size blocks of remote or HTTP files in a cache directory, with an index and garbage collection when space runs low. `RamCachingFileLoader` replaced the blocking whole-ISO RAM load with an `aheadThread_` that fills blocks in the background. It caches the compressed CSO data, which uses less memory. 88b23460f1 2015-06 “Add a file loader to perform disk caching.” · dd5c91108c 2015-12 “Use a background thread to load ISO into RAM.”

### ppsspp-125 — Persist and pre-warm a compile cache across runs
Lens: io · Source: PPSSPP
Principle: When compiling something on first use causes a stall, record which items were compiled. On the next run, compile them all up front, during loading. Even if the platform can't save binaries, saving the keys moves the cost out of the interactive phase.
Why it is faster: —
Applies elsewhere: V8 code caching, Python `.pyc` files, and precompiling Vulkan pipelines from a recorded key list during a loading screen.
Watch out: —
Seen in PPSSPP: `ShaderManager::LoadAndPrecompile` reads `<DISC_ID>.shadercache`, a header plus vertex and fragment shader IDs and linked-program pairs, and compiles them at boot. The file is saved every 256 flips and discarded when `featureFlags` differ. b246442374 2016-01 “Hook up GL shader cache. Cache linked shader programs.”

### ppsspp-126 — Fuse adjacent operations that share work
Lens: jit-interpreters · Source: PPSSPP
Principle: When two consecutive operations compute the same expensive intermediate value, such as the same sin/cos of the same input, a compiler can detect the pair and emit one computation feeding both outputs.
Why it is faster: —
Applies elsewhere: fusing `sin(x)` and `cos(x)` into `sincos`, merging adjacent map/filter stages, or combining two SQL scans of the same table.
Watch out: —
Seen in PPSSPP: The ARM JIT's `Comp_VRot` peeks at `js.compilerPC + 4`. If the next op is another `vrot` with the same source `vs`, it calls `SinCos` once, emits both shuffles (flipping the sign if `negSin` differs), and advances `compilerPC`. 80ce45f9fb 2014-12 “ARM Jit: Join adjacent vrot ops on ARM too”

### ppsspp-127 — Forward stored values to subsequent loads
Lens: jit-interpreters · Source: PPSSPP
Principle: A load from the same address you just stored to can be replaced by a register move of the stored value. That removes a memory round trip and turns a store→load chain into a cheap copy.
Why it is faster: —
Applies elsewhere: compiler mem2reg and store-to-load forwarding, or a Redux middleware returning the just-written value without re-reading the store.
Watch out: —
Seen in PPSSPP: An IR pass in `IRPassSimplify.cpp` remembers the previous instruction. On `Load32` at the same `src1`/`src2` as a preceding `Store32` or `StoreFloat`, it rewrites the load to `IROp::Mov` or `IROp::FMovToGPR` from the stored register, or drops it if the register is the same. a450a79f52 2016-05 “jit-ir: Optimize loads to transfers if possible.”

### ppsspp-128 — Constant and copy propagation over a linear IR
Lens: jit-interpreters · Source: PPSSPP
Principle: Lowering to a simple IR makes it easy to run passes that track known constants and register copies. Uses of a temporary can be rewritten to the original register, so the copy dies. Constants fold into immediates, and fewer instructions force a flush.
Why it is faster: —
Applies elsewhere: SSA optimizers in LLVM and V8, query planners folding constant predicates, and build systems propagating known config values.
Watch out: —
Seen in PPSSPP: `PropagateConstants` in `IRPassSimplify.cpp` uses the IR regcache to track `SetConst` values and fold them into later ops. `PurgeTemps` learned to replace a temp copy's source (`IRReplaceSrcGPR`) when the original register hasn't been rewritten since (`lastWrittenTo`), which removes the copy in common delay-slot patterns. 09969c0156 2016-05 “Use the regcache in a new (incomplete) pass, PropagateConstants.” · cd3f4881a5 2018-01 “irjit: Optimize out temp lhs copies.”

### ppsspp-129 — Drop guard sequences whose effect you don't model
Lens: jit-interpreters · Source: PPSSPP
Principle: A compiler can pattern-match idioms that have no observable effect in its model, such as a trap after a divide-by-zero check that only stalls, and emit nothing for them. Downstream passes then also see simpler control flow.
Why it is faster: —
Applies elsewhere: eliding debug-only assertions in release builds, or a bytecode optimizer removing a `DUP; POP` pair.
Watch out: —
Seen in PPSSPP: In the IR frontend's `BranchRSRTComp`, a likely branch of offset 4 whose delay slot is `MIPS_IS_BREAK(op)` is now eaten. It emits no branch and decrements `js.downcountAmount` so cycle counting stays balanced. 066b0b7fdf 2016-05 “jit-ir: Optimize out beql; break; sequences.”

### ppsspp-130 — Register allocator: discard dead values instead of writing them back
Lens: jit-interpreters · Source: PPSSPP
Principle: Spilling a register normally stores it to memory. If lookahead shows the value is overwritten before it is read, or the next call clobbers it anyway, or it was never modified, skip the store and just free the register.
Why it is faster: —
Applies elsewhere: a write-back cache dropping clean lines without writing them, or an OS skipping swap-out of clean file-backed pages.
Watch out: —
Seen in PPSSPP: `FindBestToSpill` in the ARM and x86 regcaches prefers registers that `MIPSAnalyst::IsRegisterClobbered` reports as overwritten soon, then calls `DiscardR` instead of `FlushArmReg`. Before a syscall, T4–T9, HI and LO are discarded because HLE fills them with `0xDEADBEEF`. ARM64 `FlushAll` stores only `if (mreg.loc == ML_IMM || ar[r].isDirty)`. f817d49dfb 2014-12 “jit: Discard clobbered registers on spill.” · 179e996b0b 2015-03 “jit: Discard unused regs before a syscall.” · 9573a791b4 2017-12 “arm64jit: Skip storing spilled but not dirty.”

### ppsspp-131 — Cache a derived host pointer in the register itself
Lens: jit-interpreters · Source: PPSSPP
Principle: If every memory access computes `base + guest_addr`, keep the register in "pointer form" by setting its upper bits to the base once. Later accesses then use a single load with an immediate offset. Invalidate that form when the register is written.
Why it is faster: —
Applies elsewhere: keeping a pre-offset base pointer to a struct array in a loop, or tagged pointers whose untagged form is cached for repeated dereferencing.
Watch out: —
Seen in PPSSPP: `Arm64RegCache::MapRegAsPointer` emits `MOVK(X0 + a, Memory::base >> 32, SHIFT_32)` once and sets `ar[a].pointerified`. Load/store, FPU `lwc1`/`swc1` and VFPU `lv.s`/`sv.s` then emit `LDR(INDEX_UNSIGNED, rt, RPtr(rs), offset)` directly. The commit reports a noticeable gain. ad648baa9c 2015-03 “ARM64 regcache: Add support to "pointerify" registers. Use in load/store to cut down instructions.” · 8eedcc7fb0 2015-03 “ARM64: Speedup fpu/vfpu load/stores too using "pointerification". Actually noticeable gain.”

### ppsspp-132 — Break false dependencies and stay in one execution domain
Lens: jit-interpreters · Source: PPSSPP
Principle: Out-of-order CPUs track dependencies per register. Instructions that only partly write a register, like MOVK or MOVSS, make the CPU wait for the old value. Mixing integer and float SIMD ops on the same data adds bypass delays. Choose full-width writes and same-domain instructions.
Why it is faster: —
Applies elsewhere: `xor eax,eax` zero idioms, preferring `movaps` for register copies, and `vxorps` instead of `pxor` in float code.
Watch out: —
Seen in PPSSPP: ARM64 `MOVI2R` now always starts with `MOVZ`, because the comment says it "kills the dependency on the register". The x86 JIT added `CopyFPReg`, which uses `MOVAPS` rather than `MOVSS` for reg→reg copies. `abs.s` switched from `PAND` to `ANDPS` to stay in the float domain. 80a813bcbd 2015-03 “ARM64 Emitter: Optimize MOVI2R by always starting with a MOVZ, killing the register dependency…” · 7e38df077f 2015-04 “x86jit: Prefer MOVAPS over MOVSS for reg->reg.” · 9069c84928 2015-04 “x86jit: Use ANDPS for abs.s.”

### ppsspp-133 — Encode known constants directly into instructions
Lens: jit-interpreters · Source: PPSSPP
Principle: If a value is known at compile time, use the instruction's immediate form (store-immediate, FMOV #imm8, the zero register) instead of materializing it in a scratch register first. That saves instructions and frees registers.
Why it is faster: —
Applies elsewhere: compilers folding `x = 0` into `str wzr`, or a SQL engine inlining literal parameters.
Watch out: —
Seen in PPSSPP: x86 `CompITypeMemWrite` now stores any immediate `rt` directly with `MOV(bits, dest, DowncastImm(gpr.R(rt), bits))`, where previously only zero was handled. ARM64 `mtc1` with a known value uses `FMOV(fs, WZR)` for zero, or `FMOV(fs, imm8)` when `FPImm8FromFloat` succeeds. 6bf2c02908 2014-12 “x86 jit: Allow storing all imms directly without bouncing to a register, not just zero.” · 8eedcc7fb0 2015-03 “ARM64: Speedup fpu/vfpu load/stores too…”

### ppsspp-134 — Use the widest load/store the data allows
Lens: memory-layout · Source: PPSSPP
Principle: Moving a 12-byte record as three 32-bit moves, or a 16-bit field as two byte loads plus an OR, costs extra instructions and ports. Use 64-bit or unaligned-capable halfword loads when the platform supports them.
Why it is faster: —
Applies elsewhere: `memcpy` implementations using the widest register, packet parsers reading `uint64_t` with `memcpy`, and NEON `ld1` for struct arrays.
Watch out: —
Seen in PPSSPP: The x64 vertex decoder copies float3 positions and normals with one 64-bit and one 32-bit `MOV` instead of three 32-bit ones. The commit reports 270%→340% for position-only vertices. ARM64 `Jit_NormalS8` replaced `LDRB`+`LDRB`+`ORR` with `LDURH`, which allows odd offsets (5x→5.66x). 0fe927aaa7 2017-05 “x86: Minor memory copy perf improvement.” · 5788e33b2f 2017-12 “arm64: Use LDRUH instead of two LDRBs.”

### ppsspp-135 — Page-bucket index for range invalidation
Lens: algorithms · Source: PPSSPP
Principle: Answering "which items overlap [addr, addr+len)?" by scanning all items is O(n) per query, which hurts when queries are frequent and small. Bucket items by fixed-size page when they are inserted, and check only the buckets the range touches.
Why it is faster: —
Applies elsewhere: spatial hashing for collision queries, interval lookup in a memory profiler, or invalidating cached tiles on a map.
Watch out: —
Seen in PPSSPP: `IRBlockCache::FinalizeBlock` appends the block index to `byPage_[page]` for each page in its range. `InvalidateICache` iterates `AddressToPage(address)..AddressToPage(address + length)` and checks only those blocks, instead of looping over all `size_`. The old scan was slow in games that do many small memcpys. 3af78883c7 2017-12 “irjit: Speed up icache block invalidation.”

### ppsspp-136 — Skip per-item teardown when the whole set is discarded
Lens: other · Source: PPSSPP
Principle: When you destroy everything at once, per-item cleanup that only keeps the remaining items consistent, like unlinking neighbors or fixing references, is wasted. Pass a "clearing all" mode that skips it.
Why it is faster: —
Applies elsewhere: arena allocators freed in one call instead of per object, `TRUNCATE` vs `DELETE`, and dropping a whole DOM subtree without detaching each listener.
Watch out: —
Seen in PPSSPP: `JitBlockCache::DestroyBlock` takes a `DestroyType` (DESTROY/INVALIDATE/CLEAR). Under `CLEAR` it skips `UnlinkBlock`. That avoids repeatedly flipping page protections on W^X platforms, since the code space is wiped anyway. 8c3b0aa89c 2017-12 “jit: Skip unlinking blocks on clear.”

### ppsspp-137 — Loop fission: count first, then fetch in one batch
Lens: algorithms · Source: PPSSPP
Principle: A loop that interleaves "decide how much" with "fetch one item" calls the fetch function per element and can't use its bulk path. Split it into a counting pass and a single bulk read, then a processing pass over the gathered buffer.
Why it is faster: —
Applies elsewhere: issuing one batched DB query instead of N+1 queries, or reading a known number of bytes with one `read()` instead of byte by byte.
Watch out: —
Seen in PPSSPP: The SAS voice mixer was split into two passes. The first collects samples into `temp[256*4]`, keeping the two history samples at the front, and the second resamples. Then the read pass stopped calling `voice.ReadSamples(&temp[tempPos++], 1)` per step. It counts `samplesToRead` and makes one `ReadSamples` call. 47df134b66 2016-12 “Optimization step 1: Separate the loops, save history from buffer.” · 6782f7c75d 2016-12 “Just figure out how many samples to read, then read it, instead of one-by-one.”

### ppsspp-138 — Stream GPU uploads through linear sub-allocated buffers
Lens: gpu-rendering · Source: PPSSPP
Principle: Creating a new image or memory allocation per upload is slow, and so is locking a resource the GPU may still be reading. Sub-allocate from big slabs or a per-frame push buffer, and tell the driver when old contents can be thrown away, so it never has to sync or copy.
Why it is faster: —
Applies elsewhere: WebGPU staging ring buffers, `GL_MAP_INVALIDATE_BUFFER_BIT`, and D3D11 `Map(WRITE_DISCARD)` for dynamic vertex buffers.
Watch out: —
Seen in PPSSPP: Vulkan textures allocate from a `VulkanDeviceAllocator` slab (`allocator_->Allocate(mem_reqs, &mem)`) instead of calling `vkAllocateMemory` each time. Hardware tessellation data moved from temporary `VulkanTexture`s to `push_->Push(…)` regions, and later to a storage buffer bound with `PushAligned`. D3D9 full-texture updates pass `D3DLOCK_DISCARD` to `LockRect`. Vulkan readback converts color while copying out of the readback buffer, which saves a copy. 27a5697a96 2016-03 “Vulkan: Use the slab allocator for textures.” · 9cd4db5f18 2017-11 “Vulkan hw tess: Use pushbuffers instead of temp images to upload data.” · 4346a54eb7 2017-11 “Vulkan: Speed up and simplify hardware tesselation by using storage buffers.” · b94ad12aea 2017-02 “D3D9: Use D3DLOCK_DISCARD when updating whole textures, may be a small speedup” · 93c785b76d 2017-10 “Color-convert directly during the readback, saves a copy.”

### ppsspp-139 — Keep container entries small and stable
Lens: memory-layout · Source: PPSSPP
Principle: Large values stored inline in a map make rehashing, node allocation and iteration move more bytes, and their addresses can't be held safely. Shrink fields to the narrowest type that fits and store big entries behind a pointer.
Why it is faster: —
Applies elsewhere: `std::map<K, unique_ptr<Big>>` in C++, struct packing for embedded tables, and Rust's `Box` for large enum variants.
Watch out: —
Seen in PPSSPP: `TexCacheEntry::maxLevel` went from `int` to `u8`. Later, `cache_` and `secondCache_` switched to storing `std::unique_ptr<TexCacheEntry>`, created with `new TexCacheEntry{}` and accessed via `iter->second.get()`. 71afaffc20 2015-03 “Pack texture cache entries just a bit tighter.” · 195eb94c8e 2017-02 “Texcache: Keep pointers to entries instead of full entries in the map.”

### ppsspp-140 — Pick the hash function per target from benchmarks
Lens: platform · Source: PPSSPP
Principle: Hash throughput depends on the word size and the microarchitecture. A 64-bit hash can beat its 32-bit sibling on 64-bit cores. Measure on each target and select by build configuration.
Why it is faster: —
Applies elsewhere: choosing CRC32C hardware instructions where available, or picking xxh3 vs murmur by platform in a content-addressed store.
Watch out: —
Seen in PPSSPP: On ARM64, `DoReliableHash` and `DoReliableHash64` are now `#define`d to `XXH64`, with `ReliableHashType` as `u64`, because benchmarks on a Cortex-A57 showed it faster than XXH32. This hash covers vertex and texture data. 7f8cf7f800 2015-07 “arm64: Use XXH64 for vertex hashing, etc.”

### ppsspp-141 — Precompute a full lookup table for a pure function of a small key
Lens: lookup-tables · Source: PPSSPP
Principle: If an expensive pure function depends on a few bytes of input, tabulate every value once. Each call becomes one memory load. The table's build time and memory are paid once and amortized over millions of calls.
Why it is faster: —
Applies elsewhere: gamma and sRGB tables, CRC tables, and per-byte popcount or bit-reversal tables.
Watch out: —
Seen in PPSSPP: The bundled xBRZ 1.2 adds `DistYCbCrBuffer`, a 256³ table indexed by packed (r,g,b) differences that holds the YCbCr distance. It replaces per-call double-precision math, which upstream credits with a 30% speedup. 3c9150231d 2015-02 “Update to xBRZ 1.2, with perf improvements.”

### ppsspp-142 — Back off polling of absent devices
Lens: io · Source: PPSSPP
Principle: Polling an empty slot every frame wastes CPU on calls that are slow precisely because nothing answers. After a miss, wait N frames before checking that slot again, and reset the delay as soon as the device shows up.
Why it is faster: —
Applies elsewhere: exponential backoff when reconnecting sockets, USB hot-plug polling, and BLE scan intervals.
Watch out: —
Seen in PPSSPP: `XinputDevice::UpdateState` sets `check_delay[i] = 30` when `XInputGetState` reports a disconnected pad, and staggers the initial delays (`check_delay[i] = i`). This cut idle CPU use when no controller is connected. 0bd31e1567 2017-04 “Windows: Reduce CPU usage with no XInput connected.”

### ppsspp-143 — Post notifications asynchronously instead of waiting for them
Lens: concurrency · Source: PPSSPP
Principle: A synchronous cross-thread call blocks the caller until the receiver handles it, which serializes both threads and invites deadlocks. When the caller doesn't need a result, enqueue the message and continue.
Why it is faster: —
Applies elsewhere: `postMessage` vs blocking RPC, `dispatch_async` vs `dispatch_sync` to the main queue.
Watch out: —
Seen in PPSSPP: `WindowsHost::NotifySymbolMapUpdated` switched from `SendMessage(mainWindow_, WM_USER + 1, …)` to `PostMessage`. Boot no longer waits on the UI thread for every symbol-map update. 4443c060ce 2017-05 “Slightly speed up booting when there are symbols to load.”

### ppsspp-144 — Rebuild derived objects only when their inputs changed
Lens: laziness · Source: PPSSPP
Principle: UI menus, loaders and other derived objects are often rebuilt on every event even when nothing they depend on changed. Compare the inputs (language ID, list contents, path) with what was used last time and return early when they match.
Why it is faster: —
Applies elsewhere: React's `useMemo`, Make-style timestamp checks, and reusing an open DB connection when the DSN hasn't changed.
Watch out: —
Seen in PPSSPP: The Windows menu code keeps `menuLanguageID` and `menuShaderInfo`. `CreateShadersSubmenu` returns early when `std::equal` shows the post-shader list is unchanged, and post-shader info is reloaded only when a menu actually opens. `GameInfo::GetFileLoader` reconstructs the loader only `if (filePath_ != gamePath)`. 6642bf3170 2017-04 “Windows: Make UI update significantly cheaper.” · 4fa529db50 2015-06 “Potentially reuse a cached file loader.”

### ppsspp-145 — Move tiny hot methods into the header so they inline
Lens: compiler-build · Source: PPSSPP
Principle: Small functions defined in a .cpp file can't be inlined across translation units without LTO, so every call pays call and return overhead and blocks optimizations around it. Define tiny, hot accessors and predicates inline in the header.
Why it is faster: —
Applies elsewhere: header-only getters in C++ libraries, `static inline` helpers in C headers, and `@inline` hints in Kotlin or Scala.
Watch out: —
Seen in PPSSPP: `IndexGenerator::Reset` and both `PrimCompatible` overloads moved from `IndexGenerator.cpp` into the class body in `IndexGenerator.h`. The `indexedPrimitiveType` table became a static class member so the inline code can reach it. efa8cfc736 2017-01 “IndexGenerator microoptimization”

### ppsspp-146 — A tight inner loop for the hot subset of commands
Lens: branching-fast-paths · Source: PPSSPP
Principle: If a stream of commands is dominated by a few opcodes, put a small dedicated loop in front of the general dispatcher. It handles only those opcodes, does minimal bookkeeping, and bails to the full path on anything else. The common case then skips the general loop's per-command overhead: validity checks, state flushes and big switches.
Why it is faster: —
Applies elsewhere: a bytecode VM that fuses the 5 most frequent opcodes into a fast loop; an HTTP parser with a fast path for plain ASCII headers; a log ingester that special-cases its most common line shape.
Watch out: —
Seen in PPSSPP: In `GPUCommon`'s PRIM handling, the "inner interpreter" that keeps consuming draw commands now also handles `OFFSETADDR`, `TEXSCALEU/V`, `TEXOFFSETU/V`, `CULL`, `BONEMATRIXNUMBER`, and a `CALL` that is detected as a 12-word bone-matrix upload (`FastLoadBoneMatrix`). Redundant `VERTEXTYPE` writes are now detected with `data ^ vertexType`, masking off the bits that don't matter. The commit reports about 4% faster in God of War. 50de13d3d7 2018-03 “Optimize the "inner interpreter" for God of War character drawing” · cfc353ee81 2018-04 “handle cull mode by indices, improve performance for GVGNP”

### ppsspp-147 — Encode a per-item flag into the data to keep a batch together
Lens: gpu-rendering · Source: PPSSPP
Principle: When a small state change would split a batch, check whether that state can be expressed in the batched data instead. Then the batch keeps going and nothing gets flushed. Each break costs a full submit, so absorbing the change into the payload keeps the batches large.
Why it is faster: —
Applies elsewhere: put a per-sprite flip bit in the vertex data instead of changing pipeline state; send a per-row "op" column in a bulk SQL insert instead of separate statements.
Watch out: —
Seen in PPSSPP: A change to the cull mode used to end a deferred draw batch. Each `DeferredDrawCall` now records its `cullMode`. `IndexGenerator::TranslateStrip` flips the triangle winding (`wind = cullMode == 1 ? 2 : 1`) for strips with the opposite cull face, and the inner interpreter now treats `GE_CMD_CULL` as non-breaking. cfc353ee81 2018-04 “handle cull mode by indices, improve performance for GVGNP, refer to #10172”

### ppsspp-148 — Group hot globals by how often they are used together
Lens: memory-layout · Source: PPSSPP
Principle: Variables that are declared next to each other usually sit next to each other in memory. Order globals and function-pointer tables by how often they are used, so the hot entries share a few cache lines. Where possible, call the resolved target directly rather than going through a trampoline.
Why it is faster: —
Applies elsewhere: put a struct's hot fields first and its rarely used fields at the end; in a plugin system, resolve function pointers once at load time instead of looking them up on every call.
Watch out: —
Seen in PPSSPP: `VulkanLoader.cpp` re-sorted its `PFN_vk*` globals into "used frequently", "every frame" and "rare" groups. A later commit loads queue, memory, fence and command functions with `vkGetDeviceProcAddr` instead of instance-level pointers. That skips the loader's dispatch bounce on every command. cd75c46d47 2018-03 “Sort Vulkan function pointers by use. Might save us a cacheline or two during command submission..” · f4d348ca0c 2019-08 “Micro-optimize the Vulkan loader: Load more device-specific function pointers to avoid a bounce in the device loader”

### ppsspp-149 — Cache recipes, not outputs, and warm the cache in the background
Lens: io · Source: PPSSPP
Principle: When outputs are expensive to build and large to store, persist the small keys that describe how to build them. On the next run, rebuild everything from those keys before it is needed, on a background thread. You avoid the first-use stall without storing huge driver blobs, and startup isn't blocked while the cache warms.
Why it is faster: —
Applies elsewhere: store the list of templates or regexes a server used last time and precompile them at boot; warm a JIT with a profile of the hot methods; prefetch the routes a user visited last session.
Watch out: —
Seen in PPSSPP: The Vulkan backend stores pipeline and shader IDs (as the GL backend already did), not the raw `VkPipelineCache` data, which had grown large. `DecVtxFormat::InitializeFromID` rebuilds vertex formats from those IDs. `GPU_Vulkan` then runs `LoadCache` on a `std::thread` and reports `IsReady()` when it finishes. Later commits add the render-pass definition to each cache entry and de-duplicate pipelines before saving. 614cabb115 2018-03 “Implement pipeline/shader cache for Vulkan, to avoid shader compile stutters on second and subsequent runs.” · 7ae4a9e977 2018-03 “Vulkan: Load shaders/pipelines on thread.”

### ppsspp-150 — Process only the delta, not the whole accumulated set
Lens: algorithms · Source: PPSSPP
Principle: If each incremental step re-walks everything collected so far, total cost grows quadratically. Collect the new items in a separate list, process only those, and append them to the main list afterwards. Do whole-set finalization once, at the end.
Why it is faster: —
Applies elsewhere: incremental builds that hash only changed files; a search index that indexes only new documents; a UI list that renders only appended rows.
Watch out: —
Seen in PPSSPP: `MIPSAnalyst::ScanForFunctions` used to insert all known functions into the symbol map on each call. It now pushes into a local `new_functions` vector and inserts only those. Separately, module loading moved hashing out of each scan and into a single `FinalizeScan` call after all ranges are scanned. That fixed a regression in loading speed. 8ddd2b6104 2020-02 “ScanForFunctions: Drastically speed up by only trying to insert the newly found functions into the symbol map.” · f14f2efa79 2018-03 “Module: Hash functions only once during loading.”

### ppsspp-151 — Size fields to the range they actually hold
Lens: memory-layout · Source: PPSSPP
Principle: If a value never needs more than 8 or 16 bits, store it in that width. Narrow fields let structs pack together, so more records fit in each cache line and queue entries copy faster. In JIT-generated code, a narrower element also means a smaller load and simpler address scaling.
Why it is faster: —
Applies elsewhere: use `uint16_t` port numbers and `uint8_t` enum tags in hot network-flow tables; store columnar data with narrow integer types.
Watch out: —
Seen in PPSSPP: `DeferredDrawCall::vertexCount` went from `u32` to `u16`. A later commit restored 32 bits because tessellation needed them, so the range really has to be known. `clearStencil` in the Vulkan render-step struct became `u8` so it fits beside the load actions. `bufw`/`dim` in the texture-match struct became `u16`. The softjit dither matrix went from `int16_t[16]` to `int8_t[16]`, which makes the address computation a single `LEA` with scale 4 and the load a `MOVSX 8`. 648b4eac51 2018-03 “Shrink the vertexCount field in DeferredDrawCall to the 16 bits required.” · 18e70e67f9 2022-06 “Small struct layout optimization” · a854fbbe76 2022-09 “fbtex matching: Shrink the matching struct” · aec41b34d6 2021-12 “softjit: Reduce ditherMatrix to 8-bit.”

### ppsspp-152 — Make rejected pixels invisible instead of discarding them
Lens: gpu-rendering · Source: PPSSPP
Principle: A `discard` in a fragment shader stops the GPU from doing depth and stencil testing before shading, and it is costly on tile-based GPUs. If the blend mode makes zero alpha a no-op (src-alpha / inv-src-alpha, no depth write, no stencil write), set alpha to 0 instead. The result looks identical and the early tests stay enabled.
Why it is faster: —
Applies elsewhere: foliage and UI rendering in game engines; WebGL sprite batchers that use premultiplied alpha instead of alpha-test.
Watch out: —
Seen in PPSSPP: `NeedsTestDiscard()` in `GPUStateUtils.cpp` checks that stencil and depth writes are off, that blend is a safe src-alpha/inv-src-alpha form with an additive equation, and that logic op is COPY. If all of that holds, it sets `FS_BIT_TEST_DISCARD_TO_ZERO`, and the generators emit `v.a = 0.0;` instead of `discard;`. Vulkan then keeps `earlyFragmentTests`. The same rule was used to drop `alpha > 0` tests completely. 5ad948df73 2018-07 “Vulkan: Avoid discard when we can blend.” · 7885a88c0c 2018-07 “GPU: Optimize > 0 alpha test using discard rules.”

### ppsspp-153 — Write each output once when the destination is uncached
Lens: memory-layout · Source: PPSSPP
Principle: Write-combined or uncached memory, such as mapped GPU buffers, is very slow to read back and to write piecemeal. Gather all the parts of an output element in registers and store each element exactly once. Never write in two passes where the second pass reads back what the first one wrote.
Why it is faster: —
Applies elsewhere: fill a `glMapBuffer` or framebuffer region sequentially; avoid read-modify-write on MMIO or PCIe-mapped memory in drivers.
Watch out: —
Seen in PPSSPP: The DXT3/DXT5 decoder used to write the colours first and then patch the alpha into the same destination. A `DXTDecoder` class now decodes colours and alpha into `colors_[4]`/`alpha_[8]` first, and `WriteColorsDXT3/5` emits the final pixel in one store. The commit says this is "significantly faster on Vulkan" and when decoding directly into uncached memory. A later commit has the CPU texture scaler, which reads back its own output, write into an aligned buffer in cached RAM and copy it to the mapped memory afterwards. 8ae2b1e6fb 2018-09 “TexCache: Optimize DXT3/DXT5 decode to single pass.” · ac909d2192 2020-03 “Vulkan: Ensure scaling happens in cached RAM.”

### ppsspp-154 — Turn divisions into multiplies and shifts
Lens: algorithms · Source: PPSSPP
Principle: Division is one of the slowest scalar operations. For a known divisor, multiply by its precomputed reciprocal. For small integer weights, use fixed-point weights that are pre-divided, then shift. When dividing by a constant, use unsigned types: the compiler's multiply-shift sequence for unsigned is shorter because it needs no sign fix-up.
Why it is faster: —
Applies elsewhere: audio gain and lerp in fixed point on microcontrollers; `x * (1.0/1e6)` in time conversions; hashing into buckets with unsigned modulo.
Watch out: —
Seen in PPSSPP: The DXT5 alpha lerp `alpha1 + (alpha2-alpha1) * (n / 7.0f)` became `(alpha1*w1 + alpha2*w2) >> 8`, where `w = (k<<8)/7`. That made the whole DXT5 decode about 8% faster. `time_now_d()` now multiplies by `(1.0 / 1000000.0)` instead of dividing, and callers read it once per frame. Bone-matrix index math switched to `u32` for `num / 12`. f65edc20a3 2018-09 “TexCache: Optimize DXT5 alpha lerp.” · e97baa503a 2020-10 “Avoid a division in time_now_d(). Minor optimizations.” · 536d881626 2020-09 “Very minor optimization (divisions by constants are faster unsigned)”

### ppsspp-155 — Convert data into the SIMD layout once, outside the hot loop
Lens: simd · Source: PPSSPP
Principle: If the inner loop needs SIMD-friendly values but the source is packed or in another format, convert everything once into aligned arrays of vector types. The hot loop then does aligned loads and math, with no unpacking.
Why it is faster: —
Applies elsewhere: convert AoS particle data to SoA before a physics step; decode packed RGBA8 into float4 once before a multi-tap filter.
Watch out: —
Seen in PPSSPP: The spline/bezier tessellator used to read `SimpleVertex` control points with `_mm_loadu_ps` and convert `color_32` on every weighted accumulation. It now fills `patch.pos/tex/col` arrays of `Vec3f/Vec2f/Vec4f` once, and `AccumulateWeighted` becomes `_mm_add_ps(out, _mm_mul_ps(in.vec, w.vec))`. 36ed61d08d 2018-01 “[spline/bezier]Pre-convert control-points for the accumulation of SIMD optimization, and avoid conversion in hot loops.”

### ppsspp-156 — Memoize parameter-only tables
Lens: lookup-tables · Source: PPSSPP
Principle: If an expensive table depends only on a few small parameters, pack those parameters into an integer key and cache the table in a map. Each parameter combination then gets computed once.
Why it is faster: —
Applies elsewhere: cache FFT twiddle factors per size, Gaussian kernels per radius, or resampler filter banks per ratio.
Watch out: —
Seen in PPSSPP: `WeightCache<T>` in `SplineCommon.cpp` wraps an `unordered_map<u32, Weight*>` keyed by `tess | count<<8 | type<<16`. Bezier and spline basis weights are now calculated once per key, and both software and hardware tessellation reuse them. 1e84a62e96 2018-01 “[spline/bezier]Implement weights cache system.” · 3c0fb44f2e 2018-07 “[spline/bezier]Improve hwtess to use cached weights.”

### ppsspp-157 — Hoist invariant products and use the smallest matrix that does the job
Lens: compiler-build · Source: PPSSPP
Principle: In nested loops, compute a product that only depends on the outer index once per outer iteration. Use a 2×2 matrix if only two components ever change. Both cut ALU work, and in generated shaders they also save registers.
Why it is faster: —
Applies elsewhere: precompute a row's base offset in image kernels; rotate 2D UI with a `mat2` instead of a full `mat4`.
Watch out: —
Seen in PPSSPP: The tessellation vertex shaders computed `weights_u[u] * weights_v[v] * points[idx]` inside the inner loop. They now compute `vec4 w = weights_u * weights_v[v]` per row, which takes the multiplications from 16 to 4. Display rotation in `VertexShaderGenerator` now uses a `mat2` on `pos.xy` instead of a 4×4 matrix of conditionals, and only on the backends that need it. de5975f13e 2018-07 “[spline/bezier]Reduce multiplications in the shaders from 16 to 4.” · 3245f54dc1 2021-10 “Optimize the rotation a bit (limit to VK/D3D11, mat2 instead of mat4)”

### ppsspp-158 — Grow-only reuse of device resources; upload constants once
Lens: allocation · Source: PPSSPP
Principle: Creating and destroying driver objects such as textures and buffers is expensive. Keep one object, reallocate it only when the required size grows, and update its contents in place. Upload data that never changes once, at creation, instead of on every frame.
Why it is faster: —
Applies elsewhere: keep a `std::vector` across iterations instead of reconstructing it; reuse a WebGL buffer with `bufferSubData`; pool DB connections.
Watch out: —
Seen in PPSSPP: GLES hardware tessellation created and deleted three float textures for every draw, about 1000 per frame in Pursuit Force. It now keeps `prevSize/prevSizeWU/prevSizeWV` and calls `TextureImage` only when a texture needs to grow, then `TextureSubImage` for the data. `PresentationCommon` now uploads its constant quad indices when the buffer is created, not every frame. 03b9492f08 2018-07 “[spline/bezier]Sharing textures to avoid heavily textures creation/deletion in one frame.” · a63706925f 2020-05 “Very minor optimization in PresentationCommon (don't upload the indices every frame).”

### ppsspp-159 — Check an atomic flag before taking a lock
Lens: concurrency · Source: PPSSPP
Principle: If a mutex-guarded path almost always finds nothing to do, keep an atomic summary of whether there is any work, and test it before locking. In the common case the cost drops from a lock/unlock pair to one relaxed load. Apply the same idea to wait primitives: return straight away when the event has already fired.
Why it is faster: —
Applies elsewhere: a "has listeners" atomic in front of an event bus; double-checked lazy init; checking a fast `isEnabled` flag before formatting log messages.
Watch out: —
Seen in PPSSPP: Every memory-check call in `Breakpoints.cpp` took the breakpoint mutex. Now `std::atomic<bool> anyMemChecks_`, kept up to date on add and remove, returns `BREAK_ACTION_IGNORE` without locking. This helps games heavy on block copies. `Event::Wait` now returns before creating the lock if `triggered_` is already set. 641bb72b40 2019-08 “Memchecks: Avoid taking the mutex lock if no memchecks active.” · 8d8a553f5b 2022-04 “Fix typo and optimize locking pattern in Event”

### ppsspp-160 — Fast path for the aligned, full-size case
Lens: branching-fast-paths · Source: PPSSPP
Principle: General code that handles every offset, size and permutation is slow for the usual case. Check for the common shape first (full 4×4, zero offset, no rotation, flat colour, no texture) and handle it with a `memcpy` or a simple loop.
Why it is faster: —
Applies elsewhere: an image blit that uses one `memcpy` when strides are equal; a matrix library with a special case for identity or affine; string search with a single-character fast path.
Watch out: —
Seen in PPSSPP: `ReadMatrix`/`WriteMatrix` for the VFPU interpreter now use `memcpy(rd, v, 64)`, or a plain transpose, when `side == 4 && row == 0 && col == 0`. The software renderer's rectangle fast path now also accepts untextured, flat-colour triangles. 8642134419 2019-08 “VFPU-Int: Add a few fast-paths to ReadMatrix/WriteMatrix.” · 361c8f966c 2021-11 “softgpu: Fast path triangles without textures.”

### ppsspp-161 — Track the last value you set and skip repeated API calls
Lens: gpu-rendering · Source: PPSSPP
Principle: Driver calls cost much more than comparing two integers. Keep a shadow copy of each dynamic state value, call the API only when the value changes, and invalidate the shadow whenever the driver resets that state (for example, on a pipeline bind).
Why it is faster: —
Applies elsewhere: a WebGL state cache for `useProgram` and `bindTexture`; skipping DOM writes when the value is unchanged; caching the last `setsockopt` value.
Watch out: —
Seen in PPSSPP: `VulkanQueueRunner` now keeps `lastStencilWriteMask/CompareMask/Reference` and calls `vkCmdSetStencil*` only when one differs. It resets them to -1 on each pipeline bind. The GL texture cache stopped binding a texture again straight after building it. ddb5208130 2019-08 “Vulkan: Reduce the amount of redundant stencil parameter sets” · dcd33a4692 2020-09 “Remove redundant bind on GL texture build.”

### ppsspp-162 — Merge consecutive passes that target the same output
Lens: gpu-rendering · Source: PPSSPP
Principle: On tiled GPUs, each render pass loads the framebuffer from memory and stores it back. When two passes in a row draw to the same target and nothing reads that target in between, join them into one pass. That removes a store and a load of the whole surface.
Why it is faster: —
Applies elsewhere: render graphs in modern engines that merge passes; combining adjacent SQL scans of the same table into one.
Watch out: —
Seen in PPSSPP: `VulkanQueueRunner` gained a `QUEUE_HACK_RENDERPASS_MERGE` step that joins render steps to the same framebuffer. It was first enabled through `compat.ini` for the God of War titles. Later fixes stop a merge when the target is read in between, when the second pass begins with a clear, or when the render areas differ. 93412aff3b 2019-08 “Vulkan: Automatically merge render passes to the same target when possible.” · 423309da50 2018-05 “Vulkan: Don't merge render passes if read from.”

### ppsspp-163 — Tell the hardware what you won't need, and fold transitions into the pass
Lens: gpu-rendering · Source: PPSSPP
Principle: Mark attachments whose contents aren't needed after a pass as "don't care" for storing, so the GPU skips writing them back. Express layout transitions through a render pass's `finalLayout` and subpass dependencies, not as separate barriers, so the driver can schedule them together with the pass.
Why it is faster: —
Applies elsewhere: `MTLStoreActionDontCare` for depth on iOS; `glInvalidateFramebuffer` on mobile GL; telling a DMA engine a buffer's previous contents can be thrown away.
Watch out: —
Seen in PPSSPP: `GetRenderPass` now takes a `finalColorLayout` in its key and builds `VkSubpassDependency deps[2]`. That replaced explicit `vkCmdPipelineBarrier` calls before framebuffer copies and sampling. Four years later, `RPKey` added store actions, so the backbuffer pass stores colour but uses `DONT_CARE` for depth and stencil. 2d3655ea23 2018-03 “Vulkan: Get rid of more explicit barriers by merging the finalLayout color transitions into renderpasses.” · 3e7723ea6d 2018-03 “Vulkan: Use finalLayout to optimize away barriers before framebuffer copies.” · b447092742 2022-08 “Some Vulkan renderpass load/store optimizations.”

### ppsspp-164 — Remove hidden copies: auto by value, pop-by-value, read-modify-write
Lens: compiler-build · Source: PPSSPP
Principle: Writing `auto x = getVector()` copies the whole container. Popping a struct from a queue copies it out. Assigning a whole struct and then adjusting its fields writes the memory twice. Use references, peek-then-advance, and constructors that write each field once.
Why it is faster: —
Applies elsewhere: `for (const auto &x : v)` in C++; `ref` returns in C#; building a JS object in one literal instead of spreading it and then mutating it.
Watch out: —
Seen in PPSSPP: `TextureCacheCommon` used `auto framebuffers = framebufferManager_->Framebuffers();`, which copied the vector on every texture lookup. It now takes a `const std::vector<...> &`. The software renderer's bin queues added `PeekNext/SkipNext` and `PeekPush/PushPeeked`, so items are used and filled in place, which "shaves a decent % off Drain time". Line expansion uses `CopyFromWithOffset` instead of copying a vertex and then adding offsets ("much better codegen"). Viewport depth clamping moved to record time to avoid copying the struct when it executes. 5ac6a9b3fd 2020-09 “Fix slowdowns due to careless use of auto” · 2539fb7c3c 2022-01 “softgpu: Tune queue push/pop to reduce overhead.” · 751afde7c9 2022-06 “Echochrome lines: Remove UV offsets, avoid reading the destination” · 5adb61a5f2 2019-10 “Move the viewport clamping back to rendermanager to avoid a struct copy in the common case.”

### ppsspp-165 — Fixed-size and stack buffers instead of heap containers on hot paths
Lens: allocation · Source: PPSSPP
Principle: A `std::vector`, `std::string` or `vector<bool>` created per call costs a heap allocation, plus bit-proxy or bounds overhead. When the maximum size is small and known, use a plain array, a stack `char[]`, or a borrowed `const char*`.
Why it is faster: —
Applies elsewhere: `char buf[64]` with `snprintf` in a logger; a `bool[256]` byte-class table in a lexer; arena-scoped string views in a compiler.
Watch out: —
Seen in PPSSPP: The GE debugger's `breakCmds` changed from `std::vector<bool>` to `bool[256]`; the commit calls the lookup "surprisingly expensive". `__sceAudio` replaced a per-frame `std::vector<int16_t>` with `int16_t srcBuffer[128]`. The GLES fragment shader generator swapped `std::string` temporaries for `char[64]` plus `truncate_cpy`. The Vulkan allocator stores `const char *tag` instead of copying a `std::string` for each allocation. 809fd6b8fa 2020-07 “Small perf optimization in debug mode (surprisingly expensive to do a vector lookup here).” · 7bced814ce 2022-09 “Remove a per-audio-frame allocation in sceAudio” · 2f1bbd314e 2020-10 “Fragment shader gen: Remove unnecessary allocations” · a9fef066b5 2020-09 “Reduce string allocations in Vulkan memory allocator”

### ppsspp-166 — Small-buffer containers for usually-tiny collections
Lens: allocation · Source: PPSSPP
Principle: If a collection almost always holds only a few elements, store the first N inline and allocate a heap vector only when it overflows. The common case never touches the allocator and stays in one cache line. Leave expensive diagnostics, such as string formatting, out of the matching loop.
Why it is faster: —
Applies elsewhere: LLVM's `SmallVector`, Rust's `smallvec`, a candidate list in a router, short header lists in HTTP parsers.
Watch out: —
Seen in PPSSPP: `TinySet` gained `push_back`/`add_back` so it can serve as a small vector. Framebuffer-to-texture matching now uses `TinySet<AttachCandidate, 6>` instead of `std::vector`, and it stopped calling `ToString()` on every candidate whenever more than one matched. a8a9fb4206 2022-08 “Much faster framebuffer matching (no more ToString every time there's more than 1 candidate, etc).” · a854fbbe76 2022-09 “use TinySet instead of vector to avoid an allocation.”

### ppsspp-167 — Switch to a modern non-cryptographic hash
Lens: algorithms · Source: PPSSPP
Principle: Older hand-tuned hashes are often slower than current SIMD-friendly designs such as XXH3, especially on short keys and on 64-bit CPUs. Change detection that has to hash many bytes per frame benefits directly. Hashing a code block is also a cheap way to tell whether it changed, without walking it word by word.
Why it is faster: —
Applies elsewhere: content-addressed caches, dedup in backup tools, ETag generation, hash-map keys in databases.
Watch out: —
Seen in PPSSPP: Vertex-cache hashes (`DrawEngineCommon::ComputeHash`), lookup hashes and hash-map hashing moved from `DoReliableHash`/`XXH32` to `XXH3_64bits`, which the commit calls "faster pretty much everywhere". The CLUT hash moved to XXH3 too, and the custom NEON xxhash was removed. The JIT block cache now stores an XXH3 `compiledHash` of each block's source so that "invalidate all" re-checks blocks cheaply. 15c25be42b 2020-08 “GPU: Switch vertex/lookup hashes to XXH3.” · 7f812a5d9a 2020-08 “GPU: Hash CLUT using XXH3 with replacements off.” · 00f9707399 2021-12 “jit: Use a hash for invalidate/clear all.”

### ppsspp-168 — Don't verify what you already know
Lens: laziness · Source: PPSSPP
Principle: Change detection costs real work. Data known to be immutable can skip the check. Data known to change every time, such as video frames, can be assumed changed without hashing it.
Why it is faster: —
Applies elsewhere: immutable fingerprinted assets served without revalidation; skipping diffing for a stream known to be live.
Watch out: —
Seen in PPSSPP: Textures at kernel addresses (the built-in PPGe font) are marked `STATUS_RELIABLE` and never hashed again (`Memory::IsKernelAddress`). In lazy texture-cache mode, video textures skip hashing completely: the stored hash is scrambled so it can't match by accident, and the function returns "changed". 3e3a40da8c 2020-09 “Mark the PPGe font texture as reliable to avoid hashing it.” · 2d9719ed26 2021-09 “TexCache: Avoid hashing videos at all in lazy mode.”

### ppsspp-169 — Generate repeating integer patterns with vector adds
Lens: simd · Source: PPSSPP
Principle: If an output sequence is a fixed pattern plus a running base, as in strip-to-list index conversion, precompute the pattern in a few vector registers. Each iteration then adds a broadcast base and stores. One iteration of 3 stores can emit 24 indices.
Why it is faster: —
Applies elsewhere: generating quad indices for sprite batchers, building gather offsets, filling interleaved stride tables.
Watch out: —
Seen in PPSSPP: `IndexGenerator::AddStrip` loads three `__m128i` offset patterns that depend on winding, then loops `_mm_storeu_si128(dst, _mm_add_epi16(ibase8, offsets0..2))` with `ibase8 += 8`, producing 8 triangles per iteration. A NEON version followed. The commit reports about 0.5% off the God of War frame. 1a2597a513 2020-09 “SSE-optimize IndexGenerator::AddStrip. Shaves about half a percent from GoW.” · be54050521 2020-09 “Also optimize IndexGenerator::AddStrip for ARM NEON.”

### ppsspp-170 — Overshoot the end instead of writing a scalar tail
Lens: simd · Source: PPSSPP
Principle: If the destination buffer has slack, for example because it is append-only and the next write will overwrite the extra, round the vector loop count up and let it write a few elements past the logical end. That removes the scalar remainder loop and its branches. Then advance the output pointer by the true count.
Why it is faster: —
Applies elsewhere: padded buffers in SIMD JSON parsers, base64 encoders that write a whole 16-byte block, string builders with spare capacity.
Watch out: —
Seen in PPSSPP: `AddStrip` changed `numChunks = numTris / 8` to `(numTris + 7) / 8` and sets `outInds += numTris * 3` afterwards. The scalar fallback now runs only on non-SIMD builds. The commit calls this "actually a decent boost". d8ddef150f 2020-09 “IndexGenerator::AddStrip: Avoid the fallback by writing a few extra indices if necessary.”

### ppsspp-171 — Vector accumulate, then a cheap horizontal reduce
Lens: simd · Source: PPSSPP
Principle: When scanning a buffer for an aggregate (AND mask, sum), keep a vector accumulator through the loop and reduce it to a scalar once at the end. Pairwise high/low adds or a scalar AND of the lanes beat per-lane extracts. A pointer-end loop condition removes index arithmetic.
Why it is faster: —
Applies elsewhere: checking whether an image is fully opaque, checksumming a packet, finding the min/max of a column.
Watch out: —
Seen in PPSSPP: `CopyAndSumMask16` and `CheckMask16` now copy 8 pixels at a time while AND-ing them into `wideMask` (SSE and NEON), then reduce with `mask & (mask >> 16)`. The NEON quick texture hash loops on `while (p < pend)` and reduces with `vadd_u32(vget_high, vget_low)`, which is 1–4% faster. 185b93058e 2022-04 “SIMD-optimize CheckMask16 / CopyAndSumMask16” · 364aabe84f 2020-08 “TexCache: Improve quicktexhash by ~1-4% on NEON.”

### ppsspp-172 — Split big flat loops across a thread pool, with a minimum chunk size
Lens: concurrency · Source: PPSSPP
Principle: Large memsets, memcpys and independent per-item loops, such as relocations or clearing rows, scale almost linearly across cores. Use a parallel-for that hands each worker a contiguous range, and set a minimum chunk size so small jobs aren't drowned in scheduling overhead.
Why it is faster: —
Applies elsewhere: zeroing large buffers on allocation, parallel image filters, bulk checksum verification.
Watch out: —
Seen in PPSSPP: Savestate RAM copy (`DoMemoryVoid`) and memory init (`ThreadPool::ParallelMemset`, with `MIN_SIZE = 128 KB`) both use `GlobalThreadPool::Loop`. ELF relocation processing runs as a parallel loop, which the commit says "can halve the load time". The software renderer splits depth and colour clears into ranges of `MIN_LINES_PER_THREAD = 128` rows each. 3400e6e517 2021-04 “SaveState: Copy RAM using threads.” · 4f96169357 2021-04 “Core: Init memory on multiple threads.” · 6ee944a0a6 2021-04 “Module: Process relocations on threads.” · 66f635cba0 2021-11 “sfotgpu: Use threads to apply clears.”

### ppsspp-173 — An unrolled vector copy when alignment is guaranteed
Lens: simd · Source: PPSSPP
Principle: When base and size are known to be multiples of 64 bytes, a loop of four 16-byte loads followed by four stores avoids `memcpy`'s size and alignment dispatch, and keeps several loads in flight at once. Rounding both chunk bounds down to the same alignment keeps every thread's chunk aligned.
Why it is faster: —
Applies elsewhere: copying fixed-size pages in a VM snapshot, ring-buffer block copies, DMA-like copies in embedded firmware.
Watch out: —
Seen in PPSSPP: `CopyAlignedFast` in `MemMap.cpp` does `l &= ~0x3F; h &= ~0x3F;` and then copies 4×`__m128i` per iteration. It falls back to `p.DoVoid` if the size or pointer isn't 64-byte aligned. c5729bbd5e 2021-04 “SaveState: Use SIMD to copy RAM faster.”

### ppsspp-174 — Cache expensive aggregates and invalidate them on write
Lens: laziness · Source: PPSSPP
Principle: Recomputing a total by walking a directory tree, or by sizing every permutation of a list, is costly. Cache the result and clear a "valid" flag whenever something that could change it happens. Reads then cost almost nothing, and only writes pay.
Why it is faster: —
Applies elsewhere: a cached disk-usage counter that is invalidated on file writes; a memoized computed property cleared when an input changes; a cached row count.
Watch out: —
Seen in PPSSPP: The memory-stick free-space query called `getDirSize("ms0:/PSP/SAVEDATA/")` every time. It now keeps `memstickCurrentUse`, and `DirectoryFileSystem` calls `MemoryStick_NotifyWrite()` on writes, deletes and renames. `SavedataButton` caches `totalSize_`/`dateSeconds_`. Sorting had been recomputing them for each comparison, which was "monstrously slow" on Android scoped storage. 906664b9d8 2021-09 “Io: Cache SAVEDATA usage between writes.” · 8c937f45f7 2021-09 “UI: Cache save size on save buttons.”

### ppsspp-175 — One bulk query instead of N per-item lookups
Lens: io · Source: PPSSPP
Principle: When a listing API can return all the needed attributes in one call, request them there instead of stat-ing each entry afterwards. Each per-item call can be an IPC round trip or a syscall, so N+1 patterns cost far more than one query.
Why it is faster: —
Applies elsewhere: the SQL N+1 problem; `readdir`+`stat` vs `getdents`/`FindFirstFile` data; batch REST endpoints.
Watch out: —
Seen in PPSSPP: Android content-URI listing went from `DocumentFile.listFiles()` plus per-file getters to a single `ContentResolver.query` on `buildChildDocumentsUriUsingTree` that asks for name, size, flags, MIME type and mtime at once ("greatly speed up"). Savedata size calculation now uses the sizes that `GetFilesInDir` already returns, instead of calling `GetFileInfo` on each file. bd99e4a287 2021-07 “Greatly speed up content URI file listings by using DocumentContract queries.” · 830d205839 2021-09 “Optimize savedata size calculation”

### ppsspp-176 — Attempt the operation instead of checking first
Lens: io · Source: PPSSPP
Principle: Checking that a file exists and then opening it costs two filesystem round trips, and it's racy. Just try the open or read and handle the failure. Once a handle is open, take metadata such as size from the handle (seek to the end) rather than looking the path up again.
Why it is faster: —
Applies elsewhere: EAFP style in Python; `fstat` on an open fd instead of `stat` on its path; conditional GET instead of HEAD then GET.
Watch out: —
Seen in PPSSPP: `SavedataParam::LoadSFO` now calls `ReadEntireFile` directly and returns a bool, dropping the separate `GetFileInfo(sfoPath).exists` check. `MetaFileSystem::ReadEntireFile` gets the size with `SeekFile(END)`/`GetSeekPos` on the open handle instead of a path-based `GetFileInfo`. 36079b86d0 2021-09 “Optimize away a couple of existence checks” · 8e2b73b05f 2021-09 “Optimize MetaFileSystem::ReadEntireFile”

### ppsspp-177 — Defer populating hidden UI
Lens: laziness · Source: PPSSPP
Principle: Filling a large widget nobody can see wastes startup time. Set a dirty flag while the widget is hidden and populate it the first time it is shown.
Why it is faster: —
Applies elsewhere: lazy tab content in web apps, virtualized lists, IDE panels that index only when opened.
Watch out: —
Seen in PPSSPP: `CDisasm::NotifyMapLoaded` used to fill the Win32 function listbox with every symbol at game start. It now sets `deferredSymbolFill_` when the window is hidden, and `CDisasm::Show(true)` fills it on first display. 84b36d6e61 2020-09 “Speed up game start under Windows by not filling the symbol listbox until the disasm window is opened.”

### ppsspp-178 — A coarse index into a sorted linked structure
Lens: algorithms · Source: PPSSPP
Principle: Finding an address in a linked list of ranges is O(n). Split the address space into fixed slices and keep an array of the first node covering each slice. A lookup then jumps to `heads[addr / SLICE]` and walks only a few nodes. Splits and merges update just the slices they touch.
Why it is faster: —
Applies elsewhere: skip lists, page tables, bucketed timer wheels, an offset index over a log file.
Watch out: —
Seen in PPSSPP: `MemSlabMap` (the debugger's memory-tagging map) gained `heads_` with 16384 slices over 1 GB, maintained by `FillHeads` on split and merge. This tracking runs on every block transfer, and a later commit also removed the heap allocation from that path. a62aa4746b 2021-02 “Debugger: Speed up lookup using an index.” · ecb84987ef 2022-09 “Remove allocation from the heaviest MemBlockInfo path (BlockTransfer).”

### ppsspp-179 — Compile asynchronously and key caches by futures
Lens: concurrency · Source: PPSSPP
Principle: Move expensive compilation onto workers and return a promise straight away. If downstream caches use the promise itself as the key, building the key doesn't block. The code waits only at the point where the compiled result is actually consumed, which can be on another thread.
Why it is faster: —
Applies elsewhere: memoizing `Promise`s instead of values in JS (deduplicates concurrent fetches); async shader and pipeline compilation in game engines; build systems keyed by task handles.
Watch out: —
Seen in PPSSPP: Vulkan pipeline creation moved to a dedicated compile thread, and the render thread waits on `compileDone_` only when a pipeline isn't ready yet. Later, `ShaderManagerVulkan` started compiling GLSL→SPIR-V through `Promise<VkShaderModule>::Spawn(&g_threadManager, ...)`. `VulkanPipelineKey` now holds `Promise<VkShaderModule> *` and calls `BlockUntilReady()` only when it creates the pipeline. In debug builds, the parallel compile is turned off because the debug allocator's lock dominates. dda425b068 2019-06 “Vulkan: Add a single background thread for pipeline creation” · fc418ee16f 2022-06 “Create shader modules (and compile GLSL) on worker threads” · e2c740827a 2022-06 “Use shader module promises as keys in pipeline cache instead of the actual shader modules.” · 6b1e4806cf 2022-09 “Don't parallelize shader compiles in debug mode”

### ppsspp-180 — A lock-free single-slot handoff in front of the queue
Lens: concurrency · Source: PPSSPP
Principle: Give each worker an atomic one-task slot. A submitter tries a compare-exchange into the slot and touches the mutex-protected queue only if the slot is full. The worker takes from the slot with an atomic exchange before it looks at the queue. For the typical one-task-at-a-time pattern, this removes most lock traffic.
Why it is faster: —
Applies elsewhere: Go's per-P `runnext` slot, the "LIFO slot" in Tokio's scheduler, mailbox registers in embedded RTOSes.
Watch out: —
Seen in PPSSPP: `ThreadContext` gained `std::atomic<Task *> private_single`. `EnqueueTaskOnThread` does `compare_exchange_weak(expected=nullptr, task)`, and `WorkerThreadFunc` starts with `private_single.exchange(nullptr)`. The commit measured about 1% in the software renderer. d15af6e9d9 2021-11 “ThreadManager: Use atomic task optimistically.”

### ppsspp-181 — Separate pools for blocking I/O and CPU work
Lens: concurrency · Source: PPSSPP
Principle: If slow I/O tasks share a pool with CPU-bound parallel loops, they occupy threads that compute work is waiting for. Tag each task by type and run blocking work on its own, oversubscribed pool, so the compute pool stays sized to the cores and never waits behind a disk read.
Why it is faster: —
Applies elsewhere: Java's separate executors for blocking calls; Tokio's `spawn_blocking`; Node's libuv threadpool vs the event loop.
Watch out: —
Seen in PPSSPP: `ThreadManager` split `queue` into `compute_queue` and `io_queue`, each with its own atomic size, and added `MIN_IO_BLOCKING_THREADS = 4`. `Task::Type()` sends each task to the right queue. f9a7ad3e3d 2021-12 “ThreadManager: Use separate pool for IO blocking.” · 8b5173350f 2021-12 “ThreadManager: Make TaskType a property of a Task.”

### ppsspp-182 — Hand workers batches, not single items
Lens: concurrency · Source: PPSSPP
Principle: Each enqueue and wake costs synchronization. Have a worker drain a whole queue of items in one task, and re-check once after clearing its "busy" flag, rather than spawning a task per primitive. Don't parallelize work that has a read-after-write hazard on the same data.
Why it is faster: —
Applies elsewhere: batched message consumers in Kafka clients; GPU command-buffer batching; bulk database inserts.
Watch out: —
Seen in PPSSPP: The software renderer's `BinManager` now schedules `DrawBinItemsTask`, which loops `while (!items_.Empty())` over a `BinQueue<BinItem, 1024>` and runs a second pass after `status_ = false`. Rendering a texture into itself stays single-threaded, because that will always be a problem to thread. 5a35525fd4 2022-01 “softgpu: Enqueue batches of prims when binning.” · 88ef2d1ac1 2022-01 “softgpu: Skip threading when rendering to self.”

### ppsspp-183 — Derive state once per state change, not once per item
Lens: laziness · Source: PPSSPP
Principle: If a per-vertex or per-pixel routine keeps reading raw configuration and deriving things from it (enable bits, colour factors, function IDs, masks), compute a compact derived struct once when the state changes, or once per batch. The hot code then reads precomputed fields and can skip whole stages that turned out to be no-ops.
Why it is faster: —
Applies elsewhere: compile a CSS selector or SQL plan once, then run it per row; prepared statements; precomputed per-request auth context.
Watch out: —
Seen in PPSSPP: The software renderer computes `PixelFuncID`/`SamplerID` once per vertex batch and passes them down, instead of computing them per primitive ("saves a decent chunk of time"). `Lighting::ComputeState` precomputes per-light factors and flags such as `anyDiffuse`/`anySpecular`, so unused terms are skipped. `ComputeTransformState` packs transform flags into bitfields. `PixelFuncID::cached` holds fog colour, the dither matrix, strides and masks for the JIT to load. In the hardware path, `ComputeFragmentShaderID` now reuses the blend setup that state mapping already computed. d7a82ab7b8 2022-01 “softgpu: Compute func IDs once per batch of verts.” · 2797e035df 2022-01 “softgpu: Precompute lighting parameters.” · 12a4c63fc7 2022-01 “softgpu: Precompute state for vertex transform.” · e9f3720e20 2022-01 “softgpu: Cache fog color draw pixel state.” · fd6ab4c495 2022-09 “Avoid recomputing the shaderblend setup in ComputeFragmentShaderID.”

### ppsspp-184 — Reject or short-circuit the common trivial case first
Lens: branching-fast-paths · Source: PPSSPP
Principle: Put the cheapest test for "nothing to do" at the top of a costly routine: fully outside the bounds, no clipping needed, lane masked off, channel unused. The common case then returns before any buffer setup, copying or sampling.
Why it is faster: —
Applies elsewhere: bounding-box rejection before precise collision tests; skipping masked lanes in SIMD shading; leaving out unused columns in a query projection.
Watch out: —
Seen in PPSSPP: The software rasterizer returns when `maxX < minX || maxY < minY` after scissoring (3–4%). The clipper ORs the three vertices' clip masks and passes the triangle straight to the binner when no near-plane clip is needed, which shrinks the clip buffers from 33 to 3 vertices. Texture sampling and colour interpolation are skipped for pixels whose mask lane is negative (about 4%). The sampler JIT skips decoding alpha when `!id.useTextureAlpha`. 12405709f0 2022-01 “softgpu: Skip processing scissored triangles.” · 89bc87a388 2022-01 “softgpu: Reduce copying during clipping.” · 7594187538 2022-01 “softgpu: Skip sample lookup if masked.” · 72aa4be879 2022-01 “samplerjit: Skip processing alpha if unused.”

### ppsspp-185 — Synchronize only when ranges overlap; skip copies when data is identical
Lens: concurrency · Source: PPSSPP
Principle: A full flush of a pipeline on every possible hazard throws away parallelism. Check whether the address range being changed overlaps what in-flight work is writing, and flush only then. When a source is re-uploaded, `memcmp` it against the current copy first. If nothing changed, skip the copy and everything that would follow it.
Why it is faster: —
Applies elsewhere: fine-grained cache invalidation by key range; React's shallow-equal bailout; rsync's block checksums.
Watch out: —
Seen in PPSSPP: `TransformUnit::FlushIfOverlap(addr, size)` compares the range against the colour and depth targets, and is now used on texture address, buffer-width and CLUT changes instead of an unconditional `Flush()`. `SoftGPU` now `memcmp`s the new CLUT against the old one and calls `NotifyClutUpdate` only if it changed. Games often re-upload an identical CLUT. cc155ec460 2022-01 “softgpu: Avoid texture/CLUT flush unless overlap.” · ce775af76d 2022-01 “softgpu: Skip new CLUT if identical.”

### ppsspp-186 — Bit-deposit and extract instructions for format conversion, gated by vendor
Lens: platform · Source: PPSSPP
Principle: BMI2 `PDEP`/`PEXT` scatter and gather arbitrary bit fields in one instruction, replacing chains of shifts and masks when expanding packed pixel formats. Some CPUs implement them slowly in microcode, so detect the vendor and family, not just the feature bit. `SHLX`/`RORX` also remove the fixed-register and flag constraints of the classic shifts.
Why it is faster: —
Applies elsewhere: Morton/Z-order encoding, bitboard move generation in chess engines, unpacking bit-packed columnar data.
Watch out: —
Seen in PPSSPP: `CPUDetect` adds `bBMI2_fast = bBMI2 && (vendor != AMD || family >= 0x19)`. The softjit decodes 565/5551 destination colours with `PDEP`/`PEXT` (about 1% overall), and the sampler JIT does the same for DXT. The MIPS x86 JIT emits `SHLX`/`SHRX`/`SARX`/`RORX` for variable shifts, which saves a `MOV` to ECX. 10bf375712 2022-01 “softjit: Use BMI2 to speed up dst color loads.” · 16dca4f69b 2022-01 “x86jit: Use BMI2 for variable shifts.”

### ppsspp-187 — Restructure compute shaders around the input, and drop needless square roots
Lens: gpu-rendering · Source: PPSSPP
Principle: For upscaling filters, run one invocation per input pixel and have it write the whole N×N output block. The neighbourhood analysis is then shared instead of repeated for every output pixel. Use an 8×8 workgroup, which suits all major vendors. Compare squared distances (`dot(v,v)`) instead of `length`, and compare packed integers instead of float-reduced colours.
Why it is faster: —
Applies elsewhere: image-processing kernels on GPUs or in WebGPU; nearest-neighbour search on squared distances; colour-difference metrics.
Watch out: —
Seen in PPSSPP: `tex_4xbrz.csh`/`tex_mmpx.csh` switched to "main runs once per INPUT pixel", with `applyScaling(xy)` writing scale×scale pixels, and the workgroup went from 16×16 to 8×8. That made 4xBRZ at 16× and MMPX at 4× faster. The xBRZ shaders now read `uint` neighbours instead of `reduce(vec4)` floats, and use `dot(YCbCr,YCbCr)` instead of `length()` squared: about 30% faster on Adreno. 281ddc2437 2021-11 “Speed up the 4xBRZ upscaler with 16x and the MMPX one with 4x by not doing redundant work” · fb3b7f8e23 2021-12 “Optimize the xBRZ texture scaling shaders by 30% on Adreno, less on Mali”

### ppsspp-188 — Shrink the work area and turn off unused features
Lens: gpu-rendering · Source: PPSSPP
Principle: A full-screen pass over a region whose touched area is known wastes fill rate. Set the scissor to the bounds of the vertices actually used. Turn off pipeline features you never use, such as primitive restart, because some hardware pays for them even when idle. And when a new buffer is about to be cleared, don't upload its previous contents.
Why it is faster: —
Applies elsewhere: dirty-rect repainting in UI toolkits; disabling unused GL capabilities; not initializing memory that will be overwritten straight away.
Watch out: —
Seen in PPSSPP: The depal (palette conversion) pass now sets the scissor from `gstate_c.vertBounds` scaled by `renderScaleFactor`, instead of the full framebuffer. `VulkanUtil` sets `primitiveRestartEnable = false` ("it has costs on some hardware"). `FramebufferManagerCommon` skips the create-time upload unless `params.isDrawing`, since a clear usually follows a new framebuffer. 9b4e94c4ae 2022-08 “Depal: Set scissor tightly around processed area.” · e57770977b 2021-10 “We don't use primitiveRestart for anything, let's not enable it.” · 0b22c98404 2022-08 “GPU: Skip fb create upload when clearing.”

### ppsspp-189 — Sequence counters instead of a side tracking table
Lens: algorithms · Source: PPSSPP
Principle: To find the most recent object for a key, stamp each object with a global, monotonically increasing sequence number when it is used, and pick the maximum among the candidates. That replaces a separate tracking structure that had to be kept in sync on every create and delete.
Why it is faster: —
Applies elsewhere: LRU by access timestamp, last-writer-wins in CRDTs, MVCC version numbers.
Watch out: —
Seen in PPSSPP: `FramebufferManagerCommon` removed `trackedDepthBuffers_` and `GetOrCreateTrackedDepthBuffer`. `GetLatestDepthBufferAt(z_address, z_stride)` now picks the framebuffer with the highest `depthBindSeq`, and texture matching uses `colorBindSeq`/`depthBindSeq` in the same way. 74f1c94ddb 2022-08 “Use sequence numbers instead of a tracking array for depth buffers”

### ppsspp-190 — Spread I/O across frames with a time budget and a loader thread
Lens: io · Source: PPSSPP
Principle: Loading assets on demand inside a frame causes hitches. Hand loads to a background thread. Where a synchronous path is unavoidable, let it do work only while a per-frame time budget lasts, and use the fallback asset until the real one is ready.
Why it is faster: —
Applies elsewhere: progressive image loading; `requestIdleCallback` work slicing; streaming texture mips in open-world games.
Watch out: —
Seen in PPSSPP: Texture replacements are loaded on a thread and waited on through a `LimitedWaitable` with `WaitFor(us)`. `TextureReplacer::FindReplacement` later gained a `budget` parameter: `PopulateReplacement` runs only while `budget > 0` when late loading is enabled. This is meant to reduce I/O stalls on Android. 09f0578a64 2021-10 “Replacement: Use a thread to load tex replacements.” · e473eb9f5e 2022-08 “Replacement: Read files only within time budget.”

### ppsspp-191 — Make hot accessors inlinable
Lens: compiler-build · Source: PPSSPP
Principle: A getter defined in a `.cpp` file costs a real call at every use unless link-time optimization is on. Expose the backing object in the header, or define the accessor inline there, so each access compiles to a single load.
Why it is faster: —
Applies elsewhere: header-only getters in C++ libraries; `static inline` helpers in C; `@inline` or final methods in JIT-ed languages.
Watch out: —
Seen in PPSSPP: `PSP_CoreParameter()` returned a file-static `coreParameter` through an out-of-line function. That is now `g_CoreParameter` with an inline accessor in `System.h`, which "eliminates a function call for each access". It is read in many places, including the compat flags. 94800f6194 2022-04 “Inline the CoreParameters accessor (eliminates a function call for each access)”

### ppsspp-192 — Emit the compact instruction encoding when operands fit
Lens: jit-interpreters · Source: PPSSPP
Principle: On ISAs with short encodings (RISC-V C, Thumb), have the code emitter pick the 16-bit form automatically whenever the registers and immediates fit. Generated code gets smaller, so more of it stays in the I-cache and fetch bandwidth goes further.
Why it is faster: —
Applies elsewhere: `-Os` / Thumb-2 builds for microcontrollers; WebAssembly-to-native compilers on RISC-V; size-optimized firmware.
Watch out: —
Seen in PPSSPP: `RiscVEmitter` gained the `Opcode16` quadrants and the `C_*` funct3 encodings (C.LW, C.ADDI, C.BEQZ, C.J, …). It can optionally compress ordinary emitted instructions automatically when they qualify. 08d82ec15b 2022-08 “riscv: Emit compressed instructions.”

### ppsspp-193 — Allocate large per-worker buffers only for workers that exist
Lens: allocation · Source: PPSSPP
Principle: Pre-sizing for the maximum possible thread count wastes memory and cache footprint. Allocate per-worker queues lazily, up to the number of workers actually running.
Why it is faster: —
Applies elsewhere: per-core arenas in allocators sized to the online CPU count; lazily created per-connection buffers.
Watch out: —
Seen in PPSSPP: `BinQueue` got a `Setup()` method that allocates `items_` separately from construction. `BinManager` calls it only for `min(GetNumLooperThreads(), MAX_POSSIBLE_TASKS)` task queues, which cuts roughly 20 MB to 5 MB. 1764111a4b 2022-01 “softgpu: Reduce wasted memory.”

### ppsspp-194 — Keep small, bounded buffers off the heap
Lens: allocation · Source: PPSSPP
Principle: If a buffer's size is known or tightly bounded, store it in a stack array or an inline fixed-capacity array with a count. Creating it is free and its data stays in one or two cache lines. A `std::vector` built in a periodic routine pays for an allocator call and a free every time the routine runs.
Why it is faster: —
Applies elsewhere: audio callbacks and ISRs, where allocating is forbidden or unpredictable. Per-request scratch space. `SmallVec`/`boost::static_vector` for short lists of pending items.
Watch out: —
Seen in PPSSPP: The audio mixing loop in `__sceAudio.cpp` built a `std::vector<int16_t> srcBuffer` and resized it to `hwBlockSize*2` on every audio frame. With the block sizes made `const`, the buffer became `int16_t srcBuffer[srcBufferSize]` on the stack. `FixedTinyVec<T, MaxSize>` was added to `TinySet.h` as an “even cheaper” sibling of `TinySet`: `T data_[MaxSize]` plus `count_`, with `begin()`/`end()` returning pointers. `push_back` returns false when the vector is full rather than spilling to the heap. 80e0bb9c1f 2022-09 “Remove a per-audio-frame allocation in sceAudio” · f178d1bd85 2023-03 “Add an even cheaper version of TinySet called FixedTinyVec that's, well, fixed.”

### ppsspp-195 — Reserve container capacity from the input size
Lens: allocation · Source: PPSSPP
Principle: When an output will be about as large as an input you already have, call `reserve()` with that size before the loop. Growing by repeated doubling reallocates and copies the buffer over and over, and the cost grows with the number of passes that rebuild it.
Why it is faster: —
Applies elsewhere: compiler passes that copy an instruction list, ETL transforms that map rows one-to-one, and `StringBuilder` with an estimated length.
Watch out: —
Seen in PPSSPP: The IR optimizer chains passes, and each pass writes a new `IRWriter`. The commit added `IRWriter::Reserve()` and calls `out.Reserve(in.GetInstructions().size())` before each pass, plus reserves for `pendingAddresses` and `checks`. The commit message reports IR compile time down by about 20–30%. 57123e8f9e 2023-08 “irjit: Reserve some arrays that churn.”

### ppsspp-196 — Append uninitialized slots and fill them in place
Lens: allocation · Source: PPSSPP
Principle: A `push_back(T{...})` zero-fills a temporary, constructs it, then copies it into the container. For large union-style command records, most of those bytes are never read. Hand out a pointer to the next raw slot and let the caller write only the fields that matter. When the buffer grows, copy it with `memcpy`.
Why it is faster: —
Applies elsewhere: command buffers in game engines, log record ring buffers, and serializers that reserve a header slot and fill it in later.
Watch out: —
Seen in PPSSPP: `GLRRenderData` lost its zero-initializing brace construction. Then the new `FastVec<T>` added `push_uninitialized()`, so `GLRenderManager` now writes `GLRRenderData &data = commands.push_uninitialized(); data.cmd = ...`. Later, `extend_uninitialized(count)` let the Vulkan draw path write descriptors directly into the packed descriptor array, which removed a `memcpy`. An early commit notes that switching containers alone was “Slower than std::vector!”. The uninitialized pushes are where the speedup came from. e4a729d371 2023-05 “Remove unnecessary zero-initialization of GLRRenderCommand structs” · 956d784bde 2023-05 “Add FastVec, start using it for InitSteps” · 47931deda7 2023-05 “Switch to FastVec for commands. Slower than std::vector!” · 37906384ee 2023-05 “Use the faster pushes.” · 183d49329a 2023-10 “Allow writing directly into the packed descriptor buffer, saving a memcpy.”

### ppsspp-197 — Match wakeups to work
Lens: concurrency · Source: PPSSPP
Principle: Signalling a worker on a fixed schedule wakes it for nothing, and each wake costs a context switch. Handling only one item per wakeup adds a full sleep interval of latency for every queued item. Notify only when something was queued, and on each wakeup keep processing until the source is empty.
Why it is faster: —
Applies elsewhere: background compile or upload threads, epoll handlers that read until `EAGAIN`, and message consumers that fetch in batches.
Watch out: —
Seen in PPSSPP: `VulkanRenderManager` called `compileCond_.notify_one()` unconditionally every frame. It now sets `needsCompile` when it pushes pipeline compile work and calls notify only if that flag is set. In `sceNetAdhoc.cpp`, the adhocctl receive thread's `if (context != NULL)` became `while (context != NULL && context->inputRunning && !Core_IsStepping())`, breaking when no datagram is available. Before, each additional player added 10 ms of delay. 54fe342f4b 2022-09 “Vulkan: Wake compile thread when enqueued.” · 2f3e9073f6 2023-10 “Process incoming packets quicker whenever available...”

### ppsspp-198 — Don't allocate, load, store or transition attachments a pass doesn't use
Lens: gpu-rendering · Source: PPSSPP
Principle: On tiled GPUs, every attachment in a render pass costs memory bandwidth to load and store, and every layout transition costs a barrier. Build pass variants that leave out unused attachments. Skip allocating depth for color-only targets. Mark depth that is never read back as transient, so the driver can keep it on-chip.
Why it is faster: —
Applies elsewhere: WebGPU or Metal render passes with `storeOp: "discard"`, mobile engines that set memoryless depth for the main pass, and offscreen UI compositing targets with no depth.
Watch out: —
Seen in PPSSPP: `CreateRenderPass` gained `RP_TYPE_COLOR` and `RP_TYPE_COLOR_INPUT` variants with `attachmentCount = hasDepth ? 2 : 1`. `VKRFramebuffer` takes `createDepthStencilBuffer`, so temp copies and depal buffers get no depth image. The backbuffer depth image is created with `VK_IMAGE_USAGE_TRANSIENT_ATTACHMENT_BIT`. The queue runner skips the depth layout transition when `!RenderPassTypeHasDepth(...)`. 8e30a7ccfc 2022-09 “Vulkan: Don't have renderpasses store/load depth buffers when we don't use them” · 753fd13494 2022-09 “Vulkan: Avoid allocating depth images for stuff like temp copies, depal buffers etc.” · 5711259b86 2023-10 “Declare the back depth buffer as "transient"...” · 60f350056c 2024-04 “Vulkan: Optimize away a barrier in case of no depth buffer”

### ppsspp-199 — Batch API calls into arrays, and narrow what each barrier waits for
Lens: gpu-rendering · Source: PPSSPP
Principle: Graphics APIs charge a fixed validation and dispatch cost on each call. Many accept arrays, so collect barriers, copy regions or descriptor allocations and submit them together. Also give each barrier only the stages and access types that actually conflict. A write hazard does not need prior reads made visible, so the GPU can overlap more work.
Why it is faster: —
Applies elsewhere: SQL multi-row INSERT, Redis pipelining, `writev`, and acquire/release fences in place of seq_cst in lock-free code.
Watch out: —
Seen in PPSSPP: `VulkanBarrierBatch` merges stage masks and collects `VkImageMemoryBarrier`s, then issues a single `vkCmdPipelineBarrier` in `Flush()`. `VulkanTexture::CopyBufferToMipLevel` adds regions to a `TextureCopyBatch`, and `FinishCopyBatch` uploads all mips with one `vkCmdCopyBufferToImage`. Descriptor sets are now allocated eight at a time into a `setCache[8]`, not one per draw. In `VulkanBarrier.cpp`, color attachment barriers dropped `COLOR_ATTACHMENT_READ` from the source access. Depth barriers dropped `DEPTH_STENCIL_ATTACHMENT_READ` and now use only `LATE_FRAGMENT_TESTS` as source and `EARLY_FRAGMENT_TESTS` as destination. b2f2e7356a 2023-05 “Add new VulkanBarrierBatch” · 2b1e87792f 2023-03 “Vulkan: During texture upload, batch the buffer->image copies to do all the mips at once.” · 8ebc08185b 2023-10 “Allocate descriptors in blocks of 8 instead of individually, to reduce overhead.” · 22fa5f787b 2024-04 “Optimize some barrier flags down to what's actually necessary.”

### ppsspp-200 — Shadow driver state and skip redundant state-setting calls
Lens: gpu-rendering · Source: PPSSPP
Principle: Keep a CPU-side copy of the last value sent for each piece of state. Call the API only when the new value differs. On older or mobile drivers a state change can be far more expensive than a compare, so this is especially worthwhile there.
Why it is faster: —
Applies elsewhere: canvas 2D wrappers that cache `fillStyle`, UI toolkits that skip identical style updates, and embedded LCD drivers that skip unchanged register writes.
Watch out: —
Seen in PPSSPP: `GLQueueRunner::PerformRenderPass` now keeps local copies of blend func, stencil func/op/mask, depth range, cull/front face and scissor, initialized to `(GLenum)-1`, and wraps each `glBlendFuncSeparate`/`glStencilFunc`/`glDepthRange` in a compare. Separately, `DIRTY_UVSCALEOFFSET` stopped being set on unrelated changes such as `TEXADDR0` writes, which “greatly reduces the amount of glUniform4fv calls”. 3b81b3ca80 2023-05 “Do dirty-tracking for stencil state” · 2081e2865a 2023-05 “Dirty track a bunch more GL state. It's so expensive to change on old devices!” · 80e47b7bd3 2023-05 “Only dirty the uniform UVSCALEOFFSET when really needed”

### ppsspp-201 — Specialize away math the current state makes unnecessary
Lens: gpu-rendering · Source: PPSSPP
Principle: When code is generated or selected from state, check whether each operation can affect the output and emit it only if it can. Two examples: a clamp is redundant if the output conversion clamps anyway, and if only one scalar of a transform is needed, it can come from a single precomputed dot product instead of a full matrix multiply.
Why it is faster: —
Applies elsewhere: ubershader permutation pruning, query planners that drop no-op predicates, and computing only the view depth for sorting.
Watch out: —
Seen in PPSSPP: `FragmentShaderGenerator` emits `clamp(v.rgb * u_texMul, 0.0, 1.0)` only if fog, color test, shader blending, logic-op simulation, color masking or blue-to-alpha will process the color afterwards. Otherwise it emits `v.rgb *= u_texMul`. `MatrixNeedsProjection` ignores `m[8]` for `GE_PROJMAP_UV`, so the common case skips texture projection. The vertex shader writes one clip distance for clamped depth instead of two. The software transform unit had a separate `POS_TO_VIEW` matrix mode just to get view-space Z for fog. That mode is gone. A precomputed `Vec4f posToFog` (the world-view Z row combined with fog end and slope) gives the fog factor with one dot product, and the main matrix always goes straight to clip space. fd43807c4d 2023-05 “New fragment shader optimization: Only clamp color output if needed.” · 106d730a20 2022-12 “GPU: Optimize out common case texture proj.” · bc3d3cf9fb 2022-10 “GPU: Optimize clip distances needed.” · 067fac6817 2022-09 “softgpu: Skip matrix multiply for fog factor calc.”

### ppsspp-202 — Canonicalize state from the actual data to reach a cheaper path
Lens: branching-fast-paths · Source: PPSSPP
Principle: Configured state is often more general than what the data in use can produce. Examples: blending with SRCALPHA when every vertex alpha is 255, an alpha test that no palette entry can fail, or a depth test of ALWAYS with writes off. Rewrite the state to its simplest equivalent before choosing a code path. A cheaper specialized routine then runs.
Why it is faster: —
Applies elsewhere: image pipelines that skip alpha compositing for opaque images, query engines that drop a filter when column statistics show it always passes, and compilers that fold flags through value-range analysis.
Watch out: —
Seen in PPSSPP: In the software rasterizer, `OptimizeFlatRasterizerState` turns SRCALPHA/INVSRCALPHA blend factors into ONE/ZERO when a flat rectangle's alpha is 0 or 0xFF. `CheckClutAlphaFull` scans the palette once (the result is cached in `CLUT_ALPHA_CHECKED`) so blending and a `NOTEQUAL 0` alpha test can be switched off. The hardware backends use `IsDepthTestEffectivelyDisabled()` to treat ALWAYS-without-write as disabled. a4c3718431 2022-09 “softgpu: Optimize rectangle sampling/blending.” · adc94b1950 2022-12 “softgpu: Use CLUT to optimize out blending more.” · eb19e24399 2022-12 “softgpu: Skip non-zero alpha test if impossible.” · 745d9ad320 2022-11 “GPU: Avoid enabling depth test pointlessly.”

### ppsspp-203 — Don't produce data that no consumer reads
Lens: laziness · Source: PPSSPP
Principle: Trace each intermediate to whoever uses it. If the final result does not depend on it, skip producing it. Three cases here: a destination pixel blended with factor ZERO, an index list for draws that are all non-indexed and clockwise, and pipeline variants for render-pass types the pipeline's flags make impossible.
Why it is faster: —
Applies elsewhere: column pruning in query engines, tree-shaking, and skipping JSON fields a client didn't request.
Watch out: —
Seen in PPSSPP: The x86 pixel JIT computes `readsDstPixel`. When it is false, the code emits `PXOR dstReg,dstReg` and does not load the framebuffer. `DrawEngineCommon` tracks `anyCCWOrIndexed_` and `seenPrims_` so it can skip `DecodeInds()` entirely. `CreateGraphicsPipeline` skips variants that need depth or an input attachment when the render-pass type has neither. 6e6535c263 2022-09 “softjit: Skip reading dst pixel where blended out.” · 61acce195c 2023-10 “Avoid decoding indices when we don't need them.” · 9f3dfe7ebe 2022-09 “Vulkan: Don't compile pipeline variants that don't make sense given their flags.”

### ppsspp-204 — Memoize per-element work when indices reuse elements heavily
Lens: memory-layout · Source: PPSSPP
Principle: When an index buffer references each vertex several times, transform every unique vertex once into a scratch array and index into that. If the index count is not larger than the referenced range, there is no reuse, and the lazy per-index path is cheaper.
Why it is faster: —
Applies elsewhere: post-transform vertex caches, dictionary-encoded columns (decode each dictionary entry once), and memoizing a pure function over repeated keys.
Watch out: —
Seen in PPSSPP: `SoftwareVertexReader` enables `useCache_` only when `vertex_count > upperBound - lowerBound + 1`. `UpdateCache()` then runs `ReadVertex` once per vertex in that range into a static `cached_` vector, and `Read()` returns `cached_[conv_(vtx) - lowerBound_]`. The commit says this matters most for splines and Béziers. 88b3b26ed3 2022-09 “softgpu: Cache reused indexed verts.”

### ppsspp-205 — Cheap pre-check before the expensive path
Lens: branching-fast-paths · Source: PPSSPP
Principle: Put an inexpensive test first that settles the common case. Examples: is anything queued, did the value change, does the text fit unwrapped. The costly work then runs only when the test fails.
Why it is faster: —
Applies elsewhere: layout engines that measure single-line text before wrapping, `if (dirty)` before re-rendering, and length comparison before byte comparison.
Watch out: —
Seen in PPSSPP: `BinManager::Flush` returns immediately when `queueRange_.x1 == 0x7FFFFFFF` (nothing queued), and the framebuffer/depth-buffer address handlers flush only `if (diff)`. `TextDrawer::MeasureStringRect` measures the plain string first and runs the wrapping path only if `plainW > bounds.w`. e2929fb395 2022-09 “softgpu: Skip flushing on no change.” · ce980af01e 2024-10 “TextDrawer: Improve performance for text wrapping (by measuring without wrapping first)”

### ppsspp-206 — Defer metadata queries until a caller needs the answer
Lens: io · Source: PPSSPP
Principle: Opening a file does not mean the caller will ask for its size or start sector. Make the metadata a lazily filled member: fetch it on first access and remember it. Opens that never seek-to-end or query info then skip a filesystem round trip.
Why it is faster: —
Applies elsewhere: ORM lazy-loaded fields, HTTP clients that issue HEAD only when a length is required, and lazy `stat()` in file managers.
Watch out: —
Seen in PPSSPP: In `sceIo.cpp`, `FileNode` gained `infoReady` and a `FileInfo()` accessor that calls `pspFileSystem.GetFileInfo` on first use. `__IoOpen` no longer calls it up front, except for TTY nodes. 4db30e7724 2022-10 “Io: Defer GetFileInfo() until it's needed.”

### ppsspp-207 — One bulk query instead of many point queries
Lens: io · Source: PPSSPP
Principle: Every existence check or stat is a syscall, and on Android content URIs it is an IPC call. Fetch a directory listing once and answer the per-file questions from it in memory. Get the file size from the handle you already opened. Drop existence checks that the next operation would fail on anyway.
Why it is faster: —
Applies elsewhere: replacing N+1 queries with one JOIN, S3 `ListObjects` instead of a HEAD per key, and using `fstat` on an open fd.
Watch out: —
Seen in PPSSPP: The savedata code replaced `GetFileInfo(dir)` followed by `GetFileInfo(dir+"/"+name)` with one `GetDirListing(path, &exists)` and a `FileFromListing()` lookup. Multi-load now lists `savePath` once to filter out missing saves. `VFSBackend::OpenFileForRead` now returns `*size`, from `ftell` or `zip_stat_index`, which removed a separate `GetFileInfo`. The texture replacer dropped a `File::Exists` per texture. 8c1acc30e4 2022-10 “Savedata: Combine file info and dir listing calls.” · 7cc59dcda1 2024-11 “Savedata multiload: Do a directory listing so we can filter missing directories faster.” · e277db63c0 2023-03 “Optimize away those extra GetFileInfo...” · e504f0db8f 2022-10 “Replacement: Avoid needless exists check per tex.”

### ppsspp-208 — Touch only the bytes the answer needs
Lens: io · Source: PPSSPP
Principle: Many questions about a file can be answered without reading or rewriting it. A format's fixed header gives the dimensions. A same-device rename moves a file by updating only metadata, while copy-then-delete rewrites every byte. Try the cheap operation first and fall back only when it fails.
Why it is faster: —
Applies elsewhere: image-size probes for HTML layout, container sniffing in media servers, `rename(2)` in installers, and S3 server-side copy.
Watch out: —
Seen in PPSSPP: `PNGHeaderPeek` is a packed struct (`magic`, `ihdrTag`, `be_width`, `be_height`, ...) with `IsValidPNGHeader()`. The texture replacer uses it in place of a `png_image_begin_read_from_memory` call. The memstick folder move uses `MoveIfFast`: for matching path types it tries `Android_MoveFile` on content URIs or `Rename`, and recurses whole directories. It falls back to `Copy` only when that fails. The commit calls it “Drastically faster.” 76c6058abe 2023-03 “Texture replacer: Faster way to get width/height from a png” · c6c0d0bf42 2024-01 “Android memstick folder moves: Do recursive moves if in the same device. Drastically faster.”

### ppsspp-209 — Load data and run scans only when first needed
Lens: laziness · Source: PPSSPP
Principle: Data and scans that only some code paths need should run the first time they are needed, not at startup. Examples are lookup tables for rare instructions, a language mapping, or a full-tree size scan. Block on the result only where it is used. Startup time and memory then reflect what actually runs.
Why it is faster: —
Applies elsewhere: lazy-loaded locale bundles and dynamic `import()`, on-demand ICU or timezone data, and IDEs that index on demand.
Watch out: —
Seen in PPSSPP: The VFPU sine/rsqrt/exp tables moved to `assets/vfpu/*.dat`. `load_vfpu_table` reads them through `g_VFS` behind a `static bool loaded = LOAD_TABLE(...) && ...` guard the first time a table is needed. `langValuesMapping` is likewise read and cached only when first requested. `MemoryStick` now starts the recursive SAVEDATA size calculation only when the `MemstickFixedFree` compat flag is set. `BlockUntilReady()` moved inside the branch that consumes the result. 098e07fc33 2023-03 “Implement load-on-demand of vfpu tables” · 93e5060137 2022-12 “Load and cache langValuesMapping on demand” · 84154e837e 2024-11 “Avoid running the initial savedata size scan except on games that need it.”

### ppsspp-210 — Pad or convert during the final copy, not in stored data
Lens: memory-layout · Source: PPSSPP
Principle: If a consumer needs data in a padded or converted layout, keep the compact original in memory and apply the transform while copying into the destination buffer. That copy happens anyway, so the transform adds little, and the stored data is smaller.
Why it is faster: —
Applies elsewhere: applying stride or row alignment while writing into a GPU staging buffer, and adding framing while writing to a socket instead of building a framed copy.
Watch out: —
Seen in PPSSPP: Replacement textures used to be padded to power-of-two sizes at load time. `ReplacedTexture::CopyLevelTo` now writes `info.w` pixels and zero-fills `outW - info.w` per row and the rows beyond `info.h`, using new `fullW`/`fullH` fields. The commit says this saves memory. 79f4e73110 2023-03 “Replacer: Perform the padding to pow2 size during CopyLevelTo, not during load. Saves memory.”

### ppsspp-211 — Cache derived results, with explicit or scoped invalidation
Lens: laziness · Source: PPSSPP
Principle: If something expensive is derived from inputs that rarely change, keep it. Drop it when a dirty flag or event says the inputs changed, or scope the cache to one operation and clear it whenever control passes to code that could modify the inputs. Either way it stays correct and avoids repeated parsing or recomputation.
Why it is faster: —
Applies elsewhere: memoized selectors in Redux, per-request DataLoader caches, and compiled regexes rebuilt only when the config changes.
Watch out: —
Seen in PPSSPP: `FramebufferManagerCommon` keeps `currentFramebufferCopy_` for self-texturing and reuses it until `GE_CMD_TEXFLUSH`, which now calls `DiscardFramebufferCopy()`. Frustum planes for bounding-box culling are recomputed in `UpdatePlanes()` only when `DIRTY_CULL_PLANES` is set. Memcheck ranges are rebuilt into `memCheckRangesRead_`/`memCheckRangesWrite_` only when breakpoints change, which speeds up JIT compilation. `SavedataParam::LoadCachedSFO` returns a `std::shared_ptr<ParamSFOData>` from a per-path cache. `ClearCaches()` is called at each entry from PSP code into `PSPSaveDialog`/`SavedataParam`. The texture replacer also shares one `levelCache_` entry between cache keys that map to the same file, with purging by `lastUsed`. 7a7117f2f7 2023-03 “Cache framebuffer copies (for self-texturing) until the next TexFlush instruction.” · 061131ec8a 2023-07 “Cache planes used for BBOX culling” · e73474e590 2023-04 “Debugger: Cache memcheck ranges.” · db5011d7f1 2022-10 “Savedata: Cache SFO data during processing.” · 0048dc2bb8 2022-10 “Replacement: Cache file data across cache keys.”

### ppsspp-212 — Reuse uploaded resources by content hash
Lens: lookup-tables · Source: PPSSPP
Principle: When identical data is uploaded repeatedly, hash it and look for an existing resource with the same hash and shape before uploading again. Hashing CPU memory is far cheaper than allocating and filling a GPU texture.
Why it is faster: —
Applies elsewhere: content-addressed asset caches (ETag), deduplicating blob storage, and sprite or glyph atlas caches keyed by hash.
Watch out: —
Seen in PPSSPP: `FramebufferManagerCommon::MakePixelsTexture` computes an XXH3 hash over the source rows. It first looks in `drawPixelsCache_` for an entry with the same `contentsHash`, width, height and format, and only falls back to overwriting an unused entry or creating a new one. An earlier commit had started keeping these textures around for reuse. 632fa1c9d6 2023-11 “Cache and hash data for DrawPixels.” · 7c4b9bac90 2023-05 “Cache textures created by MakePixelsTexture and reuse where appropriate.”

### ppsspp-213 — Flat hash tables and cheap hashes for small keys
Lens: lookup-tables · Source: PPSSPP
Principle: `std::unordered_map` allocates a node per entry and follows a pointer on every lookup. An open-addressed table keeps entries in one flat array. When keys are a few integers or pointers, rotate-XOR them together and finish with a small bit-mixer. That is cheaper than running a general-purpose buffer hash.
Why it is faster: —
Applies elsewhere: absl `flat_hash_map`, SwissTable, and tuple keys in ECS lookups. Fast mixers are not safe against adversarial input.
Watch out: —
Seen in PPSSPP: `PixelJitCache` and `SamplerJitCache` replaced `std::unordered_map<PixelFuncID, SingleFunc>` with `DenseHashMap<size_t, SingleFunc, nullptr>`, keyed by a precomputed `std::hash` of the ID. `DrawEngineCommon::ComputeDrawcallsHash` folds each draw call's `verts`, `inds`, `indexType`, `vertexCount` and `prim` with `__rotl`, then `lowbias32_r`. This replaced XXH. 778a0487cb 2022-12 “softjit: Switch to DenseHashMap.” · f5516d3248 2023-06 “Actually switch away from XXH to a custom hash, to de-risk”

### ppsspp-214 — One-entry “last lookup” cache in front of a locked map
Lens: lookup-tables · Source: PPSSPP
Principle: Lookups often repeat the previous key. Remember the last key and result, and compare against them before taking the lock and probing the map. Make the memo `thread_local` so it needs no atomics or locking.
Why it is faster: —
Applies elsewhere: the last-used font or style in a text renderer, inline caches in JS engines, and a last-route cache in a request router.
Watch out: —
Seen in PPSSPP: `PixelJitCache::GetSingle` and `SamplerJitCache::GetNearest`/`GetLinear`/`GetFetch` check `lastSingle_.key == key` before locking `jitCacheLock`. The first version used `std::atomic<LastEntry>`, which needed libatomic. It became `static thread_local LastCache`. The commit says this cut time in `ComputeRasterizerState()` “quite a bit”. 400f6abf9a 2022-12 “softgpu: Optimize lookup of last jit func.” · eda3ce556e 2022-12 “softgpu: Avoid atomic structs.”

### ppsspp-215 — Build packed pixels in a register and store one word
Lens: simd · Source: PPSSPP
Principle: To convert colors, build the whole 32-bit RGBA value in a register and store it once, not four separate bytes. From floats: multiply by 255, convert, then narrow with saturating pack instructions, which clamp for free. From 16-bit formats: expand with shifts and masks.
Why it is faster: —
Applies elsewhere: canvas pixel output, RGB565 framebuffers on microcontrollers, and float-to-PCM audio conversion.
Watch out: —
Seen in PPSSPP: `Float4ToUint8x4` in `SmallDataConvert.h` is now `_mm_mul_ps` by `exactly_255_x4`, then `_mm_cvtps_epi32`, `_mm_packs_epi32`, `_mm_packus_epi16`, `_mm_cvtsi128_si32`. `ReadColor0_8888` was also changed to return a `u32` instead of writing four bytes. The non-JIT color decoders `Step_Color565`/`5551`/`4444` replaced per-channel `c[0] = Convert5To8(...)` byte writes with `*(u32*)c = RGB565ToRGBA8888(cdata)`. The alpha check moved to a single `if (!alpha)`. e6f0f84a45 2022-12 “SSE optimize Float4ToUint8x4, some uses” · 9ac7054b01 2024-06 “Vertex decoder (non-JIT): Optimize 16-bit color decoders.”

### ppsspp-216 — Vectorize the hottest inner kernel with narrow integer or float lanes
Lens: simd · Source: PPSSPP
Principle: Once profiling shows a kernel such as a filter tap loop or bilinear blend at the top, rewrite it so one instruction processes four or eight elements. Use 16-bit fixed-point lanes when the products fit, which avoids float conversion. Replace function-pointer dispatch around the kernel with direct calls so the compiler can inline them.
Why it is faster: —
Applies elsewhere: image resizing in libyuv and pillow-simd, FIR/IIR filters in audio plugins, and codecs on mobile SoCs.
Watch out: —
Seen in PPSSPP: In `Sampler.cpp`, the bilinear blend loads the samples, uses `_mm_unpacklo/hi_epi8` against zero, multiplies by `0x10 - frac_v` and `frac_v`, then by `mul_u`. `mul_u` is built as `(frac_u ^ 0xF) + 1` to avoid poor MSVC codegen. Two adds with a shuffle and `>> 8` finish it. In `atrac3plusdsp.cpp`, the `ff_atrac3p_ipqf` inner loop now uses `_mm_loadu_ps`/`_mm_mul_ps`/`_mm_add_ps` (with a NEON twin), and `q->mdct_ctx.imdct_calc(...)` became a direct `imdct_calc(...)`. 159faaa2ec 2023-06 “softgpu: Optimize (bi-)linear texture filtering” · 7b6b7ebe3c 2024-04 “Add some SSE code for an atrac hotspot just for fun, remove function pointers”

### ppsspp-217 — Thin cross-platform SIMD wrapper types
Lens: simd · Source: PPSSPP
Principle: Wrap the vector registers in small value types such as `Vec4F32` and `Mat4F32`, with inline methods that map to SSE2 or NEON. The algorithm is written once and still compiles to tight intrinsics on every target. Prefer baseline instruction sequences (SSE2) over newer ones (SSE4) when the speed difference is small.
Why it is faster: —
Applies elsewhere: `std::experimental::simd`, Highway, glm's SIMD backend, and WebAssembly SIMD wrappers.
Watch out: —
Seen in PPSSPP: `DecodeAndTransformForDepthRaster` went from a scalar `temp[3]` loop calling `Vec3ByMatrix44` to `Vec4F32::LoadConvertS16(data).Mul(1.0f/32768.f).AsVec3ByMatrix44(mat).Store(dest + i*4)`. The commit notes “the generated assembly is great”. `ColorConv.cpp` replaced SSE4-only ops with SSE2 sequences in the new `SIMDHeaders.h`. 6a1010afb0 2024-12 “Use CrossSIMD to optimize DecodeAndTransformForDepthRaster” · d2a96dc7b5 2024-12 “Replace some SSE4-specific operations with SSE2 sequences.”

### ppsspp-218 — Write loops the autovectorizer can handle
Lens: compiler-build · Source: PPSSPP
Principle: Compilers vectorize simple reductions and element-wise loops, but only if element types are narrow and consistent and the pointers provably don't alias. Keep accumulators in the element's own type, give each element width its own loop, and mark non-overlapping pointers `restrict`. Use in-place APIs so source and destination aren't separate but possibly aliasing arguments.
Why it is faster: —
Applies elsewhere: min/max over sensor data in C, array math in Rust, where slices give no-alias for free, and NumPy-style kernels in C++.
Watch out: —
Seen in PPSSPP: `GetIndexBounds` previously tracked `int` bounds for 8-bit and 16-bit indices alike. It now has a separate branch per width with `uint8_t`/`uint16_t` accumulators, “friendlier to autovectorization. Works on x86 at least.” In the ATRAC code, `vector_fmul(dst, src0, src1, len)` became in-place `vector_fmul(av_restrict dst, const float *av_restrict src, len)` with a portable `av_restrict`, “This enables autovectorization to work here.” 45bc4d8750 2023-09 “Make GetIndexBounds friendlier to autovectorization...” · 090d1f8d0e 2024-05 “atrac3: Add a cross platform restrict modifier, use it for the vector math functions”

### ppsspp-219 — Specialize SIMD loops for the common small sizes
Lens: simd · Source: PPSSPP
Principle: A SIMD loop tuned for long inputs still pays for its full setup and stores on tiny inputs. Add early exits sized to what is actually common, such as “one store covers two triangles”. Hoist loop-invariant fields into locals so the compiler doesn't reload them through `this`.
Why it is faster: —
Applies elsewhere: `memcpy` implementations with small-size fast paths, and short-string optimizations in hashing.
Watch out: —
Seen in PPSSPP: The SSE and NEON paths of `IndexGenerator::AddStrip` used to compute and store three offset vectors no matter how many triangles there were. They now store `offsets0` only, add `offsets1` only if `numTris > 2`, and enter the loop only if `numTris > 5`. The `decodeCounter_` member is copied into a local around the decode loop. 77da36c03f 2023-06 “SSE addstrip: Add the early-outs.”

### ppsspp-220 — Fold constant branches and forward stores to loads in the IR
Lens: jit-interpreters · Source: PPSSPP
Principle: Two classic cleanup passes. If both operands of a conditional exit are known constants, the branch is always or never taken, so replace it with a jump or delete it. If a load reads the same address and width that a store just wrote, reuse the stored register in place of the load.
Why it is faster: —
Applies elsewhere: SCCP and store-to-load forwarding in LLVM, and `NODE_ENV` dead-code elimination in JS bundlers.
Watch out: —
Seen in PPSSPP: In `IRPassSimplify.cpp`, `PropagateConstants` now evaluates `ExitToConstIfEq/Neq/GtZ/...` when its sources are immediates. On pass it writes `ExitToConst` and skips the following exit. On fail it drops the instruction. The commit says this is “common for the first cycle of loops”. The new `OptimizeLoadsAfterStores` pass checks whether `Store32` is followed by `Load32` with the same base and offset, and similarly for other widths, and rewrites the load. The commit calls these loads-after-stores “surprisingly not that uncommon.” 3f97545f99 2023-08 “irjit: Reduce exits from constants.” · bd0beb68a4 2024-06 “Add new IR optimization pass, OptimizeLoadsAfterStores”

### ppsspp-221 — Peephole-fuse common instruction pairs into one op
Lens: jit-interpreters · Source: PPSSPP
Principle: Look for adjacent instruction pairs that appear together very often, and emit a single fused op for each. Every fused pair saves one dispatch in an interpreter, or one register round trip in a JIT. Likewise, don't split an op into pieces when the backend can execute it whole faster.
Why it is faster: —
Applies elsewhere: superinstructions in bytecode VMs such as CPython and Lua, macro-op fusion in CPUs, and fused multiply-add in compilers.
Watch out: —
Seen in PPSSPP: Unaligned `svl.q`/`svr.q` pairs (detected with `(nextOp.encoding ^ op.encoding) == 0x0E`) are treated as a single `sv.q`/`lv.q`. `FMovToGPR` followed by `ShrImm 8` becomes `OptFMovToGPRShr8`, which is “heavily used when writing display lists”. With `optimizeForInterpreter`, lone `lwl/lwr/swl/swr` pass through unsplit, and variable shifts skip the `AndConst 31` because the interpreter masks them itself. 159b41a0fa 2023-08 “irjit: Fuse unaligned svl.q/svr.q together.” · 0c246297d2 2024-06 “Create an IR op for a FPRtoGPR + shift-right-8, very common” · 0abcd00372 2024-06 “IR Interpreter: Don't split up lone lwr/swr/lwl/swl instructions...” · 982a83d867 2024-06 “IRInterpreter: Optimize variable shifts (no need to mask by 0x1f)”

### ppsspp-222 — Keep vector values in vector form across operations
Lens: jit-interpreters · Source: PPSSPP
Principle: On a SIMD backend, splitting a 4-lane value into scalars and recombining it costs shuffles and extra register traffic. An analysis pass can track which registers are “still a full vec4”, rewrite scalar ops to vector ops where all lanes are used, and avoid flushing lanes one at a time.
Why it is faster: —
Applies elsewhere: SLP vectorization in LLVM, and keeping data in SoA form across a pipeline in game math libraries.
Watch out: —
Seen in PPSSPP: `ReduceVec4Flush` in `IRPassSimplify.cpp` runs only with `opts.preferVec4`. It keeps `isVec4[256]`/`isUsed[256]`/`isVec4Dirty[256]` and rewrites sequences so that Vec4 registers aren't downgraded to individual lanes. 5f84887dea 2023-08 “irjit: Add a pass to keep Vec4s in Vec4s.”

### ppsspp-223 — Link compiled blocks directly to each other
Lens: jit-interpreters · Source: PPSSPP
Principle: When a translated block ends in a jump to a known address, patch that exit to branch straight into the target's native code once the target is compiled. Execution then skips the dispatcher's lookup on every hot-path transition. Keep a reverse map so links can be undone when a block is invalidated.
Why it is faster: —
Applies elsewhere: trace linking in LuaJIT and QEMU TCG, and direct-threaded interpreters.
Watch out: —
Seen in PPSSPP: `IRNativeBackend::FinalizeBlock` uses `linksTo_.equal_range(pc)` to `OverwriteExit` incoming exits, and resolves the block's own outgoing exits through `GetBlockNumberFromStartAddress`. RISC-V got this first. Invalidation rewrites the fixups. Later, `cf678a1aa6` raised `MAX_NUM_BLOCKS` to 65536*4. fcc90095f7 2023-08 “riscv: Enable block linking.” · cf678a1aa6 2024-10 “JIT: Allow more blocks to be stored in the cache...”

### ppsspp-224 — Low-level code generation habits for hot generated loops
Lens: jit-interpreters · Source: PPSSPP
Principle: Small choices in an emitter add up in hot loops. Keep loop-carried values in reserved registers and write them back once. Prefer a few simple instructions over one long-latency “do everything” instruction. Put constants in a literal pool next to the code. Align loop heads to 16 bytes with multi-byte NOPs. Leave out the prologue and epilogue when no callee-saved register is used.
Why it is faster: —
Applies elsewhere: the same rules apply to hand-written DSP or assembly kernels, `-falign-loops`, leaf-function optimization, and constant islands in ARM codegen.
Watch out: —
Seen in PPSSPP: The x86 IR backend's `Vec4Dot` replaced `DPPS ..., 0xF1` with `MULPS`, then `SHUFPS`/`ADDPS`/`MOVHLPS`/`ADDSS`. `EmitFPUConstants`/`EmitVecConstants` place `noSignMask`, `qNAN`, `mulTableVi2f` and similar values in the code buffer, which removed the `RipAccessible` fallbacks. `XEmitter::NopAlignCode16()` emits one of 16 canonical long-NOP strings before the vertex decoder's main loop, and the ARM backends got the same treatment. The ARM64 vertex JIT saves X16 only when `Step_TcU16ThroughToFloat` (texture-bounds tracking) is present, and skips `ABI_PushRegisters` entirely when there is nothing to save. The x86-64 vertex decoder keeps `gstate_c.vertexFullAlpha` in `R11`. It is loaded before the loop, cleared or ANDed by the color steps, and stored after the loop. The IR-based x86 JIT keeps `downcount` in a register (`DOWNCOUNTREG`) when the jitbase fits in 31 bits, so block checks become `TEST reg,reg` rather than `CMP [ctx+off],0`. 1abd3ac361 2023-08 “x86jit: Use a faster vdot than DPPS.” · c4d13e522b 2023-08 “x86jit: Use a literal pool to avoid rip checks.” · c4e44d66b0 2023-06 “x86/x64: Nop-align the main loop of vertex decoder loops” · 646e3b269d 2023-09 “arm64jit: Skip vertexjit prolog/epilog if possible.” · a164f77f47 2023-06 “VertexDecoderX86 (64-bit only): Avoid a memory access per loop iteration for alpha” · ee5381917a 2023-08 “x86jit: Implement downcount in register.”

### ppsspp-225 — Flat layouts for compiler bookkeeping: arenas and flag arrays
Lens: memory-layout · Source: PPSSPP
Principle: Put variable-length records into one contiguous arena and refer to them by offset. Allocations go away, locality improves, and because offsets only increase, an owner can be found by binary search. Keep rarely-set flags in their own byte array (SoA) and jump between set entries with vectorized `memchr`, without testing every struct.
Why it is faster: —
Applies elsewhere: flat AST arenas, interned string pools, dirty-bit arrays in caches, and bitmap scans with `ctz`.
Watch out: —
Seen in PPSSPP: `IRBlockCache` now owns `arena_`. `AllocateBlock(em_address, mipsBytes, instructions)` appends to it, and blocks store `GetInstructionOffset()`. The dispatcher runs `IRInterpret(mips, blocks_.GetArenaPtr() + offset)`. Offset-to-block lookup became a binary search, cross-checked against a linear scan in debug builds. `IRImmRegCache` changed `reg_[i].isImm/immVal` into separate `isImm_[]` and `immVal_[]` arrays. `FlushAll` now advances using `memchr(&isImm_[i], 1, TOTAL - i)`, because “Most of the time, lots are not.” 5526ef012c 2024-06 “Store IR instructions in a large arena vector instead of loosely in each block.” · 8d6e96d04e 2024-06 “Use binary search to find IR block offsets” · 32d8f6196f 2023-08 “irjit: Cut time flushing imm regs.”

### ppsspp-226 — Tighten an interpreter's dispatch loop
Lens: branching-fast-paths · Source: PPSSPP
Principle: In an interpreter's hottest loop, every instruction counts. Stop on a terminator op instead of also checking a counter. Copy member pointers into locals so they stay in registers. Merge a validity check and an alignment check into one mask-and-compare.
Why it is faster: —
Applies elsewhere: bytecode VMs, regex engines, and packet-filter interpreters such as eBPF and pcap.
Watch out: —
Seen in PPSSPP: `IRInterpret(ms, inst, count)` became `IRInterpret(ms, inst)` with `while (true)`, relying on every block ending in an exit op. The commit calls it “a good speed boost!”. `IRJit::RunLoopUntil` copies `mips_` into a local, and `IsValidAddress(pc) && (pc & 3) == 0` became `IsValid4AlignedAddress`, with masks like `(address & 0x3E000003) == 0x08000000`. fae846e52a 2024-05 “Remove the count parameter from IRInterpret. This is a good speed boost!” · c332d564db 2024-05 “IRInterpreter: Cache the MIPSState pointer in a local, faster address check”

### ppsspp-227 — Look a value up once and pass it down
Lens: algorithms · Source: PPSSPP
Principle: If several functions on one path each look up the same derived object, perform the lookup once at the top and pass the result as a parameter. Similarly, cache per-item metadata next to the item so inner loops don't repeat a table lookup per access.
Why it is faster: —
Applies elsewhere: resolving a user or tenant once per request and threading it through handlers, and storing a type-info pointer beside each AST node.
Watch out: —
Seen in PPSSPP: `TestBoundingBox`, `TestBoundingBoxFast`, `TestBoundingBoxThrough` and `SkipPrim` now take a `VertexDecoder *dec`, so they no longer call `GetVertexDecoder(vertType)` themselves. The IR analysis helpers take `IRInstMeta`, which carries `inst.m` alongside the instruction, in place of calling `GetIRMeta(inst.op)` on every check. The commit says this improves compile performance. 0b06cd1379 2024-12 “Actually avoid looking up the vertex decoder more than once” · cd46f0b4cb 2023-09 “irjit: Cache IR metadata lookups.”

### ppsspp-228 — Extend the current batch on a fast path instead of starting a new one
Lens: algorithms · Source: PPSSPP
Principle: When a new request is compatible with the previous one, such as the same format and contiguous data, append it to the open batch with minimal bookkeeping. Treat a re-sent identical setting as “no change” so it doesn't break the batch. Compute per-batch values once rather than per item.
Why it is faster: —
Applies elsewhere: coalescing adjacent writes in an I/O scheduler, merging consecutive same-style text runs, and batching consecutive INSERTs to one table.
Watch out: —
Seen in PPSSPP: `DrawEngineCommon::ExtendNonIndexedPrim` appends a `DeferredInds` entry that points into the last `DeferredVerts`, and bumps that entry's `vertexCount` rather than decoding again. In the `GPUCommonHW` prim loop, a `VADDR` write that doesn't change the address no longer breaks `canExtend`. The `clockwise` cull flag is now computed once per `SubmitPrim` call, not once per draw inside the decode loop. 0260aebc26 2023-10 “Implement fast-path for merging non-indexed draws quickly.” · 200575b2bc 2023-10 “Allow the new optimization through redundant VADDR instructions, very common” · 82606b6eb2 2023-10 “Move the clockwise calculation out of the AddPrim loop”

### ppsspp-229 — Replace a remapping table with direct index arithmetic
Lens: algorithms · Source: PPSSPP
Principle: If a lookup table only encodes a regular layout, such as row/column strides, compute the index arithmetically and drop the table. That removes a dependent load from every access.
Why it is faster: —
Applies elsewhere: swizzled texture addressing, matrix transpose access, and register-file mapping in emulators and VMs.
Watch out: —
Seen in PPSSPP: `ReadVector`/`WriteVector` in `MIPSVFPUUtils.cpp` went through `V(reg)`, which looks up `voffset[]`. They now index `currentMIPS->v[base + ((row+i)&3)*4]` directly, with `mtx = (reg << 2) & 0x70`. The commit reports a speedup of at most 0.5% in GTA, “But enough of these small ones and they start adding up.” 0d06af87b6 2023-10 “Interpreter: Optimize ReadVector/WriteVector by removing voffset lookups”

### ppsspp-230 — Conservative cheap culling with transformed planes
Lens: algorithms · Source: PPSSPP
Principle: A visibility test only has to be conservative: it may draw something that turns out to be invisible, but must never skip something visible. Bail out of awkward cases (offsets near the edge, VR) by answering “visible”. Transform the frustum planes into object space once, instead of transforming every vertex by every plane. The remaining test is a dot product per plane, which SIMD handles well.
Why it is faster: —
Applies elsewhere: frustum culling in game engines, map-tile visibility in web maps, and bounding-box prefilters in spatial databases.
Watch out: —
Seen in PPSSPP: `TestBoundingBoxFast` skips through-mode, indexing, morph and skinning, and returns true when `offsetOutsideEdge_` is set. The plane data was then “flipped around” into a SoA `planes_.Set(i, ...)` layout so each vertex is transformed once, and the loop got SSE and NEON paths. 89d8ef87ec 2023-12 “Use a less accurate but faster frustum cull for the general draws.” · 62c936babf 2023-12 “Flip the cull plane data around to avoid transforming each vertex multiple times.”

### ppsspp-231 — Use a hardware feature to drop a software fallback path
Lens: gpu-rendering · Source: PPSSPP
Principle: If a device extension provides a behavior natively, such as last-vertex provoking for flat shading, enable it when available. Draws then stay on the fast hardware path and skip CPU emulation of that behavior.
Why it is faster: —
Applies elsewhere: using native `Intl` or Web APIs instead of polyfills when present, and hardware CRC or AES instructions instead of table code.
Watch out: —
Seen in PPSSPP: `VulkanContext` enables `VK_EXT_provoking_vertex` with `provokingVertexLast = true`. `PipelineManagerVulkan` chains `VkPipelineRasterizationProvokingVertexStateCreateInfoEXT`, and `DrawEngineVulkan` no longer forces software transform for flat-shaded draws when the feature is present. 138193a776 2024-07 “Add support for the EXT_provoking_vertex Vulkan extension, allowing us to skip software transform for this case.”

### ppsspp-232 — Push slow, independent work onto prioritized worker tasks
Lens: concurrency · Source: PPSSPP
Principle: Pipeline compilation, asset decoding, sample loading and disk scans don't need to run on the frame or UI thread. Submit them as tasks to a pool that has priority levels, so urgent work runs first and the main loop doesn't stall. Use pool tasks instead of a dedicated `std::thread` for each job.
Why it is faster: —
Applies elsewhere: Web Workers for parsing, Android WorkManager and coroutines on Dispatchers.IO, and parallel shader compilation in game engines.
Watch out: —
Seen in PPSSPP: `VKRGraphicsPipeline::Create` became `CreateAsync`, using `pipeline->SpawnEmpty(&g_threadManager, ...)`. `ThreadManager` queues became `std::deque<Task*> queue[TASK_PRIORITY_COUNT]`. The texture replacer's “Populate” phase (VFS lookups and level setup) moved onto the loader thread. UI sound samples load through a `SampleLoadTask`. The memstick free-space scan became a `Promise<uint64_t>` task in place of a hand-managed thread. 8821d619e4 2023-02 “Create pipelines in parallel” · 0971555e51 2023-01 “ThreadManager: Add simple priority queues.” · 9b7a1a2ddc 2023-03 “Move the "Populate" phase onto the thread” · c966067bd0 2024-10 “Load UI audio effects on a background thread” · 6d2826dcb0 2024-11 “Move the MemoryStick free space calculation to a task instead of a thread”

### ppsspp-233 — Keep producer–consumer queues partly full, and parallelize when safe
Lens: concurrency · Source: PPSSPP
Principle: A producer that fills a worker queue to capacity on every drain will eventually block waiting for space. Push only up to about half capacity on ordinary drains and go to full depth only on an explicit flush. That keeps workers busy without blocking the producer. Also recognize special cases that look unsafe but aren't, and parallelize them instead of falling back to one thread.
Why it is faster: —
Applies elsewhere: bounded channels in Go or Rust pipelines, and tiled renderers that process tiles in parallel when they don't overlap.
Watch out: —
Seen in PPSSPP: `BinManager::Drain(bool flushing)` caps the batch at `QUEUED_PRIMS / 2` when not flushing and checks a new `NearFull()` to avoid `waitable_->Wait()`. `IsExactSelfRender` spots 1:1 self-texturing sprites, which only colorize the image, and lets those keep `GetNumLooperThreads()` workers. 24560eef5c 2022-09 “softgpu: Avoid waiting for a thread to drain.” · 7e7eabf86a 2022-11 “softgpu: Use threads on self-render if safe.”

### ppsspp-234 — Keep slow I/O out of critical sections
Lens: concurrency · Source: PPSSPP
Principle: Read the file into a local buffer without holding the lock, then take the lock only to publish the result. Remove locks around operations that are already thread-safe. Other threads waiting on the same mutex no longer stall behind disk I/O.
Why it is faster: —
Applies elsewhere: caches that fetch outside the lock and then insert, and database connection pools that release the pool lock before network I/O.
Watch out: —
Seen in PPSSPP: `GameInfoCache` reads into a local `std::string data`, closes the handle, then does `*contents = std::move(data)`. `ZipFileReader` dropped `lock_` around `zip_name_locate`, which is marked “this is EXPENSIVE”, and around other operations that don't use the file pointer. e29f59a102 2024-01 “GameInfoCache: Move some I/O out of the mutex. There's more to do.” · 36d10b89e2 2024-11 “Remove locks on safe zip file operations (that don't involve the file pointer)”

### ppsspp-235 — Schedule waits and long jobs for overlap and bounded latency
Lens: concurrency · Source: PPSSPP
Principle: If a frame limiter sleeps before the frame's work is submitted, the consumer idles too. Submit first, then sleep. A single huge copy in a cooperative scheduler holds everyone up, so do it in fixed-size chunks and yield between them. Latency stays bounded and total cost barely changes.
Why it is faster: —
Applies elsewhere: game loops that present before throttling, `scheduler.yield()` chunking in browsers, and incremental GC.
Watch out: —
Seen in PPSSPP: In `sceDisplay.cpp`, at end of frame the limiter now calls `g_frameTiming.DeferWaitUntil(nextFrameTime, &curFrameTime)`, not `WaitUntil`, “for some more overlap”. `WaitUntil` also skips `usleep` when `left <= 0`. The replaced memcpy/memset HLE functions in `ReplaceTables.cpp` now process `SLICE_SIZE = 32768` bytes per call when `bytes > SLICE_SIZE` (except exactly 512*272*4, to detect video). They advance `A0/A1` and decrement `A2` so the guest re-enters, which lets thread switches happen in between. e1f1af6223 2023-12 “Defer waits.” · e1eecb475a 2023-12 “HLE: Slice the very slow memset/memcpy variants.”

### ppsspp-236 — Use non-owning string views for parsing and lookups
Lens: allocation · Source: PPSSPP
Principle: Passing `const std::string&` forces callers with literals or substrings to allocate a temporary. Returning a stripped substring copies it. A `string_view` refers to the existing bytes and already knows its length. Appending pieces with `append` or `push_back` avoids building temporary concatenations.
Why it is faster: —
Applies elsewhere: Rust `&str`, Go substrings, and zero-copy HTTP header and INI/JSON tokenizers.
Watch out: —
Seen in PPSSPP: `StripSpaces` got a `string_view` overload, and `IniFile`'s `ParseLine*` now take views, building `key += sub; key.push_back('#')` instead of `line.substr(...) + "#"`. `Section::Get/Set/GetLine` keys are `std::string_view`, which “avoids recomputing length every time”. `Path` and `AndroidContentURI` methods take views and use `uri.file.append(filePath)`. 2bd2292bac 2023-09 “With string_view, stripping whitespace can be done without allocs.” · 9d0c127961 2024-05 “IniFile: Switch keys to string_view...” · d0ee5fc308 2023-11 “Path: Use string_view more to avoid allocations”

### ppsspp-237 — Block-linked byte queue with per-block scanning
Lens: memory-layout · Source: PPSSPP
Principle: A byte queue made of linked fixed-size blocks appends and consumes in bulk without moving existing data. Search it one block at a time with a tight loop over the contiguous bytes, not through a per-byte `peek(i)` that walks the block chain each call.
Why it is faster: —
Applies elsewhere: network receive buffers (Netty's composite buffers, nginx chains), and scanning for line delimiters in log tailers.
Watch out: —
Seen in PPSSPP: `CharQueue` (16 KB blocks, “optimized to be fast in debug builds”) backs `Buffer`. `Buffer::OffsetToAfterNextCRLF` used to call `data_.peek(i)` for each byte. It now calls `next_crlf_offset()`, which loops over `b->data[b->head + i]` within each block and peeks across a boundary only when it sees `'\r'`. cb27df02f3 2024-11 “Implement new fast queue data structure CharQueue” · 305453b52d 2024-11 “Buffer: Optimize scanning for the next crlf”

### ppsspp-238 — Virtualize long lists and cache their source data
Lens: laziness · Source: PPSSPP
Principle: In an immediate-mode UI, rebuilding a list of thousands of items every frame and submitting every row costs time for rows nobody can see. Fetch the data once, cache it, and use a clipper that iterates only over the visible index range.
Why it is faster: —
Applies elsewhere: react-window and virtual scrolling in web tables, and `RecyclerView` on Android.
Watch out: —
Seen in PPSSPP: The ImDebugger's symbol list called `g_symbolMap->GetAllSymbols(ST_FUNCTION)` every frame and emitted a `Selectable` for each symbol. It now fills `symCache_` once and loops over `ImGuiListClipper`'s `DisplayStart..DisplayEnd`. dd26bcf1af 2024-11 “Cache symbols and use a clipper to speed up the symbol list.”

### ppsspp-239 — Evaluate edge functions for a group of pixels at once
Lens: simd · Source: PPSSPP
Principle: A triangle rasterizer that tests each pixel against three linear edge equations can test four horizontally adjacent pixels in one vector. Seed each lane with x, x+1, x+2, x+3, then step by a splatted delta. The inner loop becomes a few vector adds and a sign-mask test, with no per-pixel branching.
Why it is faster: —
Applies elsewhere: software rasterizers for occlusion culling, point-in-polygon tests over a pixel grid in GIS tools, image-mask generation in 2D canvas libraries.
Watch out: —
Seen in PPSSPP: The depth rasterizer used for occlusion was rewritten around 4x1 pixel groups. Bounding boxes are rounded to multiples of 4, and each edge starts as `Vec4S32::Splat(A) * (Splat(p0x) + zero123) + Splat(B) * y + Splat(C)` and advances by `oneStepX = Splat(A * 4)`. The compare modes were also narrowed to a three-value enum (Greater, Less, Always), the only ones the fast path supports. 73ae6da757 2024-12 “Reimplement the depth rasterizer with SIMD.”

### ppsspp-240 — Vectorize across independent items, not within one
Lens: simd · Source: PPSSPP
Principle: Short per-item math, such as setting up one triangle or computing one vertex, has too little parallelism inside it to fill a vector. Lay the data out structure-of-arrays and put one item in each lane. Per-item operations like cross products and normalization then become plain lane-wise math.
Why it is faster: —
Applies elsewhere: physics engines that solve four contacts at a time, batch geometry kernels in CAD, columnar query engines that evaluate one predicate over many rows.
Watch out: —
Seen in PPSSPP: Depth-raster triangles are stored in groups of four (`tx[0]`, `tx[4]`, `tx[8]` are one triangle's vertices), and setup loads `Vec4S32::LoadAligned(tx)` to handle four triangles together. Spline tessellation was later changed the same way. Weights are stored one row per term, so four u-steps load as one vector, and the result is transposed into vertex order on the way out. An old SSE path that vectorized within one vertex was removed because it was “not worth it”. bcab17fcf3 2024-12 “Parallelize triangle setup.” · 0b0a30217b 2026-09 “Splines: Tessellate four vertices at a time with CrossSIMD”

### ppsspp-241 — Pre-compose a chain of transforms into one cached matrix
Lens: algorithms · Source: PPSSPP
Principle: If every point goes through a projection, then a viewport scale, then an offset, fold all of them into one matrix once. Recompute that matrix only when its inputs change, not for every use. The per-point cost drops to one matrix multiply, and the setup cost leaves the hot path.
Why it is faster: —
Applies elsewhere: scene graphs caching world matrices with dirty flags, CSS transform composition in browsers, chained affine transforms in image-processing pipelines.
Watch out: —
Seen in PPSSPP: The depth rasterizer merged the viewport scale and translation into the projection matrix with `TranslateAndScaleInplace(m, scale, translate)`. Later, bounding-box culling stopped rebuilding its viewport-adjusted matrix on every test. The cull matrix is now computed in GPUCommon when state changes, and `TestBoundingBoxFast` takes it ready-made. The same change also reads float bounding boxes in place instead of `memcpy`-ing them. 8c069917b5 2024-12 “Depth raster optimizations: Merge viewport into projection matrix” · 189dde8115 2026-05 “Cache the cull matrix. Minor bbox optimization.”

### ppsspp-242 — Make the parallel-for as thin as the work allows
Lens: concurrency · Source: PPSSPP
Principle: General task-splitting helpers pay for range arithmetic, std::function calls and shared queues. When the caller already knows how many slices it wants, put one templated task on each worker directly and wait on a single counter. When there is only one slice, run it inline.
Why it is faster: —
Applies elsewhere: game-engine job systems, per-core sharding in network servers, OpenMP static schedules.
Watch out: —
Seen in PPSSPP: `RunParallel(threadMan, func, count)` calls `func(0, 1)` directly when `count == 1`. Otherwise it enqueues `SimpleParallelTask<T>` with `EnqueueTaskOnThread(i, ...)`, one per slice, and returns a `WaitableCounter`. The lambda type is a template parameter, so the call can be inlined. 3743b5dea6 2024-12 “Add new minimal-overhead RunParallel function”

### ppsspp-243 — Use the narrowest multiply that fits the operand range
Lens: simd · Source: PPSSPP
Principle: Some ISAs have no fast 32-bit lane multiply but do have a fast 16-bit multiply-add. If you know the operands fit in 16 bits, mask one input and use the 16-bit instruction. The masked-out high half contributes zero.
Why it is faster: —
Applies elsewhere: fixed-point DSP filters, pixel blending with 8-bit channels widened to 16, quantized ML inference kernels.
Watch out: —
Seen in PPSSPP: `Vec4S32::MulAsS16` is `_mm_madd_epi16(v, _mm_and_si128(other.v, _mm_set1_epi32(0x0000FFFF)))` on SSE2, which is much faster than an emulated `mullo_epi32`. On NEON it is a full `vmulq_s32`, and a comment warns about that. 8e747dc948 2024-12 “CrossSIMD: Add a multiply-as-16bit function to Vec4S32.”

### ppsspp-244 — Move tiny hot helpers into headers so they inline
Lens: compiler-build · Source: PPSSPP
Principle: A function in another translation unit costs a call, and the compiler cannot optimize across it, unless you use LTO. For functions called per pixel, per instruction or per lookup, put the definition in the header (with its lookup tables exported). The call disappears, and constant arguments fold.
Why it is faster: —
Applies elsewhere: header-only math libraries (glm, Eigen), inline accessors in hot C++ data structures, Rust's `#[inline]` on small cross-crate functions.
Watch out: —
Seen in PPSSPP: `fast_matrix_mul_4x4` moved from a `.c` file into `fast_matrix.h`. The SoftGPU's per-pixel `GERecip` became inline, with `geRecipSegments` made non-static. `IRNativeRegCacheBase::IsValidGPR`/`IsValidFPR`, which the register cache calls constantly, moved into `IRRegCache.h`. 7d987cd78b 2026-01 “Make fast_matrix_mul_4x4 inlineable” · 3b26ba54ba 2026-10 “SoftGPU: Inline GERecip” · dd3496d63d 2026-11 “IR JIT: … inline reg checks”

### ppsspp-245 — Don't use member variables as loop counters
Lens: compiler-build · Source: PPSSPP
Principle: A loop that increments `this->count_` forces a store, and often a reload, on every iteration, because any write through another pointer might alias it. Copy the field into a local before the loop and write it back once afterwards.
Why it is faster: —
Applies elsewhere: buffer writers that append in a loop, parsers that advance `this->pos`, JS classes that keep an index on `this`.
Watch out: —
Seen in PPSSPP: `DrawEngineCommon` now reads `numDrawInds_`, `numDrawVerts_` and `numDecodedVerts_` into `const int` locals, uses them in the decode loop, and stores them back at the end. The commit comments say “Move to a local for better codegen.” 10759dc8b9 2026-01 “DrawEngineCommon: Avoid using member variables as loop counters, bad for perf”

### ppsspp-246 — Mark impossible switch defaults unreachable
Lens: compiler-build · Source: PPSSPP
Principle: A dense switch compiles to a jump table guarded by a range check. If the default case truly cannot happen, tell the compiler with `__builtin_unreachable()` or `__assume(0)`. It then drops the bounds check, and the dispatch is a single indirect jump. Keep a crash in that slot for debug builds.
Why it is faster: —
Applies elsewhere: bytecode VMs, protocol decoders over a closed enum, state machines in embedded firmware.
Watch out: —
Seen in PPSSPP: A new `UNREACHABLE()` macro expands to `Crash()` in debug builds and to the compiler hint in release builds. It replaced the `default:` of the IR interpreter's main switch (“so the compiler can optimize better (remove the range check)”) and some Vulkan queue-runner switches. c3dfddebd7 2025-10 “IR interpreter: Improve code gen for the main interpreter loop”

### ppsspp-247 — Read every source into temporaries before writing any destination
Lens: compiler-build · Source: PPSSPP
Principle: When source and destination are elements of the same array, the compiler must assume the first store may change the next load. Load all inputs into locals first, then do all the stores. It can then keep the values in registers and pair or vectorize the accesses.
Why it is faster: —
Applies elsewhere: in-place matrix and vector ops, color-channel swizzles in image code, register-file emulation in any VM.
Watch out: —
Seen in PPSSPP: IR vector ops such as half-to-float unpacking were changed from `fi[dest] = f(fi[src]); fi[dest+1] = g(fi[src]);` to computing `temp0`/`temp1` (or a `u32 temp[4]`) first, then storing them in a separate loop. 20e3a0cc70 2025-10 “Use temporaries to improve codegen in more IR vector ops”

### ppsspp-248 — Index with unsigned types to avoid sign extension
Lens: compiler-build · Source: PPSSPP
Principle: Small unsigned fields such as `u8` promote to signed `int` in C and C++. Indexing an array with them on a 64-bit target then emits sign-extension instructions. Cast to an unsigned 32-bit type, or declare locals as `u32`, and the address arithmetic gets simpler.
Why it is faster: —
Applies elsewhere: lookup tables indexed by bytes in codecs and hashers, packed instruction fields in interpreters.
Watch out: —
Seen in PPSSPP: The IR interpreter now writes `mips->fi[(u32)inst->src1 + 1]` and declares `const u32 dest = inst->dest` in place of `int`. The commit notes: “u8 promotes to int, causing signed indexing arithmetic which is completely unnecessary.” 78739104b1 2026-02 “Additional micro-optimizations (verified) in the IRInterpreter”

### ppsspp-249 — Batch instruction-cache maintenance
Lens: jit-interpreters · Source: PPSSPP
Principle: Writing code at run time requires cache cleaning and invalidation plus barriers, and the barriers cost the most. Collect the dirty code ranges (a new block plus every patched link to and from it) and flush them all with one set of barriers.
Why it is faster: —
Applies elsewhere: JS engine JITs and hot-patching runtimes on ARM, self-modifying bootloaders, any code loader on non-coherent I/D caches.
Watch out: —
Seen in PPSSPP: `FlushIcacheSections(starts, ends, count)` runs the `dc civac` loop over every range, then the `ic ivau` loop, with one barrier pair. The ARM64 backend queues ranges with `AddPendingIcacheFlush` and flushes them after linking. The commit measured recompiling 11243 blocks going from 363 ms to about 305 ms on a Galaxy S8. Two other changes in the same commit contributed: FlushAll avoids scanning 256 regs, and passes alternate between two persistent buffers. 5c2521729d 2026-11 “IR JIT: Compile faster”

### ppsspp-250 — Compact finished entries so a scan stays linear
Lens: algorithms · Source: PPSSPP
Principle: If every step of a pass walks a list of pending checks, and dead entries stay in that list, the pass becomes quadratic. Remove finished entries with an erase-remove each step, so the walk only touches live ones. When a structure is known to be empty, skip the work that feeds it.
Why it is faster: —
Applies elsewhere: active-timer lists, watcher lists in reactive UIs, live-range tracking in compilers.
Watch out: —
Seen in PPSSPP: In `PurgeTemps`, the old “skip to first live check” scan was replaced by `checks.erase(std::remove_if(..., [](c){ return c.reg == 0; }), checks.end())`. `OptimizeLoadsAfterStores` computes an instruction's destination registers only `if (!known.empty())`. The output did not change. 7079ea8d18 2026-11 “IR: Keep PurgeTemps and OptimizeLoadsAfterStores linear”

### ppsspp-251 — Use open addressing for hot integer-keyed maps
Lens: algorithms · Source: PPSSPP
Principle: Node-based hash maps allocate per entry and chase a pointer on every probe. A flat open-addressed table with linear probing keeps keys and values contiguous, so a lookup is usually one cache line.
Why it is faster: —
Applies elsewhere: symbol tables in compilers, connection tables in load balancers, entity-ID maps in game engines (abseil flat_hash_map, Rust's hashbrown).
Watch out: —
Seen in PPSSPP: The IR native backend's block-by-start-PC map changed from `std::unordered_map<uint32_t,int>` to `DenseHashMap<uint32_t,int> blockByStart_{4096}`, with the comment “Looked up for every exit and link, so open addressing rather than unordered_map's nodes.” dd3496d63d 2026-11 “IR JIT: Find native blocks by start with a DenseHashMap”

### ppsspp-252 — Keep append-heavy buffers allocation-free and reuse their capacity
Lens: allocation · Source: PPSSPP
Principle: If a hot loop appends through a `push_back` the compiler won't inline, each append costs a call. A minimal vector whose append is just a capacity compare and a store fixes that. When a pipeline rewrites a buffer over and over, alternate between two buffers that keep their capacity, so no step allocates a new one.
Why it is faster: —
Applies elsewhere: per-frame command lists in renderers, token buffers in lexers, multi-pass image filters that ping-pong between two scratch images.
Watch out: —
Seen in PPSSPP: `IRWriter` stores `FastVec<IRInst>`, and `Write()` is now inline: `IRInst &inst = insts_.push_uninitialized(); inst.op = op; ...`. In the same series, `IRApplyPasses` stopped allocating a new writer for each pass and alternates between two buffers that keep their capacity. dcc51d8b18 2026-11 “IR: Keep IRWriter's instructions in a FastVec, and inline Write” · 5c2521729d 2026-11 “IR JIT: Compile faster”

### ppsspp-253 — Record a summary while producing data, and skip passes it rules out
Lens: laziness · Source: PPSSPP
Principle: If later passes each act on only a few kinds of element, have the producer OR a bitmask of the kinds it emitted. Each pass checks the mask first and skips itself (including the copy it would make) when nothing applies.
Why it is faster: —
Applies elsewhere: compilers that skip optimization passes for functions with no loops or calls, document pipelines that skip image processing when a page has no images, Bloom-style summaries on data blocks.
Watch out: —
Seen in PPSSPP: A per-op table `irOpKinds[256]` maps each IR op to flags such as `IRKIND_LEFT_RIGHT`, `IRKIND_KERNEL_MEMORY`, `IRKIND_VEC4` and `IRKIND_PLAIN_LOAD`. `IRWriter` accumulates those flags, and `IRApplyPasses` skips a pass when its input has none of what the pass acts on, or when settings disable it. The passes took 15% less time with identical IR. 0535a69f02 2026-11 “IR: Skip the passes a block has nothing for”

### ppsspp-254 — Hash small integer keys with an inline mixer
Lens: algorithms · Source: PPSSPP
Principle: A general-purpose hash function called on a 4-byte key spends most of its time on call and setup overhead. For fixed-width integer keys, use a few inline xor-shift-multiply steps, such as the MurmurHash3 finalizer. It spreads the bits well enough for a power-of-two table at a fraction of the cost.
Why it is faster: —
Applies elsewhere: integer-keyed caches in databases, pointer hash sets in garbage collectors, PID maps in kernels.
Watch out: —
Seen in PPSSPP: `HashKey<uint32_t>` was specialized to `h ^= h >> 16; h *= 0x85EBCA6B; h ^= h >> 13; h *= 0xC2B2AE35; h ^= h >> 16;`. Before, each lookup called XXH3. This map serves JIT block lookups by PC and vertex-decoder lookups by ID. f10158dc71 2026-11 “DenseHashMap: Hash 32-bit keys inline”

### ppsspp-255 — Put a fast path for the dominant case in front of a general allocator routine
Lens: jit-interpreters · Source: PPSSPP
Principle: A general routine handles every case, but profiling often shows one simple case dominating. Check for that case first, do the small amount of work it needs, and return before the general logic runs.
Why it is faster: —
Applies elsewhere: malloc thread-local fast bins, JSON parsers with a plain-ASCII string path, HTTP routers that check exact-match routes before patterns.
Watch out: —
Seen in PPSSPP: In the IR register cache's `MapNativeReg`, most operands are GPRs that already sit alone in a native register. For those, the routine now just marks the register dirty (`nr[nreg].isDirty = true; pointerified = false; normalized32 = false;`) and returns. MapNativeReg went from 12.6% to 8.8% of ARM64 backend time, with the same generated code. 609c84bf42 2026-11 “IR regcache: Map a GPR that's already alone in a reg without the general case”

### ppsspp-256 — Track the few items that need work instead of scanning all of them
Lens: algorithms · Source: PPSSPP
Principle: If a flush or commit loop visits every slot but only a few are dirty, keep a bitset of candidates, filled where they become dirty, and iterate only that set in the original order. When repeated dead-entry cleanup has already run, return early.
Why it is faster: —
Applies elsewhere: dirty-page tracking in databases, dirty-rect lists in UI compositors, changed-field sets in ORMs.
Watch out: —
Seen in PPSSPP: At each block exit, `FlushAll` used to loop over all GPRs and special regs (on ARM64, every pair of the first 63). It now builds `NotInMemoryRegs` once from the native regs plus an `immRegs_` set, and walks `regs.ForEach(1, 63, ...)`. The set alone cut about 5% of ARM64 backend time. 0a6e23d86a 2026-11 “IR regcache: At an exit, only look at the regs that may need flushing”

### ppsspp-257 — Read trapping system registers once
Lens: platform · Source: PPSSPP
Principle: On some kernels, reading a CPU identification register from user space traps and is emulated, which costs on the order of a microsecond. Read it once into a static and clamp it to a safe value, rather than reading it on every call.
Why it is faster: —
Applies elsewhere: caching CPUID results, caching `sysconf`/page-size queries, caching cache-line sizes in allocators.
Watch out: —
Seen in PPSSPP: The ARM64 icache flush now uses `static const u64 ctr_el0 = []{ mrs ... }();` and `isize = std::min<size_t>(64, 4 << (ctr_el0 & 0xf))`. The read costs about 0.45 µs on a Galaxy S8, and the JIT flushes after every block. The 64-byte cap covers big.LITTLE cores that may have smaller lines. 371a7d0af4 2026-11 “ARM64: Read the cache line sizes once, not on every icache flush”

### ppsspp-258 — Re-validate cached results by hash only when the invalidation is ambiguous
Lens: laziness · Source: PPSSPP
Principle: Store a cheap hash of the source each cached artifact was derived from. When the invalidation signal might cover less than what actually changed, rehash and drop stale entries. When the signal is exact (you wrote the data yourself), invalidate only the stated range and skip the expensive check.
Why it is faster: —
Applies elsewhere: build systems that check content hashes only when timestamps are unreliable, CDN revalidation with ETags, template caches in web frameworks.
Watch out: —
Seen in PPSSPP: JIT blocks now always store a code hash, and `InvalidateChangedBlocks()` rehashes all blocks on an invalidation of at least the icache size. A follow-up added a `fromGame` flag to `InvalidateICacheRangeDeferred`, so the check runs only on the game's own large invalidations. The emulator's own invalidations after file reads and DMA cover exactly what was written. Streaming games issue those every second or so. c22324d80d 2026-11 “JIT: Drop blocks whose code changed on a large icache invalidation” · ac8a9c50e9 2026-11 “JIT: Only check for changed code on the game's own large icache invalidations”

### ppsspp-259 — Index intervals in a sorted array, with outliers kept apart
Lens: algorithms · Source: PPSSPP
Principle: An overlap query over intervals sorted by start must look back by the longest interval's length. A few huge intervals make that window enormous for everyone. Put the rare large intervals in a short list of their own, keep the rest in a sorted flat array, and binary-search back only by the small-interval bound.
Why it is faster: —
Applies elsewhere: memory-region lookups in allocators and debuggers, overlapping-event queries in calendars, genomic range queries.
Watch out: —
Seen in PPSSPP: Texture invalidation on block transfers used to walk the `cache_` map starting 1 MB before the address. It now uses `texRanges_` (sorted, textures up to 64 KB) and `bigTexRanges_`, with a `std::lower_bound` from `addr - BIG_TEX_RANGE`. Both are kept in sync wherever entries enter or leave the cache or change `bufw`. In MX vs. ATV, samples in `Invalidate()` fell from 889 to 27. 3863e2f90f 2026-11 “TextureCache: Find the textures a block transfer overlaps through a flat index”

### ppsspp-260 — Multiply by lane to keep a matrix in four registers
Lens: simd · Source: PPSSPP
Principle: Splatting every matrix element into its own vector needs 16 registers for a 4x4 matrix, more than many targets can keep live, so the compiler spills and reloads every iteration. Keep the four columns as vectors and use multiply-by-lane (or a shuffle) to broadcast on the fly.
Why it is faster: —
Applies elsewhere: batched vertex transforms in game engines, 4x4 color-matrix filters, any SIMD loop running into register pressure.
Watch out: —
Seen in PPSSPP: `TestBoundingBoxFast` replaced `Vec4F32 m[16]` of splats with `col0..col2` plus four splatted translations, computing `objX.MulLane<0>(col0) + ...`. `MulLane` maps to `vmulq_laneq_f32` on ARM64 and to a shuffle plus multiply on SSE. This gave about 14% fewer samples on a Cortex-A53, where it is the top C++ function on the emulation thread. 5f50174440 2026-11 “GPU: Keep the cull matrix in four registers in TestBoundingBoxFast”

### ppsspp-261 — Make the conflict test exact before waiting on a worker
Lens: concurrency · Source: PPSSPP
Principle: A producer that waits on a background worker whenever an address "might" conflict can lose a large share of its time to false positives. Rule out impossible overlaps first, for example a different memory region, so it waits only when a conflict is real.
Why it is faster: —
Applies elsewhere: range locks in file systems, hazard checks in GPU command schedulers, write-barrier filters in garbage collectors.
Watch out: —
Seen in PPSSPP: `WaitIfWriting` masked addresses into VRAM offsets, so RAM addresses aliased onto the depth buffer. It now returns immediately when `!busy_ || !Memory::IsVRAMAddress(addr)`. Ten emulated seconds of Midnight Club took 24.0 s instead of 39.6 s on a Galaxy A21. 661e9a80f7 2026-11 “Depth raster: Don't wait for the worker on RAM addresses”

### ppsspp-262 — Specialize generated shaders to drop unused features entirely
Lens: gpu-rendering · Source: PPSSPP
Principle: When you generate shader source at run time, gate every optional stage on a feature flag computed once from device and settings. Unused math and uniforms then never reach the GPU, and the uniform-upload code skips them too.
Why it is faster: —
Applies elsewhere: uber-shader permutation systems in game engines, WebGL material systems with #define variants, query compilers that emit only the predicates in use.
Watch out: —
Seen in PPSSPP: A new `GPU_USE_PRE_ROTATION` flag is set only if `vulkan->SupportsPreRotation() && g_Config.bSkipBufferEffects`. The vertex shader generator emits the rotation, and `ShaderUniforms` fills it, only when `gstate_c.Use(GPU_USE_PRE_ROTATION)` is set. 4b93a870db 2026-05 “Optimize out pre-rotation entirely from vertex shaders when not needed”

### ppsspp-263 — Hoist constant scale factors out of accumulations
Lens: algorithms · Source: PPSSPP
Principle: If every term of a weighted sum is multiplied by the same constant, accumulate the unscaled terms and multiply once at the end. Hoist the loop's stride and other invariant fields into locals while you're at it.
Why it is faster: —
Applies elsewhere: normalization in audio mixing, fixed-point to float conversion in sensor fusion, averaging in reductions.
Watch out: —
Seen in PPSSPP: The non-JIT morph decoders now do `uv[0] += (float)uvdata[0] * w;` in the loop and `out[0] = uv[0] * (1.f / 128.f);` after it, with `const int onesize = dec->onesize_` hoisted. New `LoadConvertU8` helpers widen 4 bytes to a float vector in one step. 3378faca5e 2026-01 “Optimize morph code (non-JIT vertex decoders)”

### ppsspp-264 — Re-benchmark homegrown hashes against modern libraries
Lens: algorithms · Source: PPSSPP
Principle: A custom "quick" hash that once beat the libraries may now be both slower and weaker than a current general-purpose one. Measure throughput on the target hardware, and switch when the library wins on speed and collision quality.
Why it is faster: —
Applies elsewhere: content-addressed caches, dedup in backup tools, ETag generation in web servers.
Watch out: —
Seen in PPSSPP: Texture-data hashing switched from `StableQuickTexHash` to `(u32)XXH3_64bits(checkp, sizeInRAM)`: about 37 vs 24 GB/s on a Snapdragon X, about as fast on the scalar RISC-V path, and without the old hash's collisions. Texture replacement keeps its own hash setting, so texture packs are not affected. 7de7b2d76b 2026-09 “TextureCache: Use XXH3 instead of the quick hash for texture data”

### ppsspp-265 — Use an epoch counter to validate a cache once per sync period
Lens: laziness · Source: PPSSPP
Principle: Data can only change at known synchronization points. Keep a global epoch that increments at each of those points, and stamp each cached item with the epoch in which it was last validated. Within the same epoch, skip revalidation entirely.
Why it is faster: —
Applies elsewhere: per-request memoization in web servers, generation counters in ECS change detection, RCU grace periods.
Watch out: —
Seen in PPSSPP: `sceGe` sync points do `gstate_c.curSyncDomain++`. In the texture cache, `if (entry->lastSyncDomain == gstate_c.curSyncDomain) rehash = false;`. This replaced the old minihash and the `invalidHint` heuristics. 1781f693b5 2026-06 “Implement the new SyncDomain concept for texture hashing, remove minihash”

### ppsspp-266 — Separate producer and consumer fields onto their own cache lines
Lens: concurrency · Source: PPSSPP
Principle: If a ring buffer's head (written by consumers) and tail (written by the producer) share a cache line, every push and pop bounces that line between cores. Align them apart. Idle consumers should also spin on one "new work" counter rather than polling every queue's indices.
Why it is faster: —
Applies elsewhere: SPSC queues in audio and trading systems, Disruptor-style rings, per-CPU counters in kernels.
Watch out: —
Seen in PPSSPP: In the SoftGPU `BinManager`, the tile queue now uses `alignas(64) std::atomic<uint32_t> head; alignas(64) std::atomic<uint32_t> tail;`. Waiting threads watch `distributed_`, which `DistributeItems` bumps with release order, and look at the tiles only when it changes. ce83a7d366 2026-10 “SoftGPU: Keep a tile's tail off the drawing threads' cache line”

### ppsspp-267 — Re-tune work granularity and spin time per platform
Lens: concurrency · Source: PPSSPP
Principle: The best tile size depends on the ratio of per-item setup cost to per-element cost, so it changes whenever either side gets faster. Spin-waiting before sleeping pays off only where waking a thread is expensive. Measure wall and CPU time on several workloads, and keep the constants per platform with the reasoning written next to them.
Why it is faster: —
Applies elsewhere: chunk sizes in parallel map-reduce, adaptive spinning in mutexes, batch sizes in GPU dispatch.
Watch out: —
Seen in PPSSPP: Once span drawing got faster, the SoftGPU tile rule went from 16 to 8 tiles per thread, which gave 1.7–5.6% less wall time across five games. Linger times became per platform: `DRAW_LINGER_SECONDS = 0.0` and `TRANSFORM_LINGER_SECONDS = 100e-6` on Windows (9–27% less CPU at equal wall time), 200/300 µs on Linux, and 0 on macOS and Android. 8a7a67ae25 2026-10 “SoftGPU: Use larger tiles now that pixels are cheaper” · 6245285ca7 2026-10 “SoftGPU: Don't linger in drawing threads on Windows”

### ppsspp-268 — Route to the vectorized general path when the scalar fast path won't apply
Lens: branching-fast-paths · Source: PPSSPP
Principle: A "fast" special-case routine is often fast only for a narrow set of states and falls back to slow per-element work otherwise. Choose between the paths up front: take the special case only when its fast inner loop applies, and send everything else to the general routine that processes several elements per step.
Why it is faster: —
Applies elsewhere: memcpy implementations picking the vector path by size and alignment, image libraries choosing SIMD resamplers by format.
Watch out: —
Seen in PPSSPP: 1:1 sprites now go to `DrawRectangle`, which draws four pixels in vector lanes, unless `UseDrawSinglePixel(state.pixelID)` holds and the texture function is a fast one. This cut instruction counts 4–5% in Tekken 6 and LocoRoco. e415e667d0 2026-10 “SoftGPU: Draw 1:1 sprites as rectangles unless DrawSprite has a fast path for them”

### ppsspp-269 — Generate output directly in its final format
Lens: laziness · Source: PPSSPP
Principle: If a stage writes data in an intermediate format that the next stage just converts again, have the first stage write the final layout into the final buffer. Mark the data as already converted, so the second pass does at most a copy.
Why it is faster: —
Applies elsewhere: decoders that write straight into GPU-mapped buffers, serializers writing directly into socket buffers, ETL steps emitting the warehouse's columnar format.
Watch out: —
Seen in PPSSPP: Spline and Bezier tessellation used to queue `SimpleVertex` output for a second run through the vertex decoder. The tessellator now applies the UV scale/offset and the full-alpha tracking itself and writes into `decoded_`. `DeferredVerts::predecoded` makes `DecodeVerts` only `memcpy`, and only when the backend decodes somewhere else. e94d6f9bd2 2026-09 “Splines: Skip the second vertex decode of tessellated curves”

### ppsspp-270 — Record commands on one thread, encode them on another, and merge passes first
Lens: concurrency · Source: PPSSPP
Principle: Keep all decisions (state deduplication, lazy pass starts) on the producer thread, which writes a compact command list. A second thread only replays it into the expensive API. Before replay, merge passes to the same target across independent work, so there are fewer expensive pass switches.
Why it is faster: —
Applies elsewhere: render-graph pass merging in engines, browser display-list recording versus rasterization threads, write-ahead logs replayed by a separate applier.
Watch out: —
Seen in PPSSPP: The Metal backend now records `MetalCmd` entries (BEGIN_PASS, PIPELINE, ...) on the emulation thread and encodes them on a render thread, or inline when `bRenderMultiThreading` is off. Passes are merged by the rules of `VulkanQueueRunner`. God of War went from about 41 to 18 render passes per frame, which more than halved its GPU time. 9514e02037 2026-10 “Metal: Encode on a render thread, and merge render passes”

### ppsspp-271 — Compile behind a future and persist the keys for a warm start
Lens: concurrency · Source: PPSSPP
Principle: Hand out a handle to a pipeline that may still be compiling, and block only the consumer that actually needs it, never the producer. Save the set of keys you built, so the next run can recreate them all during load instead of stalling mid-session.
Why it is faster: —
Applies elsewhere: shader precompilation in game launchers, regex or template compilation caches warmed at server start, ahead-of-time warmup lists in JVM and .NET runtimes.
Watch out: —
Seen in PPSSPP: `MetalPipelineState` is created on compile threads. Draws record the handle, and the render thread calls `-wait`. Shader IDs and pipeline keys are saved per game in `<discID>.mtlshadercache` and recreated at boot. A cold 5-second Wipeout Pure run took 1.6 s instead of 3.0 s. 51d775cb97 2026-10 “Metal: Compile shaders and pipelines in the background, and cache them”

### ppsspp-272 — Check whether a log line is enabled inline, before formatting it
Lens: laziness · Source: PPSSPP
Principle: A disabled log call should cost one load and a compare at the call site. Put the per-channel level table in a global the macro can read, and test it before calling the variadic formatter. Point the global "logging enabled" setting at a dummy true value, so the check never needs a null test.
Why it is faster: —
Applies elsewhere: `if (log.isDebugEnabled())` guards in Java, compile-time and runtime level filters in spdlog and tracing crates, kernel dynamic debug.
Watch out: —
Seen in PPSSPP: `GENERIC_LOG` became `if ((int)v <= MAX_LOGLEVEL && GenericLogEnabled(t, v)) GenericLog(...)`, where `GenericLogEnabled` is inline over `g_log[type].IsEnabled(level) && *g_bLogEnabledSetting`. An earlier commit made `g_bLogEnabledSetting` default to `&g_bDummySetting`. 8e3df92aff 2025-03 “Avoid formatting log messages in more cases.” · 0ecd72416b 2025-03 “Reduce overhead of logging further.”

### ppsspp-273 — Iterate parsed data in place instead of materializing copies
Lens: allocation · Source: PPSSPP
Principle: Converting a parsed section into a map of owned strings, or taking `const std::string&` keys, forces an allocation per item even when the caller only reads. Expose the parsed lines as views and take `string_view` keys, so lookups and scans don't allocate.
Why it is faster: —
Applies elsewhere: zero-copy JSON and CSV readers, HTTP header parsing over the receive buffer, heterogeneous lookup in C++ maps.
Watch out: —
Seen in PPSSPP: Texture replacement used to build `ToMap()` of the `[hashes]` section. It now iterates `hashesSection->Lines()`, copying each key into a stack `char k[128]` for `sscanf`. `IconCache`'s API changed to `std::string_view key` throughout. 176edb93a5 2025-04 “IniFile: Add faster way of reading everything in a section” · 419ab46172 2026-01 “Avoid needing std::string as cache keys in IconCache”

### ppsspp-274 — Parallelize bulk copies only above a size threshold, with a thread cap
Lens: concurrency · Source: PPSSPP
Principle: Splitting a memcpy across threads only pays when the copy is large enough to cover the dispatch cost, and memory bandwidth saturates after a few cores. Set a high minimum size and block size, and cap the thread count.
Why it is faster: —
Applies elsewhere: parallel buffer fills in image editors, multi-threaded checksum or compression of large files, NUMA-aware page zeroing.
Watch out: —
Seen in PPSSPP: `ParallelMemcpy` now copies inline below `threshold = 1024 * 1024` and uses `blocksize = 512 * 1024`, raised from 128 KB. `ParallelRangeLoop` gained a `maxThreads` argument, which the memcpy sets to 4 (“more than that is probably rarely useful”). e59aa11987 2026-08 “Raise the limits for when we use parallelism in ParallelMemcpy” · 831acb099e 2023-12 “Allow specifying a cap on the number of threads used in ParallelLoop”

### ppsspp-275 — Work in machine words, and defer expensive inversions
Lens: algorithms · Source: PPSSPP
Principle: Big-number code that works byte by byte wastes most of the ALU. Use 32- or 64-bit limbs. For elliptic curves, keep points in projective (Jacobian) coordinates so additions avoid modular inversion, invert once at the end, and compute u1·G + u2·Q in a single joint pass.
Why it is faster: —
Applies elsewhere: TLS signature verification, embedded secure-boot ROMs, cryptocurrency wallet libraries.
Watch out: —
Seen in PPSSPP: The KIRK ECDSA code in `ext/libkirk/bn.c` was rewritten on `u32` words (with `bn_from_bytes`/`bn_to_bytes` at the edges), Jacobian points and a joint scalar multiply. A verify went from 55 ms to 79 µs, and a 30-second XMB boot from 2.96 s to 0.79 s. Results are unchanged. 1a0cc62aa8 2026-11 “KIRK: Make the ECDSA math about 700 times faster”

### ppsspp-276 — Shape coefficient vectors so results land in storage order
Lens: simd · Source: PPSSPP
Principle: Shuffles can easily cost more than the arithmetic they feed. Build the filter's coefficient vectors so that one unaligned load times one coefficient vector yields output lanes already in their interleaved storage order. Even an extra tap is cheaper than shuffling every sample.
Why it is faster: —
Applies elsewhere: stereo resamplers in audio engines, YUV/RGB interleaving in video codecs, polyphase FIR filters in SDR software.
Watch out: —
Seen in PPSSPP: The SAS reverb upsampler builds `UpsampleTable` entries `(a, a, b, b)`. Each tap is a multiply-add on an unaligned load of `(Lf, Rf, Lf+1, Rf+1)` from a float history, producing even and odd output frames for both channels together. The comment says it “costs one tap more than using the symmetry, and saves every shuffle.” 621c2a7e21 2026-11 “sceSas: Run the reverb upsampling filter with CrossSIMD”

### ppsspp-277 — Cache expensive layout results, and look them up without allocating
Lens: laziness · Source: PPSSPP
Principle: Word wrapping measures every prefix, so re-running it every frame is wasteful. Cache the result keyed by text, style, width and scale. Reuse one key object for lookups so a cache hit doesn't allocate, and hash the key with a fast seeded hash.
Why it is faster: —
Applies elsewhere: text-shaping caches in browsers and terminal emulators, layout caches in UI toolkits, memoized markdown rendering.
Watch out: —
Seen in PPSSPP: `TextDrawer::WrapString` now uses `wrapCache_.try_emplace(LookupKey(str, flags, maxW, fontScaleX_))`, where `LookupKey` refills a member `lookupKey_`. `TextCacheKeyHash` seeds `XXH3_64bits_withSeed` with the style bits. Ten wrapped paragraphs went from 2.1 ms to 0.01 ms per frame. 02b688e4f3 2026-10 “TextDrawer: Cache wrapped text, hash the string caches”

### ppsspp-278 — Limit incremental work to what actually changed
Lens: algorithms · Source: PPSSPP
Principle: When new input arrives, redo analysis only for the affected range instead of rebuilding global indexes. Pass the changed range down, look each new item up directly, and load static tables once.
Why it is faster: —
Applies elsewhere: incremental compilers and language servers, partial reindexing in search engines, React-style reconciliation.
Watch out: —
Seen in PPSSPP: `MIPSAnalyst::FinalizeScan(insertSymbols, textStart, textEnd)` hashes and names only the module just loaded, instead of rebuilding a `hashToFunction` multimap of every function (Tekken 6's module load went from 67 ms to 13–23 ms). Savestate loads now call `RehashFunctions(g_unchangedCodeOnLoad)`, which skips functions inside code ranges the load left unchanged (2.4–6 ms down to 0.5–1 ms). 420db34118 2026-11 “MIPSAnalyst: Hash and name only the functions of the module just loaded” · f45f4d5c85 2026-10 “SaveState: Only rehash functions in code a load changed”

### ppsspp-279 — Give generated code SIMD helper routines, not integer emulation
Lens: jit-interpreters · Source: PPSSPP
Principle: If a JIT calls an out-of-line helper for a hard operation, write that helper with the host's vector unit. Choose a calling convention that clobbers only registers the JIT already keeps free, so call sites need no save/restore.
Why it is faster: —
Applies elsewhere: runtime stubs in JavaScript and Wasm engines, intrinsic fallbacks in shader compilers, math helpers in database expression JITs.
Watch out: —
Seen in PPSSPP: The ARM64 VFPU dot-product routine was integer code (“about four times the work”). It is now `vfpu_dot_neon()`, scheduled after clang's output. It clobbers `q0-q3`, `x16`/`x17`, `x30` and the flags, which both ARM64 JITs keep free of guest state. x86-64 and ARM32 got SSE2 and NEON versions in sibling commits. 3e1198bf22 2026-10 “ARM64 JITs: Make the VFPU dot routine NEON, after clang's vfpu_dot_neon”

### ppsspp-280 — Compile or skip operations that have no observable effect
Lens: laziness · Source: PPSSPP
Principle: An operation the emulator gives no effect still costs a slow fallback if the fast path doesn't recognize it. Make the translator emit nothing for it. In the same way, remove side effects such as cache invalidation from paths that only read.
Why it is faster: —
Applies elsewhere: dead-store elimination, no-op syscall short-circuits in sandboxes, skipping cache busts on GET requests.
Watch out: —
Seen in PPSSPP: The IR frontend sent unknown `cache` ops to the interpreter one at a time. Now everything except `func == 8` (icache invalidate) compiles to nothing, and a Media Engine flush loop benchmark went from 15.9 s to 1.0 s. Separately, CwCheat read operations stopped calling `InvalidateICache`. c5d2adb303 2026-10 “IR: Compile data cache ops to nothing” · b6fde72f52 2025-01 “CwCheat: Avoid instruction cache invalidations on reads. Not needed.”

### ppsspp-281 — Don't hash what you know changes every time, and dedupe linear-scan lists
Lens: laziness · Source: PPSSPP
Principle: Change detection on data that is new by definition wastes work and pollutes secondary caches. Skip it. Also keep any list that is scanned linearly free of duplicates, by refreshing an existing entry instead of appending a new one.
Why it is faster: —
Applies elsewhere: skipping ETag computation for streaming responses, not memoizing per-request random values, upserting rather than appending into small registries.
Watch out: —
Seen in PPSSPP: For video textures, `skipHash = isVideo && !(replacer_.Enabled() && replacer_.AllowVideo())` skips `ComputeTextureHash` and the secondary-cache lookup. `NotifyWriteFormattedFromMemory` now refreshes a matching `videos_` entry instead of pushing a new one. Tekken 6 went from about 53,000 entries to 18, and `IsVideo()` scans that list for every texture. 14bfd332dc 2026-09 “Stop stacking duplicate video entries, and stop hashing video textures”

### ppsspp-282 — Threaded dispatch with computed goto
Lens: branching-fast-paths · Source: PPSSPP
Principle: A switch-based interpreter funnels every opcode through one indirect jump, which the branch predictor handles poorly. With computed goto, each handler ends with its own `goto *table[next->op]`, so the predictor learns per-opcode successor patterns. Keep the switch as a portable fallback.
Why it is faster: —
Applies elsewhere: CPython's and Lua's bytecode loops, regex VMs, packet-filter interpreters.
Watch out: —
Seen in PPSSPP: Under GCC and Clang, `IR_NEXT` is `inst++; goto *dispatch[(int)inst->op];` over a static `dispatch[256]`. Ops missing from the table fall back to `&&L_switch`. A comment warns that Clang keeps one jump per op only while the shared tail stays small: a single merged jump is 10–15% slower, so the advice is to count `br`s in the disassembly after changing it. current code: Core/MIPS/IR/IRInterpreter.cpp

### ppsspp-283 — Chain directly into the next compiled block inside the interpreter
Lens: branching-fast-paths · Source: PPSSPP
Principle: Returning to an outer loop after every basic block costs a call, a lookup and a return. If the next block is already compiled and the time budget remains, jump straight into it from the exit handler.
Why it is faster: —
Applies elsewhere: trace linking in JITs, superblock chaining in QEMU's TCG, tail-call dispatch in Wasm interpreters.
Watch out: —
Seen in PPSSPP: `IR_EXIT(pc)` returns only if `mips->downcount < 0`, if the address is invalid, or if the block-shadow entry is empty. Otherwise it sets `inst = chainArena + offset` and dispatches again, “the way IRJit::RunBlocks would after we returned (minus the call and the lookup there).” current code: Core/MIPS/IR/IRInterpreter.cpp

### ppsspp-284 — Shadow array indexed by address for block lookup
Lens: memory-layout · Source: PPSSPP
Principle: Mirror the guest address space with a parallel array of block handles, one slot per instruction. Finding the compiled block for a PC is then a mask and a load, with no hashing, and unmapped ranges are simply never committed.
Why it is faster: —
Applies elsewhere: page tables, shadow memory in sanitizers, per-address metadata in garbage collectors (card tables).
Watch out: —
Seen in PPSSPP: `GetBlockShadowEntry(address)` returns `(u32 *)(blockShadow + (address & BLOCK_SHADOW_ENTRY_MASK))`. The mask folds kernel and uncached mirrors together. The ARM64 dispatcher does `ANDI2R`, `LDR`, `CBZ`, then `ADD JITBASEREG` and `BR`. An entry of zero means not compiled. current code: Core/MemMap.h, Core/MIPS/ARM64/Arm64IRAsm.cpp

### ppsspp-285 — Flat host mapping of guest memory with a fault-handler safety net
Lens: memory-layout · Source: PPSSPP
Principle: Reserve one host region laid out like the guest address space, so a guest access is `base + addr` with no bounds checks or region switch. Mask the address only on hosts that can't reserve the full range. A signal handler catches the rare invalid access and recovers.
Why it is faster: —
Applies elsewhere: Wasm runtimes using guard pages instead of bounds checks, memory-mapped databases, user-space network stacks with mmap'd rings.
Watch out: —
Seen in PPSSPP: `ReadUnchecked_U32` is `*(u32_le *)(base + address)`, or `base + (address & MEMVIEW32_MASK)` under `MASKED_PSP_MEMORY` (32-bit, UWP, ASAN, iOS). JITs keep `MEMBASEREG` (X28) pinned. `Memory::HandleFault` recognizes faults inside the PSP range and in the block-shadow lookup. current code: Core/MemMap.h, Core/MemFault.cpp, Core/MIPS/ARM64/Arm64IRRegCache.h

### ppsspp-286 — A final pass that rewrites code into interpreter-friendly variants
Lens: compiler-build · Source: PPSSPP
Principle: Run a last lowering pass that specializes common shapes into cheaper opcodes, such as a two-operand form when the destination equals the source. Move per-block bookkeeping to a fixed position so the interpreter can assume where it is.
Why it is faster: —
Applies elsewhere: superinstructions and quickening in the JVM and CPython 3.11+, peephole passes before bytecode emission.
Watch out: —
Seen in PPSSPP: `OptimizeForInterpreter` writes a leading `IROp::Downcount` and folds the block's downcount constant into it. It turns `AddConst` with `src1 == dest` into `OptAddConst`, and does the same for `AndConst` and others. `MergeLoadStore` combines adjacent byte stores at sequential offsets into wider ones. current code: Core/MIPS/IR/IRPassSimplify.cpp

### ppsspp-287 — Generate the decoder as nested switches from the opcode tables
Lens: branching-fast-paths · Source: PPSSPP
Principle: Table-driven decoding chases a pointer per level. Generating source code from the same tables produces nested switches on opcode fields that the compiler turns into jump tables with direct calls, and a regeneration step keeps the code in sync with the tables.
Why it is faster: —
Applies elsewhere: generated instruction decoders in disassemblers (LLVM TableGen), protocol parsers generated from schemas, lexers generated by re2c.
Watch out: —
Seen in PPSSPP: `InterpreterDispatch.cpp` is “AUTO-GENERATED by `PPSSPPHeadless --generate-interpreter-dispatch` from the tables in MIPSTables.cpp”. `ExecInstruction` does `switch ((op.encoding >> 26) & 0x3f)` and then sub-switches, calling e.g. `MIPSInt::Int_ShiftType(mips, op); return 1;`. current code: Core/MIPS/InterpreterDispatch.cpp

### ppsspp-288 — Hand-specialized decoders for the most common formats
Lens: algorithms · Source: PPSSPP
Principle: A generic decoder built from per-attribute step functions pays an indirect call per attribute per element. For the handful of formats that dominate real workloads, provide fully specialized routines and select them once when the decoder is created. Everything else keeps the generic path.
Why it is faster: —
Applies elsewhere: image loaders with fast paths for RGBA8, protobuf parsers specialized per message, database scan kernels per column type.
Watch out: —
Seen in PPSSPP: When the decoder is set up, `vtype == (TC_8BIT | COL_5551 | POS_16BIT)` sets `jitted_ = &VtxDec_Tu8_C5551_Ps16` (or the `_NoPrescale` variant), skipping the JIT entirely. Other formats use the JIT or the `StepFunction` tables (`tcstep_prescale[4]`, `colstep[8]`, …), which `DecodeVerts` loops over per vertex. current code: GPU/Common/VertexDecoderCommon.cpp

### ppsspp-289 — Pick a template-specialized function from a state ID
Lens: compiler-build · Source: PPSSPP
Principle: Pack the pipeline state that matters into a compact ID, then select a function instantiated for those constants once per state change. The per-pixel loop calls through one pointer with all format branches compiled away.
Why it is faster: —
Applies elsewhere: blitters in 2D graphics libraries (Skia's specialized pipelines), codec kernels chosen per bit depth, SQL operators templated on column types.
Watch out: —
Seen in PPSSPP: `GetSpanFunc(id)` switches on `id.fbFormat` to `PickSpanFunc<GE_FORMAT_565>` and the other formats. `GetSingleFunc` returns `&DrawSinglePixel<clearMode, fmt>`. The sampler is templated on `<GETextureFormat, swizzled, GEPaletteFormat>`. Values the ID doesn't hash, such as strides and fog color, sit in a `cached` sub-struct. current code: GPU/Software/DrawPixel.cpp, GPU/Software/Sampler.cpp, GPU/Software/FuncId.h

### ppsspp-290 — Dirty bits per uniform group
Lens: laziness · Source: PPSSPP
Principle: Track which state groups changed since the last upload in a 64-bit mask, and rebuild only those parts of the constant buffer.
Why it is faster: —
Applies elsewhere: React's changed-props checks, dirty flags in ECS systems, partial config reloads.
Watch out: —
Seen in PPSSPP: `BaseUpdateUniforms(ub, dirtyUniforms, ...)` is a sequence of `if (dirtyUniforms & DIRTY_TEXENV) {...}`, `DIRTY_ALPHACOLORREF`, `DIRTY_FOGCOLOR` and so on, each filling only its own fields. current code: GPU/Common/ShaderUniforms.cpp

### ppsspp-291 — Hand tasks to idle workers' private queues, with separate compute and I/O pools
Lens: concurrency · Source: PPSSPP
Principle: Schedule onto a worker whose queue is empty, under that worker's own lock, and fall back to a shared queue only when every worker is busy. Keep blocking I/O on a separate pool so it can't starve CPU-bound tasks.
Why it is faster: —
Applies elsewhere: Tokio's blocking pool, Java's ForkJoinPool combined with a cached I/O pool, Go's netpoller apart from CPU workers.
Watch out: —
Seen in PPSSPP: `EnqueueTask` scans `[minThread, maxThread)` for `thread->queue_size.load() == 0` and pushes to that thread's `private_queue[priority]`. Otherwise the task goes to `compute_queue`/`io_queue`, and one thread is notified round-robin. The pool has `min(32, cores)` compute threads, and the I/O pool is twice that size. current code: Common/Thread/ThreadManager.cpp

### ppsspp-292 — Pin the hottest guest registers to host registers
Lens: jit-interpreters · Source: PPSSPP
Principle: Statically allocate the most-used guest registers to callee-saved host registers for the whole time JIT code runs. They never need loads or spills at block boundaries, and calls into C don't clobber them.
Why it is faster: —
Applies elsewhere: pinned context and heap-base registers in Wasm and JS JITs, global register variables in Forth and Lua VMs.
Watch out: —
Seen in PPSSPP: `Arm64IRRegCache::GetStaticAllocations` maps `SP→W19`, `V0→W20`, `V1→W21`, `A0→W22`, `A1→W23` and `RA→W24` when `useStaticAlloc` is set. `JITBASEREG`, `CTXREG` and `MEMBASEREG` are fixed at X26–X28. current code: Core/MIPS/ARM64/Arm64IRRegCache.cpp, Core/MIPS/ARM64/Arm64IRRegCache.h

### ppsspp-293 — Pack color channels with multiply-add instead of shifts
Lens: simd · Source: PPSSPP
Principle: To narrow packed channels, mask them, then use a 16-bit multiply-add whose constants shift and combine two fields in one instruction. Process two vectors per iteration and saturate-pack them into one output. A scalar loop handles the tail.
Why it is faster: —
Applies elsewhere: RGBA to RGB565 conversion in display drivers, YUV packing in camera pipelines, pixel-format conversion in browsers.
Watch out: —
Seen in PPSSPP: `ConvertRGBA8888ToRGBA5551` (SSE2) masks with `0x00F800F8` and `0x8000F800`, then `_mm_madd_epi16` with `0x04000001`/`0x00400001`, then `_mm_packs_epi32` on two inputs. `ConvertBGRA8888ToRGBA8888` swaps R and B with an and-not, shifts and an or, and uses a vector path only when both pointers are 16-byte aligned. current code: Common/Data/Convert/ColorConv.cpp
