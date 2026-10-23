# WebGPU capabilities over WebGL2

Each entry: **capability** — kind · what it saves. How it helps. *WebGL2:* the closest equivalent.

Kinds: **only** = core WebGPU, no general WebGL2 equivalent. **optional** = device feature, WGSL language feature or newer API; detect it ([feature-detection.md](feature-detection.md)). **cleaner** = WebGL2 can do something similar; WebGPU does it with less overhead or code.

## Contents

1. Compute and workgroup memory
2. Storage memory and atomics
3. GPU-driven work
4. CPU cost per draw and API state
5. Uploads, copies and readback
6. Render passes and output
7. WGSL
8. Formats and optional features
9. Debugging and robustness

## 1. Compute and workgroup memory

- **Compute pipelines** — only · GPU, CPU. Dispatch any grid of invocations with no rasterizer, viewport or framebuffer; results land in buffers or textures later passes consume. Moves CPU preprocessing to the GPU and lets repeated vertex work run once (skinning reused by depth, shadow and colour passes). Compute adds dispatch, temporary buffers and synchronization; compare whole pipelines, including the consumer. *WebGL2:* fragment passes over a texture, transform feedback; one output slot per invocation.
- **Workgroup memory** — only · bandwidth. `var<workgroup>` (16 KiB default `maxComputeWorkgroupStorageSize`, up to 256 invocations) shares a tile loaded once: blurs, convolutions, reductions, depth min/max, tiled matrix multiply, neighbourhood stencils. Staging and barriers can cost more than the saved reads when the texture cache already hits; measure both. *WebGL2:* implicit texture cache only.
- **Barriers in a dispatch** — only · passes. `workgroupBarrier`, `storageBarrier`, `textureBarrier`, `workgroupUniformLoad` let multi-step algorithms (scan, reduce, sort stages, single-pass mip chains) run inside one dispatch. Barriers synchronize one workgroup and must sit in uniform control flow; a phase needing every workgroup finished is a separate dispatch. *WebGL2:* one draw per step with a texture round trip.
- **Dispatch sized to the data** — only · GPU. 1D/2D/3D grids up to 65,535 workgroups per dimension, sized to particles, tiles or voxels; `override` constants set workgroup size per pipeline. *WebGL2:* work size equals render-target pixels.
- **3D storage texture writes** — only · passes. Fill a volume (noise, SDF, fluid or light grid) in one dispatch. *WebGL2:* one draw per slice via `framebufferTextureLayer`.
- **Compute and render in one submission** — cleaner · latency. Compute and render passes encode in order into one command buffer; the implementation tracks dependencies and the CPU never waits between them.

## 2. Storage memory and atomics

- **Storage buffers** — only · memory, code. Typed structs and runtime-sized arrays (`arrayLength()`), 128 MiB per binding by default (`maxStorageBufferBindingSize`, raisable), 8 per stage by default. Readable in every stage; writable in compute and fragment. *WebGL2:* data textures + `texelFetch` with hand packing, capped by max texture size; uniform blocks guaranteed only 16 KiB.
- **Fragment-shader scatter writes** — only · passes. A fragment can write anywhere in a storage buffer and use atomics: per-pixel linked lists (exact OIT), fragment counters, virtual-texturing feedback. *WebGL2:* a fragment writes only its own pixel in bound targets.
- **Integer atomics** — only · passes, CPU. `atomicAdd/Sub/Min/Max/And/Or/Xor/Exchange/CompareExchangeWeak` on `i32`/`u32` in storage or workgroup memory: counters, histograms, append lists, binning, packed depth+ID visibility. No float or 64-bit atomics in core: sum in fixed point. Aggregate in workgroup memory or subgroups before contended global atomics. An atomic counter does not publish unrelated non-atomic data to other invocations. *WebGL2:* additive or min/max blending into a render target.
- **64-bit min/max atomics** — optional (`atomic-vec2u-min-max`) · GPU. `atomicMin/Max` on a packed `vec2u` resolves depth plus payload (point or triangle ID) in one operation; the basis of compute software rasterizers. *WebGL2:* none.
- **Read-write storage textures** — only (Chrome 124) · memory. Update a texture in place (temporal history, simulation grid) instead of ping-ponging two copies. *WebGL2:* two textures swapped per pass.
- **More storage formats** — optional (`texture-formats-tier1`/`tier2` Chrome 142, `bgra8unorm-storage`) · memory. 8- and 16-bit formats for storage writes and read-write access.
- **Vertex pulling** — cleaner · memory, code. The vertex shader indexes storage buffers by `vertex_index`/`instance_index`, so meshes can be quantized, compressed or shared without a fixed vertex layout. Vertex-stage storage is read-only. *WebGL2:* `texelFetch` from data textures.
- **Combined buffer usages** — cleaner · copies. One buffer can be `STORAGE | VERTEX | INDIRECT`: compute writes instances and draw arguments, rendering reads them in place. *WebGL2:* only transform feedback writes vertex buffers.

