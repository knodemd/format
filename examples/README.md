# Examples

Example `.knode` spaces, small and readable, for testing implementations.

## Example.knode

A space for a small side project, a garden planner, written by a person (Ada) and two agents
(`claude-code` and `codex`). It follows the [specification v0](../spec/knode-0.md) at schema
version 4 and was written by the reference library with
`cargo run -p knode-format --example example_space -- Example.knode` from
[knode/core](https://github.com/knodemd/knode/tree/main/core).

| Node | Kind | Written by | Shows |
|---|---|---|---|
| Garden planner | `note` | Ada | `[[links]]` with a label, tags, an attached image shown with `![[beds.svg]]` |
| SQLite for local data | `decision` | Ada | Headings and lists; a link to the conversation |
| Conversation: choosing the database | `conversation` | `claude-code` | A captured conversation, with its session, project and times as properties |
| Frost dates | `source` | `codex` | A GFM table |
| Ideas for v2 | `idea` | Ada, then `codex` | Task lists, an unresolved link, two versions in the history |

What it contains:

- **Links from bodies:** thirteen links in all. Seven are `mentions`, read from `[[links]]` in
  the bodies; the decision and the conversation mention each other.
- **Links made by hand:** the decision is `derived-from` the conversation and `answers` Ideas for
  v2; Frost dates and Ideas for v2 are `part-of` the Garden planner.
- **A kind of link the space defines:** `depends-on`, which reads *depends on* from its source
  and *needed by* from its target, in `link_kinds`. The Garden planner depends on the decision,
  and Ideas for v2 on the Frost dates.
- **An unresolved link:** Ideas for v2 links to `[[Companion planting]]`, which does not exist
  yet. It waits in `unresolved_links` and becomes a link when a node with that title is created.
- **Provenance:** three rows: Ada, and each agent with its session id and project folder.
- **History:** six versions: one `create` per node and an `update` of Ideas for v2 by `codex`.
- **Changes:** fifteen, one for each write, in order: five `node.create`, three `link.create`,
  the `link_kind.set` of `depends-on`, three more `link.create`, the `attachment.add` of
  `beds.svg`, the `view.set` of the graph view and the `node.update` by `codex`. Every version
  points at its change with `change_id`, and the six links made by hand are kept in
  `link_history`.
- **An attachment:** `beds.svg`, a plan of the beds, attached to the Garden planner. Its bytes are
  in `blobs/0b/fa5c...`, named by their SHA-256, with a row in `blobs` and one in `attachments`.
  `![[beds.svg]]` in the body resolves to it before any node title, so it makes no link.
- **A graph view:** `Garden planner`, filtered to three tags, with a position for each node and
  the viewport it was left at. Positions add no change.
- **Settings, folders, agents and grants:** none yet, beyond `space.id`: no icon, capture on,
  and no agent has access to the space.
- **Tags:** `project`, `architecture`, `database` (twice) and `reference`.
- **Properties:** the conversation has its agent, session id, project folder, branch, and start
  and end times as properties, not in its body.

No embeddings, Loro documents or canvases yet. The `sync/` folder is empty; git does not keep
empty folders, so an implementation creates it when it opens the space.

Look inside with the `sqlite3` command:

```sh
sqlite3 Example.knode/knode.db "SELECT id, kind, title FROM nodes"
sqlite3 Example.knode/knode.db \
  "SELECT c.id, c.action, p.actor, c.subject
   FROM changes c JOIN provenance p ON p.id = c.changed_by ORDER BY c.id"
sqlite3 Example.knode/knode.db \
  "SELECT l.kind, s.title, t.title FROM links l
   JOIN nodes s ON s.id = l.source_id JOIN nodes t ON t.id = l.target_id ORDER BY l.kind"
sqlite3 Example.knode/knode.db "SELECT node_id, name, hash FROM attachments"
sqlite3 Example.knode/knode.db \
  "SELECT n.title, snippet(nodes_fts, -1, '[', ']', '…', 8)
   FROM nodes_fts JOIN nodes n ON n.seq = nodes_fts.rowid
   WHERE nodes_fts MATCH 'wal' ORDER BY rank"
```

Reading the database with `sqlite3` changes nothing, but keep a copy if you experiment with
writes: the ids, timestamps and history are what an implementation is tested against.
