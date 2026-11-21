# memcached — technique cards (95)

Each card is exactly 7 lines. Fetch one with: grep -A 6 "^### memcached-NNN " memcached.md

### memcached-001 — Replace a general-purpose map with a purpose-built hash table on your own memory
Lens: algorithms · Source: memcached
Principle: For the one structure on the hottest path, a simple chained hash table with a power-of-two bucket array, links stored inside the objects, and nodes living in your own allocator beats a general library map on speed and on memory predictability.
Why it is faster: General ordered maps allocate internally, fragment the heap and keep their order with work you don't need. A purpose-built table gives O(1) lookups, a power-of-two size turns `hash % size` (a division) into `hash & mask`, and an intrusive `next` pointer inside each object removes a separate node allocation and one pointer chase per probe. The cost is that you lose ordered iteration and have to handle growth yourself.
Applies elsewhere: Web / backend: for a high-cardinality aggregator, a typed-array open-addressing table in JS can beat `Map` on GC pressure and memory. • Native / systems: replace `std::map` in a hot path with a flat or intrusive hash table. Linux kernel `hlist` embeds links in the object. • Data / databases: lock tables and buffer-pool page tables usually embed chain links in the entry.
Watch out: an intrusive link ties an object to one table at a time. A power-of-two mask needs a hash whose low bits are well mixed, or clustering wipes out the gain.
Seen in memcached: 825db0f941 2003-06-20 Dropped the Judy array ("judy caused memory fragmentation") for a chained hash with Jenkins one-at-a-time mixing and a power-of-two table, masking instead of mod ("mod is sooo slow!"). Chain nodes were still separately allocated `hashitem {next, item}` structs. • f6d334e03c 2003-06-20 One commit later the chain link moved into the item itself (`item->h_next`, `static item** hashtable`). Insert and delete became pointer splices with no allocation.

### memcached-002 — Grow a hash table incrementally, one bucket at a time
Lens: algorithms · Source: memcached
Principle: When a table must double, allocate the new array and migrate old buckets a few at a time. Lookups check whether their old bucket has moved yet and read whichever table is current, so no single operation ever pays O(n).
Why it is faster: A stop-the-world rehash of millions of entries stalls every request for the whole copy. Spreading migration over many small steps bounds the worst-case latency of each step and keeps the load factor (and collision chains) low as the data set grows. During migration lookups do an extra comparison, and the table briefly holds both arrays.
Applies elsewhere: Web / backend: a large in-process cache or `Map` replacement can rehash during idle callbacks instead of inside a request. • Data / databases: Redis-style incremental rehash and linear hashing in DB indexes both split one bucket at a time. • Embedded: a real-time task that can't afford an O(n) pause can migrate a fixed number of buckets per tick.
Watch out: every reader and writer must know which table a key currently lives in, and a burst of inserts can outrun the migration. If the final size is known, pre-sizing (next entry) avoids the whole mechanism.
Seen in memcached: 217dcce072 2006-11-26 Added `primary_hashtable`, `old_hashtable` and `expand_bucket`. The table starts at `hashpower=16` and doubles when items exceed 1.5x buckets. `assoc_find` reads `old_hashtable[oldbucket]` if `oldbucket >= expand_bucket`, and one bucket moves per connection state transition ("conserve memory on small caches", fewer collisions on large ones). • f1351f9bba 2008-04-02 The migration step moved to the `conn_new_cmd` transition, so it ran once per command. • 7f09e20ba5 2009-01-27 Migration left the request path for a maintenance thread that moves `hash_bulk_move` buckets per lock hold, because the per-command step added "extra contention on an already hot mutex".

### memcached-003 — Pre-size structures when the final size is known, and switch growth off
Lens: algorithms · Source: memcached
Principle: If you can predict how many entries a structure will hold, allocate that capacity once at startup instead of growing through repeated doublings.
Why it is faster: Each doubling costs a full copy or migration, extra memory while two arrays coexist, and pauses or coordination with readers. Pre-sizing removes all of that during warm-up. Turning growth off also deletes the machinery (migration thread, pauses) from the running system. The trade-off is memory committed up front, wasted if the estimate is too high.
Applies elsewhere: Web / backend: `HashMap(initialCapacity)`, `reserve()` and `Vec::with_capacity` for request-scoped collections whose size you know. • Data / databases: size hash indexes from row-count estimates before a bulk load or hash join. • Embedded: fixed-size tables sized at build time, with no growth path at all.
Watch out: with growth disabled, an underestimate degrades into long chains and slow lookups. Export load-factor stats so the size can be corrected.
Seen in memcached: 1db1de3828 2011-09-28 `-o hashpower=N` sets the initial bucket array in `assoc_init`. • 108e2cd600 2011-09-28 Exposed `hash_power_level`, `hash_bytes` and `hash_is_expanding` stats so operators could pick a value from a running instance. • 6af7aa0b15 2014-12-27 Recommends pre-sizing to anyone who wants consistent performance, since each expansion now starts with a brief global pause. • dd4acbaf76 2017-10-17 `-o no_hashexpand`: "specify an exact size, and don't try to realloc once running".

### memcached-005 — Represent small option sets as a bitmask built during the scan
Lens: algorithms · Source: memcached
Principle: Design options so each maps to one bit (single-character flags in a range under 64). Set the bit while tokenizing, and every later "is flag X present?" check becomes one AND.
Why it is faster: Without a mask, each feature check re-scans the token list or compares strings. A 64-bit mask built in the same pass gives O(1) tests, O(1) duplicate detection, and lets several flags be tested at once with one compare. Bitfields can't be compared as a group, but explicit bit manipulation can.
Applies elsewhere: Web / backend: parse feature or permission headers once into an integer mask carried on the request object. • Embedded: option letters in shells or config frames as a `uint32_t` bitset. • Data / databases: query hints and column-presence sets as bitmaps tested with AND.
Watch out: the protocol has to be designed for it (bounded flag alphabet). Adding a 65th option means widening the mask everywhere.
Seen in memcached: 6a4e8e1f20 2016-06-10 Logger event flags switched "from bit fields to bit manipulation" so they can be compared all at once. • 04613a4736 2019-11-14 `_meta_flag_preparse()` uses a `uint8_t seen[127]` table and a switch on the first byte; flags are single characters, space-delimited. • d22b66483b 2020-06-10 Proxy: `pr->t.meta.flags |= 1 << (*cur - 65)`; "the range of possible flags is deliberately < 64". • 3ef3b768e6 2024-08-06 The response inspector precomputes a `uint64_t bit` per configured flag at load time, so matching is a mask test. • 778cfd6e56 2025-02-18 mcmc's tokenizer sets `t->metaflags |= (uint64_t)1 << (*s - 65)` during the scan; `mcmc_token_has_flag_bit` is the test.

### memcached-006 — Keep head, tail and "next" cursors so you never re-walk what you have processed
Lens: algorithms · Source: memcached
Principle: When a buffer or queue is consumed or appended step by step, keep a pointer to where work resumes (read cursor, tail, next-to-send). Each step is then O(1), not O(length of what came before).
Why it is faster: Re-walking a prefix on every step turns a linear workload quadratic: memmoving the unconsumed tail after each pipelined command, walking a singly linked list to append, or rescanning already-flushed entries before each write. One extra pointer removes the rescan. When a lock is held during the walk, the saving also shortens the critical section.
Applies elsewhere: Web / backend: stream parsers that track a `readerIndex` (Netty) or offset instead of slicing and concatenating buffers per message. • Native / systems: MPSC queues with tail pointers; TCP's `snd_nxt` versus `snd_una`. • Data / databases: pipelined clients and brokers tracking "sent" versus "acknowledged" offsets separately.
Watch out: every extra cursor must be kept consistent on removal, reset and error paths (memcached resets `io_next` when a backend goes bad). Stale cursors cause skipped or duplicated work.
Seen in memcached: c9607c6ddb 2006-09-03 Added `c->rcurr`: each parsed command advances the cursor, and the remaining bytes are memmoved to the buffer front only once, just before the next socket read. The founding code memmoved after every command. • 9917c79850 2021-11-15 `extstore_submit()` keeps a `queue_tail` and finds the incoming batch's depth and tail before taking the lock, replacing a `/* FIXME: Optimize by tracking tail. */` walk. With one IO thread and 10k+ depth, "each worker thread submission walks the entire queue". • 146c6489fc 2022-12-05 Proxy backends got an `io_next` cursor into the in-flight queue, so flushes start after already-written requests instead of "looping past already flushed IO objects".

### memcached-008 — Replace timers with on-access evaluation of time-based state
Lens: laziness · Source: memcached
Principle: Store a deadline or last-update time with the object and evaluate it only when the object is touched, instead of running a timer or background job per object.
Why it is faster: Per-object timers cost memory, scheduler work and wakeups for objects nobody reads again. An integer compare on access does the same job in O(1), and unread expired objects are reclaimed by normal memory pressure. The same idea works for rate limiters: compute the tokens earned since the last check instead of refilling on a timer. The cost is that dead objects linger until touched or evicted.
Applies elsewhere: Web / backend: TTL checks on session or JWT caches at read time; API-gateway rate limiters computed per request. • Data / databases: Redis-style passive expiry; TTL visibility checks in scans. • Embedded: packet policers and debounce logic that compare against a free-running tick counter instead of arming timers.
Watch out: objects that are never read keep holding memory, so pair this with eviction or a bounded background sweep. Time must come from a monotonic source, or clock jumps expire everything or nothing.
Seen in memcached: 32f382b605 2003-05-27 Founding code: on get and on store, `if (old_it && old_it->exptime && old_it->exptime < now) item_unlink(old_it)`. No expiry timers anywhere. • 57fb9e58ea 2003-05-28 The same check applied to incr/decr. • 6f0240f424 2023-07-11 The proxy token bucket `mcp.ratelim_tbf` refills lazily on each check: `toadd = delta / tick_rate; last_update += toadd * tick_rate`, advancing only by whole ticks so rounding loses no time, then clamping to the limit. No background refill.

### memcached-009 — Invalidate everything in O(1) with a cutoff timestamp or generation number
Lens: laziness · Source: memcached
Principle: To "delete all", record a cutoff (time or generation) and treat any object older than it as missing on access. Reclaim memory lazily, and actively clean up only the part a sorted order makes cheap to find.
Why it is faster: Walking and freeing millions of objects blocks the system for seconds. Writing one global value takes constant time. Readers pay one extra compare. If objects are kept in time order, the few that need active handling sit at one end and a walk can stop at the first older object. Allowing the cutoff to be in the future lets a fleet stagger invalidation and avoid a thundering herd on the backend.
Applies elsewhere: Web / backend: namespace version keys in app caches ("user:v17:..."), where bumping the version invalidates every key at once. • Data / databases: TRUNCATE or drop-partition as a visibility cutoff; MVCC snapshot horizons; generation counters in slot maps. • Native / systems: epoch-tagged TLB or cache entries, invalidated by bumping the epoch.
Watch out: memory is not freed until objects are evicted or touched, so capacity planning must count the dead data. Version counters must not wrap into live values.
Seen in memcached: d9f9358937 2003-12-01 `flush_all`: "immediate (and fast) invalidation of all existing items", items "simply pushed away by new items". It sets `settings.oldest_live`, and get ignores `it->time <= oldest_live`. • 54c9113bf8 2005-08-09 `flush_all <time>` schedules the cutoff, so a fleet can be flushed "not at exactly the same time (to reduce load peaks)". • c5944dc2b4 2006-11-20 `item_flush_expired()`: "The LRU is sorted in decreasing time order... we only need to walk back until we hit an item older than the oldest_live time." • f593a59bce 2017-09-26 extstore gives each reused flash page a new global version. In-RAM headers carry the page version, and a mismatch on read means the object is gone, so a page is invalidated with no scan or reverse index. • 8aad5ae070 2024-03-07 Simplified to a pure timestamp test (`it->time <= oldest_live && oldest_live <= current_time`) plus the bounded head walk, removing `settings.oldest_cas` and its restart metadata.

### memcached-013 — Classify a request up front and take the cheapest path that satisfies it
Lens: laziness · Source: memcached
Principle: Scan a request's options before touching shared state to find out whether it needs the heavy path (extended locking, writes, bookkeeping). Requests that don't need it take a lighter path.
Why it is faster: A single general-purpose path pays for the most demanding option on every request: holding a lock across the operation, updating recency, preparing write-back. Most requests are plain reads. A cheap pre-scan over the already-tokenized options picks the minimal path, so lock hold time and side effects are paid only by requests that need them.
Applies elsewhere: Data / databases: plain `SELECT` versus `SELECT ... FOR UPDATE`; read-only transactions that skip write bookkeeping. • Web / backend: cache APIs offering get versus get-and-touch; middleware that skips session loading for static-asset routes. • Native / systems: copy-on-write fast paths taken when no mutation flag is present.
Watch out: two paths can drift apart. Every option that needs the heavy path must be in the classifier, or a mutating request silently runs unlocked.
Seen in memcached: 04613a4736 2019-11-14 `_meta_flag_preparse` sets `of.locked` only for the flags that mutate (N/T/R/l/h). `process_mget_command` then uses `limited_get()` for plain reads and `limited_get_locked(..., DONT_UPDATE, &hv)` only when needed. Flag `u` skips the LRU bump entirely.

### memcached-014 — Collect work from all ready sources in one loop pass, then submit once per destination
Lens: amortization · Source: memcached
Principle: Don't flush after each client or request. Let every ready source in the current event-loop iteration add to a queue, then submit the whole batch with one syscall or wakeup per destination, with a cap on batch size.
Why it is faster: Syscalls, doorbells and cross-thread wakeups have a fixed cost far larger than appending one more item. When many clients produce work in the same loop pass, combining it divides that cost by the batch size. The same holds per upstream connection: many requests to one backend can go out in one `writev`. A batch cap keeps tail latency bounded, and per-thread (not per-connection) queues are what make cross-connection batching possible.
Applies elsewhere: Web / backend: a DB or Redis proxy that gathers queries from all sockets ready in one tick and sends one pipelined write per upstream (auto-pipelining clients such as ioredis). • Native / systems: ring a NIC or NVMe doorbell after N descriptors; io_uring submit once per loop. • Data / databases: group commit of the WAL across sessions; Kafka producer batching per partition leader.
Watch out: waiting to fill a batch adds latency at low load, so always flush at the end of the loop pass, not only when the batch is full. memcached saw a slightly longer p99.9. Partial writes need per-item resume offsets.
Seen in memcached: f2a2f38717 2022-01-24 The proxy groups requests per backend, marks each backend "stacked" once per loop pass, and flushes its queue with one `writev` over `be->write_iovs`. Before, "individual requests for the same backend would use multiple syscalls". • 146c6489fc 2022-12-05 Raised `BE_IOV_MAX` from 128 to 1024 for "more IOV's per syscall". • fe2a33202a 2023-06-10 Fixed the worker-IO path to flush every queued object in the batch, not just the last. • ec1fb560fa 2024-12-29 Moved IO queues from each connection to each worker thread (connection struct ~10% smaller), so one submit can cover many connections. • 4a04d84691 2025-01-21 `drive_machine` submits async IO only after more than 20 connections have queued work (`conns_tosubmit`), otherwise after the event loop. At 500k rps on a 16-core 7950X, p50 dropped from 700us to 100us; p99 unchanged, p99.9 slightly longer. • fc354ee58a 2025-01-27 Reset the counter inside `thread_io_queue_submit` so each epoll loop can batch up to N clients.

### memcached-018 — Record hot-path side effects in a lossy per-thread buffer and replay them in bulk
Lens: amortization · Source: memcached
Principle: When a request must trigger a structural update that needs a contended lock but not right away, append a small record to a per-thread ring buffer. A background thread replays the buffer in batches. If the buffer is full, drop the hint.
Why it is faster: The request thread pays only an uncontended append. The background thread takes the shared lock once per batch instead of once per event, and replays hints with warm caches. Dropping on overflow bounds memory and keeps latency flat under bursts (for example, a fast keyspace scan), at the cost of slightly less accurate bookkeeping.
Applies elsewhere: Web / backend: batched "last seen" or access-time writes for sessions; Caffeine-style read buffers replayed by a maintenance task. • Data / databases: buffer-pool access recording batched before touching the shared LRU. • Native / systems: per-CPU pagevecs that batch LRU list insertions in the Linux kernel.
Watch out: only for hints whose loss is acceptable. Anything correctness-critical needs backpressure, not dropping. Each buffered record that pins an object (a refcount) delays its reclamation until replay.
Seen in memcached: 23b880c577 2017-01-29 "use LRU thread for COLD -> WARM bumps": each worker has an `lru_bump_buf` (a bipbuffer of `{item*, hv}`, 8192 entries). `lru_bump_async()` takes a reference and enqueues. When the buffer is full it clears `ITEM_ACTIVE` and counts `lru_bumps_dropped`. "This prevents regressions from high speed keyspace scans."

