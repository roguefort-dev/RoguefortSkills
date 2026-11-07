# Implicit state machines

Use to find every place where a lifecycle, mode or protocol is encoded in loose variables instead of an explicit state type. The code still behaves like a state machine: it has states, events and legal transitions. It just never says so, so invalid combinations are representable, transitions are scattered, and ordering rules live in comments or in nobody's head.

An implicit state machine is a **cluster** of evidence around one owner, not a single flag. One `isOpen` is a boolean; `isOpen` + `isClosing` + `closeTimer` written from four handlers is a machine.

## Kinds

| Kind | Looks like | Typical home |
|---|---|---|
| Flag soup | Several booleans describing one lifecycle: `isLoading`, `isError`, `hasLoaded`, `isSubmitting` | UI components, stores, view models |
| Unowned status | A status enum/string written from many sites, any value to any value, no transition table | Services, entities, game objects |
| Presence-encoded phase | Which nullable fields are set means the phase: `data`/`error` both null = loading; `cancelledAt`, `shippedAt` | Records, DTOs, DB rows, async results |
| Temporal coupling | Call-order rules: `init()` before `use()`, nothing after `dispose()`; enforced by flag checks, throws, comments or not at all | Clients, engines, resources, plugins |
| Counter phase | An integer that selects behavior: `step`, `stage`, `attempt`, `page` with branches per value | Wizards, onboarding, retries, tutorials |
| Mode flags in loops | Hand-rolled lexers, parsers and input handlers: `inString`, `escaped`, `inComment`, `dragging` | Parsers, tokenizers, gesture/input code |
| Async lifecycle | In-flight, cancelled and stale-response guards; retry/backoff; connect/reconnect | Fetching, sockets, workers, jobs |
| Distributed machine | Transitions spread across effects, handlers or services that react to each other's writes | React `useEffect` chains, event buses, webhooks updating a status column |
| Explicit but bypassed | A real enum/reducer/statechart exists, but some code writes the state directly or skips the transition function | Anywhere a machine was introduced later |

## Sweep signals

Run each signal from the repository root over source files only: restrict to the project's languages with `-t` and exclude tests, docs, fixtures, vendored and generated code with `-g '!…'`. On large codebases, docs and tests can outnumber source hits ten to one. Keep the explicit `.` path: without one, `rg` searches stdin when it is not a terminal, which is common in agent shells. Hits are candidates; clustering and confirmation decide. Start from the state-type and phase-literal signals: they have the fewest false positives and lead straight to owners.

```sh
# Declared state types: the fields of these types are the first owners to check
rg -n '\b(type|enum|class|interface|struct|record)\s+\w*(Status|State|Phase|Stage|Mode|Lifecycle)\b' .
# Boolean phase flags (camelCase and snake_case), declarations and assignments
rg -n '\b(is|has|was|did|should)(_[a-z]|[A-Z])\w*\s*(=|:)[^=]' .
# React/Vue/Svelte state hooks holding booleans or statuses
rg -n 'useState(<[^>]*>)?\((true|false|null|undefined|["'"'"']\w+["'"'"'])\)' .
# Status-like fields being assigned
rg -n -i '\b\w*(status|state|phase|stage|mode)\s*=[^=>]' .
# Phase literals compared, assigned, matched or unioned
rg -n -i '(===?|!==?|\bcase|\bis|\bin|=|:)\s*["'"'"'](idle|pending|loading|loaded|ready|running|started|stopped|paused|done|finished|completed?|failed|error|success|cancell?ed|active|inactive|open|opening|closed|closing|connecting|connected|disconnected|reconnecting|draft|submitted|approved|rejected)["'"'"']' .
# Lifecycle guards and ordering errors
rg -n -i '\b(if|unless|guard|assert|require)\b.*\b(initiali[sz]ed|started|ready|disposed|destroyed|mounted|running|connected|busy|processing|locked)\b' .
rg -n -i '(not|already) (been )?(initiali[sz]ed|started|running|connected|disposed|closed|opened|loaded)|must (call|be called)|before calling|call \w+ first' .
# Presence-encoded phase: timestamps and nullable results reset to empty
rg -n '\b\w+(At|_at|On|_on)\s*(=|:)' .
rg -n '\b(error|result|data|response|session|connection|socket|handle|pending\w*)\s*=\s*(null|nil|None|undefined)\b' .
# Counters that select behavior
rg -n -i '\b(step|stage|attempts?|retries|retry_?count|tries|page_?index|wave|round)\s*(\+\+|\+=|-=|=)' .
# Mode flags in loops
rg -n '\b(in|inside)(_[a-z]|[A-Z])\w*\s*=|\bescaped\s*=' .
# Stale-async and cancellation guards
rg -n -i '\b(request_?id|req_?id|generation|epoch|latest\w*|current_?token)\s*(!==|!=|===|==)|\b(cancell?ed|aborted|is_?stale)\s*=' .
# Persisted status
rg -n -i 'set\s+(status|state|phase)\s*=|where\s+(status|state|phase)\s*(=|in)' .
```

