# Redundant Copy in Block Decoder

## Location

`ruzstd/src/decoding/block_decoder.rs` - in the block content reading logic

## Issue

There is a redundant copy when reading block content:

```rust
source.read_exact(&mut buf[..]).map_err(|err| {
    DecodeBlockContentError::ReadError {
        step: block_type,
        source: err,
    }
})?;
workspace.buffer.push(&buf[..]);
```

Data is first read into an intermediate buffer (`buf`), then copied again into `workspace.buffer`. This results in an unnecessary memory copy.

## Potential Optimization

Consider reading directly into the workspace buffer to avoid the intermediate copy. This could be done by:

1. Reserving space in `workspace.buffer` for the incoming data
2. Reading directly into that reserved space

This would require extending the `DecodeBuffer` API to support direct writes or provide a mutable slice for reading into.

## Impact

This affects decompression performance, particularly for RLE and Raw block types where the pattern occurs. The impact scales with the size of the data being decompressed.
