# Changelog

## 2026-10-23

- Added the WebGPU optimization skill: a router with a measure → bound → capability → gate → validate method.
- `capabilities.md` catalogues what WebGPU offers over WebGL2 across compute, storage and atomics, GPU-driven work, per-draw CPU cost, uploads and readback, render passes, WGSL, formats and debugging, each with its WebGL2 equivalent.
- `techniques.md` covers designs these enable, with code for GPU culling with indirect draws, hardware depth testing in place of shader depth reads, and fixed-point atomic accumulation.
- `feature-detection.md` covers device features, WGSL language features, limits, newer API surfaces, fallbacks and shipping versions.
- `porting-and-measurement.md` maps bottlenecks to first experiments, porting pitfalls, where WebGL2 is equal or ahead, GPU timestamps and a validation checklist.