### memcached-020 — Batch deferred actions into a periodic sweep instead of a timer per item
Lens: amortization · Source: memcached
Principle: When many items each need an action at some later time, push them onto one list and process the list in a batch on a coarse periodic timer.
Why it is faster: A timer per item costs a timer-structure insert and a separate wakeup for each. One sweep every few seconds pays a single wakeup for the whole batch and walks a compact array linearly. The trade-off is timing precision of up to one sweep period.
Applies elsewhere: Data / databases: batched tombstone purges in LSM trees; periodic cleanup of expired locks. • Native / systems: RCU deferred frees processed per grace period; timer wheels that batch by slot. • Web / backend: one interval job flushing expired sessions instead of a `setTimeout` per session.
Watch out: a large backlog makes a single sweep a latency spike, so bound the work per sweep. Items must tolerate being handled up to one period late.
Seen in memcached: 32f382b605 2003-05-27 Delayed deletes go onto a `todelete[]` array. `delete_handler` runs on a 5-second `evtimer`, sweeps entries whose time has come and compacts the array in place. • ef3644d338 2003-06-25 One of several early correctness fixes to the sweep.

### memcached-023 — Store the compact canonical form, and render text at the edge once rendering is cheap
Lens: memory-layout · Source: memcached
Principle: Caching a pre-formatted rendering of each object's metadata speeds up reads but costs memory on every object and formatting on every write. Once formatting is fast, store the raw binary values and render them per response.
Why it is faster: Whether to pre-render depends on the cost ratio. If formatting is a slow `sprintf` and reads far outnumber writes, a stored rendering pays off. With a specialised integer-to-ASCII routine, rendering costs nanoseconds, and the stored text becomes per-object memory overhead (which lowers hit rate) plus `snprintf` on every set. Dropping it also removes the dual code paths and frees header bits.
Applies elsewhere: Web / backend: cache numbers and binary protobufs, and serialise to JSON only at the API edge, unless profiling shows rendering dominating. • Data / databases: store timestamps as int64, not ISO strings, and format them in the client or the projection. • Embedded: log binary records and decode them off-device instead of storing formatted text.
Watch out: the right answer changes as formatting speed and the read/write ratio change, so re-measure when either shifts. Pre-rendering still wins for large, expensive renderings (full HTTP headers, compressed bodies).
Seen in memcached: c9607c6ddb 2006-09-03 `item_make_header()` stores `" <flags> <bytes>\r\n"` at `ITEM_suffix(it)` and drops the `flags` struct field. A `get` sends the stored suffix and data as one iovec with no formatting. • e61c0a861f 2007-11-20 `gets` reuses the stored suffix and formats only the per-request CAS part. • 1cc77e6824 / c2027ad42a 2016-12-30 A fast `itoa_u32/u64` arrives. With `inline_ascii_response=false`, the suffix holds a 4-byte `uint32_t` flags value ("Should speed up sets, reduce memory overhead"), and `make_ascii_get_suffix()` renders at read time. • 3f3e137936 2017-07-19 Binary storage becomes the default. • 31208e8fbe 2019-05-14 The inline option is removed entirely ("Performance is fine … Code is simpler"). f28cbc5e9a reuses the freed `nsuffix` byte to widen `it_flags` to 16 bits.

### memcached-024 — Gate optional per-object fields behind a presence bit
Lens: memory-layout · Source: memcached
Principle: Don't charge every object for a feature it may not use. Store the field only when a flag bit says it is present, and compute the offsets of later fields from that bit.
Why it is faster: A fixed 8-byte field across 100M small objects is 800 MB that could hold data. A flag bit is free if the header has spare bits. Each access to later fields pays a conditional add, which is a well-predicted branch because one deployment usually uses a single setting.
Applies elsewhere: Data / databases: null bitmaps and sparse optional columns in row formats. • Native / systems: option fields in packet headers (IP options, TCP options), and an "extras" area on hot objects. • Web / backend: protobuf-style presence bits in custom binary caches.
Watch out: every offset macro must agree on the rule. Missing one (as the follow-up fixes show) corrupts data. Centralise offset computation in one place.
Seen in memcached: eda68b7 2008-11-25 "Added -C to remove the cas_id from the data structures to save space". The fixed `uint64_t cas_id` is gone. `ITEM_CAS` in `it_flags` marks the 8 bytes, and `ITEM_key()`/`ITEM_suffix()`/`ITEM_data()` add `sizeof(uint64_t)` only when the flag is set. e5260b9 (2008-11-26) counts it in `ITEM_ntotal`. • 911c9d0a6c 2011-08-08 Accounts for the optional CAS in `item_size_ok`. • f28cbc5e9a 2019-05-14 Client flags get the same treatment: an `ITEM_CFLAGS` bit, and `ITEM_data`/`ITEM_ntotal` add 4 bytes only when it is set.

### memcached-025 — Store nothing for the common default value, and make wide fields a build option
Lens: memory-layout · Source: memcached
Principle: If a field is almost always zero, let its absence mean zero. If only some users need a wider field, let them choose the width at build time so everyone else doesn't pay for it.
Why it is faster: Most objects in typical workloads carry default metadata, so eliding it saves bytes on most objects at the cost of one branch when reading. A compile-time width choice keeps the narrow layout for the majority without adding a runtime branch.
Applies elsewhere: Data / databases: default-value elision in columnar formats, and 32- versus 64-bit index builds in search engines. • Web / backend: omit default fields from cached JSON or msgpack and restore the defaults when reading. • Embedded: configurable timestamp or counter width chosen per product at compile time.
Watch out: a build-time width splits the binary into variants that must all be tested, and persisted data is not portable between them.
Seen in memcached: 830b20f7b9 2017-07-03 "save four bytes per item if client flags are 0": `nsuffix = 0` when `flags == 0`, readers return 0, and the ASCII path writes '0' directly. • f28cbc5e9a 2019-05-14 The elision moves to the `ITEM_CFLAGS` bit, so items with zero flags store no flag bytes at all. • b1e4a7afcc 2023-10-06 `--enable-large-client-flags`: `client_flags_t` is `uint32_t` or `uint64_t`, and `FLAGS_SIZE(item)` is non-zero only when `ITEM_CFLAGS` is set.

### memcached-026 — Store large objects as a chain of bounded chunks
Lens: memory-layout · Source: memcached
Principle: Don't create ever larger contiguous size classes for big values. Cap the chunk size and link chunks into a chain, so large objects reuse a few classes and pages stay small.
Why it is faster: Geometric size classes near the top waste up to one growth step (20 to 50%) per object, and a contiguous 1 MB+ value needs pages at least that large. Chaining bounds the waste to the last chunk and keeps page size independent of the maximum value size. Readers pay a pointer walk per chunk, and gather-I/O can send chunks without copying them together.
Applies elsewhere: Data / databases: TOAST or overflow pages for large column values, and fixed-size blob blocks in object stores. • Native / systems: rope strings, and segmented buffers (iovec lists) for large messages. • Web / backend: stream large uploads into fixed-size parts (multipart upload) instead of one buffer.
Watch out: every reader, the page mover and the flash tier must understand chains. Small values should stay contiguous so the common path takes no extra hop.
Seen in memcached: 0567967a92 2016-06-30 Added the `ITEM_CHUNKED` header with an `item_chunk` chain, and `slab_chunk_size_max` separate from the 1 MB `slab_page_size`. • 51a828b91d 2016-07-07 Performance testing showed "less than 5% extra CPU usage" for large items. • c6f7593f83 2016-08-08 Fixed `-I 2m` allocating 2 MB pages but using only 1 MB, which had halved capacity. b83a0a5e07 / cca7bcab6e set the default chunk max to 512 KB.

### memcached-028 — Reuse fields that mean nothing in a serialised copy
Lens: memory-layout · Source: memcached
Principle: When an in-memory record is written elsewhere (to disk or the wire), fields that only matter in RAM can carry derived data such as a hash or checksum. The serialised format doesn't grow.
Why it is faster: Storing the key hash with the copy saves rehashing on readback, and a checksum enables integrity checks, both with zero bytes added per record. The RAM-side header holds the real values and restores them on read.
Applies elsewhere: Data / databases: put checksums or LSNs in reserved words of on-disk page headers. • Native / systems: reuse padding bytes in network headers for out-of-band metadata between cooperating peers.
Watch out: readers of the serialised copy must never interpret those fields with their RAM meaning. Keep the overload documented and confined to one module.
Seen in memcached: f593a59bce 2017-09-26 The extstore write path sets `buf_it->time = hv` ("cuddle the hash value into the time field so we don't have to recalculate it") and puts the CRC32C in `exptime`. `recache_or_free` reads `uint32_t hv = (uint32_t)it->time`. • ce79ff67c0 2017-09-27 Splits the write into request and commit so the hash and CRC go into the write-buffer copy without modifying the live item. It notes that another 24 bytes per item could "technically" be saved.

### memcached-031 — Accept a text-safe encoding at the edge, but store raw bytes
Lens: memory-layout · Source: memcached
Principle: Let clients send binary identifiers (hashes, packed integers) in a text-safe encoding such as base64. Decode them at the protocol boundary and store the raw bytes.
Why it is faster: A 32-byte hash is 64 bytes as hex and 44 as base64, but 32 bytes raw. Keys sit in every object and in every hash compare, so shorter keys mean more objects per GB and cheaper compares. The decode cost is paid once per request at the edge.
Applies elsewhere: Data / databases: store SHA-256 digests as `BYTEA` or `BINARY(32)`, not hex `VARCHAR(64)`. Use packed binary row keys in Bigtable or HBase. • Web / backend: decode UUIDs to 16 bytes for Redis or in-memory map keys.
Watch out: every tool that prints keys (dumps, logs, stats) must re-encode them, or it will emit garbage or break line framing.
Seen in memcached: 1764054993 2020-11-07 "meta: allow base64'ed binary keys with 'b' flag". The protocol doc: "This allows storing and fetching of binary packed keys". • f4aee65674 2020-11-12 Keys are returned base64-encoded where needed. d0b560d91a keeps binary keys from breaking cachedump output.

### memcached-039 — Pre-build per-request contexts so the request path allocates nothing
Lens: allocation · Source: memcached
Principle: Split work into a setup phase that builds every object a request will need (coroutines, request and response slots, cached lookups) and a run phase that checks out a ready context and returns it. On return, reset only what is needed and keep the grown memory.
Why it is faster: Zero allocations per request means no allocator calls and, in a managed runtime, no garbage and therefore no GC work. Keeping a coroutine's grown stack, instead of resetting it, avoids a shrink-then-regrow malloc cycle on every request.
Applies elsewhere: Web / backend: pooled per-route handler contexts, and reused fiber or coroutine objects. • Native / systems: reuse green-thread stacks in C++ or Rust runtimes instead of allocating per task. • Embedded: statically allocated packet-processing contexts checked out from a ring.
Watch out: a context that ended in an error may hold corrupt state, so rebuild it in that case (memcached only reuses a cleanly finished coroutine). Pools sized for peak load need a trim policy.
Seen in memcached: 1518d8950e 2023-08-18 Proxy Lua API v2 (`mcp_funcgen_t`/`mcp_rcontext_t`). A request used to allocate a coroutine, request, responses and strings. "With pre-allocation it should be possible to run a request from start to finish with zero allocations (and thus zero garbage collection)." • 5fe01d0c19 2025-01-17 `_mcp_funcgen_return_rctx` replaces `lua_resetthread` with a `lua_status` check plus `lua_settop(rctx->Lc, 0)`, keeping the stack. The gain is "mostly noticable with pipelined work".

### memcached-041 — Carve small objects out of the large pooled buffers
Lens: allocation · Source: memcached
Principle: Rather than run a separate pool (and memory limit) for small objects, sub-divide one large pooled buffer into N small objects with a per-bundle refcount, and keep a "next likely free" hint.
Why it is faster: One memory budget then covers both object kinds, and bundles inherit the big pool's locality and lifecycle. Holding the last fully free bundle skips re-initialising it on the next burst. Skipping zeroing of large inline buffers removes a cost that shows in profiles.
Applies elsewhere: Native / systems: arena-backed small-object allocators in game engines and proxies that pack request contexts into 16 KB pages. • Embedded: RTOS pools where several object types share one block budget.
Watch out: one live object keeps its whole bundle allocated, so long-lived stragglers fragment the pool. This suits objects with similar, short lifetimes.
Seen in memcached: d343e53b5a 2020-05-22 `mc_resp_bundle { refcount; next_check; next; prev; mc_resp r[]; }` sized as `(READ_BUFFER_SIZE - hdr)/sizeof(mc_resp)`. The search starts at `next_check`, the last empty bundle is held ("Just hold and reuse to skip init loop"), and the memset avoids `wbuf` because zeroing it shows up "significantly under perf". • 7c2df4a70d 2020-05-29 Removed the separate response-object cache.

### memcached-042 — Give every transient-object pool a hard cap derived from a global budget
Lens: allocation · Source: memcached
Principle: Divide a global megabyte budget by the number of threads and cap each per-thread pool. When a pool is exhausted, fail or shed the request instead of growing without bound.
Why it is faster: Unbounded pools turn a traffic spike into permanent RSS growth or an OOM kill. Per-thread caps need no shared counter, and separate failure counters show which pool ran out.
Applies elsewhere: Web / backend: per-worker memory budgets in Node or Go servers that return 503 when exceeded. • Data / databases: per-session `work_mem`-style caps. • Native / systems: fixed-size per-core mempools in packet processors.
Watch out: static per-thread shares can starve one busy thread while others sit idle. Size caps with headroom and watch the OOM counters.
Seen in memcached: 05721e4b6c 2020-02-25 `cache_set_limit(me->resp_cache, resp_obj_mem_limit/num_threads/sizeof(mc_resp))`, with the same for `rbuf_cache`. Failures are counted (`response_obj_oom`, kept distinct in 7c2df4a70d).

### memcached-044 — Design APIs that don't force callers to allocate
Lens: allocation · Source: memcached
Principle: Offer API shapes that take parts separately and edit buffers in place, or return native scalars, so callers in a managed language don't build temporary strings just to make a call.
Why it is faster: Something like `flag_add("F" .. token)` makes a concatenated string that becomes garbage. `flag_add("F", token)` writes directly into the request buffer. Returning an integer token avoids string-to-number conversion and the interned string left behind.
Applies elsewhere: Web / backend: offset or length accessors on `Uint8Array` instead of `subarray().toString()`, and logger APIs that take format arguments. • Native / systems: FlatBuffers-style zero-copy accessors, and `string_view` parameters.
Watch out: in-place mutation APIs make aliasing visible. Document whether returned views stay valid after later edits.
Seen in memcached: 742125f69e 2023-01-09 Modified requests are rebuilt with a few C string copies, "faster with less lua + fewer allocations". • 3304459c09 2024-03-06 `flag_add/set/replace/del` edit the request buffer directly, in contrast to `req:flag_add("F" .. token)`, which "would create a new string to be GC'd". • e6970b9f80 2024-07-25 `r:token_int()` returns a token as a Lua integer.

### memcached-045 — Choose when to carve a fresh page: lazy bump pointer or one-time pre-split
Lens: allocation · Source: memcached
Principle: A new page can be handed out lazily from a bump pointer, or split into free-list entries once when it arrives. Lazy carving avoids touching memory early. Pre-splitting leaves the allocation fast path a single free-list pop.
Why it is faster: Bump allocation doesn't pre-touch the page (RSS grows only with use) and needs no free-list storage for untouched chunks, but every allocation carries an extra branch and state. Pre-splitting moves that work to a rare event, the arrival of a page, which shortens the hot path. memcached chose the first when free-list storage was a separate array, and the second once the free list became intrusive and branches mattered more.
Applies elsewhere: Native / systems: bump arenas in compilers (lazy), or NIC buffer pools sliced from hugepages at startup (pre-split). • Embedded: pre-split pools at boot so runtime allocation is deterministic. • Data / databases: lazily initialised extents versus pre-formatted free-page lists.
Watch out: pre-splitting touches every chunk, which faults in the whole page and defeats lazy commit on huge arenas.
Seen in memcached: 66b288105d 2003-06-19 "don't chunk new pages up onto the freelist. instead, allocate off the end of the last page": `end_page_ptr` and `end_page_free`. Follow-up fix in e38c74c9a2. • 845a4fe1c4 2012-07-14 `split_slab_page_into_freelist`. 423b9fd4b5 removed `end_page_ptr` and the third branch in `do_slabs_alloc`: "Cuts the memory assignment hotpath a tiny bit".

