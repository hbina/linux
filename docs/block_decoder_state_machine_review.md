# Block Decoder State Machine Design Review

## Overview

This document discusses a potential design issue in the `BlockDecoder` state machine located in `ruzstd/src/decoding/block_decoder.rs`. The current implementation uses runtime checks to validate state transitions, which suggests the type system could be better leveraged to make invalid states unrepresentable at compile time.

## The Problem

In `block_decoder.rs:45-51`, the `decode_block_content` function begins with a runtime state check:

```rust
match self.internal_state {
    DecoderState::ReadyToDecodeNextBody => { /* Happy :) */ }
    DecoderState::Failed => return Err(DecodeBlockContentError::DecoderStateIsFailed),
    DecoderState::ReadyToDecodeNextHeader => {
        return Err(DecodeBlockContentError::ExpectedHeaderOfPreviousBlock)
    }
}
```

The fact that this runtime check is necessary indicates that **the API allows callers to invoke `decode_block_content` in an invalid state**. In a well-designed type-safe API, this situation should be impossible to express in code.

### Current State Machine

```rust
enum DecoderState {
    ReadyToDecodeNextHeader,
    ReadyToDecodeNextBody,
    Failed,
}
```

The expected transitions are:
1. `ReadyToDecodeNextHeader` -> (call `read_block_header`) -> `ReadyToDecodeNextBody`
2. `ReadyToDecodeNextBody` -> (call `decode_block_content`) -> `ReadyToDecodeNextHeader`
3. Any state -> (on error) -> `Failed`

However, since both methods are public on the same struct, nothing prevents a caller from:
- Calling `decode_block_content` before calling `read_block_header`
- Calling `decode_block_content` twice without calling `read_block_header` in between
- Calling `read_block_header` twice without consuming the body

## Why This Matters

1. **Runtime cost**: Every call to `decode_block_content` pays for a state check that could be eliminated at compile time.

2. **API usability**: Callers must understand the implicit protocol. The compiler won't help them use the API correctly.

3. **Error handling complexity**: The error types must include variants for "you called this in the wrong order" which is a programmer error, not a data error.

4. **The `Failed` state is unused**: The code has `#[allow(dead_code)]` on `Failed` with a TODO comment, suggesting incomplete error handling.

## Possible Solutions

### Solution 1: Typestate Pattern

Use different types to represent different states, making invalid transitions impossible:

```rust
pub struct BlockDecoder<State> {
    header_buffer: [u8; 3],
    _state: PhantomData<State>,
}

pub struct ReadyForHeader;
pub struct ReadyForBody {
    header: BlockHeader,
}

impl BlockDecoder<ReadyForHeader> {
    pub fn read_block_header(
        self,
        r: impl Read,
    ) -> Result<BlockDecoder<ReadyForBody>, BlockHeaderReadError> {
        // ... read header ...
        Ok(BlockDecoder {
            header_buffer: self.header_buffer,
            _state: PhantomData,
        })
    }
}

impl BlockDecoder<ReadyForBody> {
    pub fn decode_block_content(
        self,
        workspace: &mut DecoderScratch,
        source: impl Read,
    ) -> Result<(BlockDecoder<ReadyForHeader>, u64), DecodeBlockContentError> {
        // No state check needed - type system guarantees we're in the right state
        // ... decode content ...
    }
}
```

**Pros:**
- Invalid states are impossible at compile time
- No runtime overhead for state checks
- Self-documenting API

**Cons:**
- More complex type signatures
- May require restructuring calling code
- The decoder cannot be stored in a single field without using an enum wrapper

### Solution 2: Session Types / Builder Pattern

Return a new type from each operation that only exposes the valid next operation:

