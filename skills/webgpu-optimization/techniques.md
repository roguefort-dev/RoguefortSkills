# Techniques WebGPU enables

Designs that need compute, storage writes, atomics or indirect work. Each lists the mechanism, the features it uses, and what to check. Treat each as an experiment through the method in [SKILL.md](SKILL.md).

## Contents

- GPU culling and indirect draws (code)
- Hardware depth test instead of shader depth reads (code)
- Tile-classified full-screen passes
- Atomic accumulation in fixed point (code)
- Fused post-processing
- Single-pass mip and bloom chains
- Workgroup-memory filters
- Clustered or tiled lighting
- Particles with spawn and death
- GPU sorting and prefix sums
- Software rasterization of points and tiny triangles
- Visibility buffer
- Exact order-independent transparency
- Histograms and auto-exposure
- Compute skinning and procedural geometry
- Simulation
- ML inference and image processing
- Zero-copy video
- GPU occlusion feedback
- Attachment lifetime audit

## GPU culling and indirect draws

Compute tests each instance against the frustum (and optionally a Hi-Z pyramid), appends survivors with `atomicAdd`, and the same buffer feeds `drawIndirect`. JavaScript never reads the count.

```wgsl
struct DrawArgs { vertexCount: u32, instanceCount: atomic<u32>, firstVertex: u32, firstInstance: u32 }
struct Instance { center: vec3f, radius: f32 }

@group(0) @binding(0) var<storage, read> instances: array<Instance>;
@group(0) @binding(1) var<storage, read_write> visible: array<u32>;
@group(0) @binding(2) var<storage, read_write> args: DrawArgs;
@group(0) @binding(3) var<uniform> planes: array<vec4f, 6>;

@compute @workgroup_size(64)
fn cull(@builtin(global_invocation_id) id: vec3u) {
  let i = id.x;
  if (i >= arrayLength(&instances)) { return; }
  let s = instances[i];
  for (var p = 0u; p < 6u; p++) {
    if (dot(planes[p].xyz, s.center) + planes[p].w < -s.radius) { return; }
  }
  visible[atomicAdd(&args.instanceCount, 1u)] = i;
}
```

```js
const args = device.createBuffer({ size: 16,
  usage: GPUBufferUsage.STORAGE | GPUBufferUsage.INDIRECT | GPUBufferUsage.COPY_DST });
const reset = new Uint32Array([vertexCount, 0, 0, 0]); // allocated once
// per frame
device.queue.writeBuffer(args, 0, reset);               // ordered before the submit below
const cp = encoder.beginComputePass(); cp.setPipeline(cullPipeline); cp.setBindGroup(0, cullGroup);
cp.dispatchWorkgroups(Math.ceil(instanceCount / 64)); cp.end();
// render pass: vertex shader reads visible[instance_index]
pass.drawIndirect(args, 0);
device.queue.submit([encoder.finish()]);
```

- Survivor order is nondeterministic; anything order-dependent (transparency) needs a sort.
- Several meshes: one `DrawArgs` per mesh at 16-byte offsets and one `drawIndirect` per offset; non-zero `firstInstance` needs `indirect-first-instance`.
- Hi-Z: build a depth min/max pyramid with `textureGather` in compute, test each bounding box against last frame's depth, re-test disocclusions.
- Check: culling cost plus rendering at 0%, ~50% and 100% visibility.

## Hardware depth test instead of shader depth reads

Passes that occlude against scene depth (glow, soft particles, decals, overlays) often `textureLoad` depth per fragment. A read-only depth attachment lets fixed-function early-Z do it, and the same texture stays sampleable in that pass for soft edges.

```js
const pipeline = device.createRenderPipeline({ /* … */
  depthStencil: { format: 'depth32float', depthWriteEnabled: false, depthCompare: 'less' } });
const pass = encoder.beginRenderPass({
  colorAttachments: [{ view: colorView, loadOp: 'load', storeOp: 'store' }],
  depthStencilAttachment: { view: depthView, depthReadOnly: true }, // no depth load/store ops
});
```

- A hard depth compare maps exactly; a soft fade still needs a sample, but fully hidden fragments no longer run.
- Writing `frag_depth` or `discard` can disable early-Z; keep them out of these shaders.
- Several neighbouring depth reads in one shader → one `textureGather`.
- Depth-texture loads were measurably dearer than WebGL2's in one Dawn/Metal (Apple silicon) renderer; treat them as a cost to remove, not a given.

## Tile-classified full-screen passes

A first compute pass classifies screen tiles (for example 16×16) as empty, cheap or expensive and appends tile IDs per class with atomics, writing a 3 × u32 dispatch count per class. Later passes run `dispatchWorkgroupsIndirect` only over their tiles. Fits ray marching, volumetrics, SSR, depth of field, deferred lighting.

- Per-tile depth bounds or coverage computed once in workgroup memory feed the expensive pass.
- Classification must be conservative: a missed tile is a visible hole. Diff against the unclassified pass.
- Win scales with the empty fraction; measure on the densest and the emptiest views.

## Atomic accumulation in fixed point

Additive contributions are order-independent, so compute or fragment shaders can `atomicAdd` scaled integers per pixel or bin and resolve once. Replaces blending thousands of overlapping quads (small stars, sparks, splats) or many accumulation passes.

```wgsl
@group(0) @binding(0) var<storage, read_write> accum: array<atomic<u32>>; // 3 per pixel
const SCALE = 4096.0;                                                       // 12 fractional bits

fn splat(pixel: u32, rgb: vec3f) {
  let q = vec3u(rgb * SCALE + 0.5);
  atomicAdd(&accum[pixel * 3u], q.x);
  atomicAdd(&accum[pixel * 3u + 1u], q.y);
  atomicAdd(&accum[pixel * 3u + 2u], q.z);
}
// resolve: rgb = vec3f(a0, a1, a2) / SCALE; then clear accum (clearBuffer) for the next frame
```