## 3. GPU-driven work

- **Indirect draws** — only · CPU, GPU. `drawIndirect` reads 4 × u32 (vertexCount, instanceCount, firstVertex, firstInstance); `drawIndexedIndirect` reads 5 × u32 (indexCount, instanceCount, firstIndex, baseVertex, firstInstance). Culling, LOD and compaction never return to JavaScript. Core has no multi-draw-indirect or count buffer: issue one indirect call per buffer offset (record them in a bundle) or compact into a few instanced draws. *WebGL2:* `WEBGL_multi_draw` batches CPU-supplied arrays.
- **Indirect dispatch** — only · GPU. `dispatchWorkgroupsIndirect` reads 3 × u32: a dispatch sizes itself from the list the previous one built (active tiles, live particles, surviving clusters). List building must cost less than the skipped work; test sparse and nearly full inputs. *WebGL2:* none; a fixed-size fragment pass still schedules every pixel.
- **Indirect first instance** — optional (`indirect-first-instance`) · CPU. Non-zero `firstInstance` in indirect draws, so many indirect draws share one instance buffer. Without the feature such a draw is silently a no-op.
- **GPU-readable query results** — only · latency. `resolveQuerySet` writes occlusion results into a buffer that can also be bound as storage; compute turns last frame's visibility into this frame's indirect draws. *WebGL2:* query results reach the CPU only, a frame or more late.

## 4. CPU cost per draw and API state

The explainer calls WebGPU drawing commands "individually cheaper than WebGL's": validation happens at object creation, not per draw. Matters for draw-heavy scenes and busy main threads; does nothing for GPU-bound frames.

