# Sources

Original synthesis for RoguefortSkills, written 2026-10-23 from two research reports (a WebGPU-versus-WebGL2 capability catalogue and a WebGPU optimization field guide, both reviewed 2026-10-22) and the primary documentation below. No upstream text or code is copied. MIT-licensed; see [licenses/roguefortskills-MIT.txt](licenses/roguefortskills-MIT.txt).

Mechanisms come from the specifications and API references. Workload suggestions, tradeoffs and experiments are engineering inference from those mechanisms. The 8–24% one-to-one port slowdown and the depth-read observation are measurements from one fill-bound PixiJS/Pixi3D renderer on Apple silicon (Chrome, Dawn/Metal) and are examples, not general results.

## Specifications

- [WebGPU](https://www.w3.org/TR/webgpu/) and [WGSL](https://www.w3.org/TR/WGSL/) (W3C)
- [WebGPU explainer](https://gpuweb.github.io/gpuweb/explainer/): command cost, buffer mapping, threading
- [WebGL 2.0](https://registry.khronos.org/webgl/specs/latest/2.0/), [GLSL ES 3.00](https://registry.khronos.org/OpenGL/specs/es/3.0/GLSL_ES_Specification_3.00.pdf), [WebGL 2.0 quick reference](https://www.khronos.org/files/webgl20-reference-guide.pdf), [WebGL extension registry](https://registry.khronos.org/webgl/extensions/)
- WebGL extensions: [WEBGL_multi_draw](https://registry.khronos.org/webgl/extensions/WEBGL_multi_draw/), [KHR_parallel_shader_compile](https://registry.khronos.org/webgl/extensions/KHR_parallel_shader_compile/), [EXT_disjoint_timer_query_webgl2](https://registry.khronos.org/webgl/extensions/EXT_disjoint_timer_query_webgl2/), [WEBGL_shader_pixel_local_storage](https://registry.khronos.org/webgl/extensions/WEBGL_shader_pixel_local_storage/)

## API reference

- MDN: [GPUSupportedFeatures](https://developer.mozilla.org/en-US/docs/Web/API/GPUSupportedFeatures), [GPUSupportedLimits](https://developer.mozilla.org/en-US/docs/Web/API/GPUSupportedLimits), [GPUComputePassEncoder](https://developer.mozilla.org/en-US/docs/Web/API/GPUComputePassEncoder), [dispatchWorkgroupsIndirect](https://developer.mozilla.org/en-US/docs/Web/API/GPUComputePassEncoder/dispatchWorkgroupsIndirect), [drawIndirect](https://developer.mozilla.org/en-US/docs/Web/API/GPURenderPassEncoder/drawIndirect), [GPURenderBundle](https://developer.mozilla.org/en-US/docs/Web/API/GPURenderBundle), [GPUBindGroup](https://developer.mozilla.org/en-US/docs/Web/API/GPUBindGroup), [GPUBuffer.usage](https://developer.mozilla.org/en-US/docs/Web/API/GPUBuffer/usage), [createRenderPipelineAsync](https://developer.mozilla.org/en-US/docs/Web/API/GPUDevice/createRenderPipelineAsync), [beginRenderPass](https://developer.mozilla.org/en-US/docs/Web/API/GPUCommandEncoder/beginRenderPass), [resolveQuerySet](https://developer.mozilla.org/en-US/docs/Web/API/GPUCommandEncoder/resolveQuerySet)
- GPUWeb types: [GPURenderPassEncoder](https://gpuweb.github.io/types/interfaces/GPURenderPassEncoder.html), [GPUSupportedLimits](https://gpuweb.github.io/types/interfaces/GPUSupportedLimits.html)
- Tour of WGSL: [workgroup variables](https://google.github.io/tour-of-wgsl/variables/var-workgroup/), [atomic types](https://google.github.io/tour-of-wgsl/types/atomics/atomic-types/)

## Implementation notes

- [From WebGL to WebGPU](https://developer.chrome.com/docs/web-platform/webgpu/from-webgl-to-webgpu) (Chrome)
- [WebGPU is now supported in major browsers](https://web.dev/blog/webgpu-supported-major-browsers) (web.dev)
- [What's new in WebGPU](https://developer.chrome.com/docs/web-platform/webgpu/news): [120](https://developer.chrome.com/blog/new-in-webgpu-120), [123](https://developer.chrome.com/blog/new-in-webgpu-123), [129](https://developer.chrome.com/blog/new-in-webgpu-129), [134](https://developer.chrome.com/blog/new-in-webgpu-134), [146](https://developer.chrome.com/blog/new-in-webgpu-146), [149–150](https://developer.chrome.com/blog/new-in-webgpu-149-150), [151–152](https://developer.chrome.com/blog/new-in-webgpu-151-152)
- [WebAssembly and WebGPU enhancements, part 2](https://developer.chrome.com/blog/io24-webassembly-webgpu-2) (Chrome): reduced precision and bandwidth
