# Refactor: BlockDecoder header_buffer

## Current Implementation

In `ruzstd/src/decoding/block_decoder.rs`, the `BlockDecoder` struct uses a raw byte buffer to store block headers:

```rust
pub struct BlockDecoder {
    header_buffer: [u8; 3],
    internal_state: DecoderState,
}
```

The header fields are extracted on-demand via helper methods that perform bit-shifting:

```rust
fn is_last(&self) -> bool {
    self.header_buffer[0] & 0x1 == 1
}

fn block_type(&self) -> Result<BlockType, BlockTypeError> {
    let t = (self.header_buffer[0] >> 1) & 0x3;
    // ...
}

fn block_content_size_unchecked(&self) -> u32 {
    u32::from(self.header_buffer[0] >> 3)
        | (u32::from(self.header_buffer[1]) << 5)
        | (u32::from(self.header_buffer[2]) << 13)
}
```

## Proposed Refactor

Parse the 3-byte header immediately after reading it into a structured type with named fields. This eliminates the need for repeated bit-shifting and makes the code more readable.

### Option 1: Store parsed fields directly in BlockDecoder

```rust
pub struct BlockDecoder {
    // Parsed header fields (valid after read_block_header)
    last_block: bool,
    block_type: BlockType,
    block_size: u32,
    internal_state: DecoderState,
}
```

### Option 2: Create a separate RawBlockHeader struct

```rust
struct RawBlockHeader {
    last_block: bool,
    block_type: BlockType,
    block_size: u32,
}

impl RawBlockHeader {
    fn parse(bytes: [u8; 3]) -> Result<Self, BlockHeaderReadError> {
        let last_block = bytes[0] & 0x1 == 1;
        let block_type = match (bytes[0] >> 1) & 0x3 {
            0 => BlockType::Raw,
            1 => BlockType::RLE,
            2 => BlockType::Compressed,
            3 => return Err(BlockHeaderReadError::FoundReservedBlock),
            _ => unreachable!(),
        };
        let block_size = u32::from(bytes[0] >> 3)
            | (u32::from(bytes[1]) << 5)
            | (u32::from(bytes[2]) << 13);

        if block_size > MAX_BLOCK_SIZE {
            return Err(BlockHeaderReadError::BlockSizeTooLarge { size: block_size });
        }

        Ok(Self { last_block, block_type, block_size })
    }
}
```

## Benefits

1. **Clarity**: Field access becomes `self.last_block` instead of `self.is_last()` with hidden bit manipulation
2. **Single parse point**: Bit-shifting logic is centralized in one place (the parse function)
3. **No redundant computation**: Fields are parsed once, not re-computed on each access
4. **Better separation of concerns**: Raw byte parsing is isolated from the decoder's state machine logic

## Notes

- The `reset_buffer()` method can be removed since there's no raw buffer to reset
- The existing `BlockHeader` struct (used as return value) could potentially be unified with the internal representation