```rust
pub struct BlockDecoder { /* ... */ }

pub struct PendingBody<'a> {
    decoder: &'a mut BlockDecoder,
    header: BlockHeader,
}

impl BlockDecoder {
    pub fn read_block_header<'a>(
        &'a mut self,
        r: impl Read,
    ) -> Result<PendingBody<'a>, BlockHeaderReadError> {
        // ...
    }
}

impl<'a> PendingBody<'a> {
    pub fn decode_content(
        self,
        workspace: &mut DecoderScratch,
        source: impl Read,
    ) -> Result<u64, DecodeBlockContentError> {
        // Consumes self, returning control to BlockDecoder
    }
}
```

**Pros:**
- Enforces correct ordering through lifetimes
- Header data is bundled with the pending operation
- More natural API flow

**Cons:**
- Lifetime complexity
- Cannot easily "skip" decoding a body

### Solution 3: Combined Operation

If the header and body are always decoded together, combine them:

```rust
impl BlockDecoder {
    pub fn decode_next_block(
        &mut self,
        workspace: &mut DecoderScratch,
        source: impl Read,
    ) -> Result<(BlockHeader, u64), DecodeBlockError> {
        let header = self.read_block_header(&mut source)?;
        let bytes = self.decode_block_content(&header, workspace, source)?;
        Ok((header, bytes))
    }
}
```

**Pros:**
- Simplest solution
- No state machine needed at all

**Cons:**
- Less flexible if callers need to inspect the header before deciding to decode
- May not fit all use cases

### Solution 4: Require Header as Proof

Make `decode_block_content` require ownership of the header, which can only be obtained from `read_block_header`:

```rust
pub struct BlockHeader {
    // Make fields private, only obtainable via read_block_header
}

pub fn decode_block_content(
    &mut self,
    header: BlockHeader,  // Takes ownership - can only be used once
    workspace: &mut DecoderScratch,
    source: impl Read,
) -> Result<u64, DecodeBlockContentError> {
    // No state check needed - having the header proves we read it
}
```

**Pros:**
- Minimal API change
- Header consumption proves the operation ordering
- Works with current architecture

**Cons:**
- Callers must keep the header around
- Doesn't prevent calling `read_block_header` twice

## What the Reference ZSTD Implementation Does

The reference C implementation (`zstd` by Facebook/Meta) takes a different approach:

1. **Streaming API**: Uses a `ZSTD_DStream` context with `ZSTD_decompressStream()` that handles all state internally. The caller doesn't manage block-level state at all.

2. **Block-level API**: The lower-level `ZSTD_decompressBlock()` function requires the caller to have already called `ZSTD_getFrameHeader()`. The state is managed implicitly through a `ZSTD_DCtx` context struct with internal state tracking.

3. **Error codes**: Invalid state transitions result in error codes like `ZSTD_error_stage_wrong`, similar to our runtime check.

The reference implementation chose runtime checks rather than type-level guarantees, partly because C lacks the type system features to express these constraints. Rust gives us more options.

## Recommendations

1. **Short-term**: Add the state check to `read_block_header` as well (currently commented out on lines 231-235). This provides symmetric validation and makes the state machine behavior consistent.

2. **Medium-term**: Consider Solution 4 (require header as proof) as it has the smallest API impact while improving safety.

3. **Long-term**: Evaluate Solution 1 or 2 (typestate pattern) if a larger refactor is planned. This would provide the strongest compile-time guarantees.

## Related Code Locations

- `ruzstd/src/decoding/block_decoder.rs:23-28` - `DecoderState` enum definition
- `ruzstd/src/decoding/block_decoder.rs:45-51` - Runtime state check in `decode_block_content`
- `ruzstd/src/decoding/block_decoder.rs:231-235` - Commented-out state check in `read_block_header`
- `ruzstd/src/decoding/frame_decoder.rs` - Higher-level decoder that uses `BlockDecoder`

## References

- [Typestate Pattern in Rust](https://cliffle.com/blog/rust-typestate/)
- [Session Types](https://aturon.github.io/blog/2015/09/28/session-types/)
- [Parse, don't validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/) - Philosophy behind making invalid states unrepresentable
- [ZSTD Reference Implementation](https://github.com/facebook/zstd) - See `lib/decompress/zstd_decompress.c`