Also read, without a regex: reducers and stores, socket/connection classes, job/queue workers, wizard and checkout flows, game entity update loops (`isGrounded`/`isJumping`/`isAttacking`), and any class whose methods start with a guard clause on a field.

## Cluster

Group hits by owner: the class, module, component, store, record type or table they belong to. For data passed between modules (a DB row, a shared store, an event payload), the owner is the data type and every module that writes it is part of the cluster. A cluster is a candidate when it has two or more state-carrying variables, or one variable written from two or more sites, or one variable read as a guard before an operation.

## Confirm

For each candidate, reconstruct the machine from the code:

1. **Variables.** List every field, column, hook, closure variable or counter that carries the state, with its domain (bool, enum values, null/non-null, integer range).
2. **Transitions.** Find every write. For each: the condition it runs under (from-state), the values it writes (to-state), the trigger (method, event, callback, effect, timer, message) and `path:line`.
3. **Guards.** Find every read that changes behavior: which operations are allowed, skipped or throw in which states.
4. **States.** Name each combination that the writes actually produce, by meaning (`uploading`, `failed`, `done`). Compare representable combinations (the product of the domains) with reachable ones; the difference is the set of invalid states the types allow.
5. **Gaps.** Note states with no exit (stuck), events silently ignored in some states, error states with no recovery, re-entry while a transition is in flight, concurrent or cross-thread writers, and writes that skip a guard other writes respect.

Mark **confirmed** when the reconstruction yields at least two named states and two transitions, and at least one of:

- an invalid combination is representable (`isLoading && error != null`);
- the state decides which operations are legal;
- transitions are written from two or more sites with no single owner;
- an ordering rule is enforced only by convention, a comment, or a runtime check.

Mark **likely** when the states are clear but some transitions run through code you could not trace (dynamic dispatch, reflection, external callers, another service); say what is untraced.

**Reject**, with a one-line reason:

- Orthogonal flags: every combination is valid and meaningful (`isAdmin`, `darkMode`, `isPinned`). Verify by enumerating, not by name.
- Configuration and feature flags that do not change at runtime.
- Values derived on read from other state (fine; *stored* derived state that can drift is a finding).
- One-way latches with one write and no ordering consequence (`didWarnOnce`).
- Explicit machines already in place (state type plus a single transition function, reducer, or statechart) with no bypassing writes.

## Severity

| Severity | Evidence |
|---|---|
| High | Invalid states are reachable or defensively handled (`// should never happen`, impossible-combination checks, related TODO/FIXME); async, concurrent or cross-thread writers; stuck states; persisted status with scattered writes; explicit machine bypassed |
| Medium | Three or more variables or write sites; spread across modules; ordering enforced only by convention; silent ignores of events |
| Low | Local to one owner, two variables, one or two write sites, short-lived |

Raise severity when the owner is on a hot path for bugs (frequent fix commits on those lines, when history is cheap to check). Lower it when the machine is test-only scaffolding.

## Report

Open with one line: scope covered, candidates swept, confirmed / likely / rejected counts. Then a summary table ranked by severity:

| ID | Severity | Kind | Owner | Where | States | Confidence |
|---|---|---|---|---|---|---|
| SM-01 | High | Async lifecycle | `UploadPanel` | `src/upload/UploadPanel.tsx:24` | idle, uploading, failed, done | confirmed |