- **Pre-validated pipelines** — cleaner · CPU. Shaders, blend, depth/stencil, topology and vertex layout validated and compiled once. *WebGL2:* mutable state checked per draw, often in ANGLE and the driver.
- **Bind groups and shared layouts** — cleaner · CPU. A resource set validated at creation, bound in one call. Group by update frequency (frame, material, object); share explicit layouts across pipelines so low-frequency groups stay bound through pipeline switches. Cache groups; recreating them per frame wastes the benefit. *WebGL2:* per-resource binds; UBO `bindBufferRange` is closest.
- **Dynamic offsets** — cleaner · CPU, memory. One big uniform or storage buffer; select each draw's slice with an offset in `setBindGroup`. *WebGL2:* `bindBufferRange` per draw.
- **Render bundles** — only · CPU. Record a draw sequence once, replay with `executeBundles` in compatible passes (same formats and sample count); buffer contents may still change. No GPU saving; per-frame rebuilds erase the benefit. *WebGL2:* VAOs and multi-draw capture part of it.
- **Base vertex and first instance in every draw** — cleaner · CPU, memory. `draw`/`drawIndexed` take `firstVertex`, `baseVertex`, `firstInstance`: many meshes in one vertex and index buffer with no rebinding. *WebGL2:* draft `WEBGL_draw_instanced_base_vertex_base_instance` or re-pointing attributes.
- **No synchronous calls in the frame** — cleaner · latency. Nothing in the frame loop blocks on the GPU process; results arrive as promises. *WebGL2:* `getError`, `getParameter`, `checkFramebufferStatus`, `getShaderParameter`, `readPixels` without a PBO each round-trip to the browser's GPU process.
- **Asynchronous errors** — cleaner · latency, code. `pushErrorScope`/`popErrorScope` and `uncapturederror`, reported with object labels. *WebGL2:* synchronous `getError`.
- **Async pipeline creation** — cleaner · latency. `createRenderPipelineAsync`/`createComputePipelineAsync` compile off the main thread; create every variant before first use. *WebGL2:* `KHR_parallel_shader_compile` polling; using an unready program blocks.
- **Override constants** — cleaner · CPU, GPU. `override` values specialize one module per pipeline (toggles, loop counts, workgroup size); the compiler folds them and removes dead branches. They cannot toggle `enable` directives. *WebGL2:* `#define` patching and recompiling.
- **Immediates** — optional (Chrome 149–150, `immediate_address_space`) · CPU. `setImmediates` + `var<immediate>` for up to `maxImmediateSize` (64) bytes per draw without a buffer. *WebGL2:* uniform setters (also cheap).
- **One device, many canvases** — cleaner · memory, CPU. One `GPUDevice` renders to any number of canvases and shares every buffer, texture and pipeline. *WebGL2:* a context per canvas; no sharing, so uploads and compiles duplicate.
- **No global state machine** — cleaner · code. State travels with each command; no bind-to-edit, no leakage between libraries sharing a device.

## 5. Uploads, copies and readback

- **`mappedAtCreation`** — cleaner · copies. Fill a new buffer during creation with no staging copy. `writeBuffer` costs "at least an extra copy" versus mapping (explainer). *WebGL2:* `bufferData` copies.
- **Uploads that never stall** — cleaner · latency. `writeBuffer`/`writeTexture` stage the data and queue the copy, so updating a buffer the GPU is reading never waits. A `mapAsync` staging ring removes the extra copy. *WebGL2:* `bufferSubData` on an in-use buffer can stall or force driver renaming.
- **Async readback** — cleaner · latency. `copyTextureToBuffer`/`copyBufferToBuffer` → `mapAsync` resolves when the GPU is done; `onSubmittedWorkDone()` is a completion promise. Use a ring of readback buffers; a mapped buffer cannot be used by the GPU. *WebGL2:* `readPixels` blocks; the async path is PBO + `fenceSync` + polling + `getBufferSubData`.
- **External textures** — only · memory, latency. `importExternalTexture` samples a video element or WebCodecs `VideoFrame` without copying, with YUV→RGB in the sampler. The object expires at the end of the task; import every frame. *WebGL2:* `texImage2D(video)` copies each frame.
- **One-step image upload** — cleaner · CPU. `copyExternalImageToTexture` copies `ImageBitmap`, canvas or video into a texture region with flip, premultiply and colour space handled in the GPU copy.
- **Direct copy commands** — cleaner · passes. Buffer↔buffer, buffer↔texture, texture→texture (depth included), encoded with other GPU work. *WebGL2:* `copyBufferSubData`; textures need a framebuffer plus `blitFramebuffer`/`copyTexSubImage`.
- **Texture views** — cleaner · memory, code. View a mip range, array layer, or a reinterpreted format (sRGB vs linear via `viewFormats`) without copying; render into mip N while sampling N−1 through two views. *WebGL2:* mutate `TEXTURE_BASE_LEVEL`/`MAX_LEVEL`; no format reinterpretation.
- **Usage flags and large buffers** — cleaner · memory. Declared usages let the implementation place memory; `maxBufferSize` 256 MiB default, raisable.

