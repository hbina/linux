# Architectural Review of ruzstd Library

This document provides a comprehensive analysis of the ruzstd library's design, identifying strengths, weaknesses, and recommendations for improvement.

## Overview

ruzstd is a pure Rust implementation of the Zstandard compression format (RFC8878). The library provides complete decompression support and partial compression support (only "Fastest" level implemented).

## Architectural Strengths

### 1. Clean Separation of Concerns

The library follows a well-structured layered architecture:

```
StreamingDecoder (high-level, io::Read)
    └── FrameDecoder (frame-level state machine)
            └── BlockDecoder (block-level processing)
                    ├── LiteralsSectionDecoder
                    ├── SequenceSectionDecoder
                    └── SequenceExecution
```

Each layer has clear responsibilities and minimal coupling to adjacent layers.

### 2. Excellent no_std Support

The conditional compilation strategy is well-implemented:

- `io_std.rs`: Re-exports `std::io` types when `std` feature enabled
- `io_nostd.rs`: Provides minimal `Read`/`Write` traits for no_std
- All heap allocations use `alloc` crate, not `std`
- API surface remains nearly identical between std and no_std modes

### 3. Workspace Reuse Pattern

The `DecoderScratch` struct efficiently reuses allocations across blocks:

```rust
pub struct DecoderScratch {
    huf: HuffmanScratch,
    fse: FSEScratch,
    buffer: DecodeBuffer,
    offset_hist: [u32; 3],
    literals_buffer: Vec<u8>,
    sequences: Vec<Sequence>,
    block_content_buffer: Vec<u8>,
}
```

This avoids per-block allocations, which is critical for performance.

### 4. Forward-Compatible Error Types

All public error enums use `#[non_exhaustive]`, allowing new variants without breaking changes:

```rust
#[non_exhaustive]
pub enum FrameDecoderError {
    // ...
}
```

### 5. Extensible Compression via Matcher Trait

The `Matcher` trait allows custom compression algorithms:

```rust
pub trait Matcher {
    fn get_next_space(&mut self) -> &mut [u8];
    fn commit_space(&mut self, used: usize);
    fn start_matching(&mut self, source: &[u8], start_idx: usize, ...) -> Sequence;
    // ...
}
```

## Areas for Improvement

### 1. Incomplete State Machine (Critical)

**Location:** `ruzstd/src/decoding/block_decoder.rs:27`

```rust
pub enum DecoderState {
    ReadyToDecodeNextHeader,
    ReadyToDecodeNextBody,
    Failed, //TODO put "self.internal_state = DecoderState::Failed;" everywhere an unresolvable error occurs
}
```

**Problem:** The `Failed` state is defined but never transitioned to. When errors occur, the decoder remains in an inconsistent state rather than properly marking itself as failed.

**Impact:**
- Subsequent operations may produce undefined behavior
- Users cannot check if the decoder is in a failed state
- Violates state machine semantics

**Recommendation:** Audit all error paths and add `self.internal_state = DecoderState::Failed;` before returning errors.

### 2. Multi-Frame Limitation (High Priority)

**Location:** `ruzstd/src/decoding/streaming_decoder.rs:22-28`

```rust
// NOTE: This is based on a very simple "there is only one frame" assumption!
// Once this limit is removed, this could easily handle multi-frame data and skipable frames.
```

**Problem:** The zstd specification allows multiple concatenated frames in a single archive. `StreamingDecoder` only handles single frames.

**Impact:**
- Users must manually recreate decoders for each frame
- `SkipFrame` errors must be handled externally
- No built-in support for common multi-frame archives

**Recommendation:** Add a `MultiFrameDecoder` wrapper:

```rust
pub struct MultiFrameDecoder<R: Read> {
    source: R,
    current_decoder: Option<StreamingDecoder<...>>,
}

impl<R: Read> Read for MultiFrameDecoder<R> {
    fn read(&mut self, buf: &mut [u8]) -> io::Result<usize> {
        // Handle frame transitions automatically
    }
}
```

### 3. Incomplete Compression Levels (Medium Priority)

**Location:** `ruzstd/src/encoding/mod.rs`

```rust
pub enum CompressionLevel {
    Uncompressed,
    Fastest,      // Implemented
    Default,      // NOT IMPLEMENTED
    Better,       // NOT IMPLEMENTED
    Best,         // NOT IMPLEMENTED
}
```

**Problem:** Accepting unimplemented levels silently may produce unexpected results.

**Recommendation:** Either:
- Return `Result<_, UnimplementedLevelError>` for unimplemented levels
- Document clearly which levels are implemented
- Add `#[non_exhaustive]` and remove unimplemented variants