Then one block per finding:

```md
### SM-01 · `UploadPanel` upload lifecycle · High · confirmed

**Encoded as:** `isUploading` (bool), `progress` (number), `error` (Error | null), `url` (string | null) — `src/upload/UploadPanel.tsx:24-27`

**States:** idle · uploading(progress) · failed(error) · done(url)

| From | Event | To | Site |
|---|---|---|---|
| idle | `start()` | uploading | `UploadPanel.tsx:41` |
| uploading | XHR progress | uploading | `useUpload.ts:18` |
| uploading | XHR error | failed | `useUpload.ts:25` |
| failed | `retry()` | uploading | `UploadPanel.tsx:63` |

**Problems:**
- 16 combinations representable, 4 reachable; `isUploading && error` is rendered as both spinner and error banner (`UploadPanel.tsx:88`).
- `retry()` has no guard and can start a second upload while one is in flight (`UploadPanel.tsx:63`).
- No transition out of done; a new file requires remounting.

**Suggested model:** discriminated union `{ kind: 'idle' } | { kind: 'uploading'; progress } | { kind: 'failed'; error } | { kind: 'done'; url }` driven by a `useReducer` that ignores `retry` unless `failed`.
```

Keep the suggested model to a type sketch and the owner of transitions; full code belongs to the hand-off. Close with the rejections grouped by reason and counted, and any areas not covered.

## Hand-off: making a machine explicit

Only when the user asks to fix findings. One finding at a time, behavior preserved unless the user decides a current behavior is a bug:

1. **Pin behavior.** Add tests over the reconstructed transition table, including the invalid-combination and gap cases; ask the user what should happen where current behavior looks wrong.
2. **Introduce the state type.** One type, one value per state, carrying only the data valid in that state.
3. **Own the transitions.** Route every write through one transition function, reducer or method set that rejects or ignores illegal events explicitly. Reads switch exhaustively over the state.
4. **Remove the old variables** and any guards the type now makes unnecessary.
5. **Persisted state:** map each old combination to a new value in a migration, constrain the column (enum type or CHECK), and make writes conditional on the from-state (`UPDATE … SET status = 'shipped' WHERE id = ? AND status = 'paid'`) so concurrent writers cannot skip a transition. Plan rollout so old and new code can read each other's rows.

Use a statechart or state-machine library only when the machine has real nesting, parallel regions, delayed transitions or needs visualization, and prefer one the project already depends on.

| Language | Explicit form |
|---|---|
| TypeScript | Discriminated union + `switch` with a `never` exhaustiveness check; `useReducer` in React |
| Rust | `enum` with per-variant data + `match`; typestate for compile-time call ordering |
| Swift | `enum` with associated values + `switch` |
| Kotlin | `sealed interface` + `when` |
| Java 21+ | `sealed interface` of `record`s + pattern `switch` |
| C# | `abstract record` hierarchy + `switch` expression, or `enum` + `switch` in one owner |
| Python 3.10+ | `Enum`, or a union of `@dataclass` states + `match` |
| Go | Named type + `const` block + `switch` in one owner; interface with one struct per state when states carry data |
| C++ | `std::variant` + `std::visit` |
| C | `enum` tag + `union` + `switch` in one function |
| SQL | Enum type or CHECK constraint; conditional `UPDATE … WHERE status = from`; optional allowed-transitions table |

## Primary references

- [Making Impossible States Impossible](https://www.youtube.com/watch?v=IcgmSRJHu_8) (Richard Feldman): replacing flag combinations with types that admit only valid states.
- [Parse, don't validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/) (Alexis King): encoding established facts in types instead of re-checking them.
- [State](https://github.com/munificent/game-programming-patterns/blob/master/book/state.markdown) (Game Programming Patterns): from boolean flags to finite state machines, hierarchical and pushdown variants.
- [Stop using isLoading booleans](https://kentcdodds.com/blog/stop-using-isloading-booleans) (Kent C. Dodds): status enums for async UI.
- [Statecharts](https://statecharts.dev/): when nesting, parallel regions and delayed transitions justify a statechart.
- [TypeScript discriminated unions](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#discriminated-unions): exhaustive narrowing over state variants.
