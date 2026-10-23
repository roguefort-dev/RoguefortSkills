# Feature detection and fallbacks

Four kinds of support, each checked separately. An adapter's support enables nothing until it is requested on the device.

| Kind | Check | Enable |
|---|---|---|
| Device features (`shader-f16`, `subgroups`, `timestamp-query`, …) | `adapter.features.has(name)` | `requiredFeatures` in `requestDevice`, then `device.features.has(name)` |
| WGSL language features (`packed_4x8_integer_dot_product`, `immediate_address_space`, …) | `navigator.gpu.wgslLanguageFeatures.has(name)` | `requires name;` in WGSL where needed |
| Limits | `adapter.limits.x` | `requiredLimits`, then read `device.limits.x` |
| Newer API surfaces | presence test, e.g. `'TRANSIENT_ATTACHMENT' in GPUTextureUsage` | use directly |

## Device request

```js
async function requestGpu() {
  const adapter = await navigator.gpu?.requestAdapter({ powerPreference: 'high-performance' }).catch(() => null);
  if (!adapter) return null;                                   // → WebGL2 path
  const wanted = ['shader-f16', 'subgroups', 'timestamp-query', 'rg11b10ufloat-renderable',
    'dual-source-blending', 'indirect-first-instance', 'float32-filterable'];
  const device = await adapter.requestDevice({
    requiredFeatures: wanted.filter(f => adapter.features.has(f)),
    requiredLimits: { maxComputeWorkgroupStorageSize: adapter.limits.maxComputeWorkgroupStorageSize,
                      maxStorageBufferBindingSize: adapter.limits.maxStorageBufferBindingSize },
  }).catch(() => null);
  if (!device) return null;
  device.lost.then(info => { if (info.reason !== 'destroyed') scheduleRecreate(); });
  return {
    device,
    f16: device.features.has('shader-f16'),
    subgroups: device.features.has('subgroups'),
    packedDot: navigator.gpu.wgslLanguageFeatures.has('packed_4x8_integer_dot_product'),
    transient: typeof GPUTextureUsage !== 'undefined' && 'TRANSIENT_ATTACHMENT' in GPUTextureUsage,
  };
}
```

- Request only what a code path uses; each requested limit or feature narrows the devices that succeed. A feature the renderer cannot run without (for example `indirect-first-instance` for engines that need it) belongs in the adapter check, falling back to WebGL2.
- `enable f16;` and `enable subgroups;` are directives, not `override`s: build two shader variants (string or module selection) and choose at pipeline creation.
- Size dispatches and buffers from `device.limits`, not from the defaults below.
- Keep one fallback per optional path, written and tested: `f32` for `f16`, workgroup memory for subgroups, `rgba16float` for 11-bit targets, stored-and-discarded attachments for transient ones.

## Default limits (core)

| Limit | Default |
|---|---|
| `maxBufferSize` | 256 MiB |
| `maxStorageBufferBindingSize` | 128 MiB |
| `maxUniformBufferBindingSize` | 64 KiB |
| `maxStorageBuffersPerShaderStage` | 8 |
| `maxStorageTexturesPerShaderStage` | 4 |
| `maxBindGroups` | 4 |
| `maxColorAttachments` | 8 |
| `maxColorAttachmentBytesPerSample` | 32 |
| `maxInterStageShaderVariables` | 16 |
| `maxComputeWorkgroupStorageSize` | 16,384 bytes |
| `maxComputeInvocationsPerWorkgroup` | 256 |
| `maxComputeWorkgroupSizeX / Y / Z` | 256 / 256 / 64 |
| `maxComputeWorkgroupsPerDimension` | 65,535 |
| `maxImmediateSize` | 64 bytes |

Compatibility-mode devices (`core-features-and-limits` absent) have lower limits and extra restrictions; do not assume the core resource model there.

## Optional device features

`atomic-vec2u-min-max`, `bgra8unorm-storage`, `clip-distances`, `core-features-and-limits`, `depth-clip-control`, `depth32float-stencil8`, `dual-source-blending`, `float32-blendable`, `float32-filterable`, `indirect-first-instance`, `primitive-index`, `rg11b10ufloat-renderable`, `shader-f16`, `subgroups`, `subgroup-size-control`, `texture-component-swizzle`, `texture-compression-astc`, `texture-compression-astc-sliced-3d`, `texture-compression-bc`, `texture-compression-bc-sliced-3d`, `texture-compression-etc2`, `texture-compression-unaligned`, `texture-formats-tier1`, `texture-formats-tier2`, `timestamp-query`.

## Shipping versions (Chrome)

| Release | Addition |
|---|---|
| 113 | WebGPU on desktop (Android from 121) |
| 120 | `shader-f16` |
| 121 | timestamp queries |
| 123 | packed 4×8 integer dot products |
| 124 | read-only and read-write storage textures |
| 129 | HDR canvas tone mapping (`extended`) |
| 130 | dual-source blending |
| 131 | clip distances |
| 132 | `float32-blendable` |
| 134 | subgroups |
| 142 | texture format tiers, `primitive-index` |
| 143 | texture component swizzle |
| 146 | transient attachments, compatibility mode |
| 149–150 | immediates |
| 151–152 | subgroup size control |

Other browsers: Safari 26 (macOS, iOS, iPadOS, visionOS); Firefox 141 on Windows and 145 on Apple-silicon macOS. Feature sets differ per browser and device; detect at runtime rather than by version.
