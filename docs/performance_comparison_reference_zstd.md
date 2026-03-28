# Performance Comparison: ruzstd vs Reference Zstd

**Date:** 2026-03-14
**Reference:** `docs/zstd/` (facebook/zstd C implementation)
**Basis:** Flamegraph hotspot analysis of `ruzstd-cli decompress`

This document compares each flamegraph hotspot against the equivalent code in the reference zstd C implementation to identify algorithmic and structural gaps.

---

## 1. Sequence Execution & Match Copying

### ruzstd
- `decoding/sequence_execution.rs:5` — `execute_sequences()`
- `decoding/decode_buffer.rs:67` — `repeat()` / `repeat_in_chunks()`

### Reference zstd
- `lib/decompress/zstd_decompress_block.c:1008` — `ZSTD_execSequence()`
- `lib/decompress/zstd_decompress_block.c:914` — `ZSTD_execSequenceEnd()` (slow path)
- `lib/common/zstd_internal.h:218` — `ZSTD_wildcopy()`
- `lib/decompress/zstd_decompress_block.c:806` — `ZSTD_overlapCopy8()`

### Key Difference: Two-Path Execution (Fast / Slow)

Reference zstd aggressively separates the common-case fast path from edge cases:

```c
// ZSTD_execSequence: fast path preconditions (line ~1038)
// Asserts that: oend_w > op + WILDCOPY_OVERLENGTH
//               match + WILDCOPY_OVERLENGTH <= iend (match is fully in src)
// If these hold, unsafe wildcopy is used without per-byte bounds checks.
// Edge cases (last few sequences near end of block) fall to ZSTD_execSequenceEnd.
```

ruzstd has a single unified loop that bounds-checks on every sequence:
```rust
// sequence_execution.rs:15-19 - checked on every sequence
if high > scratch.literals_buffer.len() {
    return Err(ExecuteSequencesError::NotEnoughBytesForSequence { ... });
}
```

**Impact:** For a typical compressed block with hundreds of sequences, only the last ~2 sequences can actually overflow. The fast path covers ~98% of sequences with zero bounds checks.

### Key Difference: Overlapping Copy (`offset < match_length`)

Reference uses lookup tables to handle short-offset overlapping matches in fixed 8-byte strides:

```c
// zstd_decompress_block.c:806-827 — ZSTD_overlapCopy8()
static const U32 dec32table[] = { 0, 1, 2, 1, 4, 4, 4, 4 };  // line 811
static const int dec64table[] = { 8, 8, 8, 7, 8, 9, 10, 11 }; // line 812

// Copies 8 bytes from match → op, then advances pointers using the lookup
// tables so the next COPY8 can proceed. For offset=1, this replicates the
// single byte into 8 using the table offsets, not a loop.
```

ruzstd copies in `min(offset, remaining)` chunks linearly:
```rust
// decode_buffer.rs:107-128
while copied_counter_left > 0 {
    let chunksize = usize::min(offset, copied_counter_left);
    self.buffer.extend_from_within_unchecked(start_idx, chunksize);
    // loops ceil(match_length / offset) times
}
```

For `offset=1, match_length=10000`, reference runs the 8-byte loop 1250 times. ruzstd calls `extend_from_within_unchecked` 10000 times.

### Key Difference: `ZSTD_wildcopy` 16-byte Unrolled Loop

For non-overlapping matches, reference uses 16-byte unrolled double-`COPY8` iterations:
```c
// zstd_internal.h:243-246
do {
    COPY8(op, ip);   // 8-byte copy, advance both pointers
    COPY8(op, ip);   // 8-byte copy, advance both pointers
} while (op < oend);
```

ruzstd delegates to `copy_bytes_overshooting` which uses `u128` (16-byte) but wraps it in a `write_unaligned` loop that the compiler may not vectorize as well as the explicit unroll.

---

## 2. FSE State Updates — Dual-State Pipelining

### ruzstd
- `fse/fse_decoder.rs:43` — `FSEDecoder::update_state()`
- Called 3× per sequence (LL decoder, ML decoder, OF decoder)

### Reference zstd
- `lib/common/fse.h:532` — `FSE_updateState()` (identical structure)
- `lib/decompress/fse_decompress.c:173` — `FSE_decompress_usingDTable_generic()` ← **key difference**

### Key Difference: Dual Interleaved State Machines

Reference zstd decompresses generic FSE data using **two independent state machines** running in lock-step:

