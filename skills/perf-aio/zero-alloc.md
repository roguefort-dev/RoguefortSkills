# Zero-allocation hot paths

**Target where allocation/GC is a measured bottleneck: 0 bytes allocated per steady-state frame, within retained-memory and latency budgets.** Allocate at load, spawn-wave, or level-change time and reuse while needed. A hot path is anything run per frame, per entity per frame, per physics step, or per event in a burst. Measure total retained memory and pool/scratch lifecycle as well as allocation rate; use [benchmarking.md](benchmarking.md) for full-path acceptance.

Allocation-free is a measured property. Code that "looks" allocation-free is only a hypothesis.

## Verify

| Stack | How to see per-frame allocations | How to lock it in |
|---|---|---|
| Unity | Profiler → CPU Hierarchy **GC Alloc** column, with allocation call stacks on. Memory module → **GC Allocated in Frame**. | `Assert.That(() => Tick(), Is.Not.AllocatingGCMemory())` (`UnityEngine.TestTools.Constraints`) in an edit- or play-mode test. At runtime, use `ProfilerRecorder.StartNew(ProfilerCategory.Memory, "GC Allocated In Frame")` and log when it is > 0. |
| .NET (non-Unity) | `GC.GetAllocatedBytesForCurrentThread()` before/after, dotnet-counters, BenchmarkDotNet `[MemoryDiagnoser]` | Benchmark or test asserting 0 B |
| Browser JS/TS | DevTools → Performance (look for *Minor GC* events and a saw-tooth JS heap), Memory → *Allocation sampling* / *Allocations on timeline* | Heap delta (`performance.memory.usedJSHeapSize`, Chrome only) across N frames in a perf test |
| Rust | `dhat`, or a counting global allocator in tests | Test asserting the count is unchanged |
| C++ | Heaptrack, Tracy memory profiling, overridden `operator new` counter | Same counter in tests |

Unity's incremental GC (Player Settings → Use Incremental GC; `gcIncremental: 1` in ProjectSettings.asset) spreads collection over frames, which shrinks spikes. It does not remove the cost; allocation still costs CPU and eventually collection time.

## Universal patterns

- **Preallocate + reuse:** clear a collection and reuse it, don't allocate a new one. Hold scratch objects in fields or module scope.
- **Object pools** for things with identity (bullets, enemies, VFX, UI rows): get/release plus a reset. Prewarm to the expected peak during loading. Cap the size.
- **Free-list / index handles** over object references for large homogeneous sets: store entities in arrays, hand out integer IDs, keep a free-index stack.
- **Flat typed storage** (`Float32Array`, `NativeArray<T>`, `T[]` of structs, `Vec<T>`) for particles and simulation can avoid per-element heap objects. Choose AoS/SoA/blocked layouts by access pattern through [cpu-data-and-simd.md](cpu-data-and-simd.md); dense identity/handle patterns are in [optimization-patterns.md](optimization-patterns.md).
- **Ring buffers** for event queues, trails and history.
- **Out-parameters / "into" APIs:** `add(a, b, out)` rather than `return new Vec(...)`.
- **Swap-remove** (`items[i] = items[last]; last--`) instead of splice/RemoveAt in the middle when order doesn't matter.

## C# / Unity (C# 9, .NET Standard 2.1)