### 4. API Inconsistencies (Medium Priority)

#### Redundant Methods

**Location:** `ruzstd/src/decoding/frame_decoder.rs`

`init()` and `reset()` are identical - both call `reset()`. This is confusing.

**Recommendation:** Deprecate `init()` or clarify semantic difference.

#### Dictionary Management Confusion

Two mechanisms exist for dictionary handling:
1. Automatic: `dictionaries: BTreeMap<u32, Dictionary>`
2. Manual: `force_dict(dict: Option<Dictionary>)`

**Recommendation:** Unify into a single, clear API:

```rust
impl FrameDecoder {
    pub fn add_dictionary(&mut self, dict: Dictionary) -> Result<(), DictError>;
    pub fn set_dictionary(&mut self, dict: Dictionary);  // Force specific dict
    pub fn clear_dictionaries(&mut self);
}
```

### 5. Error Type Proliferation (Low Priority)

Currently 10+ error types exist:
- `FrameDecoderError`
- `ReadFrameHeaderError`
- `BlockHeaderReadError`
- `BlockTypeError`
- `BlockSizeError`
- `DecodeBlockContentError`
- `DecodeSequenceError`
- `ExecuteSequencesError`
- `DecodeBufferError`
- `GetBitsError`
- etc.

**Problem:** High cognitive overhead for users handling errors.

**Recommendation:** Consolidate into fewer, broader error types:

```rust
pub enum DecodeError {
    Frame(FrameError),
    Block(BlockError),
    Sequence(SequenceError),
    Buffer(BufferError),
    Io(io::Error),
}
```

### 6. Missing Builder Pattern (Low Priority)

Current construction requires knowing all parameters upfront:

```rust
let mut decoder = FrameDecoder::new();
decoder.add_dict(dict)?;
// No way to set max_window_size, etc.
```

**Recommendation:** Add builder pattern:

```rust
let decoder = FrameDecoder::builder()
    .max_window_size(64 * 1024 * 1024)
    .with_dictionary(dict)
    .checksum_validation(true)
    .build()?;
```

### 7. Hardcoded Limits (Low Priority)

Several limits are hardcoded without configuration options:

| Constant | Value | Location |
|----------|-------|----------|
| Max window size | 100 MB | `frame_decoder.rs` |
| Default slice size | 128 KB | `match_generator.rs` |
| Min match length | 5 bytes | `match_generator.rs` |

**Recommendation:** Make these configurable via builder or constants that users can override.

### 8. Generic Complexity (Low Priority)

**Location:** `FrameCompressor<R, W, M>` and `StreamingDecoder<READ, DEC>`

The generic parameters add complexity without clear benefit for most users.

**Recommendation:** Add type aliases for common cases:

```rust
pub type DefaultCompressor<R, W> = FrameCompressor<R, W, MatchGeneratorDriver>;
pub type DefaultDecoder<R> = StreamingDecoder<R, FrameDecoder>;
```

## Outstanding TODOs

The codebase contains 30+ TODO comments indicating incomplete work:

| Location | TODO |
|----------|------|
| `block_decoder.rs:27` | State machine `Failed` state never used |
| `encoding/` | Make use of offset history |
| `encoding/` | Store previously used tables |
| `encoding/` | Check if new table is better |
| `ringbuffer.rs` | Branchless implementation |
| Various | Performance optimizations |

## Performance Considerations

### Identified Hotspots

1. **Sequence Section Decoder** - Complex FSE/RLE decoding with many conditionals
2. **Match Generator** - Hash table lookup and window management
3. **Decode Buffer** - Chunk-based copying for overlapping regions

### Optimization Opportunities

1. **SIMD for copy operations** - Sequence execution does byte-by-byte copying in some paths
2. **Memory pooling** - No top-level allocator abstraction for `FrameDecoder`
3. **Branchless operations** - Several places marked for branchless optimization

## Recommendations Summary

| Priority | Recommendation | Effort |
|----------|----------------|--------|
| Critical | Fix state machine - use `Failed` state | Low |
| High | Add `MultiFrameDecoder` | Medium |
| Medium | Error on unimplemented compression levels | Low |
| Medium | Unify dictionary management API | Medium |
| Low | Consolidate error types | High |
| Low | Add builder pattern | Medium |
| Low | Make limits configurable | Low |
| Low | Add type aliases for generics | Low |

## Conclusion

The ruzstd library has a solid architectural foundation with clean separation of concerns, excellent no_std support, and good performance characteristics. The main areas needing attention are:

1. Completing the state machine implementation
2. Supporting multi-frame archives at the high-level API
3. Cleaning up API inconsistencies

The library is well-positioned for incremental improvement without requiring major architectural changes.