```c
// fse_decompress.c:184-215 (paraphrased)
FSE_initDState(&state1, &bitD, dt);
FSE_initDState(&state2, &bitD, dt);

for ( ; (BIT_reloadDStream(&bitD) == BIT_DStream_unfinished) & (ptr > ptr_end+1) ; ptr -= 2) {
    ptr[-2] = FSE_decodeSymbol(&state1, &bitD);   // decode from state1
    // reload here allows state2 decode to proceed without stall
    if (sizeof(bitD.bitContainer)*8 < FSE_MAX_TABLELOG*2 + 7)
        BIT_reloadDStream(&bitD);
    ptr[-1] = FSE_decodeSymbol(&state2, &bitD);   // decode from state2, independent of state1
    if (sizeof(bitD.bitContainer)*8 < FSE_MAX_TABLELOG*4 + 7)
        BIT_reloadDStream(&bitD);
}
```

The two states are completely independent — after `state1` issues its table lookup, `state2` can begin its lookup before `state1`'s result is needed. This hides the memory access latency of the table lookup via CPU out-of-order execution.

In the context of zstd sequence decoding, the reference further extends this: within each sequence, the three FSE decoders (LL, ML, OF) are **already independent**. The reference updates all three states after reading all three values, but the key is the order and interleaving within the decode loop that allows the CPU to overlap the dependent loads.

ruzstd calls `update_state` on each decoder sequentially with no overlap opportunities explicitly given to the CPU. The compiler may not reorder across the calls since they share the `BitReaderReversed` mutable borrow.

**Impact:** This is the single largest optimization gap. FSE state update is 12.83% of execution in the flamegraph. Reference's pipelining is specifically designed to hide the ~5ns table lookup latency.

---

## 3. Bit Reader — Refill Strategy

### ruzstd
- `bit_io/bit_reader_reverse.rs:43-87` — `refill()`

### Reference zstd
- `lib/common/bitstream.h:412` — `BIT_reloadDStream()`
- `lib/common/bitstream.h:400` — `BIT_reloadDStreamFast()` (fast variant, no bounds check)
- `lib/common/bitstream.h:384` — `BIT_reloadDStream_internal()`

### Key Difference: Three-Tier Reload with Status Codes

Reference provides three reload functions for different contexts:

```c
// BIT_reloadDStreamFast: used in the inner loop, no bounds check (line 400)
// Assumes caller has validated stream position. Used for the 98% common case.

// BIT_reloadDStream: safe reload with status (line 412)
// Returns: BIT_DStream_unfinished, BIT_DStream_endOfBuffer,
//          BIT_DStream_completed, BIT_DStream_overflow
// Status lets the caller (FSE_decompress_usingDTable_generic) exit the fast
// loop as soon as the stream boundary is reached.

// BIT_reloadDStream_internal: core reload logic (line 384)
// Called by both of the above.
```

The `BIT_reloadDStreamFast` usage in the inner loop is critical: it avoids the bounds-checking arithmetic of the safe variant for every reload in the common case, keeping the hot path lean.

ruzstd's `refill()` is a single function that handles all cases:
```rust
// bit_reader_reverse.rs:43-87
// Handles: normal refill, partial bytes at end, stream exhaustion
// All in one function, branching on `self.index` each time
// Marked #[cold] correctly, but the conditional logic in the hot
// "pre-refill" check (line 93) is still paid on every get_bits call.
```

The `get_bits` pre-check at line 93 (`if self.bits_consumed + n > 64`) cannot be avoided, but the fast reload variant in the reference skips the boundary arithmetic inside refill, which is the more expensive part.

---

## 4. Output Window / Buffer Management

### ruzstd
- `decoding/ringbuffer.rs` — custom ringbuffer with head/tail tracking
- `len()` computes from `data_slice_lengths()` every call

### Reference zstd
- `lib/decompress/zstd_decompress_block.c` — uses raw pointer arithmetic

### Key Difference: Linear Buffer vs. Ringbuffer

Reference zstd does **not** use a ringbuffer. It manages output as a simple linear array with raw pointers:

```c
// zstd_decompress_block.c (sequence execution setup, ~line 1013)
BYTE* op = ostart;          // current output position
const BYTE* const oend = ostart + dstCapacity;
const BYTE* const oend_w = oend - WILDCOPY_OVERLENGTH;

// Length of data in window: simply op - prefixStart
// No branch, no arithmetic on wrapped indices.
```

The window is maintained via a `prefixStart` pointer. When the output fills a buffer, the next frame starts with the tail of the previous frame copied to the start of the new buffer (`ZSTD_setBasePrefixes`).

For the streaming use case ruzstd targets, a ringbuffer is architecturally necessary. But the cost is significant:
- `len()` requires a conditional and two subtraction/addition operations
- `extend_from_within_unchecked` needs 2–4 branches to handle wrap-around cases
- All pointer arithmetic goes through `(tail + offset) % cap` modulo operations

Reference achieves the equivalent with a single pointer subtraction.

