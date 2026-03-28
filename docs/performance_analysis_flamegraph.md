# Performance Analysis — ruzstd CLI Decompression

**Date:** 2026-03-14
**Tool:** `cargo flamegraph` with `CARGO_PROFILE_RELEASE_DEBUG=true`
**Workload:** `decompress ~/Documents/20240201_TOPS1.6.pcapng.1gb.zst /dev/null`
**Throughput observed:** ~66–84 MiB/s

---

## Flamegraph Hotspot Summary

| Function | % Samples | File |
|---|---|---|
| `execute_sequences` | 19.05% | `decoding/sequence_execution.rs` |
| `FSEDecoder::update_state` | 12.83% | `fse/fse_decoder.rs:43` |
| `RingBuffer::extend_from_within_unchecked` | ~17% (two frames) | `decoding/ringbuffer.rs:292` |
| `DecodeBuffer::repeat` | ~14% (two frames) | `decoding/decode_buffer.rs:67` |
| `DecodeBuffer::push` | 8.86% | `decoding/decode_buffer.rs:62` |
| `RingBuffer::extend` | 8.61% | `decoding/ringbuffer.rs:148` |
| `BitReaderReversed::get_bits` | 6.47% | `bit_io/bit_reader_reverse.rs` |
| `decode_sequences_without_rle` | ~10% (two frames) | `decoding/sequence_section_decoder.rs` |
| `copy_bytes_overshooting` | ~7% (two frames) | `decoding/ringbuffer.rs:556` |
| `BitReaderReversed::refill` | 3.55% | `bit_io/bit_reader_reverse.rs` |
| `RingBuffer::len` | ~4% (two frames) | `decoding/ringbuffer.rs:41` |
| `RingBuffer::reserve_amortized` | 2.90% | `decoding/ringbuffer.rs:72` |
| `RingBuffer::data_slice_lengths` | 2.76% | `decoding/ringbuffer.rs:196` |
| `core::ptr::write_unaligned` | 2.65% | (from `copy_bytes_overshooting`) |
| `twox_hash` checksum | 2.44% | (external) |
| `do_offset_history` | 2.39% | `decoding/sequence_execution.rs:59` |
| `FSETable::build_decoding_table` | 1.89% | `fse/fse_decoder.rs:141` |

---

## Issue 1: `RingBuffer::len()` is Computed, Not Stored (~6% overhead)

**Files:** `decoding/ringbuffer.rs:41-44`, `decoding/ringbuffer.rs:196-209`

```rust
// Every call to len() branches on head vs tail and does arithmetic
pub fn len(&self) -> usize {
    let (x, y) = self.data_slice_lengths();  // branch here
    x + y
}
```

`len()` appears in the flamegraph both directly (3.01%) and via `data_slice_lengths` (2.76%), totaling ~6%.
It is called on **every** `repeat()` invocation (`decode_buffer.rs:68,71`) and every `push()`.
The fix is straightforward: add a `len: usize` field to `RingBuffer` and maintain it incrementally in `extend`, `drop_first_n`, `clear`, and `reserve_amortized`. The `len()`, `free()`, and `reserve()` methods then become branchless.

Similarly, `free()` calls `free_slice_lengths()` which has the same branching pattern, contributing to the 8.86% `DecodeBuffer::push` cost.

---

## Issue 2: `repeat_in_chunks` Uses Linear Loop Instead of Exponential Doubling (~3–4% overhead)

**File:** `decoding/decode_buffer.rs:101-129`

```rust
// TODO this can be optimized further I think.
while copied_counter_left > 0 {
    let chunksize = usize::min(offset, copied_counter_left);
    // copies offset bytes, then offset bytes, then offset bytes...
    unsafe { self.buffer.extend_from_within_unchecked(start_idx, chunksize) };
    copied_counter_left -= chunksize;
    start_idx += chunksize;
}
```

For overlapping matches where `offset < match_length`, this loops `ceil(match_length / offset)` times.
For `offset=1, match_length=65536` (common in run-length encoded data), this is 65536 loop iterations each copying 1 byte — meaning 65536 calls through the ringbuffer copy path.

**The correct approach is exponential doubling:** after copying `offset` bytes, the available contiguous run is now `2*offset` bytes. Copy that, then `4*offset`, etc. This reduces the loop from O(match_length/offset) to O(log(match_length/offset)).

Example fix pattern:
```rust
let mut available = offset;
let mut remaining = match_length;
while remaining > 0 {
    let chunk = available.min(remaining);
    // copy chunk bytes from start_idx
    remaining -= chunk;
    start_idx += chunk;
    available = available.saturating_mul(2).min(remaining + chunk);
}
```

---

## Issue 3: `reserve()` Called Per-Operation Instead of Per-Block (~2–3% overhead)

**Files:** `decoding/decode_buffer.rs:75`, `decoding/ringbuffer.rs:61-68`

Every call to `DecodeBuffer::push` and `DecodeBuffer::repeat` calls `RingBuffer::reserve()`, which computes `free()` (via the branching `free_slice_lengths`), then conditionally calls `reserve_amortized`. Even when no reallocation is needed, the overhead of computing `free()` is paid every time.

