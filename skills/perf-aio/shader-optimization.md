# Shader and GPU optimization

Engine-agnostic (GLSL / HLSL / WGSL / MSL). Engine hooks are at the end.

For compute kernels, tiled filters, reductions, workgroup sizing, shared-memory barriers, or autotuning, load [compute-kernels.md](compute-kernels.md). Its benchmarking and bottleneck experiments also apply to graphics shaders; keep graphics-specific coverage and depth behavior here.

The graphics cost model in one line: **GPU time ≈ shaded fragments × cost per fragment + bytes moved + vertices × cost per vertex**. Instruction count is a weak proxy. Judge every change by A/B GPU milliseconds on the target device.

## Find the bound

Run the cheapest experiment that separates the candidates:

| Experiment | If frame time drops a lot → |
|---|---|
| Halve the render resolution (or viewport) | Fragment-bound or bandwidth-bound |
| Replace the suspect material's fragment shader with a constant color | That shader's per-fragment ALU/texture cost |
| Swap textures for 1×1 or a lower mip (mip bias) | Texture bandwidth / cache misses |
| Disable post-processing passes one at a time | Fullscreen-pass bandwidth |
| Cut vertex count (LOD 0 → LOD 2) with the same screen coverage | Vertex-bound or micro-triangle waste |
| Skip draw submission but keep simulation | CPU/submit-bound (draw calls, state changes), not the shader |
| GPU is idle between frames in a capture | CPU-bound; shader work will not help |

Tools:
- **macOS/iOS/Metal:** Xcode GPU frame capture (per-pass and per-draw cost, shader profiler with per-line cost) and Instruments → Metal System Trace. RenderDoc cannot capture Metal.
- **Windows/Linux/Android:** RenderDoc, PIX (D3D12), Nsight Graphics (NVIDIA), Radeon GPU Profiler/Analyzer (AMD), Snapdragon Profiler (Adreno), Arm Performance Studio/Streamline + Mali Offline Compiler (Mali).
- **Web:** Chrome Performance panel with GPU track, Spector.js (WebGL command capture), `EXT_disjoint_timer_query_webgl2` (WebGL2 timer queries), WebGPU `timestamp-query`.
- **Unity:** Frame Debugger for draw/pass structure, Profiler GPU module, `FrameTimingManager` for CPU/GPU frame times in builds.

## Fragment cost

- **Overdraw first.** Every covered layer pays full shading. Sort opaque front-to-back. Add a depth prepass when the fragment shader is expensive and depth complexity is high. Shrink transparent quads to the sprite's visible shape. Render particles/smoke at half resolution and composite.
- **Quad shading.** GPUs shade 2×2 pixel quads, so triangles smaller than a few pixels waste up to 4× fragment work. Use LODs to keep triangles ≳ 8–10 px on screen.
- **`discard` / alpha test.** On tile-based and hidden-surface-removal GPUs (Apple, PowerVR, Mali, Adreno), `discard` disables early depth rejection for that draw. Keep alpha-tested draws few, draw them after opaques, and prefer tighter geometry over transparent texels. Writing `gl_FragDepth`/`SV_Depth` has the same effect.
- **Move work up the frequency ladder:** constant → per-draw uniform (CPU) → per-vertex (interpolated) → per-fragment. Compute anything linear across the triangle in the vertex shader.
- **Transcendentals** (`sin`, `cos`, `exp`, `log`, `pow`, `rsqrt`, `atan`) run on special-function units at a fraction of full rate. `pow(x, y)` is `exp2(y * log2(x))`, so write `x*x`, `x*x*x` for small integer powers. `normalize` costs a `rsqrt`; skip it where the input is already unit length.
- **Precision (mobile).** `mediump`/`half` gives up to 2× ALU throughput and halves register use on Mali, Adreno and Apple GPUs. Desktop GPUs typically ignore GLSL `mediump`. Keep in full precision: positions, depth, UVs into large textures (> ~2048 texels), and accumulating time. Wrap time (`mod(time, period)` on the CPU) before feeding `sin(time * k)`, or it loses precision after hours.
- **Register pressure → occupancy.** Big shaders with many live values reduce how many waves/warps run at once, which exposes texture latency. Splitting a monster shader or reducing simultaneously live temporaries can speed it up even with more instructions.

## Branching

- A branch on a **uniform** or dynamically uniform value (same for the whole draw/wave) is nearly free. Use it to skip whole features.
- A **divergent** branch (different pixels in one wave go different ways) executes both sides for that wave. It still wins when one side is expensive and most waves are coherent (e.g. early-out on sky pixels).
- Small `if`s compile to selects anyway; rewriting them as `mix`/`step` is rarely a gain by itself. Measure.
- Implicit derivatives (`texture()`, `dFdx`) are undefined inside divergent control flow. Sample before the branch, or use `textureLod`/`textureGrad`/`SampleLevel`/`SampleGrad` inside it.
- Loops: constant trip counts unroll well. For data-dependent loops, cap the count and break early.

