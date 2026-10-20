# Specification

| Version | Status | Document |
|---|---|---|
| 0 | Draft | [knode-0.md](knode-0.md) |

[knode-0.md](knode-0.md) covers the package layout, `manifest.json`, the full `knode.db` schema
(nodes, properties, links, provenance, history, full-text search), identifiers and timestamps,
the Markdown dialect and `[[links]]`, space settings and the folders that feed a space, agents
and their grants, change history and undo, typed links and saved views, semantic and hybrid
search, attachments, concurrent editing with Loro, and how a space migrates from one version to
the next.

The reference implementation is
[knode-format](https://github.com/knodemd/knode/tree/main/core/crates/knode-format) in Rust.
