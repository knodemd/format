<img src="https://raw.githubusercontent.com/knodemd/brand/main/09%20Banners/README%20Headers/PNG/readme-format.png" alt="Knode format" width="100%">

# The .knode format

An open format for a local, searchable memory shared by people and AI agents. A `.knode` file is one
space: notes, ideas and conversations as nodes, the links between them, where each one came from, and
the indexes to search them.

Anyone can read and write `.knode` files. Only Knode may change the format and still call it `.knode`.

## What's here

| Path | What |
|---|---|
| `spec/` | The specification, versioned |
| `examples/` | Example spaces, to test an implementation against |

## Status

Version 0, in design. It is frozen as 1.0 before Knode's public beta; until then it can change.
Follow the [roadmap](https://github.com/knodemd/handbook/blob/main/06-roadmap.md).

## Implementations

- [knode-format](https://github.com/knodemd/knode/tree/main/core/crates/knode-format), the reference
  library in Rust, in the Knode repository.

## Licence

[Apache 2.0](LICENSE); keep the [NOTICE](NOTICE).