`reserve_amortized` appears at 2.90% — it is marked `#[cold] #[inline(never)]` but still registers because the ringbuffer grows during decoding.

The fix: at the start of each block, compute the maximum decompressed output size (available from the block header `content_size` field) and call `buffer.reserve(block_content_size)` once. This pre-allocates all needed space so the per-operation `reserve()` calls hit the fast-path immediately (free >= amount, return early).

---

## Issue 4: `copy_bytes_overshooting` Manual Loop is Slower than `memcpy` for Medium Copies (~4–5% overhead)

**File:** `decoding/ringbuffer.rs:555-596`

The function uses `u128` (16-byte) chunks for SSE2/NEON targets. For small copies (≤ 16 bytes), this is excellent. For medium copies (17–256 bytes), it loops with manual `write_unaligned` / `read_unaligned` calls, which prevents AVX-512 autovectorization.

The flamegraph confirms this: `write_unaligned` appears at 2.65%, while the `__memcpy_avx512_unaligned_erms` fallback path (via `copy_nonoverlapping`) appears at 2.49% for the **same workload** — the fallback is achieving similar throughput despite fewer calls.

For the multi-iteration loop path (when `min_buffer_size >= copy_multiple` but `copy_at_least > COPY_AT_ONCE_SIZE`), consider lowering the threshold where `copy_nonoverlapping` is used, e.g.:

```rust
// If more than ~32 bytes, trust the compiler/memcpy
if copy_at_least > 2 * COPY_AT_ONCE_SIZE {
    dst.0.copy_from_nonoverlapping(src.0, copy_at_least);
    return;
}
```

---

## Issue 5: `FSEDecoder::update_state` — Dependent Memory Access Chain (12.83%)

**File:** `fse/fse_decoder.rs:43-51`

```rust
pub fn update_state(&mut self, bits: &mut BitReaderReversed<'_>) {
    let num_bits = self.state.num_bits;          // load 1
    let add = bits.get_bits(num_bits);            // variable bit read
    let base_line = self.state.base_line;         // load 2
    let new_state = base_line + add as u32;       // compute
    self.state = self.table.decode[new_state as usize];  // table lookup (load 3)
}
```

This creates a serial dependency chain: each state update depends on `num_bits` from the previous state, which was computed from the table lookup before that. CPUs cannot pipeline these.

This is 12.83% because `update_state` is called **3 times per sequence** (for LL, ML, and OF decoders) and each sequence is processed in the inner loop. There is no easy algorithmic fix here — this is fundamental to FSE. However:

- The three decoders (LL, ML, OF) are **independent of each other**. Interleaving their updates may allow out-of-order execution to pipeline across decoders:
  ```
  instead of: LL.update → ML.update → OF.update (serial per-decoder)
  try:        LL.peek_bits → ML.peek_bits → OF.peek_bits → LL.consume → ML.consume → OF.consume
  ```
- Ensure `Entry` is packed to fit in a cache line. Currently `Entry` has `base_line: u32 + num_bits: u8 + symbol: u8` — this likely pads to 8 bytes. Three entries per table lookup = 24 bytes, fine for cache.

---

## Issue 6: `FSETable::build_decoding_table` at 1.89% (Should Be Initialization-Only)

**File:** `fse/fse_decoder.rs:141`

This function appears in the hot profile at 1.89%, suggesting FSE tables are rebuilt frequently. This happens when compressed blocks use `Compressed_Block` mode with a new FSE header (not `Repeat_Mode`). For the PCAP test file which likely contains many blocks, each new FSE distribution triggers a full table rebuild.

There is no fast fix here — this is spec-mandated — but verifying that `Repeat_Mode` detection is working correctly in `sequence_section_decoder.rs` could reduce unnecessary rebuilds.

---

## Issue 7: Checksum Overhead (2.44%)

**Relevant to:** `decoding/decode_buffer.rs` (`drain_to`)

The `twox_hash::xxhash64` checksum is computed during draining (`drain_to` at line 277-290). This is a streaming hash over all output bytes. At 2.44% overhead it is already fairly efficient (xxHash64 is fast), but it can be disabled by compiling without the `hash` feature flag. Only relevant if checksum validation is not needed.

---

## Prioritized Recommendations

| Priority | Change | Expected Gain | Difficulty |
|---|---|---|---|
| 1 | Add `len` field to `RingBuffer` | ~6% | Low |
| 2 | Exponential doubling in `repeat_in_chunks` | ~3–4% for repetitive data | Low |
| 3 | Pre-reserve buffer per block | ~2–3% | Medium |
| 4 | Tune `copy_bytes_overshooting` fallback threshold | ~1–2% | Low |
| 5 | Interleave FSE decoder updates | ~2–3% | High |
| 6 | Verify FSE `Repeat_Mode` is used when possible | ~1–2% | Medium |

Implementing items 1–4 could reasonably improve throughput by **10–15%** with relatively low implementation risk.