| Allocates | Allocation-free alternative |
|---|---|
| Lambda capturing locals/`this` (closure + delegate on every call) | A delegate cached in a field, or a `static` lambda (C# 9) with state passed explicitly |
| Method group passed as a delegate: `list.ForEach(Handle)`, `Sort(Compare)` | Cache `Action<T> _handle = Handle;` once. C# 9 allocates a new delegate per conversion; the caching of static method groups arrived in C# 11. |
| LINQ (`Where`, `Select`, `Any`, `ToList`, `OrderBy`) | `for` loops over `List<T>`/arrays |
| `foreach` over an interface (`IEnumerable<T>`, `IList<T>`, `IReadOnlyList<T>`) — boxes the enumerator | `foreach` over the concrete `List<T>`/array (struct enumerator), or indexed `for` |
| Boxing: value type → `object`/interface, `string.Format(fmt, int)`, non-generic collections, `Enum.HasFlag` on older runtimes | Generic APIs; bit tests `(flags & X) != 0`; custom `IEqualityComparer<TEnum>` if the profiler shows enum-key boxing |
| `params T[]` calls | Fixed-arity overloads |
| String concat, interpolation, `ToString()`, `name`, `tag` getters | `CompareTag`, cached strings, `StringBuilder` reused, TextMeshPro `SetText(string format, float a, ...)` / `SetText(StringBuilder)` |
| Unity APIs returning arrays: `GetComponents<T>()`, `GetComponentsInChildren<T>()`, `mesh.vertices`, `Physics.RaycastAll`, `OverlapSphere` | List-taking overloads (`GetComponents(List<T>)`), `mesh.GetVertices(List<Vector3>)`, `Physics.RaycastNonAlloc` / `OverlapSphereNonAlloc` with a preallocated buffer |
| `yield return new WaitForSeconds(x)` | Cache the `WaitForSeconds` instance; `yield return null` is free |
| `async Task` state machines | Unity 6 `Awaitable` (pooled), or plain update-driven state |
| `Instantiate`/`Destroy` per event | `UnityEngine.Pool.ObjectPool<T>`; `ListPool<T>.Get()`/`Release` for temporary lists |
| `new T[n]` temp buffers | `stackalloc` into `Span<T>` (small, fixed-size), `ArrayPool<T>.Shared.Rent/Return`, or a reused field |
| `Debug.Log` in shipped paths | `[Conditional("UNITY_EDITOR")]` wrappers or define-stripped logging |

Jobs/Burst/`NativeArray` avoid the managed heap entirely, but they need the Collections/Burst packages. Confirm they are in the project manifest before recommending them.

## JavaScript / TypeScript (V8, SpiderMonkey, JSC)

Allocation per frame shows up as frequent minor GCs (scavenges). Each is short, but together they cause frame-time jitter, and promoted garbage triggers major GCs.

| Allocates | Allocation-free alternative |
|---|---|
| Object/array literals in the loop (`{x, y}`, `[a, b]`, returning tuples) | Module-scope scratch objects; out-params |
| Math-library temporaries: `new THREE.Vector3()`, `v.clone()`, `new Point()` | Module-level `const _v = new Vector3()` reused; `copy`, `set`, `addScaledVector`, `lerpVectors` |
| `map`/`filter`/`slice`/`concat`/spread/`Object.entries` | Index `for` loops writing into preallocated arrays; `array.length = 0` to reuse |
| Closures created per call (`arr.forEach(x => ...)` inside `update`) | Hoisted named functions, or a plain `for` loop |
| String building/template literals for keys | Numeric keys, or strings interned at init |
| Iterators over `Map`/`Set` (`for...of`, `.forEach`) in very hot paths | Parallel arrays or an array of values maintained next to the Map |
| Growing arrays by `push` beyond capacity | Preallocate to the expected max, or use typed arrays with a count |

What does **not** allocate: local numeric variables and `const` numbers (small integers and doubles in locals live in registers/stack after JIT). Don't "optimize" them away. Stable object shapes also matter for speed, though not for allocation: initialize every field in the constructor and never `delete` properties.

Library pointers: three.js pooling in [threejs/rules/object-pooling.md](threejs/rules/object-pooling.md); PixiJS pooling, `destroy` patterns and the GC systems in [pixijs/pixijs-performance/GUIDE.md](pixijs/pixijs-performance/GUIDE.md).

## Rust

`Vec::with_capacity` and `clear()`-and-reuse, over collecting into a new `Vec` each frame. Use `SmallVec`/`ArrayVec` for small bounded sets and a bump/arena allocator (`bumpalo`) reset per frame for frame-scoped data. Avoid `clone()` and `to_string()`/`format!` in loops. Iterator chains are allocation-free until `collect()`.

## C++

`reserve()` up front and `clear()` to reuse (it keeps capacity). Use `std::pmr::monotonic_buffer_resource` or a custom linear allocator reset per frame, `std::array`/small-buffer containers, and `emplace_back` into reserved vectors. Keep `std::function` holding large captures out of hot paths (it may heap-allocate); use templates or function pointers instead.
