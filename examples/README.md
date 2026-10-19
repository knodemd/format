# Examples

Example `.knode` spaces, small and readable, for testing implementations.

## Example.knode

A space for a small side project, a garden planner, written by a person (Ada) and two agents
(`claude-code` and `codex`). It follows the [specification v0](../spec/knode-0.md) and was
written by the reference library with
`cargo run -p knode-format --example example_space -- Example.knode` from
[knode/core](https://github.com/knodemd/knode/tree/main/core).

| Node | Kind | Written by | Shows |
|---|---|---|---|
| Garden planner | `note` | Ada | `[[links]]` with a label, tags |
| SQLite for local data | `decision` | Ada | Headings and lists; a link to the conversation |
| Conversation: choosing the database | `conversation` | `claude-code` | A captured conversation, with its session, project and times as properties |
| Frost dates | `source` | `codex` | A GFM table |
| Ideas for v2 | `idea` | Ada, then `codex` | Task lists, an unresolved link, two versions in the history |

What it contains:

- **Links from bodies:** ten links in all. Seven are `mentions`, read from `[[links]]` in the
  bodies; the decision and the conversation mention each other.
- **Links made by hand:** the decision is `derived-from` the conversation; Frost dates and Ideas
  for v2 are `part-of` the Garden planner.
- **An unresolved link:** Ideas for v2 links to `[[Companion planting]]`, which does not exist
  yet. It waits in `unresolved_links` and becomes a link when a node with that title is created.
- **Provenance:** three rows: Ada, and each agent with its session id and project folder.
- **History:** six versions: one `create` per node and an `update` of Ideas for v2 by `codex`.
- **Tags:** `project`, `architecture`, `database` (twice) and `reference`.
- **Properties:** the conversation has its agent, session id, project folder, branch, and start
  and end times as properties, not in its body.

The `blobs/` and `sync/` folders are empty in this version; git does not keep empty folders, so
an implementation creates them when it opens the space.

Look inside with the `sqlite3` command:

```sh
sqlite3 Example.knode/knode.db "SELECT id, kind, title FROM nodes"
sqlite3 Example.knode/knode.db \
  "SELECT n.title, snippet(nodes_fts, -1, '[', ']', '…', 8)
   FROM nodes_fts JOIN nodes n ON n.seq = nodes_fts.rowid
   WHERE nodes_fts MATCH 'wal' ORDER BY rank"
```

Reading the database with `sqlite3` changes nothing, but keep a copy if you experiment with
writes: the ids, timestamps and history are what an implementation is tested against.
