# Refactor `LiteralsSection::parse_from_header` to Static Function

## Current Implementation

**Location:** `ruzstd/src/blocks/literals_section.rs:117`

```rust
pub fn parse_from_header(&mut self, raw: &[u8]) -> Result<u8, LiteralsSectionParseError> {
    // ... parses header and mutates self ...
}
```

**Usage:** `ruzstd/src/decoding/block_decoder.rs:136-137`

```rust
let mut section = LiteralsSection::new();
let bytes_in_literals_header = section.parse_from_header(raw)?;
```

## Problem

The current design requires:

1. Creating a mutable `LiteralsSection` with dummy default values
2. Calling `parse_from_header` to mutate it in place
3. The variable must be declared `mut` even though it's only "mutated" once during initialization

This is an anti-pattern in Rust. The `section` variable is never actually mutated after parsing - it's used immutably for the rest of its lifetime. The `mut` keyword here is misleading about the variable's actual usage.

## Proposed Solution

Convert `parse_from_header` to a static constructor function:

```rust
impl LiteralsSection {
    /// Parse a literals section header from raw bytes.
    /// Returns the parsed section and the number of bytes consumed.
    pub fn parse_from_header(raw: &[u8]) -> Result<(Self, u8), LiteralsSectionParseError> {
        let mut br: BitReader<'_> = BitReader::new(raw);
        let block_type = br.get_bits(2)? as u8;
        let ls_type = Self::section_type(block_type)?;
        let size_format = br.get_bits(2)? as u8;

        let byte_needed = Self::header_bytes_needed_inner(ls_type, size_format);
        if raw.len() < byte_needed as usize {
            return Err(LiteralsSectionParseError::NotEnoughBytes {
                have: raw.len(),
                need: byte_needed,
            });
        }

        // ... rest of parsing logic, building the struct at the end ...

        Ok((
            LiteralsSection {
                regenerated_size,
                compressed_size,
                num_streams,
                ls_type,
            },
            bytes_consumed,
        ))
    }
}
```

## Benefits

### 1. Immutable by Default

The call site becomes cleaner and more idiomatic:

```rust
// Before (mutable, misleading)
let mut section = LiteralsSection::new();
let bytes_in_literals_header = section.parse_from_header(raw)?;

// After (immutable, clear intent)
let (section, bytes_in_literals_header) = LiteralsSection::parse_from_header(raw)?;
```

### 2. No Dummy Initialization

The current `new()` function creates a struct with meaningless default values:

```rust
pub fn new() -> LiteralsSection {
    LiteralsSection {
        regenerated_size: 0,        // Meaningless
        compressed_size: None,      // Meaningless
        num_streams: None,          // Meaningless
        ls_type: LiteralsSectionType::Raw,  // Meaningless
    }
}
```

These values are immediately overwritten. With a static function, no dummy values are needed.

### 3. Clearer Ownership Semantics

The function signature `fn parse_from_header(raw: &[u8]) -> Result<(Self, u8), ...>` clearly communicates:
- Input: borrowed byte slice
- Output: owned `LiteralsSection` + bytes consumed
- No hidden mutation

### 4. Potential for `const fn` or Compile-Time Guarantees

A static constructor is easier to reason about and could potentially be made `const fn` in the future if needed.

## Additional Cleanup

### Remove `new()` and `Default`

If `parse_from_header` becomes the only way to construct a valid `LiteralsSection`, consider:

1. Removing `pub fn new()`
2. Removing the `Default` impl
3. Making fields private (if not already used externally)

This enforces that all `LiteralsSection` instances are properly parsed, preventing invalid states.

### Refactor `header_bytes_needed`

The current `header_bytes_needed` method takes `&self` but only uses the result of `section_type()` called on the input byte. It could become a static function:

```rust
// Current (requires instance)
pub fn header_bytes_needed(&self, first_byte: u8) -> Result<u8, LiteralsSectionParseError>

// Proposed (static)
pub fn header_bytes_needed(first_byte: u8) -> Result<u8, LiteralsSectionParseError>
```

## Migration

This is a breaking change to the public API. Options:

1. **Deprecation path**: Keep old method, add new static function, deprecate old
2. **Major version bump**: Remove old method entirely

Since `LiteralsSection` is in the `blocks` module which appears to be internal (not re-exported from lib.rs), this may not be a breaking change for external users.

## Similar Pattern in Codebase

Check if `SequencesHeader` in `sequence_section.rs` has the same pattern - it likely does and could benefit from the same refactoring.