- Range: per-channel sums must stay below 2³² / SCALE (1,048,576 at 4096). Pick SCALE from the HDR peak; check quantization at the dimmest contribution.
- Contention: many writers on one pixel serialize; aggregate per workgroup or subgroup first.
- Packing two 16-bit channels in one `u32` halves atomics but carries overflow between channels; guard the range.

## Fused post-processing

Tonemap, grading, vignette, grain, composite and other per-pixel steps that each read the same colour and depth run as one compute dispatch (or one fragment pass), reading inputs once and writing the output once. Neighbourhood steps (blur, glow) load a tile plus apron into workgroup memory. Saves full-resolution intermediates and their bandwidth. Check: register pressure and occupancy of the fused kernel; the slowest branch now runs for every pixel.

## Single-pass mip and bloom chains

One dispatch builds every mip level (AMD single-pass downsampler style): each workgroup reduces its tile through workgroup memory; a global atomic counter lets the last workgroup finish the top levels. Replaces one render pass per level. Bloom upsample chains can fuse similarly. Check: the "last workgroup" pattern needs storage barriers and atomics on a counter; compare with the per-level chain on small targets, where it may not win.

## Workgroup-memory filters

Separable and bilateral blurs, SSAO, depth of field, convolutions and FFTs (ocean waves, large kernels) load each tile plus apron once. Gains appear when the kernel is wide enough that cache reuse fails; a linear-sampled Gaussian in a fragment pass can still win for small radii.

## Clustered or tiled lighting

Compute bins lights into screen tiles or depth clusters with atomics (per-tile counters in workgroup memory, then a global append), and shading loops over each cluster's list only. Clustered shading exists in WebGL2 with CPU-built lists; GPU list building is the new part. Check: list overflow handling and the worst cluster.

## Particles with spawn and death

Compute emits into free slots and kills by compacting with atomic counters, writes the live count into draw arguments, and renders with `drawIndirect`. Neighbour grids (counting sort by cell) enable SPH fluids, flocking and collisions. WebGL2 transform feedback already handles independent updates; the gain is in emission, death and neighbour queries.

## GPU sorting and prefix sums

Radix sort and scan from workgroup memory and subgroups; decoupled look-back needs forward-progress guarantees WebGPU does not give, so prefer reduce-then-scan. Underpins Gaussian-splat rendering, transparency sorting, stream compaction and spatial hashing. If installed, `perf-aio`'s `parallel-primitives.md` and `compute-kernels.md` cover the algorithms.

## Software rasterization of points and tiny triangles

Compute projects each point (or pixel-sized triangle) and writes `atomicMin` of packed depth+ID per pixel; a resolve pass shades from the ID. With `atomic-vec2u-min-max` depth and a 32-bit payload fit; without it, pack quantized depth in the high bits of a `u32` and the ID in the low bits. Beats hardware rasterization when primitives are about one pixel, since hardware shades 2×2 quads. Check: depth quantization against z-fighting, and the fallback to hardware for large primitives.

## Visibility buffer

Rasterize only a triangle ID (and instance ID) per pixel, then shade each pixel once in compute or a full-screen pass, fetching attributes from storage buffers. Removes overdraw shading and G-buffer bandwidth. `primitive-index` provides the triangle ID; without it, pass a flat ID attribute. Check: derivative reconstruction for texture LOD in the shading pass.

## Exact order-independent transparency

Fragment shaders append (colour, depth, next) nodes to a storage-buffer pool and swap a per-pixel head pointer with `atomicExchange`; a resolve pass sorts each pixel's list and blends. Check: pool overflow policy and the worst-case list length; weighted blended OIT (possible in WebGL2) may be enough.

## Histograms and auto-exposure

Each workgroup builds a luminance histogram in workgroup memory with atomics, merges into a global histogram, and one small dispatch computes exposure. Replaces a reduction mip chain and is exact.

## Compute skinning and procedural geometry

Skin or morph vertices once per frame into a `STORAGE | VERTEX` buffer, reused by depth, shadow and colour passes. Terrain, grass, hair and tessellated surfaces are generated in compute and drawn indirectly; neither API has tessellation or geometry shaders, so compute fills that role. Check: the extra buffer write and read against saved vertex work.

## Simulation

Cloth and soft bodies (PBD), grid fluids, reaction–diffusion and n-body run in compute with storage buffers and read-write storage textures (in-place updates). Separate dispatches act as global barriers between solver phases.

## ML inference and image processing

Tiled matrix multiply through workgroup memory, `f16` storage and math, subgroup reductions and packed int8 dot products. ONNX Runtime Web and Transformers.js take this path. Compare at equal output quality, not across precision changes.

## Zero-copy video

`importExternalTexture` each frame and sample camera or video directly for filters, background replacement and AR. Check: per-frame import (the object expires) and colour-space handling.

## GPU occlusion feedback

Occlusion queries resolve into a storage-capable buffer; compute converts results into next frame's indirect draws without the CPU. Accept one frame of latency and re-test newly visible objects.

## Attachment lifetime audit

For every render pass, list each attachment and whether its contents are needed after the pass. Set `storeOp: 'discard'` for depth and intermediates that die with the pass, `loadOp: 'clear'` where old contents are irrelevant, `TRANSIENT_ATTACHMENT` where supported for pass-local targets, and resolve MSAA in-pass. Free on desktop, often significant on tile-based GPUs. Check: nothing samples a discarded attachment later.
