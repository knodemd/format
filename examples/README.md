# Examples

Example `.knode` spaces, small and readable, for testing implementations.

## Example.knode

A space for a small side project, a garden planner, written by a person (Ada) and two agents
(`claude-code` and `codex`). It follows the [specification v0](../spec/knode-0.md) at schema
version 4. It was written at schema version 3 by the reference library with
`cargo run -p knode-format --example example_space -- Example.knode` from
[knode/core](https://github.com/knodemd/knode/tree/main/core), then moved to schema version 4 with
migration 4, and Ada's last four changes (a kind of link, a typed link, an attachment and a graph
view) were written with SQL, following sections 18 to 20.

| Node | Kind | Written by | Shows |
|---|---|---|---|
| Garden planner | `note` | Ada | `[[links]]` with a label, tags, an attachment shown with `![[beds.svg]]` |
| SQLite for local data | `decision` | Ada | Headings and lists; a link to the conversation |
| Conversation: choosing the database | `conversation` | `claude-code` | A captured conversation, with its session, project and times as properties |
| Frost dates | `source` | `codex` | A GFM table |
| Ideas for v2 | `idea` | Ada, then `codex` | Task lists, an unresolved link, two versions in the history |

What it contains:

- **Links from bodies:** eleven links in all. Seven are `mentions`, read from `[[links]]` in the
  bodies; the decision and the conversation mention each other.
- **Links made by hand:** the decision is `derived-from` the conversation; Frost dates and Ideas
  for v2 are `part-of` the Garden planner; Ideas for v2 `depends-on` the decision.
- **A kind of link of the space:** `depends-on`, defined in `link_kinds`, reads "depends on" from
  its source and "needed by" from its target.
- **An unresolved link:** Ideas for v2 links to `[[Companion planting]]`, which does not exist
  yet. It waits in `unresolved_links` and becomes a link when a node with that title is created.
- **Provenance:** three rows: Ada, and each agent with its session id and project folder.
- **History:** seven versions: one `create` per node, an `update` of Ideas for v2 by `codex`, and
  an `update` of Garden planner by Ada, who pasted the sketch.
- **Changes:** thirteen, one for each write, in order: five `node.create`, three `link.create`,
  the `node.update` by `codex`, then Ada's `link_kind.set`, `link.create`, `node.update` and
  `view.set`. Every version points at its change with `change_id`, and the four links made by
  hand are kept in `link_history`.
- **An attachment:** `beds.svg`, a sketch of the beds, attached to Garden planner. Its file is
  `blobs/38/0e6bdfbf932054edc571bf2f25be679803d749ce541410535fb30eab1f5af4`, named by its SHA-256,
  with a row in `blobs` (size and media type) and one in `attachments` (node and name).
- **A graph view:** "Garden map", with a saved position for each of the five nodes.
- **Settings, folders, agents and grants:** none yet, beyond `space.id`: no icon, capture on,
  and no agent has access to the space.
- **Tags:** `project`, `architecture`, `database` (twice) and `reference`.
- **Properties:** the conversation has its agent, session id, project folder, branch, and start
  and end times as properties, not in its body.

There are no embeddings (they are made by a model on the device, section 19) and no Loro
documents (made when a node is first edited through Loro, section 21), so `sync/` is empty; git
does not keep empty folders, so an implementation creates it when it opens the space.

Look inside with the `sqlite3` command:

```sh
sqlite3 Example.knode/knode.db "SELECT id, kind, title FROM nodes"
sqlite3 Example.knode/knode.db \
  "SELECT c.id, c.action, p.actor, c.subject
   FROM changes c JOIN provenance p ON p.id = c.changed_by ORDER BY c.id"
sqlite3 Example.knode/knode.db \
  "SELECT n.title, snippet(nodes_fts, -1, '[', ']', '…', 8)
   FROM nodes_fts JOIN nodes n ON n.seq = nodes_fts.rowid
   WHERE nodes_fts MATCH 'wal' ORDER BY rank"
```

Reading the database with `sqlite3` changes nothing, but keep a copy if you experiment with
writes: the ids, timestamps and history are what an implementation is tested against.
