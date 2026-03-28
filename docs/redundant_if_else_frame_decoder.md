# Potentially Redundant If-Else in Frame Decoder

## Location

`ruzstd/src/decoding/frame_decoder.rs` - in the read logic

## Issue

There is a potentially redundant if-else branch:

```rust
if state.frame_finished {
    state.decoder_scratch.buffer.read_all(target)
} else {
    state.decoder_scratch.buffer.read(target)
}
```

## Analysis

The distinction between `read_all` and `read` when `frame_finished` is true may not be necessary. If `read_all` and `read` behave identically when all data has been written to the buffer (i.e., no more data is coming), this conditional could be simplified.

## Questions to Investigate

1. What is the semantic difference between `read_all` and `read` on `DecodeBuffer`?
2. Does `read_all` have special behavior that only matters when the frame is finished?
3. Could `read` handle both cases correctly, making the branch unnecessary?

## Potential Simplification

If the methods are equivalent in the finished state, this could be simplified to a single call to `read(target)`.