## 6. Render passes and output

- **Explicit load/store ops** — cleaner · bandwidth. Per attachment `loadOp` `clear|load` and `storeOp` `store|discard`. On tile-based GPUs (mobile, Apple silicon) `clear` skips reading old tiles and `discard` skips writing them back. Audit every attachment: `load` and `store` only where contents must survive. *WebGL2:* `invalidateFramebuffer` and clears as hints.
- **Transient attachments** — optional (Chrome 146, `GPUTextureUsage.TRANSIENT_ATTACHMENT` with `RENDER_ATTACHMENT`) · memory, bandwidth. A pass-local target (depth, MSAA colour) can stay in tile memory and may need no VRAM. Requires `clear`/`discard`; cannot be sampled or be a resolve destination; a transient MSAA target may resolve into a persistent one. *WebGL2:* none.
- **In-pass MSAA resolve** — cleaner · bandwidth. `resolveTarget` resolves at pass end; with `discard` or a transient multisampled texture, samples never reach memory. Core sample counts are 1 and 4. *WebGL2:* store all samples, then `blitFramebuffer`.
- **Read-only depth** — cleaner · GPU, copies. `depthReadOnly: true` (and `stencilReadOnly` for combined formats) lets a pass depth-test against a texture it also samples: soft particles, decals, glow occlusion. Hardware early-Z rejects hidden fragments instead of shader depth loads. *WebGL2:* sampling the bound depth texture is a feedback loop; copy depth first.
- **Per-target blend and write mask** — cleaner · passes. Each colour target has its own blend state and write mask. *WebGL2:* needs `OES_draw_buffers_indexed`.
- **[0, 1] depth** — cleaner · precision. Native [0, 1] clip depth plus `depth32float` gives full reversed-Z precision. *WebGL2:* needs `EXT_clip_control`; [−1, 1] wastes float precision.
- **Depth-only formats** — cleaner · bandwidth. `depth16unorm`, `depth32float` and `stencil8` carry no unused aspect; prefer them when the stencil is unused.
- **Sample mask and per-sample shading** — cleaner · quality per cost. `@builtin(sample_mask)` in/out, `sample_index`, `@interpolate(…, sample)`: supersample only where needed, custom coverage. *WebGL2:* not in GLSL ES 3.00; alpha-to-coverage only.
- **Dual-source blending** — optional (`dual-source-blending`, Chrome 130) · passes. A second fragment output (`@blend_src(1)`) feeds blend factors: per-channel transmittance, subpixel text in one draw.
- **Unclipped depth** — optional (`depth-clip-control`) · GPU. `unclippedDepth` clamps instead of clipping: pancaked shadow casters, tighter shadow frusta.
- **Primitive index** — optional (`primitive-index`, Chrome 142) · memory. Per-triangle ID in the fragment shader; a visibility buffer without an extra attribute.
- **Clip distances** — optional (`clip-distances`, Chrome 131) · GPU. Hardware clip planes instead of `discard`. *WebGL2:* `WEBGL_clip_cull_distance`.
- **HDR canvas** — optional (Chrome 129) · quality. `rgba16float` canvas with `toneMapping: { mode: 'extended' }` draws brighter than white on HDR displays. *WebGL2:* no core HDR output mode.
- **Preferred canvas format** — cleaner · bandwidth. Render in `navigator.gpu.getPreferredCanvasFormat()` (often `bgra8unorm`) to avoid a present-time conversion copy; `getCurrentTexture()` each frame, no `preserveDrawingBuffer` copy.

## 7. WGSL

