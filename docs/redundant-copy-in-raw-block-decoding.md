# Redundant Copy in Raw Block Decoding

## Location

[block_decoder.rs:88-94](../ruzstd/src/decoding/block_decoder.rs#L88-L94)

## Issue

In the `BlockType::Raw` handling within `decode_block_content`, there's a redundant copy happening:

```rust
for _ in 0..full_reads {
    source.read_exact(&mut buf[..]).map_err(|err| {
        DecodeBlockContentError::ReadError {
            step: block_type,
            source: err,
        }
    })?;
    workspace.buffer.push(&buf[..]);  // <-- Copy here
}
```

## Problem

1. `source.read_exact(&mut buf[..])` reads data from the source into the temporary `buf`
2. `workspace.buffer.push(&buf[..])` then copies from `buf` into the workspace buffer

This results in **double copying**: `source` → `buf` → `workspace.buffer`

## Potential Optimization

We could eliminate the intermediate buffer by reading directly into the workspace buffer, achieving a single copy: `source` → `workspace.buffer`

### Possible Approach

If `DecodeBuffer` (the type of `workspace.buffer`) provides a method to expose a mutable slice or allows direct writing, we could:

```rust
// Pseudocode - would need to verify DecodeBuffer API supports this
for _ in 0..full_reads {
    let dest_slice = workspace.buffer.get_writable_slice(BATCH_SIZE)?;
    source.read_exact(dest_slice).map_err(...)?;
    workspace.buffer.advance(BATCH_SIZE)?;
}
```

## Considerations

1. **DecodeBuffer API**: Need to check if `DecodeBuffer` in [decode_buffer.rs](../ruzstd/src/decoding/decode_buffer.rs) supports direct mutable access
2. **Performance Impact**: Raw blocks are uncompressed data, so this is a pure copy operation - eliminating the redundant copy could have measurable performance benefits for streams with many raw blocks
3. **Complexity vs. Benefit**: The current approach is simple and clear. Would need benchmarking to confirm the optimization is worthwhile
4. **no_std Compatibility**: Any solution must work in both std and no_std contexts

## Next Steps

- [ ] Benchmark current implementation with raw block heavy workloads
- [ ] Investigate `DecodeBuffer` API for direct write support
- [ ] If beneficial, implement optimization while maintaining clarity and correctness
- [ ] Add regression tests to ensure correctness

## Related Code

- Same pattern appears in the `single_read_size` handling (lines 97-104)
- `DecodeBuffer` implementation: [decode_buffer.rs](../ruzstd/src/decoding/decode_buffer.rs)
