# Porting, bottlenecks and measurement

## Bottleneck → first experiment

| Observed cost | First WebGPU experiment | Check |
|---|---|---|
| CPU issuing many stable draws | Render bundles; cached bind groups; dynamic offsets; shared vertex/index buffers with `baseVertex` | CPU frame time with GPU work held constant; bundle rebuild rate |
| Invisible or inactive work | Compute culling/compaction + `drawIndirect` / `dispatchWorkgroupsIndirect` | Saved work minus list building, at visibility extremes |
| CPU builds and uploads GPU data each frame | Compute + storage buffers; keep intermediates resident | Full data flow, transfers, consumer cost |
| Repeated full-screen passes over the same inputs | Fusion into one compute dispatch; workgroup tiling | GPU time, traffic, register pressure of the fused kernel |
| Expensive per-pixel effect on mostly empty screen | Tile classification + indirect dispatch | Empty fraction; classification is conservative |
| Shader depth reads for occlusion | Read-only depth attachment + hardware depth test; `textureGather` | Exact match for hard compares; early-Z kept |
| Bandwidth on tile-based GPUs | `clear`/`discard`, transient attachments, in-pass MSAA resolve, 11-bit float targets | Memory and GPU time on the tile-based device itself |
| Collective math or contended counters | Workgroup aggregation; subgroups | Correct with different subgroup sizes and inactive lanes |
| ALU-bound with tolerant math | `f16`, packed int8 dot products | Worst-case error, conversions, total traffic |
| Main-thread hitches | Async pipeline creation up front; remove synchronous patterns | Cold first-use stalls measured apart from warm frames |

If the frame is fragment-bound and none of the rows apply, the API switch alone will not speed it up.

## Porting from WebGL2

- **Restructure, don't translate.** A one-to-one port keeps every pass and pixel and adds validation and shader translation. One fill-bound renderer measured 8–24% slower on WebGPU (Dawn/Metal, Apple silicon) until passes were restructured.
- Remove WebGL-era workarounds that now cost: data textures (→ storage buffers), ping-pong pairs (→ read-write storage textures), depth copies for sampling (→ read-only depth), per-slice volume draws (→ 3D storage writes), `#define` variants (→ `override`), readbacks used to size later work (→ indirect).
- Map conventions: clip-space depth is [0, 1] (adjust the projection; reversed-Z becomes straightforward). NDC Y still points up, but `@builtin(position)` and texture coordinates start at the top-left, so code that flips Y for render-to-texture or reads `gl_FragCoord.y` needs adjusting.
- Create pipelines and bind groups at load time, asynchronously; per-frame object creation is the WebGPU equivalent of redundant GL state changes.
- Upload through `writeBuffer`/`writeTexture` from preallocated typed arrays; allocate nothing per frame.

## Where WebGL2 is equal or ahead

Equal: MRT, float render targets (`EXT_color_buffer_float`), integer, 3D and array textures, instancing, UBOs, VAOs, sampler objects, MSAA, occlusion queries, transform feedback and ping-pong simulation, compressed textures, float filtering/blending and clip distances via extensions, `OffscreenCanvas` rendering in a worker, and any fragment-bound pass.

Ahead: `WEBGL_multi_draw` (core WebGPU has no multi-draw); reach and driver maturity through ANGLE; synchronous start-up (no adapter, device or pipeline promises); draft `WEBGL_shader_pixel_local_storage` (on-chip per-pixel storage; not compute, not shipped everywhere).

Neither (in browsers): mesh/task shaders, hardware ray tracing, bindless resources, geometry and tessellation shaders, multiple queues, device-wide barriers, multi-draw-indirect-count, 64-bit floats, float atomics, one device used from several workers.

## Measuring GPU time

Command-encoding time is not GPU time. With `timestamp-query`:

```js
const qs = device.createQuerySet({ type: 'timestamp', count: 2 });
const resolve = device.createBuffer({ size: 16, usage: GPUBufferUsage.QUERY_RESOLVE | GPUBufferUsage.COPY_SRC });
const readback = device.createBuffer({ size: 16, usage: GPUBufferUsage.MAP_READ | GPUBufferUsage.COPY_DST });

const pass = encoder.beginRenderPass({ /* attachments */
  timestampWrites: { querySet: qs, beginningOfPassWriteIndex: 0, endOfPassWriteIndex: 1 } });
// … draws …
pass.end();
encoder.resolveQuerySet(qs, 0, 2, resolve, 0);
encoder.copyBufferToBuffer(resolve, 0, readback, 0, 16);
device.queue.submit([encoder.finish()]);

await readback.mapAsync(GPUMapMode.READ);
const [t0, t1] = new BigUint64Array(readback.getMappedRange());
const passMs = Number(t1 - t0) / 1e6;   // nanoseconds → ms; quantized for privacy
readback.unmap();
```

- Use a ring of readback buffers; a buffer pending `mapAsync` cannot be copied into, so a single buffer drops frames.
- Average many frames and report the median; timestamps are coarsened and can be absent.
- Without the feature: time a fixed number of frames with `onSubmittedWorkDone()` and subtract an empty-frame baseline, or vary one pass and watch total frame time.
- Keep warm-up, shader compilation and first-use allocation outside steady-state numbers; report start-up separately.
- Compare against an optimized WebGL2 baseline on the same scene, inputs, resolution and devices, including a tile-based GPU when bandwidth changes are involved.

## Validation checklist

Run before accepting any speed-up:

- Output against the quality contract (for example SSIM and a per-pixel tolerance) on representative scenes.
- Empty, single-element and full inputs; partial last workgroups; counts beyond buffer capacity.
- Nondeterministic atomic ordering: results must not depend on it, or must be sorted.
- Reduced precision (`f16`, 11-bit floats, fixed point) at range extremes and over long accumulation.
- Every fallback path executed at least once (force-disable each optional feature).
- Device loss and recreation.
- Results across the target browsers and GPU vendors; differences within noise are not wins.
