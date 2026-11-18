# What's new

## 18 November 2026

Perf AIO now covers real-time audio. It helps keep audio callbacks within their deadline by keeping locks, allocation and I/O off the audio thread, passing parameters and state to it safely, and preventing CPU spikes from very small floating-point values as sound fades out. It includes guidance for Web Audio worklets.

The CPU guide also covers instruction-level costs: dependency chains, expensive math, lookup tables, pointer chasing and code layout.

## 3 October 2026

Perf AIO now helps optimize compute shaders and GPU kernels across GPU vendors. It guides realistic measurements, memory reuse, workgroup tuning, and correctness checks, and keeps hardware-specific choices tied to the devices being tested.

It also looks for useful optimization ideas across domains, creatively adapting patterns from kernels, SIMD, graphics, and other areas wherever they fit, then checking correctness and measuring the result.

New guides cover data-oriented design, avoiding unnecessary work, CPU/SIMD processing, parallel jobs and pipelines, shared algorithms, realistic benchmarking, and runtime tuning. Optimization choices account for speed, memory, correctness, and responsiveness.
