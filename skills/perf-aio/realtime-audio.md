# Real-time audio and hard-deadline callbacks

Use for audio callbacks, DSP graphs, synthesizers, AudioWorklets, MIDI/control threads, or any callback that must finish before a fixed deadline. The budget is the worst case, not the average: one late callback is an audible glitch (xrun/underrun). The deadline is `bufferFrames / sampleRate` (for example 128 frames at 48 kHz ≈ 2.67 ms), shared with every other node in the graph and the host. Use [benchmarking.md](benchmarking.md) for the measurement contract and [cpu-data-and-simd.md](cpu-data-and-simd.md) for the inner-loop work.

## Measure against the deadline

- Time each callback on the real thread and record the maximum and high percentiles against the buffer period. Log from a preallocated ring read by a non-real-time thread, never from the callback itself.
- Count xruns/underruns over sustained sessions, with realistic voice/plugin counts, parameter automation, UI activity and background load. Test the smallest buffer size you support; small buffers expose jitter that large buffers hide.
- Compare cold first-callback behavior (page faults, lazy initialization, JIT tiering) separately from steady state, and remove it by preparing before playback starts.

## Nothing unbounded on the audio thread

Each item below can block for an unbounded time or wait on a lower-priority thread. Treat any occurrence in the callback path as a defect until measurement proves its bound.

| Avoid in the callback | Use instead |
|---|---|
| Heap allocation/free, GC-producing code, growing containers | Preallocate at prepare/start time for the maximum block size and voice count; fixed pools; see [zero-alloc.md](zero-alloc.md) |
| Mutexes, condition variables, waiting on another thread | Lock-free single-producer/single-consumer rings; atomics for scalar parameters; a `try_lock` with a defined fallback only when skipping the update is acceptable |
| File/network I/O, logging, console output, system calls | A non-real-time worker thread that streams through a ring; preloaded or prefetched sample data |
| Untouched memory (first-touch page faults) | Touch/zero buffers before playback; lock memory where the platform supports it and the budget justifies it |
| Calls into code with unknown cost (UI, scripting, generic library code) | Calls whose implementation and worst case you know |

Priority inversion: a lock that the audio thread shares with a lower-priority thread lets that thread delay the callback for as long as it holds the lock. Ownership transfer and lock-free handoff remove the shared wait. Concurrent containers still need the lifetime and memory-order contracts in [concurrency-and-pipelines.md](concurrency-and-pipelines.md).

## Handing state to the audio thread

- **Scalar parameters:** one atomic per value, read once per block. Smooth toward the target (per-sample or per-block ramps) so stepped changes do not click; ramp coefficients are part of the numerical contract.
- **Larger state (filter tables, wavetables, sample maps, graph topology):** build it off the real-time thread, then publish a pointer with an atomic swap or a triple buffer. Free the old version on a non-real-time thread after the audio thread has released it (deferred reclamation); freeing on the audio thread brings the allocator back.
- **Events (notes, MIDI, automation):** timestamped messages in a bounded SPSC ring, applied at their sample offset inside the block. Define overflow behavior; dropping a note-off is a correctness bug.

## DSP inner loops

- Process in blocks: hoist per-block work (coefficient updates, parameter reads, branch decisions) out of the per-sample loop, and keep per-sample loops homogeneous so they vectorize. Voices or channels in SoA layout let one SIMD lane handle one voice.
- **Denormals:** decaying feedback paths (IIR filters, reverbs, envelopes, delay feedback) drift into subnormal floats, which take a slow microcode path on many CPUs, so CPU time can spike exactly when the signal fades to silence. Set flush-to-zero/denormals-are-zero on the audio thread where the platform allows (x86 MXCSR FTZ/DAZ, Arm FPCR FZ; these flags are per thread). Where you cannot (for example WebAssembly), flush small values explicitly or add an inaudible offset in feedback paths, then verify by profiling a decaying tail.
- Feedback recursions (IIR, one-pole smoothing) are loop-carried dependencies: vectorize across voices or channels, not across samples, or use a block-parallel formulation whose numerical behavior you have checked.
- Replace expensive per-sample functions (`sin`, `exp`, `pow`, `tan`) with wavetables, recurrences or polynomial approximations within the audible error budget. Check aliasing and drift over long sessions, not only per-sample error.

## Web Audio specifics

- `AudioWorkletProcessor.process()` runs on the audio rendering thread with a 128-frame render quantum by default. Allocate nothing per call: reuse typed arrays, avoid closures, array literals and per-call `postMessage`. Create objects in the constructor.
- `postMessage` copies or transfers data and is delivered asynchronously; use it for infrequent control. For streaming or high-rate data, use a `SharedArrayBuffer` ring with `Atomics`, which requires cross-origin isolation (COOP/COEP headers).
- Heavy DSP can move to WebAssembly (with SIMD where supported) called from `process()`. Keep the WASM memory preallocated and avoid crossing the JS/WASM boundary per sample.

## Verification

Test the smallest supported buffer size, maximum polyphony, rapid automation, preset/graph changes during playback, silence after loud input (the denormal tail), long sessions, and a loaded system. Accept a change when the worst-case callback time and xrun count improve on sustained runs and the output matches the reference within its tolerance.

Cross-domain transfer: frame-budget pacing → callback deadline budgets; double-buffered render resources → atomically swapped DSP state; GPU ring buffers → SPSC audio/event rings.

## Primary references

- [Real-time audio programming 101](http://www.rossbencina.com/code/real-time-audio-programming-101-time-waits-for-nothing): why locks, allocation and I/O break audio deadlines.
- [AudioWorkletProcessor.process()](https://developer.mozilla.org/en-US/docs/Web/API/AudioWorkletProcessor/process): render quantum and processing contract.
- [SharedArrayBuffer security requirements](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer#security_requirements): cross-origin isolation for shared memory.

Original synthesis for this bundle; examples are hypotheses, with no assumed speedup.