### memcached-047 — Allocate a large object piece by piece as data arrives, and fit the last piece
Lens: allocation · Source: memcached
Principle: Allocate the next chunk only when the current one is full, and size the final chunk to the bytes that remain. If no tight-fitting class is available, fall back to a larger one rather than fail.
Why it is faster: Allocating the whole object up front evicts a large amount at once and rounds the tail up to a full chunk. Before the change, a 17 KB item with a 16 KB chunk max used 32 KB. Incremental allocation spreads eviction over the upload and right-sizes the tail. The fallback keeps all-large workloads working when small classes have no pages.
Applies elsewhere: Web / backend: multipart uploads that allocate parts as the stream arrives. • Native / systems: growable arenas whose final block is sized to the tail, and size-class allocators that fall back to the next class up. • Data / databases: extent allocation that right-sizes the last extent of a file.
Watch out: allocation can now fail mid-upload. The protocol path must clean up partial chains and drain the rest of the body.
Seen in memcached: ae84d77181 2017-02-22 "refactor chunk chaining for memory efficiency": `do_item_alloc_chunk(ch, bytes_remain)` picks `slabs_clsid(min(remain+hdr, chunk_max))`, and `read_into_chunked_item()` allocates on demand. It also removed branches from the slab hot path. • 597645db6a 2022-08-25 If the tight class can't supply a chunk, retry with `slab_chunk_size_max`: "If items in a cache are all exclusively large, these slab classes could be empty."

### memcached-048 — Reserve the arena up front and back it with huge pages
Lens: allocation · Source: memcached
Principle: Allocate the whole cache arena as one region aligned to, and advised for, huge pages, then bump-allocate slab pages from it.
Why it is faster: Random access over gigabytes of 4 KB pages misses the TLB constantly, and every miss is a page-table walk. 2 MB or larger pages cut TLB misses and page-table memory sharply. Preallocation also gives each class a page from the start and makes memory use predictable. Fresh anonymous memory is already zeroed, so the per-page memset can be skipped.
Applies elsewhere: Data / databases: PostgreSQL `huge_pages` for shared buffers, and huge-page hash indexes. • Native / systems: DPDK hugepage mempools, JVM `-XX:+UseLargePages`, and huge-page JIT code heaps. • Web / backend: large in-process caches (V8 or JVM heaps) on hosts with THP set to `madvise`.
Watch out: huge pages may be unavailable or fragmented, so always have a fallback. Preallocation commits the full limit immediately, and THP compaction can itself cause latency stalls.
Seen in memcached: a6b35b4444 2008-02-25 `-L`: `getpagesizes()` plus `memcntl(MC_HAT_ADVISE)` (Solaris), malloc of `mem_limit` at once, and `memory_allocate()` bump-allocating aligned pages from `mem_base`. • 73628d3e9a 2009-09-16 Linux support through libhugetlbfs: "Increasing the memory page size could reduce the number of TLB misses". • c979d21325 2012-07-29 Preallocates one page per class under `-L`. • 94ca8d4c4c 2017-12-13 `alloc_large_chunk_linux()` reads `Hugepagesize`, then `posix_memalign` plus `madvise(MADV_HUGEPAGE)`, falling back to normal allocation. • 73e46a6f4e 2018-11-04 / b452670309 2019-10-26 FreeBSD `mmap(MAP_ALIGNED_SUPER)`, with the slab-page memset skipped there. a7aafbbfe3 (2022-01-25) fixes large-page detection on RedHat.

### memcached-049 — Don't zero memory that is already zero, but do wipe memory that might not be
Lens: allocation · Source: memcached
Principle: Fresh anonymous pages from the OS are already zero. Use `calloc` or `mmap` instead of malloc plus memset, skip memset on fresh slab pages, and avoid zeroing large buffers that will be overwritten. Keep an explicit wipe for recycled memory.
Why it is faster: A memset over a large table is O(n) work that also commits every page at once. `calloc` on large sizes maps zero pages lazily. Skipping the zeroing of a 1 KB inline write buffer per response removes a cost that showed up in profiles.
Applies elsewhere: Web / backend: `new ArrayBuffer(n)` is already zeroed, and `Buffer.allocUnsafe` suits buffers that are fully overwritten. • Data / databases: large bitmaps or hash directories from `calloc` or `mmap`. • Native / systems: skip zeroing of DMA receive buffers the device fully writes.
Watch out: recycled memory (pools, restart files, freed pages) carries old data. Skipping its wipe leaks data or corrupts state.
Seen in memcached: 217dcce072 2006-11-26 `assoc_expand` uses `calloc` for the new bucket array. 984053cd68 (2008-06-18) does the same in `assoc_init`: "Use calloc for allocating the hash table vs. malloc+memset". • b452670309 2019-10-26 `do_slabs_newslab` skips memset on FreeBSD super-page mmap memory. • cb0de81149 2020-03-26 Pages from the global pool are always wiped, because they are not always fresh from mmap. • d343e53b5a 2020-05-22 The response-object reset skips the inline `wbuf`, whose zeroing shows up "significantly under perf".

### memcached-050 — Lock the working set in RAM
Lens: allocation · Source: memcached
Principle: For a memory cache, a page swapped to disk is slower than a miss. Use `mlockall` to keep all current and future pages resident.
Why it is faster: One swapped-out page turns a sub-microsecond hit into a millisecond disk read and stalls the event-loop thread. Locking removes that tail. The cost is that the OS can no longer reclaim the memory under pressure.
Applies elsewhere: Data / databases: in-memory databases that lock their heap. • Native / systems: latency-critical trading or audio processes. • Embedded: real-time Linux tasks that lock memory to avoid page-fault jitter.
Watch out: this needs privileges or a raised `RLIMIT_MEMLOCK`, and it must run before privileges are dropped. Over-committing locked memory gets the process OOM-killed instead of swapped.
Seen in memcached: 32f382b605 2003-05-27 `-k` calls `mlockall(MCL_CURRENT | MCL_FUTURE)`. a8e605d94f (2003-08-11) adds autoconf support. • 72a2eb4d70 2008-02-11 Moves the lock before privileges are dropped ("Seems like -k never worked").

### memcached-051 — Rebalance memory between size classes by moving whole pages
Lens: allocation · Source: memcached
Principle: When the object-size mix shifts, take a whole page from a class that doesn't need it, clear its objects, and give it to a class under pressure. Prefer the oldest data, skip already processed chunks, and don't evict when free memory exists.
Why it is faster: Without rebalancing, memory stays stuck in classes whose objects no longer arrive, and the hit rate falls while RAM sits idle. Moving pages fixes this without a restart. Picking the oldest page disturbs the least valuable data. A per-chunk "done" bitmap stops repeat passes from relocking finished chunks. Evicting from the LRU tail rather than from whatever happens to be in the page sacrifices the coldest items.
Applies elsewhere: Native / systems: moving extents between size-class bins or per-CPU caches in allocators. • Data / databases: resizing buffer-pool partitions per tablespace or page size, and shard rebalancing by whole-page migration. • Web / backend: moving capacity between per-tenant cache partitions.
Watch out: uniform page sizes (needed so any page fits any class) cost flexibility. A mover that blocks on busy items can stall, so pair it with skip and retry logic and with reap-on-read.
Seen in memcached: b7b9d5f4b9 2003-06-23 / d72b1a2d8d 2003-06-24 `slabs_reassign(srcid, dstid)` walks the page, unlinks live items, filters free slots inside it, and hands it over as a bump region. 0c84015a53 (2004-07-16) adds `memcached-tool move`, and 51b1074bb7 marks the moved chunks free. • 10698bae63 2011-12-18 Background rebalancing (`slab_rebalance_start/_move/_finish`, `slabs reassign`). With `-o slab_reassign`, every page is `item_size_max` so any page fits any class, and a page is processed `slab_bulk_check` items at a time. • d5185f9c25 2015-09-28 Shifts the page list down after a move so it stays age-ordered: "Now we always chew the oldest page." • 397464bedd 2019-09-23 A `slab_rebal.completed` byte per chunk skips finished chunks, and unreferenced busy items are freed in place instead of going through the free list. • 4c56c8d3e2 2024-11-14 New `slabs_mover.c`: evictions are allowed only if `mem_limit_reached`, otherwise a page is allocated. Reclaimed pages pull from the LRU tail, and completed chunks skip the slabs lock.

### memcached-052 — Rescue live objects during page moves, and keep a global pool of free pages
Lens: allocation · Source: memcached
Principle: When moving a page, copy its live objects into free chunks of the same class instead of evicting them. Return pages from classes with surplus free chunks to a shared pool that any class can draw from, and keep that pool prefilled.
Why it is faster: Eviction during a move throws away valid data, which costs hit rate. Rescue keeps the data at the cost of a memcpy. A global pool decouples "who has spare memory" from "who needs memory", so a needy class takes a page immediately. A flap threshold (2.5 pages) stops pages bouncing back and forth.
Applies elsewhere: Native / systems: compacting or moving collectors and allocator defragmentation that relocate live objects (as Redis activedefrag does). • Data / databases: a free-extent pool shared across tablespaces, and online table compaction.
Watch out: relocation needs every reference to the object to be updatable, which here means a hash chain and an LRU link under the item lock. With external raw pointers, rescue isn't possible.
Seen in memcached: 004e221190 2015-09-28 "slab mover rescues valid items with free chunks". • d6e9646705 2015-09-29 Global page pool (class 0): classes with more than 2 pages of free chunks give one back. 1c3f0a3d0f raises the threshold to 2.5 to avoid flapping. b1debc4c96 (2015-10-09) reclaims inline to keep rescuing. • ee461d1102 2016-07-11 Rescue for chunked items. • 97b52b1866 2017-12-18 Prefills the global pool (`slab_automove_freeratio`).

