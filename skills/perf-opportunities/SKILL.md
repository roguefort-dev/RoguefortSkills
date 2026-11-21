---
name: perf-opportunities
description: Run a complete performance review of a codebase (or a directory, file, or diff), reading every source file and cross-referencing the code against a library of 4,300+ optimization techniques mined from the commit histories of Redis, nginx, SQLite, RocksDB, HAProxy, DuckDB, PostgreSQL, Seastar, curl, Netty, Tokio and ten console emulators, then write every optimization opportunity found to a plain performance-opportunities.html report. Use when the user asks to "find optimization opportunities", "what could be faster here", "performance audit", "perf review", "where is this slow", or wants a performance opportunity document. This skill finds and documents opportunities; it does not apply them unless asked.
---

# Performance opportunities

You are reviewing code for ways to make it faster or lighter, and writing up everything you find. Two things define this review.

It is complete. Every source file in scope gets read, and every place where a known technique applies gets reported, whether or not that code is on a hot path. The user wants the full inventory of what could be optimized and will decide for themselves what is worth doing. Do not skip code because it runs rarely, and do not drop an opportunity because its gain looks small; record how often the code runs so the reader can weigh it.

It is grounded. Every opportunity cites a technique that a mature, performance-critical project actually shipped. The `references/` folder holds 4,308 such techniques, each a domain-generic principle with the evidence of how the source project used it. Your job is to read the target code closely, find every site whose code meets a technique's precondition, and document each match so a developer can act on it.

The reader of the report is a developer who knows their codebase but has not seen the technique library. They will decide what to implement from your document alone, so each entry has to show the exact code, say what would change, and be honest about how sure you are.

## The technique library

- `references/README.md` lists the lenses and the source projects with counts.
- `references/index/<lens>.md` has one line per technique: `id | title | first sentence of the principle`. There are 24 lenses (algorithms, laziness, caching, allocation, memory-layout, concurrency, io, simd, and so on). These are what you scan.
- `references/cards/<project>.md` has the full card for every technique from that project. The id prefix is the file name: `redis-042` lives in `cards/redis.md`. Each card is exactly seven lines (heading, lens and source, principle, why it is faster, where else it applies, pitfall, how the source project used it), so `grep -A 6 "^### redis-042 " references/cards/redis.md` returns one whole card. Cards from the emulator catalogues have no "why" or "watch out" text and show a dash there.

The index files total about 660 KB and the cards about 4 MB. Read index files whole; pull cards by id. Reading a whole cards file is rarely worth it.

The source projects are mostly C and C++ systems software, plus Netty (Java/JVM) and Tokio (Rust), but the principles were written to be domain-generic and each card names applications elsewhere (web backends, frontends, data pipelines, embedded). A technique from an emulator's renderer can be the right answer for a React list or a Python ETL job. Match on the shape of the work, and ignore the source domain.

## How to run the review

### 1. Inventory the code

Work out the scope from the user's request: the whole repository by default, or the directory, file or diff they named. List every source file in scope, along with build and bundler configuration, database schemas and queries, and shaders or other assets that carry logic. Exclude only vendored dependencies, generated code and build output, and say so in the report.

Group the files into units you can review one at a time (a module, a package, a subsystem). This list is your coverage checklist: the review is finished when every unit has been read and cross-referenced, and the report shows the list so the reader can see nothing was skipped.

Also learn what the program does and what "fast" means for it (throughput, latency, frame time, startup, memory, battery, bundle size), since that tells you which costs the reader cares about.

### 2. Read each unit and note what it does

Read every file in the unit in full. Skimming finds only the obvious; the less obvious opportunities sit in ordinary-looking code. As you read, note each piece of code that does any of the following, with file and line:

- iterates, searches, sorts, or looks things up
- computes something that could be reused, skipped, or deferred
- allocates, copies, converts, formats, or builds strings and collections
- branches on conditions, dispatches dynamically, or handles a general case
- crunches numbers, bytes, pixels or samples
- takes locks, spawns tasks, awaits, or shares state
- touches files, databases, sockets, the network, or another process
- interprets, parses, serializes, or generates code
- draws, lays out, or updates UI
- runs at startup, on a timer, or in response to events

For each, note how often it runs (per frame, per request, per item, at startup, on a rare user action, once) and how large its inputs can get. This is recorded for the reader's benefit and is never grounds for skipping a site. If profiler output or benchmarks exist, use them for these notes; otherwise infer from the code and say so.

### 3. Cross-reference against the library

Choose the lenses that have any bearing on the unit. For a complete review this is most of them; leave out only lenses whose subject matter is absent from the entire scope (for example `gpu-rendering` for a headless service, or `storage-engine` for a project with no persistence). When unsure, read the lens.