**Potential mitigation for ruzstd:** Track `len` as a separate field (eliminates the per-call computation). Maintain `cap` as a power-of-two and replace `% cap` with `& (cap - 1)` (eliminates division). These do not change the architecture but reduce the constant factor.

---

## 5. `repeat_in_chunks` vs. `ZSTD_overlapCopy8` — Detailed Comparison

This is the most algorithmically distinct difference.

### Reference: Two-Stage Copy with Lookup Tables

For the case where `offset < 8` (the most expensive overlapping case):

```c
// ZSTD_overlapCopy8 (line 806)
static const U32 dec32table[] = { 0, 1, 2, 1, 4, 4, 4, 4 }; // offset: 0 1 2 3 4 5 6 7
static const int dec64table[] = { 8, 8, 8, 7, 8, 9,10,11 }; // offset: 0 1 2 3 4 5 6 7

void ZSTD_overlapCopy8(BYTE** op, BYTE const** ip, size_t offset) {
    assert(offset < 8);
    MEM_write64(*op, MEM_read64(*ip));    // Copy 8 bytes (may overshoot, that's fine)
    *ip += dec32table[offset];            // Advance source by table-driven amount
    *op += 8;
    *ip -= dec64table[offset];            // Potentially reverse source for next iteration
}
```

After this setup, the match continues with `ZSTD_wildcopy` which handles the rest in 16-byte strides. The total loop count is `ceil(match_length / 8)` regardless of offset.

### ruzstd: Chunk Loop

```rust
fn repeat_in_chunks(&mut self, offset: usize, match_length: usize, start_idx: usize) {
    while copied_counter_left > 0 {
        let chunksize = usize::min(offset, copied_counter_left);
        self.buffer.extend_from_within_unchecked(start_idx, chunksize);
        copied_counter_left -= chunksize;
        start_idx += chunksize;
    }
}
```

| Scenario | ruzstd loop iterations | Reference loop iterations |
|---|---|---|
| `offset=1, match_length=1024` | 1024 | 128 (8×) |
| `offset=3, match_length=1024` | 342 | 128 (8×) |
| `offset=8, match_length=1024` | 128 | 128 (equal) |
| `offset=16, match_length=1024` | 64 | 64 (equal) |

The reference approach is always ≤ ruzstd for the same workload.

**Recommended fix for ruzstd:** After the initial `min(offset, remaining)` copy, double the available run length on each iteration (exponential growth):

```
Iteration 1: copy offset bytes   (available = offset → 2*offset)
Iteration 2: copy 2*offset bytes (available = 2*offset → 4*offset)
Iteration 3: copy 4*offset bytes (available = 4*offset → 8*offset)
...
```

This gives `O(log(match_length/offset))` calls to `extend_from_within_unchecked` and closely matches the reference's fixed-stride approach.

---

## Summary: Gap Analysis

| Hotspot | ruzstd | Reference | Status | Estimated Impact |
|---|---|---|---|---|
| Sequence fast/slow path split | Unified loop with bounds checks every sequence | Separate fast path (98% sequences) + slow path (last 2) | ❌ Missing | 5–10% |
| Overlapping copy loop count | O(match_length / offset) iterations | O(match_length / 8) iterations always | ❌ Missing | 2–5% on short-offset data |
| `ZSTD_wildcopy` 16-byte unroll | `u128` write_unaligned loop | Explicit `COPY8 + COPY8` unrolled double loop | ⚠️ Partial | 1–3% |
| FSE dual-state pipelining | 3 sequential `update_state` calls per sequence | Two parallel state machines hiding table lookup latency | ❌ Missing | 10–15% |
| Bit reload fast path | Single `refill()` with all cases | `BIT_reloadDStreamFast` (no bounds check) for hot loop | ⚠️ Simplified | 1–2% |
| Output buffer `len()` | Branch + 2 arithmetic ops on every call | Pointer subtraction (1 op) | ❌ Avoidable | 3–6% |
| `% cap` in ringbuffer | Integer modulo (`%`) | Pointer arithmetic (no modulo) | ⚠️ Mitigatable with `& (cap-1)` | 1–2% |
| ARM64 prefetch | Absent | `PREFETCH_L1(match)` before match copy | ❌ Missing | Up to 5% on ARM64 |

### Highest-ROI Changes

1. **FSE dual-state pipelining** — 10–15% potential, but requires significant refactor of `decode_sequences_without_rle`
2. **Track `len` as a field in `RingBuffer`** — ~6% with minimal risk
3. **Exponential doubling in `repeat_in_chunks`** — 2–5%, very low risk
4. **Sequence fast/slow path split** — 5–10%, moderate complexity
5. **`% cap` → `& (cap-1)` in RingBuffer** (requires cap always power-of-two, which it already is per `next_power_of_two()` at line 78) — 1–2%, trivial