### memcached-053 — Decide page moves from windowed eviction and age statistics, with hysteresis
Lens: allocation · Source: memcached
Principle: Move memory from where it is idle to where pressure is highest. Base the decision on a sliding window of per-class evictions and eviction ages, require repeated confirmation, and react to bursts that dominate the window.
Why it is faster: Marginal hit-rate gain is highest where items are evicted youngest. Equalising eviction age across classes approximates the best split of memory. Requiring the signal over several windows avoids oscillation. A burst rule (more than 25% of the window's evictions) still reacts quickly to sudden shifts.
Applies elsewhere: Web / backend: multi-tenant cache partition sizing by equalising eviction age or marginal hit rate, and autoscalers with stabilisation windows. • Data / databases: splitting memory between query cache and buffer pool by observed reuse distance. • Native / systems: allocator arena rebalancing driven by per-class miss statistics.
Watch out: background scanner items or stale samples skew age signals. Filter instrumentation artefacts out of the inputs, and prototype the policy offline before building it in.
Seen in memcached: 99fc043af9 2012-01-03 `slab_automove_decision` every 10 s: the source needs 0 evictions for 3 checks and more than 2 pages, and the destination must be the top evicter 3 times. c2e0023d02 rotates the source class "so as to not bias against the smaller classes". 2db1bf462c adds an aggressive mode. • 7d72a92cf1 2017-04-08 Python prototype. 5c43b80577 (2017-06-20) adds `slab_automove.c` with per-class `window_data{age,dirty,evicted}`, moving when `youngest_age < oldest_age * max_age_ratio`. 48b07bd320 adds an extstore-aware variant. • 7bb69b80b1 2020-07-02 "if more than 25% of the total evictions in the window appear in this slab it becomes a candidate", ported to C in e86a585d5a. • e183fa37d7 2022-03-21 Ignores crawler placeholder items when sampling tail ages and includes the current window. 15ff009464 (2022-08-22) uses the latest window data.

### memcached-054 — Let foreground readers drain a region that is being evacuated
Lens: allocation · Source: memcached
Principle: A background mover can wait forever on hot objects that are always referenced. Have readers that touch an object inside the region being moved treat it as a miss and unlink it, and have the mover delete objects that stay busy too long.
Why it is faster: Popular items are pinned almost constantly, so a mover that waits for refcount 0 may never finish. Turning reads into evacuation help guarantees progress at the cost of a few extra misses on those keys.
Applies elsewhere: Native / systems: read barriers in moving garbage collectors that evacuate on access. • Data / databases: online defragmentation or shard draining that redirects reads to the new location.
Watch out: evacuation-by-miss costs hit rate on the hottest keys, the most expensive ones to miss. Keep the evacuation window short.
Seen in memcached: b3630e1a2c 2012-01-05 `do_item_get`: if `slab_rebalance_signal` is set and the item lies within `[slab_start, slab_end)`, unlink it and return NULL ("prevents popular items from jamming in busy wait"). • d67d18791f 2017-06-15 The mover deletes items that stay busy so it can't get stuck.

### memcached-055 — Resize the memory limit at runtime and give freed pages back to the OS
Lens: allocation · Source: memcached
Principle: Let the memory limit grow or shrink while running. Shrinking frees whole pages through the rebalancer back to the OS instead of needing a restart.
Why it is faster: A cache that only ever grows ties up host memory after its workload shrinks, and a restart to shrink it loses the warm data. Page-granular release reuses the existing mover. Accurate accounting matters: a counter that drifts upward stops allocation early.
Applies elsewhere: Native / systems: JVM G1 periodic uncommit, and allocator purge calls after load drops. • Data / databases: online buffer-pool resizing (MySQL `innodb_buffer_pool_size`). • Web / backend: in-process caches that shrink when the container memory limit changes.
Watch out: shrinking evicts data and takes time. Rapid down-and-up cycles thrash the mover.
Seen in memcached: ec937e5ed1 2015-10-21 Fixed `total_malloced` growing on every page reuse, which had stopped malloc too early. • 31541b371a 2016-06-23 `cache_memlimit` raises the limit at runtime, and with `-o modern` lowers it: "pages are free()'ed back to the OS via the slab rebalancer as memory is freed up".

### memcached-056 — Use a segmented LRU with lazy, flag-based promotion
Lens: caching · Source: memcached
Principle: Split the recency list into segments (hot, warm, cold). On a hit, only set an "active" bit. Relink an item only when it reaches a segment's tail. New items start in a probation segment, and only items hit again get promoted.
Why it is faster: A hit becomes one plain store with no list lock and no pointer writes. Reordering happens in bulk at the tails, off the request path. Segments also resist scans: one-hit items flow through hot to cold and are evicted without displacing the frequently used working set. Requiring two hits before promotion strengthens that filter.
Applies elsewhere: Data / databases: CLOCK or second-chance buffer-pool replacement, InnoDB's young/old split, and 2Q. • Web / backend: in-process caches (S3-FIFO, segmented LRU, TinyLFU-style) that mark entries on read and reorder lazily. • Native / systems: page caches with active and inactive lists.
Watch out: segment sizes are now tuning parameters. Fixed ratios can leave memory idle (see auto-sizing segments), and some urgent promotions still need a path that doesn't wait for the tail.
Seen in memcached: fb26989756 2015-01-03 The HOT/WARM/COLD flow: new items enter HOT, and HOT and WARM drain into COLD. A COLD-tail item with `ITEM_ACTIVE` has the bit cleared and moves to WARM. 9bce42f27c makes `do_item_update` a no-op in this mode. • 4c80e2fd4c 2017-01-22 Two hits are required (`ITEM_FETCHED` then `ITEM_ACTIVE`), citing segmented-LRU papers. ebbc7d00e8 (2017-01-22) lets active HOT items flow to WARM, and bafd13d33c (2017-01-01) moves COLD hits straight to WARM. • e3d97fd12a 2019-09-30 Removed `do_item_update_nolock()`: the bump writes `it->time = current_time` directly, and warm shuffles relink with `do_item_unlink_q`/`do_item_link_q` ("we were calling extra functions and branches for no real reason").

### memcached-058 — Keep one eviction list per size class
Lens: caching · Source: memcached
Principle: When memory is partitioned by size class, evicting the global LRU victim may free a chunk of the wrong size. Keep a recency list per class so evicting a tail always frees a usable chunk in O(1).
Why it is faster: A single global LRU would have to scan for a victim of the right class, or evict several objects to free one usable chunk. Per-class lists make eviction a single unlink. The trade-off is that recency is only comparable within a class, so balance between classes needs page moves.
Applies elsewhere: Native / systems: per-bucket LRUs in GPU texture caches and allocator page caches. • Data / databases: separate eviction lists per page size in buffer pools that mix page sizes. • Web / backend: per-tier LRUs in CDN edge caches.
Watch out: classes compete blindly, and one can churn while another holds stale data. Pair per-class lists with cross-class rebalancing.
Seen in memcached: 60d709425f 2003-05-30 `items.c` holds `heads[LARGEST_ID]` and `tails[LARGEST_ID]`. When `slabs_alloc` fails, `item_alloc` unlinks from `tails[id]`.

### memcached-059 — Reuse already-dead entries before allocating or evicting live ones
Lens: caching · Source: memcached
Principle: Before allocating fresh memory or evicting a live entry, check whether the eviction-list tail holds an expired or invalidated entry. If so, take its memory directly.
Why it is faster: The dead entry's memory is reused without a free-then-allocate trip through the allocator lock, and a live item that would have been evicted survives, which raises the hit rate. A counter shows how often this path saves memory.
Applies elsewhere: Web / backend: TTL caches (lru-cache) that recycle an expired node for the next insert. • Data / databases: buffer pools that take clean or invalid frames before evicting dirty or valid ones. • Native / systems: object pools that recycle timed-out session objects.
Watch out: only check a bounded number of tail entries. A deep search for expired items rarely pays off in large caches (see bounded victim search).
Seen in memcached: d3807d06f0 2009-03-17 `do_item_alloc` checks the tail for an expired item before `slabs_alloc`: "just steal the item to avoid to grab the slab mutex twice". • bc340cdc82 2010-01-12 Added the `reclaimed` stat. • 44c0e100b8 2011-07-18 Also reclaims items invalidated by `flush_all` (`search->time < oldest_live`).

### memcached-060 — Bound the victim search and skip anything busy
Lens: caching · Source: memcached
Principle: Look at only a few entries from the eviction tail. Skip any entry that is pinned or whose lock is held, and accept an occasional allocation failure instead of waiting or scanning further.
Why it is faster: A bounded scan keeps worst-case allocation cost and lock hold time constant. Deep searches for expired entries almost never find one in a large cache. Trylock-and-skip avoids lock-order deadlocks and convoys behind request threads. Holding the item lock while checking the refcount means no one can grab a new reference during the check.
Applies elsewhere: Data / databases: buffer-pool clock sweeps with a maximum scan and skipping of pinned or latched pages. • Web / backend: Redis-style sampled eviction (look at N candidates, not all). • Native / systems: GC sweepers and connection reapers that skip in-use objects.
Watch out: if the whole tail is pinned, a bounded search always fails. Add a way to move pinned entries out of the way.
Seen in memcached: 60d709425f 2003-05-30 "search up from tail an item with refcount==0 and unlink it; give up after 50 tries". • ca5016c541 2011-10-02 Removed the 50+50 deep search: "shorter code and benches a bit faster as it holds the lock for less time." • 2db1bf462c 2012-08-17 `tries = 5` with `item_trylock(hv)` per candidate: "We walk up *only* for locked items. Never searching for expired. Waste of CPU for almost all deployments." • a039084732 2015-01-04 `lru_pull_tail` skips trylock failures and items with `refcount_incr != 2`: "you want them to fall out and cause occasional OOM's, rather than internally work around them".

### memcached-061 — Keep eviction from getting stuck on pinned or leaked entries
Lens: caching · Source: memcached
Principle: If the eviction tail fills with entries that are referenced but not evictable, move them to the head (with a large separate try budget) so the scan reaches evictable entries. Treat a very old referenced tail entry as a leak and reclaim it.
Why it is faster: Clients that request thousands of keys and never read their socket pin the tail, and every allocation in that class then fails. Rotating pinned items away restores eviction. Reclaiming leaked references avoids the alternative of monitoring and restarting the process. Once the leaks were fixed, the repair could be turned off by default.
Applies elsewhere: Data / databases: buffer pools that re-age pinned pages, and orphaned-transaction cleanup. • Web / backend: lease timeouts on distributed locks, and pool watchdogs that reclaim abandoned connections.
Watch out: forced reclaim of a reference that is actually still live corrupts memory. Use it only with a generous age threshold, make it observable, and switch it off once the leaks are fixed.
Seen in memcached: 4ad6da605d 2009-03-28 If the tail item has `refcount != 0` and `time + TAIL_REPAIR_TIME < current_time`, it is reset and unlinked (counted in the `tailrepairs` stat). • 058af0d85b 2012-09-04 Made the threshold configurable. • 59bd02ce29 2014-04-27 Disabled by default once leaks were fixed. • f2a4e5b491 2014-10-12 When `refcount_incr != 2`, the item is relinked at the head and `tries` refunded, up to `tries_reflocked = 1000`. 13340b1a12 (2014-08-20) adds the `reflocked` stat.

### memcached-062 — Keep short-lived entries in their own unpromoted list
Lens: caching · Source: memcached
Principle: Route entries whose TTL is about to expire into a separate list. They are never bumped or moved between segments, and they are reclaimed from first.
Why it is faster: Short-TTL items would otherwise churn through promotion and demotion and crowd long-lived items out of the protected segments. Isolating them removes those moves and lets maintenance reclaim expired memory from one list without scanning everything.
Applies elsewhere: Web / backend: separate stores for ephemeral tokens and durable cached objects. • Native / systems: young generations in garbage collectors, and timing-wheel buckets for short timers. • Data / databases: separate TTL partitions dropped wholesale.
Watch out: a short-TTL item that is hot gets no protection. That is fine when TTLs really are short, wrong when the threshold is set too high.
Seen in memcached: 4de89c8c69 2015-01-08 `expirezero_does_not_evict` introduces a separate NOEXP list. • 9d51fdd2de 2017-01-02 The NOEXP list becomes a "transient" list for `exptime - now <= transient_ttl`, "to relieve some pressure off the LRU crawler and do fewer cross-LRU juggles". fa8b42560a renames it `TEMP_LRU`.

### memcached-063 — Size recency segments by idle age, not only by fixed percentages
Lens: caching · Source: memcached
Principle: Cap each protected segment by how long its tail entry has been idle compared with the cold segment's tail age, so memory leaves segments whose entries aren't being reused.
Why it is faster: Fixed ratios can hold memory in a segment whose items sit idle, so the cache ran at exactly 32% utilisation (the HOT size) for some workloads. Tying segment limits to relative age moves capacity to where hits happen, approximating adaptive policies such as ARC with a cheap rule.
Applies elsewhere: Data / databases: InnoDB-style old-block timing, and ARC-style recency/frequency splits. • Native / systems: generation sizing by survival age in GCs. • Web / backend: adaptive window sizing in W-TinyLFU caches.
Watch out: age ratios are sensitive to clock resolution and to background scanner placeholders at tails. Validate them against real hit-rate data.
Seen in memcached: e2b1421353 2017-01-24 "autotuning for HOT/WARM": WARM items may be idle up to 2x the COLD tail age. • 1174994a6c 2017-06-24 Replaced the static `hot_max_age` with `hot_max_factor` (20% of COLD age), after caches were observed stuck at 32% utilisation.

### memcached-064 — Promote from a slower tier only for proven-hot entries, and only a sample of them
Lens: caching · Source: memcached
Principle: Bring an entry back from the slower tier (flash) into RAM only if it is linked, was hit at least twice and was touched recently. Even then, promote 1 in N, and only if its lock is free right now.
Why it is faster: Promoting every read churns RAM with one-off reads and costs writes back to the slow tier. Requiring repeated hits filters one-hit wonders. Sampling caps promotion bandwidth. Trylock means the read path never waits; skipping a promotion costs nothing.
Applies elsewhere: Web / backend: CDN second-hit admission, and browser memory-cache promotion from disk cache. • Data / databases: SSD-to-DRAM promotion in hybrid caches, and tiered storage that rehydrates only repeatedly read objects.
Watch out: high sample rates make warm keys slow to return to RAM after a workload shift. Tune N against flash read capacity.
Seen in memcached: f593a59bce 2017-09-26 `recache_or_free()` requires `ITEM_LINKED|ITEM_FETCHED|ITEM_ACTIVE` ("Item must be recently hit at least twice to recache"), a time within `ITEM_UPDATE_INTERVAL`, and `recache_counter++ % ext_recache_rate == 0`, all under `item_trylock` ("opt to throw away rather than wait on a lock").

### memcached-065 — Keep the cache warm across restarts with a file-backed arena and one-pass pointer fixup
Lens: caching · Source: memcached
Principle: Place the item arena in a shared mmap of a memory-backed file. On restart, map it again. If the base address changed, rebase every stored pointer in a single linear pass and rebuild the index from the linked items.
Why it is faster: A cold restart costs the whole hit rate and floods the backend while the cache refills. Reattaching saved memory restores it almost instantly. Fixing pointers once at startup avoids translating them on every access.
Applies elsewhere: Web / backend: in-process caches that survive deploys through a shared-memory segment. • Data / databases: LMDB-style mmap stores, and buffer-pool dump and reload on restart. • Native / systems: persisted JIT code caches and save-state snapshots relocated at load.
Watch out: the binary layout must match exactly across versions, or the restart must refuse to reuse the file. Recycled memory from the file isn't zeroed, so wipe pages that are reused.
Seen in memcached: b5ea901605 2019-09-17 "restartable cache" (`-e /path/to/tmpfsmnt/file`): `restart_mmap_open()` uses `MAP_SHARED`. `restart_fixup()` rewrites `it->next/prev` and chunk links by `- orig_addr + mmap_base` and calls `do_item_link_fixup`. Its comment: do this "once on startup to avoid having to do pointer fixup on every" access. • c0a868b5b5 2019-09-19 `msync` at close.

### memcached-072 — Take per-byte checks out of scanning loops and use vectorised primitives
Lens: cpu-microopt · Source: memcached
Principle: In byte-scanning loops, get the bound once with a SIMD-optimised library call (`strlen`, `memchr`). The inner loop then tests only for the delimiter, with one branch per character.
Why it is faster: Testing for a terminator on every byte adds an unpredictable branch to every iteration. libc `strlen` and `memchr` use SSE/AVX or NEON to examine 16-64 bytes at a time. Once the bound is known, the loop is tight and easy to predict.
Applies elsewhere: Web / backend: in JS, use `indexOf` or `Buffer.indexOf` rather than char-by-char loops when parsing headers or CSV. • Native / systems: `memchr`-driven line splitting, as in simdjson or picohttpparser. • Embedded: compute the frame length once before decoding, rather than testing the end on every byte.
Watch out: `strlen` needs a guaranteed terminator. On untrusted, unterminated buffers, use `memchr` with an explicit length.
Seen in memcached: ce96a20854 2007-04-17 The tokenizer is rewritten as a two-pointer scan with one branch per character and the end-of-string check folded in. • 67b6bb1d04 2011-09-30 "The \0 test in the loop was accounting for 2% of memcached's CPU usage according to callgrind. strlen is an SSE4 instruction and can sniff out that null byte quickly."

### memcached-073 — Put wide fields on their natural alignment
Lens: cpu-microopt · Source: memcached
Principle: Round record sizes to the strictest alignment of any member, and order fields in headers and wire formats so 64-bit values sit at 8-aligned offsets.
Why it is faster: Misaligned 64-bit loads can split across cache lines, run slower and lose atomicity. On SPARC and some ARM and MIPS cores they trap outright. Aligned layouts let the compiler use single load and store instructions and let structs be read directly from buffers.
Applies elsewhere: Data / databases: on-disk record formats read through mmap should place 8-byte fields at 8-aligned offsets. • Native / systems: order struct members from largest to smallest, and check the layout with `pahole` or `static_assert(offsetof(...))`. • Embedded: DMA buffers and protocol headers on Cortex-M0, which faults on unaligned access.
Watch out: alignment padding costs bytes for every object. Reorder fields to reclaim the padding instead of padding blindly.
Seen in memcached: c9607c6ddb 2006-09-03 Slab class sizes are rounded to `CHUNK_ALIGN_BYTES`. • f7a126a441 2008-02-11 Sets the alignment to 8 because "the cas_id member cause the item-struct to require 8 byte boundaries" on 32-bit builds. • 1107f5d1f3 2008-03-19 Reorders the binary-protocol extras so the 64-bit CAS sits at an 8-aligned offset.

### memcached-074 — Pay for alignment fix-ups only on platforms that need them
Lens: cpu-microopt · Source: memcached
Principle: Detect at build time whether the target traps on unaligned access. On strict platforms, `memcpy` into a local copy, or realign the buffer only when the pointer is actually misaligned. Everywhere else, keep the direct pointer cast.
Why it is faster: x86 and modern ARM handle unaligned loads almost for free, so an unconditional copy or memmove is wasted work there. A compile-time flag removes the fix-up from those builds, while strict-alignment CPUs stay correct. The flag also keeps the check out of the hot path at run time.
Applies elsewhere: Embedded: parse network headers on ARM or MIPS with `memcpy`, which compiles to a plain load where that is safe. • Native / systems: Rust `read_unaligned`, or C `memcpy` idioms that optimise to single loads on x86. • Web / backend: in JS, read unaligned binary through `DataView` and use typed-array views where offsets are aligned.
Watch out: configure-time probes can be silently optimised away, as happened here. Run CI on a real strict-alignment target.
Seen in memcached: 5268604ae6 2008-12-04 `AC_C_ALIGNMENT` sets `NEED_ALIGN`. `try_read_command` memmoves the input buffer only `if ((long)c->rcurr % 8 != 0)`. ca51c3d4e8 2009-08-23 and 2fa661a77f 2011-01-19 refine the detection. • 8333182bc8 2018-03-15 Fixes `NEED_ALIGN` detection, which GCC had optimised away, and an unaligned access in logger.c. 91102e15c8 2018-05-22 adds extstore alignment fixes. 93dfe276c7 2018-08-07 adds `ITEM_schunk()`, which pads the first chunk only under `NEED_ALIGN`. • 9f9d286d31 2025-03-27 Fixes a crawler issue on strict-alignment ARM. 1f1d030f6c 2026-05-21 adds a `FLAGS_CONV` memcpy variant and logger padding, both only under `NEED_ALIGN`. `item_hdr` reads `memcpy` on those builds and cast the pointer elsewhere.

### memcached-076 — Write accumulation loops in memory order so the compiler can vectorise them
Lens: cpu-microopt · Source: memcached
Principle: When summing a struct of counters, add the fields in the order they are laid out in memory, so adjacent loads and adds can merge into SIMD instructions.
Why it is faster: Compilers combine neighbouring scalar operations into vector operations (SLP vectorisation) only when the accesses are contiguous and in order. Shuffled field order breaks the pattern and leaves scalar code. The fix is free, but the gain is small unless the loop is hot.
Applies elsewhere: Native / systems: folding per-CPU stats in kernels, and merging histograms in metrics libraries. • Embedded: DSP accumulator loops on NEON or Helium. • Data / databases: aggregate-state merges in columnar engines.
Watch out: check the generated code. A small refactor can silently undo the vectorisation.
Seen in memcached: 68d0b9eba6 2024-05-15 `item_stats_totals`/`item_stats` move the `evicted` and `reclaimed` adds to match the `itemstats_t` field order, based on a godbolt comparison.

### memcached-078 — Split a global lock: first by subsystem, then stripe it by hash
Lens: concurrency · Source: memcached
Principle: Break a global mutex up in stages. First give each independent subsystem its own lock. Then stripe the main data lock across an array indexed by key hash, sized from the thread count. Finally give each independent list its own lock, so the global lock disappears entirely.
Why it is faster: A single lock serialises unrelated operations and its cache line bounces between cores. With striping, two keys conflict only if they share a stripe, which is rare when stripes outnumber threads. The stripe array costs a few hundred KB, far less than the data set. Too few stripes cause collisions, and too many waste memory and risk false sharing between adjacent mutexes.
Applies elsewhere: Web / backend: striped mutexes for per-user rate limiters or session maps (Java `ConcurrentHashMap` segments, Guava `Striped`). • Data / databases: per-bucket latches in a buffer-pool page table, and per-partition locks in a lock manager. • Native / systems: per-list locks for allocator size classes.
Watch out: operations that span stripes need a lock order, or they deadlock. Pad mutexes to a cache line if profiling shows false sharing.
Seen in memcached: 5b304997cc 2007-04-16 Separate locks are created for cache, slabs, stats, connections and the queue freelist. The docs judge a coarse cache lock "not a huge bottleneck" at small core counts. • 8fe5bf1fe1 2011-10-03 Adds 4000 item locks indexed by hash. Get, store and delta take the item lock, and `cache_lock` shrinks to LRU and hash pointer surgery. 97409b31af 2011-10-12 sizes the table at 2^10-2^13 depending on `nthreads`. 1c94e12c3d 2012-08-18 makes item locks the official hash-bucket locks. • 9bce42f27c 2015-01-02 "splitting cache_lock into a lock-per LRU". 90a59871d9 2015-01-09 deletes `cache_lock` ("ding-dong the cache_lock is dead"). • 4dc24dcccd 2016-12-19 Raises the item-lock hashpower to 13, 14 or 15 for up to 10, up to 20, or more threads, and notes possible false sharing between adjacent entries.

### memcached-081 — Use atomics to avoid a lock, but drop them once a lock already covers the field
Lens: concurrency · Source: memcached
Principle: A reference count updated with atomic add or subtract lets a release skip the mutex. If a later redesign puts every access under a lock anyway, go back to plain increments. Where 64-bit atomics are missing, fall back to a mutex.
Why it is faster: An atomic RMW avoids lock acquire and release and the convoying that comes with them. It still forces exclusive cache-line ownership and a fence, which is expensive across sockets. When a mutex is already held, the atomic adds that cost on top for nothing. Encoding "linked in the table" as one held reference makes decrement-to-zero an unambiguous signal to free.
Applies elsewhere: Native / systems: use Rust `Rc` instead of `Arc` in single-owner contexts, and plain fields inside an already locked page header. • Data / databases: atomic pin counts for buffer-pool pages when there is no latch, and plain ones under the latch. • Embedded: on 32-bit MCUs without 64-bit atomics, guard 64-bit counters with a critical section.
Watch out: removing atomics relies on an invariant ("always under the lock") that later code can silently break. Document it, or assert it in debug builds.
Seen in memcached: f4983b2068 2012-01-08 "Moving the cache_lock around item_remove caused a 30% performance drop, so it's been reimplemented with GCC atomics. refcount of 1 now means an item is linked but has no reference". 3b961388d1 2012-01-10 adds portable `refcount_incr`/`decr` (Solaris `atomic.h` or a mutex fallback), and de1f30c5d5 2012-01-25 adds configure detection. • 60e98600ca 2016-06-28 Logger sequence IDs: "I'd rather use atomics where available then fall back to a mutex when necessary". 8333182bc8 2018-03-15 fixes a 64-bit atomics configure test that "possibly never worked before". • ea773779cd 2017-01-09 Refcounts become `++(it->refcount)` because every call now holds the item lock: "Calling atomics is pretty expensive, especially in multicore/multisocket scenarios. This yields a notable performance increase."

### memcached-085 — Per-worker hand-off queues with intrusive links
Lens: concurrency · Source: memcached
Principle: Give each worker thread its own inbound queue and node freelist instead of a global one, and embed the list link in the work object itself so handing it over allocates nothing. Keep one queue per thread rather than one per connection.
Why it is faster: A global freelist or queue is a single contended mutex. Per-worker queues reduce contention to one producer and one consumer per lock. An intrusive link removes a wrapper alloc and free per message and the mutex round trips that came with them. A per-thread queue also lets a single submit cover many connections, and it shrinks every connection struct.
Applies elsewhere: Native / systems: per-core run queues in thread pools, and kernel-style intrusive lists for request structs. • Web / backend: actor runtimes that avoid per-message envelope objects, and per-worker job queues in Node `worker_threads`. • Embedded: per-core message queues whose nodes live inside the message buffers.
Watch out: an intrusive link means an object can sit in only one queue at a time. Clear ownership rules are required.
Seen in memcached: 5b304997cc 2007-04-16 Hand-off queue items are allocated 64 at a time and recycled through a freelist ("to reduce fragmentation"). • 331dca5d64 2021-07-28 Removes the global `cqi_freelist` and its lock. Each worker's `CQ` gets its own cache and `STAILQ` ("help scalability a bit"). • 48e6702582 2023-06-27 `io_pending_t.iop_next` is an intrusive link, with a dedicated `ion` eventfd, lock and head for IO returns, so CQ_ITEM wrapping goes away. Up to "10%+ improvement" under heavy extstore or proxy use. • ec1fb560fa 2024-12-29 Moves `io_queues[]` from every `conn` to the worker thread, adding `resps_suspended` counters. This "Reduces connection structure size ~10%" and enables batching submits across connections.

### memcached-088 — Let the owner thread keep the accounting, and tolerate a small overshoot
Lens: concurrency · Source: memcached
Principle: Track "outstanding batches" or memory used in a field that only the owning thread touches. Apply remote contributions when the object comes back through the owner's own queue. Soft limits may be exceeded briefly.
Why it is faster: A shared counter needs a mutex or atomic on every submit and completion, sometimes several times per request. Partitioning by ownership turns these into plain loads and stores. Enforcement becomes approximate, which is acceptable for soft limits.
Applies elsewhere: Web / backend: an actor or async task counts its own fan-out replies, and per-thread quota counters are reconciled lazily. • Native / systems: per-CPU byte counters in allocators, folded in on free. • Data / databases: a parallel-scan coordinator counts completions on its own thread.
Watch out: hard limits such as security quotas or strict memory caps can't tolerate overshoot. Size the slack explicitly.
Seen in memcached: 5d4785936e 2020-10-20 `c->io_queues_submitted` is owned by the worker and `io_queue_t.count` by the side thread after submit: "All of this is to avoid having to hit a mutex owned by the connection thread". 57493bfca4 2021-07-30 and 3fc8775bf0 2021-08-02 extend this to flow mode. • 31176ee383 2025-09-04 Deletes `proxy_limit_lock`, which was taken "a minimum of 4 times per request and could be dozens". Buffer memory is counted when results return to the worker.

### memcached-089 — Split global limits into per-worker budgets
Lens: concurrency · Source: memcached
Principle: Divide a global concurrency or memory limit by the number of worker threads and enforce each share locally, with no shared counter.
Why it is faster: Checking a global limit means a contended atomic or lock on every request. Local budgets turn the check into a plain compare on thread-owned data. Under overload, each worker sheds load independently instead of everyone competing for one counter.
Applies elsewhere: Web / backend: per-core token buckets in load balancers, and per-worker connection caps in Node cluster mode. • Native / systems: per-core mempools in DPDK. • Data / databases: per-session or per-worker `work_mem`-style caps.
Watch out: skewed load can exhaust one worker's share while others sit idle. Use this for loosely balanced workloads, or let workers rebalance their budgets.
Seen in memcached: 05721e4b6c 2020-02-25 `cache_set_limit(resp_cache, resp_obj_mem_limit/num_threads/sizeof(mc_resp))`, and the same for read buffers. OOMs are counted per pool. • 6c80728209 2023-03-04 `mcp.active_req_limit` and `mcp.buffer_memory_limit`: "Divides by the number of worker threads and creates a per-worker-thread limit".

### memcached-090 — Take each lock once per operation, and take the heavy path only when needed
Lens: concurrency · Source: memcached
Principle: Don't look up under a lock, release it, and then lock again to update. Pass an intent flag so the update happens inside the first critical section. Before touching shared state, scan the request to decide whether it needs the heavier extended-hold path, and give plain reads a lighter lookup.
Why it is faster: Each extra acquire and release costs atomics and possible contention, and a second lookup re-hashes the key. Pre-scanning means the common read pays only for the cheap path.
Applies elsewhere: Native / systems: map `entry()`/`compute()` APIs instead of lookup-then-insert. • Data / databases: `UPDATE ... RETURNING` instead of SELECT then UPDATE, and a plain read versus `SELECT ... FOR UPDATE` chosen per request. • Web / backend: a cache client that offers `getAndTouch`.
Watch out: fusing operations widens the API and can lengthen a single hold. Keep the fused work small.
Seen in memcached: e87ab394f3 2017-01-09 `do_item_get(..., DO_UPDATE)` does the LRU bump inside the lookup lock and removes the separate `item_update()` calls: "Good performance bump by inlining the LRU bump". • 04613a4736 2019-11-14 `_meta_flag_preparse` sets `of.locked` only for the N/T/R/l/h flags. Otherwise the get uses `limited_get` instead of `limited_get_locked`.

### memcached-092 — Pick lock-stripe bits as a subset of the bucket bits so one stripe covers a split
Lens: concurrency · Source: memcached
Principle: If the lock index is the low M bits of the hash and the bucket index is the low N bits, with N at least M, then one lock covers an old bucket and every bucket it splits into. A resize can then migrate data while holding only that lock.
Why it is faster: Resizing no longer needs a global lock. Each step blocks only the requests that hash to one stripe, and if that stripe is busy the migrator sleeps and retries rather than stalling everyone.
Applies elsewhere: Native / systems: online resizing of striped concurrent hash maps. • Data / databases: linear-hashing bucket splits under a single parent latch.
Watch out: the invariant (stripe count no larger than bucket count, using the same low bits) must hold at all times. Breaking it silently allows races.
Seen in memcached: 1c94e12c3d 2012-08-18 Item locks become the hash-bucket locks. • d2676b4aa5 2014-10-24 `assoc_maintenance_thread` calls `item_trylock(expand_bucket)` per old bucket, sleeps 100 ms when it is busy, and drops the global and cache locks from the migration loop. Expansion is skipped if `item_lock_hashpower >= hashpower`.

### memcached-093 — Prefer blocking mutexes to trylock spin loops
Lens: concurrency · Source: memcached
Principle: Busy-waiting on trylock can help when critical sections are a few hundred cycles long and threads have dedicated cores. Under oversubscription it backfires. Measure before you keep it, and default to a blocking (adaptive) mutex.
Why it is faster: Spinning avoids futex sleep and wake, but it burns the waiter's timeslice and can keep the lock holder off the CPU. Modern mutexes already spin briefly before sleeping. Once the critical sections had been shortened, plain blocking locks won.
Applies elsewhere: Web / backend: avoid user-space spinlocks in containers with CPU quotas. • Native / systems: parking-lot or adaptive mutexes rather than hand-written spin loops. • Embedded: spin only on true multicore parts with very short sections. On a single core, spinning is pure waste.
Watch out: spinlock benchmarks on idle dedicated machines don't predict production behaviour on shared hosts.
Seen in memcached: 45e0e9500f 2011-10-02 `mutex_lock()` becomes `while (pthread_mutex_trylock(m));`. It "appears to significantly help prevent performance dropoff from additional threads, but only when the locks are frequently contested and are short." • 0aa1a82a4c 2015-01-09 Reverts to `pthread_mutex_lock`: "spinlocks never seem to help in benchmarks… killing the spinlock only makes things better."

### memcached-095 — Assign connections to threads by NIC receive queue
Lens: concurrency · Source: memcached
Principle: Choose the worker for a new connection from the hardware RX queue its packets arrive on (`SO_INCOMING_NAPI_ID`), so each thread serves one NIC queue.
Why it is faster: Interrupt handling, socket buffers and application processing then stay on one core and its cache. This reduces context switches and cross-core traffic, and makes epoll busy-polling effective. It needs RSS and IRQ affinity to be configured to match.
Applies elsewhere: Native / systems: SO_REUSEPORT with CPU-affinity BPF steering in nginx or Envoy, and RSS-aligned cores in DPDK or Seastar. • Data / databases: brokers or DB servers that pin network threads to IRQ cores.
Watch out: when the thread count doesn't match the RX queue count, or traffic hashes unevenly, load becomes skewed.
Seen in memcached: 4de258ed89 2020-08-18 `select_thread_by_napi_id()` maps each newly seen NAPI id to the next thread and falls back to round-robin (counting `round_robin_fallback`). It is enabled with `-N`, and "streamlines the flow of data from the NIC to the application" with "reduced context switches" when busy polling is enabled.

### memcached-097 — Dispatch to the shallowest queue, reading depths without a lock
Lens: concurrency · Source: memcached
Principle: Send each unit of work to the worker or connection with the smallest queue, stopping at the first idle one. Read depths without locking, since a stale guess is harmless. Consumers take only small batches so the depths stay meaningful.
Why it is faster: Round-robin lets slow queues build backlogs under sustained overload while others sit empty. Join-shortest-queue keeps latency even. Unlocked depth reads keep the selection cheap. If a consumer drains huge batches, the depth signal stops reflecting real load.
Applies elsewhere: Web / backend: least-connections load balancing, and routing to the least-busy worker in a cluster. • Native / systems: NVMe submission-queue selection, and thread-pool dispatch by queue length. • Data / databases: gRPC or DB channel pools that pick the least-loaded connection.
Watch out: scanning every queue is O(N) per dispatch. With many queues, sample two at random (power of two choices).
Seen in memcached: 566610d41b 2018-10-02 extstore `_get_io_thread()` takes the first zero-depth queue or the minimum: "during sustained overload some queues can get behind while other stays empty". The present-day code ignores the lock "since being wrong isn't fatal", and IO threads "chew small batches" to keep depths even. • fd4e933e9f 2023-03-23 The proxy keeps up to 8 sockets per backend, and `proxy_choose_beconn` picks the one with the smallest depth, skipping bad connections.

### memcached-098 — Move housekeeping to background threads and keep a bounded inline fallback
Lens: scheduling · Source: memcached
Principle: Run maintenance such as rehashing, list balancing, reclaim and flushing on dedicated threads so request threads rarely do it. Keep a small, bounded inline "direct reclaim" path so a request never blocks waiting for the background thread.
Why it is faster: Maintenance done inside request handling adds latency to every request and contention on hot locks. A background thread can work in tunable batches when the system is idle. The inline fallback covers bursts that the background thread can't keep up with, and it must stay small: evict from one list and stop.
Applies elsewhere: Native / systems: Linux kswapd plus direct reclaim, and concurrent GC with allocation assists. • Data / databases: checkpointer and background writer threads, where a foreground backend flushes a page only when no clean page exists. • Web / backend: resize or prune in-process caches from a timer or worker instead of inside request handlers.
Watch out: background threads add synchronisation and pause-coordination complexity. If the inline fallback is too generous, it quietly becomes the main path again.
Seen in memcached: 217dcce072 2006-11-26 Incremental hash expansion first migrates one bucket per request transition, inside the request path. • 7f09e20ba5 2009-01-27 Moves this to `assoc_maintenance_thread`, because per-command migration caused "extra contention on an already hot mutex". It moves `hash_bulk_move` buckets per lock hold. • 10698bae63 2011-12-18 A background slab rebalancer moves pages `slab_bulk_check` items at a time. • fb26989756 2015-01-03 Adds the LRU maintainer thread. a039084732 2015-01-04 "Only way to do eviction case fast enough is to inline it": allocation retries up to 5 times, pulling tails inline and counting `direct_reclaims`. e1d9af2bcf 2017-01-08 trims the inline path to "evict from COLD; only pull HOT if that produced no reclaims". • 954f4e044b 2018-07-11 Moves flash writes to a dedicated storage writer thread. 277f54dfeb 2022-02-21 puts writes and compaction IO on their own extstore thread, which improves tail latency for foreground reads, and runs the compaction checker only once per slab page of bytes written.

### memcached-099 — Pace background work adaptively: short sleeps while productive, exponential backoff when idle or stuck
Lens: scheduling · Source: memcached
Principle: Make a background thread's sleep depend on what its last pass achieved. Drop to a minimal sleep while it is making progress, grow the sleep exponentially (up to a tunable cap) while idle, and keep the backoff per work unit so one busy unit doesn't make the thread poll all the others. Reset the backoff only when real progress is made.
Why it is faster: A fixed tick wastes CPU and wakeups when idle and falls behind when busy. A sleep-after-burst pattern causes flood-then-idle oscillation. Resetting the backoff on partial progress makes a thread spin on stuck items.
Applies elsewhere: Web / backend: queue pollers with backoff (like SQS long-poll), and cache-refresh workers that speed up under churn. • Embedded: sensor tasks that lengthen their sleep when readings are quiet, which saves power. • Data / databases: autovacuum or compaction schedulers with a separate backoff per table.
Watch out: a long maximum sleep delays the response to sudden demand. Pair adaptive sleep with a condition-variable wake for urgent work.
Seen in memcached: 57a9856a2f 2012-07-15 The rebalancer uses `usleep(50)` instead of `sleep(1)` when it hits busy items. • a039084732 2015-01-04 The LRU maintainer's sleep grows by 1 ms when idle and resets on moves. 2266c8402b 2017-01-08 adds per-slab-class `next_juggles[]`/`backoff_juggles[]`, so the thread sleeps until the soonest class is due. f0a3c4954f 2017-02-17 makes the backoff exponential (`+= x/8`) and raises the logger cap to 1 s to stop wakeups on idle machines. • 954f4e044b 2018-07-11 The storage writer uses a `counter % backoff[x]` skip and halves all backoffs after long sleeps so classes don't drift apart. • 397464bedd 2019-09-23 The slab mover backs off from 1 µs to 1 ms (`backoff_timer *= 2`) when blocked. ae3efecb15 2022-02-21 the compaction thread drops to `MIN_STORAGE_COMPACT_SLEEP` while compacting and ramps toward a tunable `ext_max_sleep`. • 4c56c8d3e2 2024-11-14 / ffe8d03d64 2025-03-03 The mover sleeps 100 ms while productive and 1 s only after about a second with no work, fixing "flooding moves and then sleeping for a full second, causing large oscillations". 8c02ee82c9 2025-08-21 makes `slab_rebalance_move` return 0/1/2 so only completed pages reset the backoff, and raises the cap from 1000 to 5000. Frequent resets had been "causing the thread to use a lot of CPU when items are stuck".

### memcached-102 — Scan incrementally with a cursor that survives concurrent changes
Lens: scheduling · Source: memcached
Principle: Walk a shared structure in small steps with the lock released between steps. Either put a sentinel cursor node into the linked list and move it forward one item per step, or iterate hash buckets one at a time while blocking resizes.
Why it is faster: A full scan under one lock blocks every request for its duration. A cursor that is itself a list node stays valid while other threads insert and remove around it, so each step needs only a brief lock hold, with an optional sleep between steps. A bucket walk visits each item exactly once, while LRU walks can miss items that get reordered.
Applies elsewhere: Data / databases: background TTL sweepers and checksum scrubbers that latch one page or tuple at a time. • Web / backend: Redis `SCAN`-style cursors for admin exports, rather than a full `KEYS`. • Native / systems: incremental GC marking by region.
Watch out: a cursor node must be skipped by every other list walker, and needs a marker that real items can never have.
Seen in memcached: 0d1f505c15 2014-04-14 The LRU crawler links a `crawler` pseudo-item, which shares the item header layout, at each class's tail. Each step locks, swaps the cursor past one item, trylocks that item, frees it if expired, unlocks and sleeps `lru_crawler_sleep`, interleaving classes round-robin. 649f7f0149 2014-04-15 adds a reclaims stat. • d184b4b08f 2017-05-29 Schedules sub-LRUs independently and makes the crawler drop its lock often. • b2cee81823 2020-11-09 Hash-table walk mode: `assoc_get_iterator()` holds `maintenance_lock` so the table can't resize, locks one bucket at a time and sleeps between buckets. It "doesn't suffer from missing items because of LRU reordering or high lock contention".

### memcached-103 — Schedule scans by their predicted yield
Lens: scheduling · Source: memcached
Principle: While scanning, record a histogram of when items will become reclaimable. Before the next scan, add up the buckets that have since come due, and run only if the expected yield passes a threshold. Otherwise back off, up to a long ceiling.
Why it is faster: Blind periodic scans spend CPU and lock time walking data that is still alive. A cheap histogram from the previous pass predicts the payoff, so scans run when they will reclaim a lot and rarely when they won't.
Applies elsewhere: Data / databases: TTL-index cleanup and LSM compaction triggered by the predicted garbage fraction. • Web / backend: session-store sweepers scheduled from expiry histograms. • Native / systems: GC heuristics that predict survivors from the previous cycle.
Watch out: a stale histogram mispredicts after workload shifts, so keep a maximum interval between scans.
Seen in memcached: e708513abc 2015-01-06 `lru_maintainer_crawler_check()` keeps `histo[60]` (one bucket per minute for the next hour). It re-crawls a class when `available_reclaims > seen/100 + 1`, and otherwise grows the wait by 60 s up to 3600 s. • d184b4b08f 2017-05-29 Tracks and schedules each sub-LRU independently.

### memcached-104 — Run garbage collection between requests, not during them
Lens: scheduling · Source: memcached
Principle: In an embedded managed runtime, stop the collector while a request runs. Between event-loop iterations, run small incremental steps that grow with how far memory is over target, plus a slow timer step so idle VMs still free memory. Tell the GC about native memory it can't see, and release pinned native resources at scope exit instead of waiting for collection.
Why it is faster: Each request produces little garbage, so collecting mid-request only adds latency. Running between iterations moves the cost to where it doesn't block a response. Native allocations are invisible to the GC's pacing, so reporting fake allocations makes sure cycles actually run. Deterministic release frees scarce native resources, such as item references that block eviction, without depending on GC timing.
Applies elsewhere: Web / backend: Node `adjustExternalMemory` for native buffers, and `using`/`Symbol.dispose` for handles. • Native / systems: game engines that run GC or reset arenas between frames, and .NET `GC.AddMemoryPressure`. • Data / databases: embedded scripting (Lua or JS UDFs) in DB servers, collected between statements.
Watch out: with GC stopped, one request that allocates heavily can balloon memory. Keep a hard ceiling that forces a collection.
Seen in memcached: 103890d807 2023-09-14 Adds a periodic timer GC step. • e58b8dcc62 2024-02-13 The worker loop becomes `event_base_loop(EVLOOP_ONCE)` followed by `proxy_gc_poke`, with `lua_gc(L, LUA_GCSTOP)` during requests. A cycle starts when memory doubles, and steps scale with bloat. Per-request native kb feed `proxy_vm_extra_kb`. • 38ac376875 2024-05-02 Tunes the stepping so the GC can catch up. d164eb151a 2024-11-11 only lets GC speed rise within a run, removes branches and gives one owner the decision of when to run. • 6fdc1e6470 2024-05-30 "Adds a VM fudge when freeing a request slot. This tricks the GC to run". • 0954b53cd1 2024-05-24 Adds Lua 5.4 `<close>` support for result objects ("references must be held for a short and temporary period of time"). 5d97fda21f 2024-05-29 adds an explicit `res:close()`. • d164eb151a 2024-11-11 GC speed only ramps up within a run, with fewer branches and one owner of the run decision. 998e18c783 / b4a061ccaa configurable ratio.

### memcached-105 — Trigger maintenance from a periodic tick or a trylock signal, not a per-operation counter
Lens: scheduling · Source: memcached
Principle: Don't maintain a shared counter on every insert just to decide when to resize. Check an existing total from a once-per-second timer. When a hot path does notice the need, signal the maintenance thread through a trylock, so the hot path never blocks and needs no racy flag.
Why it is faster: Checking a threshold on every insert or delete means a shared counter and a lock in the hottest path. A periodic check reuses numbers that are already maintained and costs nothing per operation. If the trylock succeeds, the hot path signals. If it fails, someone is already handling it.
Applies elsewhere: Web / backend: autoscaling or cache-resize decisions from sampled metrics instead of per-request counters. • Native / systems: allocators that check decay or purge thresholds on a timer, and GC-trigger hints from allocation paths. • Data / databases: periodic compaction and statistics refresh checks.
Watch out: a slow tick reacts late to bursts, and the table can overfill between checks. Choose the interval against the worst-case insert rate.
Seen in memcached: 5d7dc88da3 2015-02-06 After `cache_lock` was removed, a mutex had to be added around `hash_items`. • 5c0face158 2018-02-12 Removes `hash_items` and its lock from `assoc_insert`/`delete`. `assoc_start_expand(curr_items)` runs from `clock_handler`, which also fixes expansion failing to start again after a burst of sets. • 4575e1f966 2019-11-09 Removes the racy `started_expanding` bool: `if (pthread_mutex_trylock(&maintenance_lock) == 0) { if (curr_items > hashsize*3/2) pthread_cond_signal(...); unlock; }`.

### memcached-106 — Let foreground operations help a stuck background task
Lens: scheduling · Source: memcached
Principle: When a background mover can't finish because hot objects are always in use, have foreground readers of those objects do part of the work, such as treating an object in the region being moved as a miss and unlinking it. If something stays busy too long, the background task is allowed to take a harsher action.
Why it is faster: Busy-waiting on popular items can stall a background move forever. The request that holds the object is in the best position to release it. The cost is a few extra misses during the move.
Applies elsewhere: Native / systems: read barriers in moving GCs, where mutators evacuate objects they touch, and allocation-triggered GC assists. • Data / databases: draining a shard during rebalancing by redirecting reads, and online defragmentation.
Watch out: foreground assists add latency and misses to user requests. Limit them to the region being moved and to the duration of the move.
Seen in memcached: b3630e1a2c 2012-01-05 `do_item_get`: if a rebalance is active and the item lies within `[slab_start, slab_end)`, the reader unlinks it and returns NULL. The code calls this an "Optimization for slab reassignment. prevents popular items from jamming in busy wait." • d67d18791f 2017-06-15 The slab mover deletes items that stay busy so it can't get stuck.

### memcached-107 — Record finished sub-units so retry passes skip them
Lens: scheduling · Source: memcached
Principle: When a background sweep over a region must loop because some entries are busy, keep a per-entry "done" bitmap or byte vector. Later passes then skip finished entries without relocking or rechecking them.
Why it is faster: Without a completion record, each retry pass redoes lock acquisition and validation for every entry, so a single stuck item makes the cost quadratic. The bitmap costs a byte per entry for one region at a time.
Applies elsewhere: Data / databases: vacuum skipping already processed pages through a visibility map, and resumable backfills that track finished ranges. • Native / systems: defrag or compaction passes in allocators, and GC evacuation retrying pinned objects. • Web / backend: batch migrations that checkpoint completed IDs.
Watch out: an entry marked done must not be able to become not-done again without the map being reset.
Seen in memcached: 397464bedd 2019-09-23 `slab_rebal.completed = calloc(perslab, 1)`, and the loop checks `if (completed[offset] == 0 && …)`. Unreferenced items are freed in place: "Avoids looping … Avoids re-locking and re-checking already completed memory". • 4c56c8d3e2 2024-11-14 The rewritten slabs_mover.c skips the slabs lock for chunks already marked done in `rebal.completed[offset]`.

### memcached-108 — Spread periodic scans over time and sleep until the earliest deadline
Lens: scheduling · Source: memcached
Principle: Walk a large table in slices with sleeps between them, rather than in one burst. Compute the next wakeup from the earliest possible deadline instead of a fixed tick. Ask the owning thread to act rather than touching its objects directly.
Why it is faster: Full-table scans cause CPU and cache bursts that show up as latency spikes. Slicing smooths the load out, and deadline-based sleeps avoid pointless wakeups. Signalling the owner keeps the scanner lock-free with respect to connection state.
Applies elsewhere: Web / backend: idle-session reapers and WebSocket ping sweeps. • Embedded: timer-wheel style RTOS sweeps. • Native / systems: lease-expiry scanners in lock services.
Watch out: slicing makes timeout enforcement less precise. Size the slices so the worst-case lag is acceptable.
Seen in memcached: 83ba6bd953 2014-09-03 `conn_timeout_thread` scans `CONNS_PER_SLICE 100` per step with `usleep(1000000 / (max_fds / CONNS_PER_SLICE))` and sleeps `idle_timeout - (now - oldest_last_cmd) + 1`. Kills are sent as a `'t'` message to the owning worker's notify pipe.

### memcached-109 — Give background retries a total time budget with a per-item cap
Lens: scheduling · Source: memcached
Principle: Bound how long a background phase waits overall, then cap the wait for each item, so it neither gives up too early (losing work) nor falls hopelessly behind.
Why it is faster: A fixed small retry count drops items under transient load. Unlimited patience lets the phase fall behind until other mechanisms, such as eviction, take over anyway. A two-level budget adapts to both cases.
Applies elsewhere: Web / backend: retry budgets in RPC clients, with a per-call timeout plus an overall deadline. • Data / databases: time-boxed LSM compaction or replication catch-up phases. • Embedded: flash-erase or bus retries bounded per operation and per cycle.
Watch out: budgets hide chronic overload. Export a stat whenever the budget is exhausted.
Seen in memcached: 2037c5f3bb 2026-03-03 `storage_compact_readback` now waits up to 5 s total per readback phase, then at most 10 ms per item, replacing 10 retries of 1 ms. Too little patience had caused key loss, and too much put compaction behind and caused evictions anyway.

### memcached-117 — Suspend a request on slow I/O, batch the submissions, resume on completion
Lens: io · Source: memcached
Principle: When a request needs slow I/O (flash, an upstream server), reserve its response slot, queue the I/O, take the connection off the event loop and keep serving others. Submit queued I/O in batches. Resume the connection on its owning thread when the last result arrives.
Why it is faster: A blocking wait idles the whole event-loop thread. Stacking every I/O for a multiget and submitting together overlaps device latency. Batching submission across several ready connections pays the wakeup and syscall once per batch. Doing the I/O bookkeeping in one fixed allocation with a per-thread queue keeps the per-request overhead small.
Applies elsewhere: Web / backend: DataLoader-style batching in GraphQL; a Node proxy collecting upstream calls from all ready sockets and flushing once per tick. • Native / systems: io_uring submissions batched per loop pass; NIC doorbells after N descriptors. • Data / databases: Postgres pipeline mode; WAL group commit across sessions.
Watch out: bound the batch or the p99.9 grows (memcached saw a slightly longer p99.9). Responses must still go out in request order, so reserve their slots up front.
Seen in memcached: 31e7d8f5fe 2016-08-02 Side threads can redispatch a connection back to its worker. c053584ad1 2017-04-25 tags queue entries by type. • extstore (state at 2018): `_get_extstore()` allocates the destination item, reserves an iovec placeholder, chains `io_wrap`s, then `event_del` plus one `extstore_submit` for the whole stack. The callback calls `redispatch_conn` when `io_wrapleft == 0`. • 0d4cd8af75 2020-08-30 The generic `io_pending_t` carries a 120-byte payload that each subsystem casts to its own struct: "what might require 3 allocs stays as just 1". • ec1fb560fa 2024-12-29 IO queues move from each connection to each worker thread ("Reduces connection structure size ~10%"), so one submit can cover many connections. • 4a04d84691 2025-01-21 Submit only every ~20 connections or after the loop pass: p50 went from 700 µs to 100 µs at 500k rps on 16 cores, with p99 unchanged. fc354ee58a 2025-01-27 resets the counter per submit. • 83425f30ef 2025-01-14 I/O completed on the owning thread returns directly, and resumption is scheduled with `event_active` instead of a recursive call: "5%+ performance bump".

### memcached-118 — Prove a new I/O interface beats a tuned baseline, or delete it
Lens: io · Source: memcached
Principle: Adopt a new kernel interface (io_uring and the like) behind a switch, size it for backpressure, measure it against the best version of the old path, and remove it if it doesn't win.
Why it is faster: Fewer syscalls per operation only matter if syscalls were the bottleneck. Once a readiness loop is tuned (persistent events, batched writes), its overhead may be small enough that the new path adds only complexity. When used, an oversized completion queue and reaping all completions before advancing the ring once keep it from blocking.
Applies elsewhere: Native / systems: benchmark SIMD, GPU or async-IO paths against an optimized scalar or readiness baseline before keeping them. • Web / backend: feature-flag experimental fast paths and delete the losers.
Watch out: benchmarking against an untuned baseline makes almost any new interface look good.
Seen in memcached: d22b66483b 2020-06-10 Proxy IO thread with optional io_uring: CQ 16384 vs SQ 2048 via `IORING_SETUP_CQSIZE` ("backpressure issues can cause us to block on SQ submissions"), reaping with `io_uring_for_each_cqe` plus one `io_uring_cq_advance(count)`. • 305b7864e6 2021-12-03 timeouts. 5e5f18a893 2022-01-26 made it opt-in. • d15bbb16d6 2022-12-14 Moving writes behind the ring "does not result in any performance uplift". • 595758ad02 2023-03-13 Removes 663 lines: after the event handler rewrite the IO thread "scales much better (up to 8-12 worker threads), leaving the io_uring code in the dust".

### memcached-119 — Tier cold values to flash, keeping a tiny in-RAM pointer header
Lens: io · Source: memcached
Principle: Keep keys and metadata in RAM, write cold values to SSD, and replace each in-memory object with a small header (page id, page version, offset). A hit then costs exactly one device read.
Why it is faster: RAM holds far more keys when values live on flash, so the hit rate rises for the same memory budget. Because the index stays in RAM, a miss costs no I/O and a hit costs one read. Writes run on a background thread from the LRU tail, so the request path never waits on flash writes.
Applies elsewhere: Data / databases: key-value separation (WiscKey, BlobDB); hybrid RAM/SSD caches. • Web / backend: in-memory index with large payloads in IndexedDB/disk. • Embedded: index in SRAM, records in external flash.
Watch out: each flashed item still costs a header, so tiering tiny values saves little and can run away (see the flush clamp entry below).
Seen in memcached: f593a59bce 2017-09-26 extstore base: the LRU maintainer writes COLD-tail items out, and the originals become `ITEM_HDR` items whose data is an `item_hdr`. • ce79ff67c0 2017-09-27 Split write into request + write so hash and CRC go into the write-buffer copy and the live item is untouched. • 954f4e044b 2018-07-11 Writes move to a dedicated storage writer thread. • Present-day storage.c reuses the on-flash copy's meaningless `time` field for the key hash and `exptime` for the CRC32C, so the record doesn't grow.

### memcached-120 — Write flash append-only through page buffers, and invalidate with page versions
Lens: io · Source: memcached
Principle: Copy objects into a large in-RAM buffer for the active page and flush it sequentially. Stamp each reused page with a new version so stale pointers fail a cheap compare instead of needing a scan or reverse index. Serve reads of not-yet-flushed data from the buffer.
Why it is faster: Large sequential writes are what flash does best, and they reduce write amplification and wear. Version stamps make "free a whole page" O(1) no matter how many RAM headers point into it. Reading from the write buffer avoids blocking on, or racing with, the pending flush.
Applies elsewhere: Data / databases: LSM memtables flushed as segments with read-through; generation counters on slot maps. • Embedded: FTL-style block versioning on raw NAND or NOR. • Native / systems: write-back caches in disk controllers.
Watch out: append-only storage needs compaction to reclaim partially dead pages. Without lifetime-aware placement (next entry), compaction rewrites a lot of live data.
Seen in memcached: f593a59bce 2017-09-26 extstore.c: `_store_wbuf` per active page, `store_page.version` from `_next_version()` (0 = freed). Reads, deletes and checks compare `p->version == page_version`. • Present-day extstore.c `_read_from_wbuf`: when `p->active && offset >= p->written`, copy from RAM (iovec-aware). Write buffers retire only from the head so offsets stay valid. Unused tails are zeroed "to allow blind readback of data".

### memcached-121 — Place data in separate write streams by expected lifetime
Lens: io · Source: memcached
Principle: Route writes into different pages by kind: short-TTL, compacted survivors, chunked, cold, old. Whole pages then die together and need no copying.
Why it is faster: Mixing short- and long-lived data means pages are never fully empty, so compaction keeps rewriting live bytes. Segregating by lifetime turns reclamation into "drop the page". Buckets also let cold data go to cheaper devices.
Applies elsewhere: Data / databases: time-partitioned tables dropped by partition; Cassandra time-window compaction. • Native / systems: F2FS hot/cold segments; generational heaps. • Web / backend: separate stores for ephemeral sessions and durable data.
Watch out: too many streams leave many partially filled active pages and buffers. Predicted lifetimes must be roughly right.
Seen in memcached: f593a59bce 2017-09-26 doc/storage.txt: a separate bucket for objects rewritten by the compactor. • 46a297cb0b 2017-11-16 `ext_low_ttl`: items expiring soon go to `PAGE_BUCKET_LOWTTL`, which compaction skips. • 277f54dfeb 2024-03-18 (authored 2022) Tiered buckets: default, compacted, chunked, low-TTL, cold, old, assignable to different devices.

### memcached-122 — Compact by reclaimable bytes, throttle the checker, and drop unread data under pressure
Lens: io · Source: memcached
Principle: Compact the page with the most reclaimable bytes. Scale the trigger by how few free pages remain. Run the selection logic only once per page-worth of writes. When space is critical, drop never-read objects rather than copying them. Give readback a total time budget.
Why it is faster: Compaction is pure overhead I/O. Picking the most fragmented page frees the most space per byte rewritten. Rate-limiting the CPU-heavy checker saves cycles. Dropping cold unread data avoids copying bytes that would be evicted anyway.
Applies elsewhere: Data / databases: LSM compaction prioritized by garbage ratio; cost-benefit cleaning in log-structured file systems. • Web / backend: retry budgets with a total deadline in RPC clients.
Watch out: foreground reads must not queue behind compaction I/O. Give compaction its own thread or queue.
Seen in memcached: f593a59bce 2017-09-26 Compaction limit 0.9 × free-page fraction. Reads whole write-buffer-sized ranges. • 73351ead77 2017-09-28 Skip unhit objects when fewer than 2 free pages. 81c83e803d / 2adab846a3 2017-11-28 make `ext_drop_unread` and `ext_compact_under` tunable. • 277f54dfeb 2024-03-18 Tracks `bytes_fragmented` per bucket and picks the most fragmented page. `check_compact -= chunk_size` wakes the checker once per page of writes. Writes and compaction IO move to a dedicated background IO thread "improving tail latency". • 2037c5f3bb 2026-03-03 Readback waits up to 5 s per phase, then at most 10 ms per item, replacing 10×1 ms retries. Too little patience lost keys, and too much led to evictions.

### memcached-123 — Flush to the slower tier ahead of need, driven by a free-memory watermark
Lens: io · Source: memcached
Principle: Keep a reserve of free memory and start demoting data to the slower tier whenever the reserve dips, instead of waiting until allocation fails. Clamp demotion where it saves little memory.
Why it is faster: Demoting on demand puts eviction-like stalls on the allocation path and evicts data that could have been tiered. A watermark keeps allocations instant. Flushing small items leaves a header behind and saves little, so capping it stops metadata growth from eating the gain.
Applies elsewhere: Native / systems: kswapd low watermarks; `dirty_background_ratio` writeback. • Data / databases: keep a free-frame reserve in a buffer pool; LSM memtable flush triggers. • Web / backend: don't offload objects smaller than their index entry to a remote tier.
Watch out: a watermark set too high wastes RAM and flash bandwidth. Too low and you're back to on-demand stalls.
Seen in memcached: 8421b6c7a3 2017-12-03 Per-class `ext_free_memchunks`: when below target and at the memory limit, the writer ignores item age and flushes at once. 97b52b1866 2017-12-18 keeps the global page pool prefilled. • 3d6d74a340 2022-08-18 Replaces per-class targets with `global_pool_watermark = total_pages × free_ratio` (min 2). Without it the server evicted when it could have tiered. 7b7ffaecf6 2022-08-19 re-allows aggressive moves when much memory frees at once. • e0e415b7b2 2023-05-07 `target_pages` from the deficit, and minimum write sleep lowered 500→200 so the flusher keeps up. • ffe8d03d64 2025-03-03 Cap pages reclaimed per pass for small classes (3 under 500 B, 4 under 1000 B, 5 under 2000 B). Begin gentle flushing at the pool minimum, not below it. d918990f86 2025-02-26 waits one window before acting on a live-tuned watermark.

### memcached-124 — Drive devices independently instead of striping them
Lens: io · Source: memcached
Principle: Give each storage device its own active pages and route writes per device, rather than putting them behind RAID striping.
Why it is faster: With striping, one SSD's garbage-collection pause stalls every write. Independent devices isolate stalls and add bandwidth.
Applies elsewhere: Data / databases: Kafka `log.dirs`; Cassandra JBOD data directories. • Native / systems: object stores placing shards per disk.
Watch out: a device failure loses its share of data. That's fine for a cache, but durable stores need replication above the device layer.
Seen in memcached: f593a59bce 2017-09-26 doc/storage.txt: "with a RAID setup you risk any individual disk doing a GC pause stalling all writes". • 89bf7ab1cf 2018-07-16 extstore JBOD support.

### memcached-125 — Let pipelined requests suppress uninteresting replies
Lens: networking · Source: memcached
Principle: Offer quiet or no-reply variants that answer only when there is something to say (a hit, an error). Add short opaque tags for correlation and a no-op terminator to mark the end of a batch and force a flush.
Why it is faster: In a batch, "OK" and "miss" replies are wasted server writes, wire bytes and client parsing. Opaque tags are cheaper to echo than full keys. Fire-and-forget writes let clients stream without waiting.
Applies elsewhere: Web / backend: batch APIs returning only failures; Redis `CLIENT REPLY OFF`; HTTP/2 stream IDs as correlation tokens. • Embedded: IoT telemetry acknowledged only on error. • Data / databases: bulk-load protocols reporting only rejected rows.
Watch out: without a terminator, a client can't tell "no reply yet" from "no reply coming". Quiet errors must still be reported.
Seen in memcached: d9ece78091 2008-02-22 `noreply` on update commands: `out_string()` returns to the read state without queuing output. Shipped with `devtools/bench_noreply.pl`. • 1a37377 2008-12-19 Binary `GETQ`/`GETKQ` send nothing on a miss. The header carries an `opaque` field. • 1e1462867d 2019-04-30 Meta text protocol. 04613a4736 2019-11-14 adds `q`, `O(opaque)` and `k`. `mn` "forces immediate writeback flush" (d866c7429a 2020-03-12). • 46bfa8292f 2021-06-09 protocol.txt: O allows "easier pipelining of requests while saving bytes on the wire". 493bb69af5 2022-09-27 removes stray response spaces. fa745db8ff 2022-03-01 the proxy honors quiet mode end to end.

### memcached-126 — Move read-modify-write and fetch-plus-touch onto the server
Lens: networking · Source: memcached
Principle: Provide atomic server-side primitives (increment, append, compare-and-swap, get-and-touch) so clients don't do get-modify-set across multiple round trips.
Why it is faster: Client-side RMW costs two round trips, resends the whole value and races. A server primitive is one round trip and one lookup. Counters can even be rewritten in place when the new digits fit.
Applies elsewhere: Web / backend: Redis `INCR`/`GETEX`; S3 conditional writes with ETags. • Data / databases: `UPDATE x = x + 1` or `UPDATE … RETURNING` instead of select-then-update.
Watch out: in-place mutation is only safe when the caller holds the only reference. Every primitive you add is protocol surface you have to keep.
Seen in memcached: 57fb9e58ea 2003-05-28 incr/decr rewrite digits in place, space-padded, when they fit. • 7283d85b57 / 6091c6dea7 2007-10-03 append/prepend server-side. fc2e8effbb 2007-10-03 CAS for optimistic updates. e4a459655b 2007-11-15 CAS id as a plain global counter. • 7f4e0246e5 2017-10-05 `gat`/`gats` (get and touch) under a single locked lookup. • a71af5d1f3 2020-03-06 Meta arithmetic command.

### memcached-127 — Return only what the client asked for, and skip the payload if the client already has it
Lens: networking · Source: memcached
Principle: Make every returned field opt-in, and support conditional fetch: if the client's version matches, send only a header.
Why it is faster: A catch-all response formats and ships fields nobody reads. Opt-in fields mean metadata-only checks never move the value. Conditional fetch saves bandwidth and copying for unchanged data, like an HTTP 304.
Applies elsewhere: Web / backend: GraphQL selections, gRPC `FieldMask`, ETag/If-None-Match. • Data / databases: projection pushdown; HEAD-style existence checks. • Embedded: version-checked asset or config sync.
Watch out: too many optional fields make responses harder to parse. Keep the flag vocabulary small.
Seen in memcached: 1e1462867d 2019-04-30 A catch-all metaget was rejected because it would become "a slow command" in production. 04613a4736 2019-11-14 sends `VA` only with `v`, else `HD`. Each token is formatted only if requested. • 836d736dbb 2025-07-29 `mg`: `if (of.has_cas && ITEM_get_cas(it) == of.req_cas_id) of.value = false;`

### memcached-128 — Prevent thundering herds inside the cache protocol
Lens: networking · Source: memcached
Principle: On a hot miss, let exactly one client win the right to recompute (an atomically created placeholder plus a win token) while others get the placeholder or a stale value. Allow early recache before expiry, and stagger mass invalidations.
Why it is faster: When a hot key expires, every client misses at once and hammers the backend. Coordinating in the cache, where the key's lock already lives, cuts recomputation to one per key with no extra round trips or external locks.
Applies elsewhere: Web / backend: Go singleflight; nginx `proxy_cache_lock`; Varnish grace; jittered TTLs. • Data / databases: materialized-view refresh with a single refresher and stale reads.
Watch out: if the winner dies, the placeholder must expire, or the key stays blocked for everyone.
Seen in memcached: 54c9113bf8 2005-08-09 `flush_all <time>`: flush backends "not at exactly the same time (to reduce load peaks)". • 1e1462867d 2019-04-30 / 04613a4736 2019-11-14 `N` vivify-on-miss under the held lock with a win token, `R` win-if-TTL-below (guarded by `ITEM_TOKEN_SENT`), `I` mark stale instead of deleting. • 99dfb1ead6 2020-03-01 mset mode switches. a71af5d1f3 2020-03-06 meta arithmetic.

### memcached-129 — Tokenize once, in place, into a compact offset index
Lens: networking · Source: memcached
Principle: Scan a request line once and record each token's offset (and length) in a small fixed array. Don't copy, split or re-scan. Don't modify the buffer, so the original can be forwarded verbatim.
Why it is faster: Repeated `sscanf`/`strncmp` passes re-read the same bytes and copy keys. A single pass touches each byte once. A `uint16` offset array fits in a cache line and stays valid if the buffer moves. Hoisting the end-of-string test out of the per-character loop removes a branch.
Applies elsewhere: Web / backend: picohttpparser-style header slices; routing proxies that inspect a key and forward raw bytes. • Embedded: shell/AT-command parsers storing offsets. • Data / databases: CSV/log scanners returning offset arrays instead of substrings.
Watch out: fixed token arrays need a hard cap and a fallback path for long lines (multigets with many keys).
Seen in memcached: 217dcce072 2006-11-26 `tokenize_command()` fills `token_t{value,length}`. Before, `get` ran `sscanf` per key, copying each one. ce96a20854 2007-04-17 tightens it to a two-pointer scan. • d9cd01ede9 2009-11-02 Oversized lines: check only a bounded prefix for "get "/"gets " before swallowing. • 67b6bb1d04 2011-09-30 One `strlen` (SIMD in libc) bounds the loop. The per-char `\0` test "was accounting for 2% of memcached's CPU usage". • d22b66483b 2020-06-10 Proxy `uint16_t tokens[]`: "fast small (<= cacheline) index", parse in place "to forward an unmodified request". • 80fa3c7654 2023-01-09 context-free parse. 778cfd6e56 2025-02-18 one non-modifying tokenizer for requests and responses (4471 lines deleted). 7d8ef8fcd9 2026-07-09 fixed-length token-to-integer converters.

### memcached-130 — Use a fixed-size binary header with explicit lengths
Lens: networking · Source: memcached
Principle: Frame every message with a fixed header that carries opcode and lengths, so the receiver reads exactly the header and then exactly the body. No delimiter scanning, no number parsing, binary-safe keys.
Why it is faster: Text protocols must scan for delimiters and parse decimal numbers. A fixed header is read in one shot and decoded with loads. Building response headers directly in the output buffer saves a copy.
Applies elsewhere: Data / databases: Postgres/Kafka length-prefixed framing. • Embedded: CAN or UART frames with fixed headers. • Native / systems: FlatBuffers-style fixed layouts.
Watch out: binary protocols are harder to debug and evolve. memcached later moved its main investment to a compact text protocol (meta) that keeps most of the gains.
Seen in memcached: 7d296f3eb3 2007-12-14 spec draft. 6aafe58eec 2007-12-11 initial binary server. • 74b3a8057f 2008-03-18 "Avoid a memcpy by writing directly to wbuf". 1107f5d1f3 2008-03-19 reorders extras so the 64-bit CAS is 8-aligned. • 5da8dbab2a 2008-07-28 `protocol_binary.h`: 24-byte union of magic, opcode, keylen, extlen, bodylen, opaque, CAS.

### memcached-131 — Detect the protocol from the first byte, per connection
Lens: networking · Source: memcached
Principle: Serve multiple protocols on one port by peeking at the first byte, then fix the parser for the life of the connection.
Why it is faster: No negotiation round trip, no second listener and no per-request branch on protocol. The decision is made once.
Applies elsewhere: Web / backend: HTTP/2 prior-knowledge vs HTTP/1 detection; TLS-or-plaintext sniffing on one port. • Data / databases: Postgres SSLRequest handling.
Watch out: the magic bytes must never be a valid first byte of the other protocol.
Seen in memcached: cba691f466 2008-03-01 `negotiating_prot`: binary magic 0x80 vs ASCII, replaying the byte already read. 839eb26ad0 2008-03-01 removes the `-B` option.

### memcached-133 — Offer a connectionless transport for small fan-out reads
Lens: networking · Source: memcached
Principle: For huge client-by-server fan-out, UDP removes per-connection state and handshakes. Keep datagrams under the MTU, add a small reassembly header, and give each thread its own socket.
Why it is faster: Thousands of clients times hundreds of servers means millions of TCP connections, each with kernel buffers and a conn struct. UDP avoids that. Staying under about 1400 bytes avoids IP fragmentation. A descriptor per thread avoids thundering-herd wakeups on a shared socket.
Applies elsewhere: Web / backend: statsd metrics; DNS-style small RPC. • Native / systems: SO_REUSEPORT per-worker sockets; game state snapshots under the MTU.
Watch out: no congestion control or retransmit. Large replies fragment into many datagrams that all must arrive. UDP reflection makes public exposure dangerous.
Seen in memcached: c9607c6ddb 2006-09-03 `UDP_MAX_PAYLOAD_SIZE 1400`, 8-byte header (request id, sequence, count). `maximize_sndbuf()` binary-searches the largest `SO_SNDBUF`. The first TCP reply message is also capped at 1400 bytes. • 2439472aae 2008-02-27 UDP on by default. • c60ca35b2d 2011-03-01 One thread per UDP port when there are several. 70c1b5f61b 2014-01-14 `dup(sfd)` so "each worker thread has its own UDP file descriptor".

### memcached-135 — Shed connection overload explicitly instead of spinning or hanging
Lens: networking · Source: memcached
Principle: When out of descriptors, stop polling the listener until a connection closes. At a configured connection cap, accept, write an error and close, so clients fail fast and retry elsewhere.
Why it is faster: On EMFILE the listener stays readable and the loop spins on failing accepts at 100% CPU. Leaving clients in the backlog makes them wait for a timeout. Explicit shedding bounds both CPU waste and client latency.
Applies elsewhere: Web / backend: 503 load shedding in proxies; Node `server.maxConnections`. • Data / databases: admission control in connection poolers. • Embedded: mask an interrupt source while its queue is full.
Watch out: keep a few reserved descriptors so you can still accept-and-reject, log and serve admin connections.
Seen in memcached: 217dcce072 2006-11-26 `accept_new_conns(0)` on EMFILE (`update_event(listen_conn, 0)`, `listen(sfd, 0)`), re-enabled from `conn_close`. • 7d010a855e 2009-03-15 configurable listen backlog (`-b`) for bursts. • d1f9d992ce 2011-09-27 `maxconns_fast`: write "ERROR Too many open connections" and close, keeping `reserved_fds` headroom and counting `rejected_conns`.

### memcached-136 — Index connection state by file descriptor
Lens: networking · Source: memcached
Principle: Descriptors are small dense integers, so a pointer array indexed by fd gives O(1), lock-free lookup and reuse of connection structs.
Why it is faster: A shared free list of connection objects needs a mutex on every connect and disconnect. The fd table needs none, because the kernel already guarantees a descriptor is unique while open.
Applies elsewhere: Native / systems: fd-indexed handler tables in epoll servers. • Embedded: handle tables in an RTOS. • Web / backend: slot maps keyed by small dense ids.
Watch out: the table is sized by the fd limit. With very high limits, allocate lazily or in chunks.
Seen in memcached: 70c1b5f61b 2014-01-14 `conns = calloc(max_fds, sizeof(conn *))`, reusing `conns[sfd]`. Removes the locked freelist.

### memcached-137 — Pipeline many requests over a few persistent upstream connections
Lens: networking · Source: memcached
Principle: Keep a small number of long-lived connections per backend, pipeline outstanding requests on each, and match in-order responses to the head of a per-backend FIFO. Track a "next to write" cursor, and choose among the connections by least depth.
Why it is faster: A connection or round trip per request wastes handshakes and serializes latency. Pipelining fills the pipe. A write cursor avoids rescanning already-flushed entries, which would make deep queues quadratic. Several sockets per backend avoid head-of-line blocking on one TCP stream.
Applies elsewhere: Web / backend: Redis/memcached client libraries; gRPC channel pools. • Data / databases: pgbouncer/ProxySQL multiplexing. • Native / systems: TCP's `snd_nxt` vs `snd_una` split.
Watch out: one slow response delays everything behind it on that socket. Bound depth per connection and fail fast (see backend health).
Seen in memcached: d22b66483b 2020-06-10 Proxy: per-backend `io_head` FIFO and `depth`, with responses parsed off the backend "in the correct order". • d273bd2063 2022-05-31 Static per-backend read buffer parses several responses per read. • 146c6489fc 2022-12-05 `be->io_next` cursor fixes IO-thread CPU that "would bloat while looping past already flushed IO objects". • fd4e933e9f 2023-03-23 Up to 8 sockets per backend. `proxy_choose_beconn` takes the min depth and exits early on depth 0.

### memcached-138 — Give each worker its own upstream sockets instead of a central I/O thread
Lens: networking · Source: memcached
Principle: Let each worker thread own its backend connections so request handling and upstream I/O stay on one core.
Why it is faster: Handing every request to a shared I/O thread costs a lock, a wakeup and cache-line transfers each way, and that thread becomes the throughput ceiling. Per-worker sockets scale with the number of workers.
Applies elsewhere: Native / systems: thread-per-core designs (Seastar, Glommio). • Data / databases: per-worker DB connection pools instead of one global locked pool.
Watch out: upstream connection count multiplies by the number of workers. Large fleets may need the shared thread to cap connections.
Seen in memcached: 6442017c54 2023-01-13 Worker-run backend IO as a pool option. • b80e2a5385 2024-01-08 Default: "We no longer use the iothread by default… better scaling and performance by default."

### memcached-139 — Call a co-located service in-process instead of over loopback
Lens: networking · Source: memcached
Principle: When the router and the data engine share a process, execute requests directly against the engine instead of looping back through sockets.
Why it is faster: Loopback still pays socket syscalls, serialization, a second parse and context switches. A direct call costs a function call.
Applies elsewhere: Web / backend: in-memory gRPC transports; in-process sidecar modes. • Data / databases: embedded SQLite instead of a local DB server.
Watch out: you lose process isolation, and engine code must not depend on per-connection state.
Seen in memcached: fccf7b9efd 2023-01-09 / 0e87c6dd86 2023-01-11 Cache commands refactored to need only the worker thread, not a client connection. • f012c9cb35 2023-01-12 `mcp.internal(r)`. 33cf4ccd32 2024-12-26 exposed as a normal pool-like handle.

### memcached-140 — Fail fast on unhealthy or overloaded upstreams
Lens: networking · Source: memcached
Principle: Connect and validate upstreams in the background, mark failing ones bad so queued requests fail immediately, cap each backend's queue depth, and back off retries of flapping backends multiplicatively. Shard global limits into per-worker shares.
Why it is faster: Requests to a dead or slow backend otherwise hold resources until timeout and consume the global concurrency budget, so healthy traffic slows too. Immediate failure keeps latency bounded and lets clients route around trouble.
Applies elsewhere: Web / backend: circuit breakers with exponential half-open probes; bulkheads per upstream; Envoy outlier detection. • Data / databases: pool validation queries (HikariCP). • Native / systems: bounded per-device queues in drivers.
Watch out: over-eager bad-marking during a brief blip can cascade load onto the remaining backends.
Seen in memcached: d22b66483b 2020-06-10 `if (be->bad)` returns queued IO at once. 3614f3cf49 2022-02-03 `backend_failure_limit`. • cedaf55868 2022-09-23 The IO thread opens connections and validates them with `version\r\n`. Dead backends "flip into a bad fast-fail state quicker". • 6c80728209 2023-03-04 `active_req_limit`/`buffer_memory_limit` divided per worker thread. • 6a27618a0b 2023-04-17 connect timeout during retries. 312bab92c6 2023-07-12 flap detection with backoff ramp 1.5 up to 3600 s. • 4a75ac2415 2024-05-15 `backend_depth_limit`: flush with errors when depth exceeds the cap (`request_failed_depth`).

### memcached-141 — Return from a fan-out as soon as the answer is known
Lens: networking · Source: memcached
Principle: When querying several replicas, return on the first good result, and wait for more only on misses. Dispatch writes whose results nobody needs in the background and return immediately.
Why it is faster: Waiting for all replicas makes latency track the slowest one. Early return makes it track the fastest. Background writes take replication off the client's critical path.
Applies elsewhere: Web / backend: hedged requests; async cache-warming writes. • Data / databases: Cassandra ONE vs QUORUM reads.
Watch out: background writes silently fail. Count and expose those errors.
Seen in memcached: 774630ea7a 2022-02-16 Await modes `AWAIT_FIRST`, `AWAIT_OK`. • 1ba54b05a4 2022-09-23 `AWAIT_FASTGOOD`: "returns early on a hit, else waits for N non-error responses". • 1ba5df8410 2022-11-22 `AWAIT_BACKGROUND` returns once dispatched.

### memcached-142 — Distribute keys with cheap, stable hashing, and co-locate related keys
Lens: networking · Source: memcached
Principle: Use jump consistent hashing by default (no memory, minimal remapping). Where ring compatibility is needed, binary-search a sorted continuum. Allow a hash tag so related keys land on one server and multi-key requests go to a single backend.
Why it is faster: Modulus remaps almost everything when servers change, which causes a miss storm. Jump hash remaps the minimum with no table. Hash tags turn a scattered multiget into one batched upstream request.
Applies elsewhere: Web / backend: Redis Cluster `{tag}` keys; client-side sharding. • Data / databases: distribution columns in Citus/Vitess.
Watch out: jump hash only supports appending or removing at the end of the server list, and hash tags can create hot shards.
Seen in memcached: 4b78197360 2022-02-05 "default hash dist is now jump". Key hash filters hash only the portion inside tags or before a stop char. • 2a903f04d1 2022-02-24 ketama continuum `qsort`ed and binary-searched. Weights unsupported because they "would shift unrelated servers".

### memcached-143 — Cut TLS cost per connection and per response
Lens: networking · Source: memcached
Principle: Let the TLS library drop its buffers on idle connections, coalesce each response into one TLS record, reuse sessions, and offload encryption to the kernel where supported.
Why it is faster: Per-connection TLS buffers cost tens of KB each, which is gigabytes across many idle clients. One `SSL_write` per iovec fragment means one record (header, MAC, syscall) per fragment. Session resumption skips full handshakes. kTLS lets sendfile-style paths and NIC offload work.
Applies elsewhere: Web / backend: Node TLS servers with many idle sockets; nginx `ssl_buffer_size` and session tickets. • Native / systems: kTLS plus sendfile for content serving.
Watch out: releasing buffers costs a re-allocation when traffic resumes. Coalescing adds a memcpy, which only pays off when it saves records.
Seen in memcached: ee1cfe3bf9 2019-04-15 `ssl_sendmsg()` memcpys the iovecs into a per-thread `ssl_wbuf`, then makes one `SSL_write`. • 4e79f166fc 2020-02-19 `ssl_session_cache`. • 69e732ab6f 2022-04-06 `SSL_OP_ENABLE_KTLS`. • e31abc34f9 2022-08-26 `SSL_MODE_RELEASE_BUFFERS`: with many clients the buffers "can quickly add up to gigabytes of memory".

### memcached-144 — Rate-limit with a lazily refilled, tick-quantized token bucket
Lens: networking · Source: memcached
Principle: Refill on check rather than on a timer: add tokens for the whole ticks elapsed, advance the timestamp by whole ticks only, and clamp.
Why it is faster: O(1) per check with no background thread or timer per limiter. Advancing by whole ticks loses no time to rounding.
Applies elsewhere: Web / backend: API gateway and client-side throttles. • Embedded: packet policers in firmware.
Watch out: a shared bucket across threads needs synchronization. Per-worker buckets avoid it at the cost of fairness.
Seen in memcached: 6f0240f424 2023-07-11 `mcp.ratelim_tbf`: `toadd = delta / tick_rate; last_update += toadd * tick_rate`.

### memcached-145 — Log by writing compact binary records into per-thread buffers drained by a background formatter
Lens: observability · Source: memcached
Principle: Workers append small fixed-shape binary records to their own ring buffer. One background thread formats text and fans it out. When a buffer is full, drop and count rather than block.
Why it is faster: Formatting, locking and write syscalls leave the request path. An uncontended per-thread buffer costs a memcpy. Dropping under overload keeps logging from ever adding latency. The drain thread should sleep on a condition variable when nobody is listening.
Applies elsewhere: Native / systems: per-core trace buffers (LTTng, ftrace). • Web / backend: async loggers with ring buffers (Log4j2 async). • Embedded: binary logging decoded off-device (defmt-style).
Watch out: lossy logging is unsuitable for audit trails. Expose the drop count.
Seen in memcached: 916fff3609 2015-11-18 "Logs are written to per-thread buffers. A new background thread aggregates the logs". `bipbuf_request`, binary `logentry`, dropped records counted. • 60e98600ca 2016-06-28 Global log sequence via 64-bit atomics where available, mutex fallback otherwise. • d63db64889 2021-08-09 The logger thread "would wake up once per second looking for watchers… Now sleeps on a condition."

### memcached-147 — Fan out by rendering once and copying into a buffer per consumer
Lens: observability · Source: memcached
Principle: For several filtered consumers, render each record once into scratch space and memcpy it into a private buffer for each matching consumer. Don't track shared iovecs and partial writes across readers.
Why it is faster: The extra copy is cheaper than loops managing per-reader offsets into shared buffers. The drain side becomes a trivial write. A slow consumer only loses its own data (reported as skipped) and never blocks others.
Applies elsewhere: Web / backend: WebSocket broadcast with per-client queues; log shippers with per-sink buffers. • Native / systems: pub/sub with a queue per subscriber.
Watch out: with many consumers of large records, copies dominate, and shared immutable buffers with refcounts win.
Seen in memcached: d5f7f89ad5 2016-06-11 tried shared iovecs. a4f8b982fa 2016-06-12 "use one bipbuffer per watcher. remove iovec code…" because it removes "all of the loops and management of iovecs". • 1b03a9c6a4 2016-06-12 A full watcher prints `[skipped: n]` rather than blocking.

### memcached-148 — Maintain statistics incrementally instead of scanning on demand
Lens: observability · Source: memcached
Principle: Update a histogram on insert and remove while tracking is enabled, so queries cost O(buckets) rather than a full scan under locks.
Why it is faster: A stats command that walks millions of items blocks the server for seconds. Incremental updates spread a tiny cost over operations, and only while someone has enabled them.
Applies elsewhere: Data / databases: catalog row counts and size histograms instead of `COUNT(*)`. • Web / backend: Prometheus histograms instead of scanning logs.
Watch out: enabling tracking mid-flight sees only new data. Correct for the pre-existing population, or document the gap.
Seen in memcached: ae6f4267fe 2016-06-21 "online hang-free 'stats sizes'": `stats_sizes_hist[ntotal/32]` updated in link/unlink, toggled at runtime. • 8d82383f0f 2016-06-22 Uses CAS to ignore items that existed before tracking began. • d7fb022db0 2016-06-21 Paired with explicit slab sizes, so the measured histogram directly drives tuning.

### memcached-149 — Make bulk introspection non-blocking, bounded and lean
Lens: observability · Source: memcached · Also: jemalloc
Principle: Dumps of internal state must never hold fine-grained data locks across network I/O. Buffer while locked and write after unlocking, in batches. Bound the dump to the population at start. Use trylock and return "try again" when a long maintenance operation holds the lock. Emit the leanest format the consumer can use.
Why it is faster: A slow dump client must not stall request threads behind item locks. Bounding the work guarantees termination under constant writes. Lean formatting cuts dump CPU.
Applies elsewhere: Data / databases: backup and vacuum threads copying under latch and writing afterwards. • Web / backend: export endpoints streaming newline-delimited IDs instead of full JSON. • Native / systems: kernel debug dumpers.
Watch out: a dump that runs unlocked between batches is not a consistent snapshot.
Seen in memcached: 37bfa412de 2018-02-12 metadump limited to the items in the LRU when it started. • a22be2bdbf 2023-02-18 Growable buffer, writes moved outside the item lock, `MIN_ITEMS_PER_WRITE 16`: "Item locks must NOT be held for long periods of time". • d2e0d66899 2023-02-18 `lru_crawler mgdump` emits ready-to-send `mg <key>` lines: "30-40% less CPU time". • 788f592f4a 2024-05-24 Iterator creation uses trylock on `maintenance_lock` and returns "ERROR locked try again later". • c464311059 2025-03-24 metadump formatting with literal memcpy plus itoa: "Increases dump speed with no crawler sleep by 30%". • Also seen in jemalloc (jemalloc-159): 4f37ef693e 2014-01-16 Heap-profile dumping became two-pass and calls `prof_leave()` before the second pass, "This avoids write(2) system calls while holding critical prof resources". • 1a71533511 2019-05-22 `background_thread_stats_read` uses trylock and skips a thread whose lock is held, to avoid "latency spikes". • a835d9cf85 2020-04-13 The last-N profiling dump moves the list out with `ql_move`, unlocks, emits, relocks and splices it back.

### memcached-151 — Shrink limits in debug builds so rare performance paths get exercised
Lens: observability · Source: memcached
Principle: Lower batching caps, pool-trim thresholds and wait times in debug builds so tests and benchmarks hit the overflow and recycle paths constantly.
Why it is faster: Optimizations add edge paths (IOV caps, pool trimming) that production-sized limits almost never trigger in tests. Making them common surfaces their bugs before they become latency or correctness incidents.
Applies elsewhere: Web / backend: tiny pool and buffer limits in CI. • Native / systems: fuzz builds with small buffers; JITs forcing slow paths with flags.
Watch out: debug-only constants must not leak into release builds or into benchmark numbers.
Seen in memcached: f3700374f4 2024-05-29 `#if MEMCACHED_DEBUG #define BE_IOV_MAX 128` plus an assert. The bug triggered easily at ≤128 but almost never at ≥256. Debug `FGEN_FREE_PRESSURE_MAX 100`, `FGEN_FREE_WAIT 0`. • 38ac376875 2024-05-02 The debug binary recycles slots "10x more often", which exposed a GC lag.
