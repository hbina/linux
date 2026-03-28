# Dictionary Compression

ruzstd supports compression with pre-trained zstd dictionaries via `FrameCompressor::add_dict()`.

## How it works

A zstd dictionary bundles entropy tables and a block of "raw content" that acts as history
pre-loaded before the first byte of real input. The encoder inserts that raw content into its
match window at the start of each frame so the LZ77 matcher can emit back-references into the
dictionary. The dictionary ID is written into the zstd frame header so the decoder knows which
dictionary to load.

### Encoder side (`MatchGeneratorDriver`)

* `set_dict_content(content)` — stores the raw dictionary bytes and increases
  `max_window_size` by `content.len()` so the dict and at least one normal block can coexist.
* `clear_dict()` — removes the dictionary and restores `max_window_size`.
* `reset()` — after clearing the window, chunks the dict content through
  `MatchGenerator::add_data` + `skip_matching`, inserting every suffix into the hash table
  without generating sequences. Subsequent real blocks can then match against those entries.
* `window_size()` — returns the base window size *excluding* the dict length, which is the
  value written into the frame header.

### Frame header

`FrameHeader.dictionary_id` is set to `Some(dict_id as u64)` when a dictionary is active,
causing the correct `Dictionary_ID_Flag` bits and `Dictionary_ID` field to be serialized.

## Usage

```rust
use ruzstd::encoding::{FrameCompressor, CompressionLevel};

let dict_bytes = std::fs::read("my.dict").unwrap();
let input: &[u8] = b"data to compress";
let mut output = Vec::new();

let mut compressor = FrameCompressor::new(CompressionLevel::Fastest);
compressor.add_dict(&dict_bytes).unwrap();
compressor.set_source(input);
compressor.set_drain(&mut output);
compressor.compress();
```

To decompress, load the same dictionary into `FrameDecoder::add_dict()`.

## Limitations

* Only the raw content portion of the dictionary is used for matching. The entropy tables
  stored in the dictionary are not applied to the encoder output (the encoder always uses its
  own entropy coding).
* Dictionary compression is only available for `MatchGeneratorDriver` (the default matcher),
  not for custom `Matcher` implementations.
