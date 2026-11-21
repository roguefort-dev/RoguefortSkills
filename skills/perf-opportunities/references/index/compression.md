# Lens: compression (11 techniques)

Lightweight encodings, per-block codec choice, operating on encoded data.

Format: `id | title | principle (first sentence)`. The id prefix names the cards file: `../cards/<prefix>.md`.

- duckdb-067 | Keep data encoded through execution and work once per distinct value | Let in-memory batches carry their encoding (constant, sequence, dictionary plus indices) from storage through operators to writers.
- duckdb-068 | Let codecs compete per block: analyze cheaply, pick the smallest, fall back to raw | Don't hard-code one encoding per type. For each block, let every candidate codec estimate (or actually produce) its output size, keep the winner, and store the block uncompressed when nothing beats raw.
- duckdb-069 | Narrow values before materializing them (frame of reference on intermediates) | Before data enters anything that buffers, hashes, sorts or spills it, use known value ranges to subtract the minimum and cast to the smallest unsigned type that fits.
- duckdb-070 | Bitpack integers with frame-of-reference and delta modes | Per small group (such as 32 values), subtract a reference (the min, or the previous value for deltas) and store each value in the minimum number of bits.
- duckdb-071 | Dictionary-encode repetitive strings per block, with a purpose-built builder | Store each distinct value once per block and replace occurrences with small (bitpacked) codes.
- duckdb-072 | Encode runs, constants and sparse bitmaps compactly | Store repetitive data as (value, run length) pairs.
- duckdb-073 | Push filters and row selections into the decoder | Give each codec two operations: "which rows match this predicate" (evaluated on encoded data where possible) and "decode only these rows".
- duckdb-074 | Write each decoder for its common case | When scans are decode-bound, rewrite each encoding's inner loop for the dominant shape: fewer per-value branches, bulk copies for long runs, structure-of-arrays metadata unpacked up front, and table-driven…
- duckdb-075 | Use heavyweight compression only where it pays: cold or large values, and adaptively for spill | General-purpose codecs (gzip, zstd) get great ratios but cost CPU on every access.
- duckdb-076 | Compress strings with a per-block symbol table that keeps each string separately decodable (FSST) | Replace frequent 1-8 byte substrings with 1-byte codes from a small per-segment symbol table.
- duckdb-077 | Compress floats by exploiting their structure: XOR with similar neighbours, or recover hidden decimals (Chimp/Patas, ALP) | Consecutive floats share sign, exponent and high mantissa bits, so XOR each value with a well-matching recent value and store only the meaningful bits.