## Textures and bandwidth

- **Mipmaps** on every minified texture. Without them, sampling thrashes the cache and aliases.
- **Compressed formats:** BC1/BC3/BC5/BC7 (desktop), ASTC (Apple, modern Android), ETC2 (GLES3 fallback); on the web, KTX2 + Basis Universal transcodes to whichever the device supports. Compressed textures cut both memory and bandwidth.
- **Channel packing:** pack occlusion/roughness/metallic (ORM) into one texture so one fetch serves three values.
- **Render-target formats** dominate fullscreen-pass cost. Use `R11G11B10F`/`RG11B10_UFLOAT` instead of `RGBA16F` for HDR color when alpha is unused. Use single-channel targets for masks and AO.
- **Dependent reads** (UVs computed in the fragment shader) cost more on older mobile GPUs than varyings-driven UVs. Pass static UV transforms from the vertex shader.
- Prefer ALU over a lookup texture for cheap math. A LUT wins only for genuinely expensive functions or authored curves (color grading).
- `textureGather`/`GatherRed` fetches 4 texels for the price of one when you need a neighborhood of one channel (PCF shadows, SSAO, edge detect).

## Fullscreen passes (post-processing)

- **Bandwidth-bound by default:** each pass reads and writes the whole screen. Merge compatible passes into one "uber" pass: tonemap + grade + vignette + grain + chromatic aberration + dither belong together.
- **Downsample** before wide blurs, bloom, SSAO, volumetrics, DoF. Half or quarter resolution plus a bilateral/depth-aware upsample is the standard.
- **Separable filters:** an N×N Gaussian becomes two N-tap passes, and **bilinear tap merging** cuts each pass to about N/2 fetches (see [shaders/post-fx-extras.md](shaders/post-fx-extras.md)).
- **Pyramids beat wide kernels:** mip-chain bloom and dual-Kawase blur get large radii from many cheap small-kernel passes.
- **Compute + groupshared** for very large kernels on desktop: load a tile plus apron once into shared memory.
- **Tile-based GPUs:** every render-target switch flushes the tile to memory. Set load/store actions so unneeded attachments are neither loaded nor stored (`DontCare`, memoryless depth on Metal). Avoid reading back the full-resolution target mid-frame.

## Vertex cost

- Vertex count × attribute bytes: strip unused channels (tangents, vertex colors, second UV set) and use compact formats (`half` UVs, `snorm8x4`/`unorm8x4` normals and colors, octahedral normals).
- Keep varyings/interpolators few; pack scalars into the `.w` of existing `vec4`s.
- Instancing and batching are CPU wins (fewer draw calls); they do not reduce per-vertex GPU cost.

## Variants and compilation hitches

- Every keyword combination is a separate compiled program. Unused variants cost build time and memory; used-for-the-first-time variants cause hitches at runtime.
- Strip what the project never enables. Warm the rest during loading: Unity `ShaderVariantCollection.WarmUp()` and PSO tracing/`GraphicsStateCollection` in Unity 6; three.js `renderer.compileAsync(scene, camera)`; PixiJS `renderer.prepare`.

## Correctness guards that also save time

- `normalize(vec3(0))`, `pow(negative, y)`, `log(0)`, `sqrt(negative)`, and division by zero produce NaN/Inf. A NaN fed into bloom spreads across the whole screen. Clamp inputs (`max(x, 1e-5)`) where they can reach zero or go negative.
- Output HDR values are bounded by the target format. Clamp before writing into `R11G11B10F` (no sign bit, max ~65000).

## Engine hooks

- **Unity URP:** SRP Batcher compatibility needs all per-material properties inside `CBUFFER_START(UnityPerMaterial)`. Use `half` in URP HLSL for colors and normals. Prefer `shader_feature` (stripped when unused) over `multi_compile`, and `dynamic_branch` for rarely toggled features. Shader Graph has per-node precision. See [unity/GUIDE.md](unity/GUIDE.md).
- **three.js:** `new WebGLRenderer({ precision: 'mediump' })` or `material.precision`, `onBeforeCompile` to patch chunks, `renderer.info.render` for draws/triangles, TSL for WebGPU. See [threejs/rules/shader-mobile.md](threejs/rules/shader-mobile.md) and [threejs/rules/postprocessing-optimization.md](threejs/rules/postprocessing-optimization.md).
- **PixiJS:** filter `resolution`, `padding` and `filterArea` bound the filtered pixel count. `TexturePool` provides scratch render textures. See [pixijs/pixijs-filters/GUIDE.md](pixijs/pixijs-filters/GUIDE.md) and [pixijs/pixijs-performance/GUIDE.md](pixijs/pixijs-performance/GUIDE.md).
