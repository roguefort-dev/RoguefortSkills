# Lens: batch-processing (8 techniques)

Vectorised execution: batch-at-a-time loops, per-batch checks, typed arrays over boxed values.

Format: `id | title | principle (first sentence)`. The id prefix names the cards file: `../cards/<prefix>.md`.

- duckdb-108 | Process data a cache-sized batch at a time, and keep batches full | Run every operator over a fixed-size batch (around 1-2K values per column) instead of one item at a time.
- duckdb-109 | Specialize the inner loop per case at compile time | Turn per-element runtime questions (is the input constant?
- duckdb-110 | Check once per batch, then run a check-free loop (no-NULL and dense fast paths) | Decide per batch, or per plan from statistics, whether the slow cases can occur at all: any NULLs, a selection, an incomplete coverage.
- duckdb-111 | Let batches carry cheap encodings, read through one zero-copy accessor | Allow a batch to be flat, constant (one value), a sequence (start plus step) or a dictionary (indices into another vector).
- duckdb-112 | Compute once for constant inputs | If every input to a deterministic operation is constant across the batch, evaluate it once and emit a constant result.
- duckdb-113 | Keep hot paths on raw typed arrays, not boxed dynamic values | A generic tagged value object (type tag + union + owned string) per element costs allocation, type dispatch and conversion.
- duckdb-114 | Split mixed per-row work into tight single-purpose passes | Instead of one loop that hashes, branches on state kind, checks NULLs and updates structures per row, first classify rows into selection vectors or compute an intermediate array.
- duckdb-115 | Use SIMD and SWAR (8 bytes per register) for byte scanning, and lean on vectorized libraries | Classify or search 8 (SWAR) or 16-64 (SIMD) bytes per instruction with bit tricks or vector compares, and drop to per-byte logic only for flagged positions.