- **Texture gather** — only · GPU. `textureGather`/`textureGatherCompare` return one channel of four texels per fetch: PCF, depth-aware upsampling, depth min/max tiles, 2×2 reductions. *WebGL2:* not in GLSL ES 3.00.
- **`f16`** — optional (`shader-f16`, Chrome 120, `enable f16;`) · GPU, memory. Explicit half type for math and storage; on Apple and mobile GPUs it can raise ALU throughput, cut register pressure and halve buffer traffic. Keep positions, depth and long accumulations in `f32`; validate worst-case error. *WebGL2:* `mediump` is a hint desktop drivers usually ignore.
- **Subgroups** — optional (`subgroups`, Chrome 134; `subgroup-size-control` 151–152; `enable subgroups;`) · GPU, barriers. `subgroupAdd`, ballots, broadcasts, shuffles and quad ops in compute and fragment: reductions and scans without workgroup memory, aggregation before global atomics. Do not assume 32 lanes or full activity; keep a workgroup-memory fallback. *WebGL2:* derivatives only.
- **Packed 8-bit dot products** — optional language feature (`packed_4x8_integer_dot_product`, Chrome 123) · GPU. `dot4I8Packed`/`dot4U8Packed` for quantized inference. A built-in does not guarantee a hardware instruction; include scale and zero-point costs.
- **Pack/unpack built-ins** — cleaner · memory. `pack4x8unorm/snorm`, `pack2x16float/unorm/snorm` and unpack partners decode dense storage in one call. *WebGL2:* 2×16 only.
- **Any-vertex flat interpolation** — cleaner · GPU. `@interpolate(flat, either)` lets the hardware pick the provoking vertex. *WebGL2:* last vertex fixed; backends whose hardware uses the first may emulate.
- **Language structure** — cleaner · code. Structs and runtime arrays in buffers, pointers, one module with vertex, fragment and compute entry points, no implicit conversions, async `getCompilationInfo()` with line/column.
- **Uniformity analysis** — cleaner · correctness. Derivatives, implicit-LOD sampling and barriers in non-uniform control flow are compile errors, not driver-dependent undefined results.

## 8. Formats and optional features

- **11-bit float render targets** — optional (`rg11b10ufloat-renderable`) · bandwidth. 4-byte HDR targets with blending and MSAA, half of `rgba16float`, for bloom chains and alpha-free lighting buffers. 6/6/5-bit mantissas: check banding against the quality contract.
- **Float32 filtering and blending** — optional (`float32-filterable`, `float32-blendable` Chrome 132). *WebGL2:* `OES_texture_float_linear`, `EXT_float_blend`.
- **Compressed textures** — optional (`texture-compression-bc`, `-etc2`, `-astc`; `-bc-sliced-3d`, `-astc-sliced-3d` for volumes). *WebGL2:* 2D/array compression via extensions, no 3D.
- **Texture component swizzle** — optional (`texture-component-swizzle`, Chrome 143). Remap channels per view instead of storing extra channels. *WebGL2:* `TEXTURE_SWIZZLE_*` removed from ES 3.0.
- **`depth32float-stencil8`** — optional. Float depth with stencil.
- **Compatibility mode** — newer (Chrome 146). A restricted feature level for OpenGL ES 3.1 and D3D11-class devices; `core-features-and-limits` marks a full-core device. Code written for compatibility runs on both.

## 9. Debugging and robustness

- **Per-pass GPU timestamps** — optional (`timestamp-query`, Chrome 121) · profiling. `timestampWrites` on a pass descriptor; values are quantized for privacy. Snippet in [porting-and-measurement.md](porting-and-measurement.md). *WebGL2:* `EXT_disjoint_timer_query_webgl2`, often unavailable.
- **Labels, debug groups, markers** — cleaner. Every object takes `label`; `pushDebugGroup`/`insertDebugMarker` show in errors and native capture tools.
- **Device loss as a promise** — cleaner. `device.lost` resolves with a reason; recover by requesting a new device. *WebGL2:* `webglcontextlost`/`restored` and manual rebuild of all state.
- **Consistent validation** — cleaner. One set of validation rules across platforms; fewer driver-specific workarounds.
