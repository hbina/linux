# Dictionary Interoperability Test

## Overview

`scripts/dict_interop_test.py` tests that the zstd dictionary format used by
ruzstd is fully compatible with the reference C `zstd` implementation.  It does
this by exercising the complete cross-codec round-trip pipeline in both
directions.

## Pipeline A (zstd-first)

```
raw
 └─[zstd compress -D dict]──► compressed_a.zst
                                └─[ruzstd decompress -D dict]──► mid.raw
                                                                   └─[ruzstd compress -D dict]──► compressed_b.zst
                                                                                                   └─[zstd decompress -D dict]──► final.raw
                                                                                                                                   └─[compare]──► raw == final?
```

## Pipeline B (ruzstd-first)

```
raw
 └─[ruzstd compress -D dict]──► compressed_a.zst
                                  └─[zstd decompress -D dict]──► mid.raw
                                                                   └─[zstd compress -D dict]──► compressed_b.zst
                                                                                                  └─[ruzstd decompress -D dict]──► final.raw
                                                                                                                                    └─[compare]──► raw == final?
```

## What this validates

| Property | How it is tested |
|---|---|
| ruzstd writes valid dict-compressed frames | zstd can decompress them in Pipeline B step 1 |
| ruzstd reads dict-compressed frames from zstd | ruzstd decompresses zstd output in Pipeline A step 1 |
| Dictionary IDs are correctly embedded in frames | Both codecs auto-select the right dictionary from the frame header |
| End-to-end data integrity | SHA-256 of the final output must match the original raw input |

## Prerequisites

- Rust toolchain (`cargo`)
- Reference `zstd` CLI (`apt install zstd` / `brew install zstd`)
- Python 3.7+

## Running

```bash
# Build + run all tests (recommended)
python3 scripts/dict_interop_test.py

# Skip the cargo build step (if already built)
python3 scripts/dict_interop_test.py --skip-build

# Verbose: print each subprocess invocation
python3 scripts/dict_interop_test.py --skip-build -v

# Use a custom dictionary
python3 scripts/dict_interop_test.py --dict /path/to/my.dict
```

## Default dictionary

The script uses the dictionary shipped with the ruzstd test corpus:

```
ruzstd/dict_tests/dictionary
```

It also re-uses the raw files in `ruzstd/dict_tests/files/` as part of its
corpus of test inputs, in addition to synthetically generated data
(all-zeros, sequential bytes, text patterns, random bytes) at various sizes.

## CLI changes required

To support dictionary operations from the command line, `ruzstd-cli` was
extended with a `-D` / `--dict` option for both the `compress` and
`decompress` subcommands:

```
ruzstd-cli compress -D dictionary input.txt output.zst
ruzstd-cli decompress -D dictionary output.zst input.txt
```

This mirrors the interface of the reference `zstd` CLI.

## Known issues discovered during testing

Running the script against the `dict_tests/files` corpus exposed a panic in
the FSE encoder (`fse_encoder.rs:304`) for very small inputs (< ~300 bytes).
This is a pre-existing bug in ruzstd unrelated to dictionary handling, and the
test correctly reports it as an `ERROR`.
