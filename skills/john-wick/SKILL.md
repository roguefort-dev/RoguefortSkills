---
name: john-wick
description: Codebase cleanup sweep. Use when asked to clean up, audit or untangle a codebase, or to find implicit state machines — boolean flag soup (isLoading/isError/hasLoaded), status strings or enums mutated from many places, nullable fields or timestamps that encode a phase, init/dispose ordering guards, step counters, parser mode flags, retry/reconnect lifecycles. Finds and reports every instance with locations, reconstructed states and transitions, and a suggested explicit model. Reports only; does not edit code unless asked.
---

# John Wick

A cleanup sweep that finds every target in the codebase and leaves none behind. Each kind of target is a **pass** with its own guide. Load only the guide(s) for the passes being run.

## Method

Every pass follows the same loop. Each step ends on its criterion:

1. **Scope** the codebase: languages, frameworks, entry points, and what to skip (vendored, generated, build output, lockfiles, migrations history). Done when the included paths are listed. Tests are read as evidence, not reported as findings.
2. **Sweep** with the pass's lexical signals across every included path. Done when each signal has been searched in each included language and every hit is recorded against its owner (class, module, component, store, record or table).
3. **Confirm** each candidate by reading the code: reconstruct what the pass asks for and apply its confirmation criteria. Done when every candidate is marked confirmed, likely, or rejected with a one-line reason.
4. **Report** findings ranked by severity using the pass's report format. Done when every confirmed and likely finding has file:line locations and nothing found in the sweep is silently dropped (rejections are summarized by reason).

Rules for every pass:

- **Find, don't fix.** A sweep reports. Change code only when the user asks; then follow the pass's hand-off section, one finding at a time, with behavior preserved.
- **Be exhaustive, not noisy.** Cover the whole included scope; for large codebases, sweep per top-level area and say which areas were covered. A finding needs evidence from the code, not a naming hunch.
- **Cite precisely.** Every claim points at `path:line`. Quote at most the few lines that prove it.
- **Match the codebase.** Suggested fixes use the language's and project's own idioms and existing libraries before introducing new ones.

## Passes

| Pass | Finds | Load |
|---|---|---|
| Implicit state machines | Lifecycles and modes encoded in loose flags, nullable fields, status strings, counters or call-order rules instead of an explicit state type with defined transitions | [implicit-state-machines.md](implicit-state-machines.md) |

With no pass named, run all passes.

Changes: [docs/CHANGELOG.md](docs/CHANGELOG.md). License: [MIT](licenses/roguefortskills-MIT.txt).