| What the code does | Lenses |
|---|---|
| Loops over collections, searches, sorts, nested iteration, lookups by key | algorithms, lookup-tables |
| Recomputes things that rarely change; does work whose result may go unused | laziness, caching |
| Creates objects, strings, buffers or closures | allocation, memory-layout |
| Handles many small items, calls or writes one at a time | amortization, batch-processing |
| Branch-heavy code, generic paths handling a dominant common case | branching-fast-paths, cpu-microopt |
| Numeric or byte-crunching kernels, pixel/audio/array loops | simd, cpu-microopt, compiler-build |
| Threads, async tasks, locks, queues, shared state | concurrency, scheduling |
| Files, databases, sockets, HTTP, serialization | io, networking, storage-engine, compression |
| Query builders, ORMs, planners, rule engines | query-planning, algorithms |
| Interpreters, VMs, template engines, expression evaluators, code generators | jit-interpreters, lookup-tables |
| Rendering, canvas/WebGL/GPU, UI redraw, layout | gpu-rendering, laziness |
| Build configuration, bundlers, compiler flags, target platforms | compiler-build, platform |
| Instrumentation, logging, metrics, benchmarks | observability |
| Anything | other |

Read each chosen `references/index/<lens>.md` from first line to last. For every technique line, ask whether its precondition describes any site in your notes, and record each `(technique id, code location)` match. Work in both directions: after going through the index, go back through your notes and ask of each site whether any technique you have read covers it. A site that looks improvable but has no match yet is a reason to check another lens.

Then verify each match:

1. Pull the card and read the principle, the pitfall and the source project's usage.
2. Go back to the code and confirm the precondition actually holds. Read the function and its callers; check whether the result is really recomputed, reallocated or searched the way you think. Search for an existing implementation of the same idea elsewhere in the codebase, since a cache or pool may already exist one layer up.
3. Check the card's pitfall against this code and carry it into the report as the stated risk.

Drop a match only when it is wrong: the technique is already in place, the precondition does not hold on a close read, or the change would alter behaviour the program depends on. Do not drop it for being cold or small. Every reported entry must be true, because a false entry costs the reader an investigation; within that constraint, report everything.

Several techniques often point at the same change; record them as one opportunity citing all the ids. When one technique applies at several sites, write one entry listing every location.

On anything beyond a small codebase, delegate if you can: one subagent per unit, each told to read every file in its unit, given the path to this skill's `references/` folder and the lenses to read in full, and asked to return every verified match with file, line, technique ids, frequency notes and reasoning. Tell them the review is exhaustive and cold code counts. Check that each returned its whole unit, and spot-check their findings against the code before they go in the report.

### 4. Assess each opportunity

For every opportunity, settle:

- **Runs**: how often the code executes and on what input sizes (per frame, per request, at startup, rarely).
- **Impact** (high, medium, low): how much cost it removes, considering both the saving per execution and how often it runs, and by what mechanism. State the mechanism ("removes one allocation per row", "turns an O(n²) pass over the open orders into O(n)"). Give a number only if you measured it or the code makes it derivable; otherwise describe the expected direction and scale in words. Invented percentages mislead.
- **Confidence** (high, medium, low): how sure you are that the precondition holds. Say what you verified and what you assumed.
- **Effort** (small, medium, large): rough size and invasiveness of the change.
- **Risk**: what could go wrong, drawing on the card's pitfall, plus any behaviour change the user would see.
- **How to verify**: the measurement that would confirm the win, such as a specific benchmark, a counter, or a profile of a named scenario.

Order the report by impact, then confidence, with effort as the tie-breaker. Ordering is the only use of these ratings; low-impact entries stay in.

### 5. Write the report

Write `performance-opportunities.html` in the root of the reviewed project unless the user names another location. Copy the structure of `assets/report-template.html`: plain semantic HTML with no stylesheet, no scripts and no external resources. Browser defaults are the intended look.

The report contains, in order:

1. **Header**: project, scope reviewed, date, the number of opportunities, and whether frequency notes are measured or inferred.
2. **Summary table**: one row per opportunity, ordered, with number, title, location, runs, impact, confidence and effort, each linking to its detail section.
3. **Opportunities**: one section each, with
   - location as `path/to/file.ext:line` (every site if several),
   - what the code does now, with a short excerpt of the actual code,
   - the technique or techniques: id, title and the principle in a sentence, plus one line on how the source project used it,
   - the proposed change, concrete enough to implement, with a sketch of the new code where that helps,
   - runs, impact with its mechanism, confidence, effort, risk, and how to verify.
4. **Coverage**: every unit reviewed with its file count and number of opportunities, plus what was excluded (vendored, generated).
5. **Method note**: which lenses were read and what was and was not measured.

Write up opportunities found. In the lens list, name only the lenses that were read; do not enumerate techniques that did not apply. Escape `<`, `>` and `&` in code excerpts.

Before finishing, check the coverage list against the inventory from step 1. If any unit was not fully read, go back and review it, or state plainly in the report which files were not covered.

### 6. Reply to the user

After writing the file, tell the user where it is, how many opportunities it holds, and the top few in a sentence each. Do not modify the reviewed code unless the user asked for the fixes as well; if they then ask you to implement an opportunity, measure before and after where a measurement is available.

## Adding to the library

The references are generated from HTML catalogues by `C:\Users\kato\Documents\Optimization findings\skill\build-references.mjs` (run with `node`). It reads every `<Project>/index.html` in that folder and rewrites `skill/perf-opportunities/references/`. If the user adds a catalogue, rerun the script and copy `skill/perf-opportunities` over the installed copies in `~/.claude/skills/` and `~/.codex/skills/`.
