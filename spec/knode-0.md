# The .knode format - version 0

**Status:** draft. Version 0 can still change; it is frozen as 1.0 before Knode's public beta.
The reference implementation is
[knode-format](https://github.com/knodemd/knode/tree/main/core/crates/knode-format), with
[knode-md](https://github.com/knodemd/knode/tree/main/core/crates/knode-md) for the Markdown rules.

A `.knode` file is one **space**: a local memory shared by a person and their AI agents. It holds
**nodes** (notes, ideas, decisions, captured conversations), the **links** between them, who wrote
each one and from where (**provenance**), structured metadata such as the agent and session of a
conversation (**properties**), every earlier version and who made it (**history**), a full-text
index to search them, and, from schema version 3, its settings, the folders that feed it, and
what each agent may do in it (**grants**).

The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are to be read as in
[RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

## Contents

1. [The package](#1-the-package)
2. [manifest.json](#2-manifestjson)
3. [Identifiers and timestamps](#3-identifiers-and-timestamps)
4. [knode.db](#4-knodedb)
5. [Tables](#5-tables)
6. [Markdown](#6-markdown)
7. [Links](#7-links)
8. [Plain text](#8-plain-text)
9. [Full-text search](#9-full-text-search)
10. [Provenance](#10-provenance)
11. [History](#11-history)
12. [Writing a node, step by step](#12-writing-a-node-step-by-step)
13. [Versions and migrations](#13-versions-and-migrations)
14. [Reserved for later versions](#14-reserved-for-later-versions)
15. [Space settings and folders](#15-space-settings-and-folders)
16. [Agents, keys and grants](#16-agents-keys-and-grants)
17. [Change history and undo](#17-change-history-and-undo)
18. [Appendix: the schema](#appendix-the-schema)

## 1. The package

A space is a folder whose name ends in `.knode`. On macOS the Knode app declares it as a package,
so the Finder shows it as one file.

```text
Example.knode/
  manifest.json   format version, id and name of the space
  knode.db        SQLite database: nodes, links, provenance, history, search
  blobs/          attachments, by hash (reserved, see section 14)
  sync/           Loro updates for sync (reserved, see section 14)
```

- A writer MUST create all four entries when it creates a space.
- A reader MUST NOT fail when `blobs/` or `sync/` is missing (git, for example, drops empty
  folders); a writer that opens such a space SHOULD create them.
- While a space is open, SQLite also keeps `knode.db-wal` and `knode.db-shm` next to the
  database. They are part of the database: copying a space while it is open MUST include them, or
  be done with SQLite's backup API. When the last connection closes they are folded back into
  `knode.db` and removed.
- Readers MUST ignore any other file in the package. Names starting with `.` are reserved for
  temporary files.
- Only one process SHOULD write to a space at a time. In Knode that process is Knode Service;
  everything else talks to it. SQLite's locking keeps a space consistent even when this is not
  followed, but concurrent writers may wait on each other.

## 2. manifest.json

A UTF-8 JSON object that identifies the space. It is read before the database is opened.

```json
{
  "format": 0,
  "id": "01M5B425TGSX5KHPW5TZNBNP9B",
  "name": "Example",
  "created_at": "2026-10-19T22:21:08.816Z",
  "generator": "knode-format 0.1.0"
}
```

| Field | Type | Required | Meaning |
|---|---|---|---|
| `format` | integer | yes | The format version: `0` for this specification |
| `id` | string | yes | The id of the space, a ULID (section 3). It never changes, even when the package is renamed or moved |
| `name` | string | yes | The name people see. Not empty |
| `created_at` | string | yes | When the space was created, RFC 3339 in UTC with a `Z` suffix and milliseconds |
| `generator` | string | no | The program that last wrote the manifest, such as `knode-format 0.1.0` |

- A reader MUST check `format` first, and MUST refuse a space whose `format` it does not
  support, without changing anything.
- A reader MUST ignore fields it does not know, and a writer MUST keep them when it rewrites the
  manifest.
- A writer SHOULD replace the manifest atomically: write a temporary file in the package (such
  as `.manifest.json.tmp`) and rename it over `manifest.json`.
- The `id` is also stored in `knode.db` (section 5.1). A reader MUST refuse a space whose two
  ids differ: the database does not belong to that manifest.

## 3. Identifiers and timestamps

**Ids are ULIDs.** The space and every node have a [ULID](https://github.com/ulid/spec): 26
characters of Crockford base32, the first 10 encoding the creation time in milliseconds. Ids are
written in upper case. Sorting ids as text sorts them by creation time.

- A writer SHOULD generate ids monotonically within one process, so two ids made in the same
  millisecond still sort in the order they were made.
- Readers MUST accept lower-case ids as input and compare them in upper case.

**Timestamps are UTC.** In `knode.db` every timestamp is an INTEGER: milliseconds since
1970-01-01T00:00:00Z. In JSON (`manifest.json`) timestamps are RFC 3339 strings in UTC, such as
`2026-10-20T09:30:00.000Z`. Local time zones are only for display.

## 4. knode.db

`knode.db` is a [SQLite](https://sqlite.org) 3 database. It needs SQLite 3.37 or newer
(`STRICT` tables) with the FTS5 extension.

**Connection settings.** Every connection that writes MUST use:

```sql
PRAGMA journal_mode = WAL;   -- stored in the file; readers and a writer work at once
PRAGMA foreign_keys = ON;    -- deleting a node removes its links and tags
```

and SHOULD use `PRAGMA synchronous = NORMAL` and a busy timeout of a few seconds.

**Schema version.** SQLite's `PRAGMA user_version` holds the schema version of the database:
`3` for the schema in this document. See section 13.

**Text** is UTF-8. **Booleans** are not used. All tables except the full-text index are
`STRICT`.

## 5. Tables

| Table | What it holds |
|---|---|
| `settings` | Key-value settings of the space |
| `provenance` | Who made a change and from where; shared by nodes, links and history |
| `nodes` | Every node: kind, title, Markdown body, extracted plain text, timestamps |
| `tags` | Tags of each node |
| `links` | Typed links between nodes, by id |
| `unresolved_links` | `[[links]]` whose target does not exist yet |
| `history` | Every version of every node, including deleted ones |
| `nodes_fts` | The FTS5 full-text index of titles and plain text |
| `properties` | Structured metadata of each node, by key (schema version 2) |
| `folders` | The folders that feed the space (schema version 3, 15.2) |
| `agents` | The agents that may have access to the space (schema version 3, 16.3) |
| `agent_keys` | Fingerprints of the agents' keys, never the keys (schema version 3, 16.4) |
| `grants` | What each agent may do in the space: nothing, read or write (schema version 3, 16.5) |
| `changes` | One row per change, with who made it; what undo works on (schema version 3, 17.1) |
| `link_history` | Every link made or removed by hand (schema version 3, 17.3) |

The full SQL is in the [appendix](#appendix-the-schema). A writer MUST create exactly that schema
for the schema version it writes. A reader MUST NOT depend on the order of columns; it reads them by name.

### 5.1 settings

| Column | Type | Meaning |
|---|---|---|
| `key` | TEXT, primary key | A dotted name |
| `value` | TEXT | The value |

Defined keys:

| Key | Value |
|---|---|
| `space.id` | The `id` from `manifest.json`. Written when the space is created; never changes |
| `space.icon` | The icon of the space (schema version 3, 15.1) |
| `capture.enabled` | Whether captured conversations are written to the space (schema version 3, 15.1) |

Keys a reader does not know MUST be kept. Later versions add settings here (section 14).

### 5.2 provenance

One row per distinct author and origin. Rows are shared: a hundred changes from the same agent
session point at the same row. Rows are never updated or deleted.

| Column | Type | Meaning |
|---|---|---|
| `id` | INTEGER, primary key | Row id, referenced by `nodes`, `links` and `history` |
| `actor_kind` | TEXT | `person`, `agent` or `system` (Knode itself) |
| `actor` | TEXT | The name of the person or agent: `Ada`, `claude-code`, `codex` |
| `session` | TEXT, `''` for none | The conversation or session the change came from, such as an agent session id |
| `source` | TEXT, `''` for none | Where the content came from: a file path, a URL, a project folder |

`(actor_kind, actor, session, source)` is unique. Absent values are stored as the empty string,
not NULL, so the uniqueness holds. See section 10.

### 5.3 nodes

| Column | Type | Meaning |
|---|---|---|
| `seq` | INTEGER, primary key | Internal row number; the rowid of the full-text index. Never shown, never used to refer to a node |
| `id` | TEXT, unique | The node id, a ULID (26 characters) |
| `kind` | TEXT | What the node is (below). Default `note` |
| `title` | TEXT | The title, not empty |
| `title_key` | TEXT | The title as compared by links (section 7.2) |
| `body` | TEXT | The text, in Markdown (section 6) |
| `plain` | TEXT | The plain text of the body, for search (section 8) |
| `created_at` | INTEGER | When the node was created, ms UTC |
| `updated_at` | INTEGER | When the node last changed, ms UTC |
| `created_by` | INTEGER | The provenance of the change that created it |
| `updated_by` | INTEGER | The provenance of the last change |

Indexes: `title_key`, `updated_at`, and `(kind, updated_at)`.

**Titles** are trimmed and runs of whitespace become one space. A title MUST NOT be empty and
MUST NOT contain `[`, `]` or `|`, which `[[links]]` use. Titles need not be unique.

**Kinds** are names: lower-case ASCII letters, digits and `-`, starting with a letter, at most 32
characters. This version defines:

| Kind | Meaning |
|---|---|
| `note` | A note written by a person or an agent. The default |
| `conversation` | A conversation with an agent, captured as it happened |
| `decision` | A decision and the reasons for it |
| `idea` | An idea, not yet worked out |
| `source` | Something read: a web page, a document, a book |

A reader MUST keep and show kinds it does not know.

**Metadata is never in the body.** Dates, authors, sources, tags, properties (5.9) and
permissions live in columns and tables. A body MUST NOT be given YAML frontmatter by a writer; frontmatter is only produced
when a space is exported to Markdown files, and removed on import.

### 5.4 tags

| Column | Type | Meaning |
|---|---|---|
| `node_id` | TEXT | The node, deleted with it |
| `tag` | TEXT | One tag |

Primary key `(node_id, tag)`; indexed by `tag`. A tag is stored trimmed, in lower case, without
a leading `#`, and contains no whitespace (at most 64 characters).

### 5.5 links

| Column | Type | Meaning |
|---|---|---|
| `source_id` | TEXT | The node the link starts from |
| `kind` | TEXT | The type of link (below) |
| `target_id` | TEXT | The node the link points at |
| `created_at` | INTEGER | When the link was made, ms UTC |
| `created_by` | INTEGER | The provenance of the change that made it |

Primary key `(source_id, kind, target_id)`: two nodes have at most one link of each kind in each
direction. Indexed by `(target_id, kind)` for backlinks. Deleting either node deletes the link.

**Link kinds** follow the same naming rule as node kinds. This version defines:

| Kind | Meaning |
|---|---|
| `mentions` | The source's body has a `[[link]]` to the target. Written only from the body (section 7) |
| `related` | The nodes are related. The default for links made by hand |
| `part-of` | The source is part of the target, such as a note of a project |
| `derived-from` | The source was made from the target, such as a decision from a conversation |

A writer MUST NOT create or delete `mentions` links except by reading a body. Every other kind is
made and removed by hand, and is never touched when a body changes.

### 5.6 unresolved_links

A `[[link]]` whose target does not exist yet waits here until a node with that title appears.

| Column | Type | Meaning |
|---|---|---|
| `source_id` | TEXT | The node whose body has the link, deleted with it |
| `target_key` | TEXT | The title key it waits for (section 7.2) |
| `target` | TEXT | The target as written, for display |

Primary key `(source_id, target_key)`; indexed by `target_key`.

### 5.7 history

One row per change to a node, with the whole node as it was after the change. For a delete, the
row holds the node as it was just before it was deleted.

| Column | Type | Meaning |
|---|---|---|
| `seq` | INTEGER, primary key, autoincrement | Order of the change in the space; it only grows |
| `node_id` | TEXT | The node. Not a foreign key: history outlives the node |
| `op` | TEXT | `create`, `update` or `delete` |
| `changed_at` | INTEGER | When, ms UTC |
| `changed_by` | INTEGER | The provenance of the change |
| `kind` | TEXT | The node's kind |
| `title` | TEXT | The node's title |
| `body` | TEXT | The node's Markdown body |
| `tags` | TEXT | The node's tags, a JSON array of strings, sorted |
| `change_id` | INTEGER | The change the row belongs to (schema version 3, 17.2) |
| `restored_from` | INTEGER, or NULL | The row whose content this one puts back (schema version 3, 17.2) |

Indexed by `(node_id, seq)` and, from schema version 3, `change_id`. See sections 11 and 17.

### 5.8 nodes_fts

An FTS5 table indexing `nodes.title` and `nodes.plain`, stored as an external-content table
over `nodes` (rowid `nodes.seq`), so the text is not stored twice. Triggers on `nodes` keep it
in step. See section 9.

### 5.9 properties

Added in schema version 2. Structured metadata of a node, one value per key, such as the agent
and session of a captured conversation.

| Column | Type | Meaning |
|---|---|---|
| `node_id` | TEXT | The node, deleted with it |
| `key` | TEXT | The property name (below) |
| `value` | TEXT | The value |
| `updated_at` | INTEGER | When the value was last set, ms UTC |
| `updated_by` | INTEGER | The provenance of the change that last set it |

Primary key `(node_id, key)`: a node has at most one value for each key. Indexed by
`(key, value)`, so finding the nodes with a given value, such as the conversation of one agent
session, is fast in any space.

**Keys** are lower-case ASCII letters, digits, `_`, `.` and `-`, starting with a letter, at most
64 characters. Dotted prefixes group keys by what they describe. **Values** are text; a
timestamp is written as RFC 3339 in UTC with milliseconds, such as `2026-10-20T09:30:00.000Z`,
so values of one key sort by time.

Setting a property does not change the node's `updated_at` and adds nothing to `history`;
properties carry their own provenance, like links. A reader MUST keep keys it does not know.
This version defines the keys of a `conversation` node:

| Key | Value |
|---|---|
| `conversation.agent` | The agent, as in provenance: `claude-code`, `codex` |
| `conversation.session` | The agent's id for the session |
| `conversation.project` | The folder the conversation was held in |
| `conversation.branch` | The git branch at the start |
| `conversation.model` | The last model that answered |
| `conversation.started_at` | When the conversation started |
| `conversation.ended_at` | When the conversation last had a message |

A conversation captured from an agent session is identified by `conversation.agent` and
`conversation.session` together: a writer that captures the same session again updates that
node instead of creating another one.

## 6. Markdown

The body of every node is Markdown, stored as text in `nodes.body`. The dialect is:

- [CommonMark](https://spec.commonmark.org) 0.31;
- with the [GitHub Flavored Markdown](https://github.github.com/gfm/) extensions for tables,
  strikethrough (`~~text~~`), task lists (`- [ ]`, `- [x]`) and footnotes (`[^1]`);
- with `[[links]]`, below.

GFM autolinks of bare URLs and raw HTML are allowed in a body, but carry no meaning for the
format: HTML is ignored by plain-text extraction, and a bare URL is plain text.

**`[[links]]` syntax.** A link is `[[destination]]` or `[[destination|label]]`:

| Written | Points at | Shows |
|---|---|---|
| `[[Roadmap]]` | the node titled Roadmap | Roadmap |
| `[[Roadmap\|the plan]]` | the node titled Roadmap | the plan |
| `[[Roadmap#Q4]]` | the node titled Roadmap, at its heading Q4 | Roadmap#Q4 |
| `[[01M5B425TNF334HAF0FXPS05AZ]]` | the node with that id | its current title |
| `![[Diagram]]` | the node titled Diagram, embedded | the node's content |

- The destination is everything between `[[` and the first `|` (or `]]`), trimmed. The
  label is the rest. An empty destination is not a link.
- `[[links]]` inside code spans and code blocks are text, not links.
- `[[` and `]]` follow the rules of the reference parser,
  [pulldown-cmark](https://github.com/pulldown-cmark/pulldown-cmark) 0.13 with its wiki-links
  option. An implementation in another language MUST match the
  [knode-md tests](https://github.com/knodemd/knode/tree/main/core/crates/knode-md).

## 7. Links

Links are stored by id; the text keeps showing the current title. Markdown is read when a node is
written, never when searching or browsing: search and the graph only read tables.

### 7.1 From body to links

Whenever a node's body is written (created or changed), in the same transaction:

1. Read every `[[link]]` of the body (section 6).
2. Resolve each one (7.2). The set of resolved targets is the node's new set of `mentions`
   links.
3. Delete the node's `mentions` links whose target is not in the set; insert the new ones.
   Links that stay keep their `created_at` and `created_by`.
4. Replace the node's rows in `unresolved_links` with the links that did not resolve.

A link from a node to itself is allowed.

### 7.2 Resolving a target

The **title key** of a title is the title trimmed, with runs of whitespace as one space, in
lower case by Unicode's default case mapping: `"  Café  de IDEAS"` has the key
`"café de ideas"`. It is stored in `nodes.title_key`.

A destination resolves to:

1. the node whose id it is, if it is a ULID of an existing node; else
2. the oldest node (lowest `seq`) whose `title_key` equals the destination's key; else
3. when the destination has a `#`, the oldest node whose `title_key` equals the key of the
   part before the first `#` (titles may themselves contain `#`, so the whole destination is
   tried first);
4. otherwise it does not resolve. It is stored in `unresolved_links` with the key of the part
   before the first `#`, or of the whole destination when there is none.

### 7.3 When a title appears

When a node is created, or its title changes, every row of `unresolved_links` whose
`target_key` is the node's title key becomes a `mentions` link from its `source_id` to the
node, and the row is deleted. The link's `created_by` is the provenance of the change that made
the title appear.

### 7.4 Renaming

When a node's title changes from *old* to *new*, every node with a `mentions` link to it
(other than itself) has each `[[old]]` in its body rewritten to `[[new]]`, keeping any
`#heading`, `|label` and `!`: `[[Old#Q4|see]]` becomes `[[New#Q4|see]]`. Destinations are
compared by title key. Links written by id are left as they are. Each rewritten node gets a new
`plain`, `updated_at`, `updated_by` (the provenance of the rename) and a history row; its links
do not change. This keeps every body readable as Markdown with the current titles.

### 7.5 Deleting

When a node is deleted, each node with a `mentions` link to it (other than itself) gets a row in
`unresolved_links` for the deleted title, so the link comes back if a node with that title is
created. The links and tags of the deleted node go with it (foreign keys).

## 8. Plain text

`nodes.plain` is the text of the body as a person reads it, without Markdown syntax. It is
extracted when the body is written, and is what the full-text index searches.

- Kept: the text of headings, paragraphs, list items, block quotes, table cells, footnote
  definitions, inline code and code blocks; the text of links (the label of a `[[link]]`, or its
  destination when it has none); the description of images.
- Dropped: Markdown markup, raw HTML, link and image URLs, task list markers, thematic breaks.
- Each block (paragraph, heading, list item, code block, table row, footnote definition) is one
  line; lines are joined with `\n`. Inside a block, runs of whitespace (including soft and hard
  line breaks) become one space. Table cells of a row are joined with a space. Empty lines are
  dropped.

For example:

```markdown
# Plan

Ship **v0.1**, see [[Roadmap|the roadmap]].

- [x] Spec
- [ ] Code
```

has the plain text:

```text
Plan
Ship v0.1, see the roadmap.
Spec
Code
```

## 9. Full-text search

`nodes_fts` is created as:

```sql
CREATE VIRTUAL TABLE nodes_fts USING fts5 (
    title, plain,
    content = 'nodes', content_rowid = 'seq',
    tokenize = 'unicode61 remove_diacritics 2',
    prefix = '2 3'
);
INSERT INTO nodes_fts (nodes_fts, rank) VALUES ('rank', 'bm25(5.0, 1.0)');
```

- **Tokens** are Unicode words, compared without case or diacritics: `cafe` finds `Café`.
- **Prefix indexes** of 2 and 3 characters make search-as-you-type fast.
- **Ranking** is BM25, with a match in the title weighing five times one in the plain text. The
  ranking is stored in the table (`rank` option), so `ORDER BY rank` uses it.
- **Kept in step by triggers** `nodes_fts_insert`, `nodes_fts_delete` and `nodes_fts_update`
  (after an update of `title` or `plain`), defined in the appendix. A writer MUST NOT write to
  `nodes_fts` otherwise; `INSERT INTO nodes_fts (nodes_fts) VALUES ('rebuild')` rebuilds it
  from `nodes` if ever needed.

**Querying (recommended).** The reference library turns the words people type into an FTS5
query where each word is quoted (so `AND`, `-` or `*` are words, not operators), every word is
required, and the last word is a prefix unless the query ends with a space:
`knode format` becomes `"knode" "format"*`. Results come from

```sql
SELECT rowid, snippet(nodes_fts, -1, '**', '**', '…', 16), rank
FROM nodes_fts WHERE nodes_fts MATCH ?1 ORDER BY rank LIMIT ?2
```

joined to `nodes` afterwards. Querying `nodes_fts` alone lets FTS5 sort by rank itself and run
`snippet()` only on the rows returned, which keeps a search of 10,000 nodes under 10 ms.
Snippets mark matches with `**` (Markdown bold) and cuts with `…`.

## 10. Provenance

Every write records who made it and from where:

- **Who:** `actor_kind` and `actor`. A person is `person` with their name; an agent is `agent`
  with a stable, lower-case name (`claude-code`, `codex`, `gemini-cli`, `cursor`); Knode's
  own changes (such as a migration that rewrites data) are `system` with actor `knode`.
- **From which conversation:** `session`, such as the session id of the agent conversation.
- **From where:** `source`, such as the project folder an agent worked in, a file path, or a URL.

A writer looks up the row with the same four values and inserts it if missing; nodes
(`created_by`, `updated_by`), links (`created_by`) and history (`changed_by`) point at it.

From schema version 3, an agent that writes through Knode Service is recorded with the name of
the agent its key belongs to, not a name it gives for itself (16.7), and every write is also a
row of `changes` (17.1).

## 11. History

Every change to a node adds one row to `history`, in the same transaction as the change:

- creating a node adds a `create` row with the new node;
- changing its kind, title, body or tags adds an `update` row with the node after the change,
  including bodies rewritten by a rename (7.4); a change that leaves every field as it was adds
  nothing;
- deleting it adds a `delete` row with the node as it was.

History rows are never changed. The history of a deleted node stays. In schema versions 1 and 2,
link changes made by hand are not part of the history; links carry their own provenance. From
schema version 3, every row belongs to a change, links made or removed by hand are kept in
`link_history`, and changes can be undone (section 17).

## 12. Writing a node, step by step

Each write is one SQLite transaction, so a failed write changes nothing.

**Create.**

1. Check the kind, title and tags (section 5). Make a new ULID.
2. Read the body: plain text (section 8) and `[[links]]` (section 6).
3. Find or add the provenance row.
4. Insert into `nodes` with `created_at = updated_at = now` and
   `created_by = updated_by = provenance`; insert its tags and properties.
5. Apply its links (7.1), then resolve links waiting for its title (7.3).
6. Add a `create` row to `history`.

**Update.** Compare each new field with the current one and stop if nothing changes. Write the
changed fields (a new title also writes `title_key`; a new body also writes `plain` and applies
its links), set `updated_at` and `updated_by`, add an `update` row to `history`, and on a new
title rewrite the nodes that link here (7.4) and resolve links waiting for it (7.3).

**Delete.** Turn the mentions of the node into unresolved links (7.5), add a `delete` row to
`history`, then delete the row from `nodes`.

**From schema version 3**, each of these writes also adds its row to `changes` first (17.1), and
points its `history` rows at it; a delete also adds a `link_history` row for each of the node's
links other than `mentions` (17.3).

## 13. Versions and migrations

Two numbers describe a space:

- **The format version**, `format` in `manifest.json`: the major version of this
  specification. It changes only when a reader of the old version could not safely read the
  new one. This document is format `0`.
- **The schema version**, `PRAGMA user_version` of `knode.db`: how many migrations have run.
  This document describes schema version `3`.

**Migrations** are numbered from 1. Migration *n* takes the database from schema version *n − 1*
to *n*. Version 0 is an empty database; migration 1 creates the tables of version 1, migration 2
adds `properties`, and migration 3 adds settings, folders, agents, grants and changes. All are in
the appendix.

| Migration | Schema version | What it does |
|---|---|---|
| 1 | 1 | Creates `settings`, `provenance`, `nodes`, `tags`, `links`, `unresolved_links`, `history` and `nodes_fts` |
| 2 | 2 | Adds `properties` (5.9) |
| 3 | 3 | Adds `folders` (15), `agents`, `agent_keys` and `grants` (16), `changes` and `link_history`, and `change_id` and `restored_from` to `history` (17). Makes every existing version a change of its own |

**Moving to schema version 3.** Migration 3 only adds, as the rules below require:

- Nothing written before it changes meaning. Every existing `history` row becomes one change,
  numbered like the row, with the row's time, provenance and operation (`node.create`,
  `node.update` or `node.delete`); this records what was already there, so it is not a
  `system` change.
- A space with no `space.icon`, `capture.enabled` or folders behaves as in version 2: no icon,
  capture on, captured conversations go to the default space.
- A space with no agents and no grants is invisible to every agent (16.6). An implementation
  that upgrades from a version without keys SHOULD offer the person to connect the agents
  already set up and to give them the access they had, rather than give it silently.
- Readers of schema version 2 or older refuse a space at version 3 (step 2 below), so no older
  writer can write to it without adding changes.

When a space is opened:

1. If `format` is not supported, refuse to open it.
2. If `user_version` is greater than the newest version the reader knows, refuse to open it:
   it was written by a newer version, and writing to it could lose data.
3. If `user_version` is lower, run each missing migration in order, each in its own
   transaction that also sets `user_version` to its number. A migration that fails rolls back
   and leaves the space at the version before it.
4. Check `settings.space.id` against the manifest's `id` (section 2).

Rules for later versions:

- Within format 0, a new schema version only adds: tables, columns with defaults, indexes,
  settings keys, node and link kinds. Data written by an older writer stays valid.
- A migration that must change existing data records it as provenance `system`/`knode`.
- Each new schema version is described in this document's next revision, with its migration as
  SQL, before an implementation ships it.
- Format 1.0 starts from the schema of the last format 0 version.

## 14. Reserved for later versions

These are part of the design but not of version 0 yet. Readers of version 0 MUST leave them as
they are.

| What | Where | Planned |
|---|---|---|
| Attachments by hash | `blobs/` | v0.3 |
| Sync as Loro updates | `sync/` | v0.4 |
| Embeddings for semantic search (sqlite-vec) | new tables in `knode.db` | v0.3 |
| Saved positions on the map and canvas | new tables | v0.3 |
| Encryption metadata | `manifest.json` and settings | v0.4 |

## 15. Space settings and folders

Added in schema version 3. Each space keeps its own settings and the list of folders that feed
it, so a space read on another machine, or by another implementation, behaves the same.

### 15.1 Settings

Settings live in the `settings` table (5.1). Schema version 3 defines these keys:

| Key | Value | When absent |
|---|---|---|
| `space.id` | The `id` from `manifest.json` (5.1) | Never absent |
| `space.icon` | The icon of the space: one emoji, such as `🌱` | No icon: readers show their default icon |
| `capture.enabled` | `on` or `off`: whether conversations captured from agents are written to this space at all (15.5) | `on` |

- The **name** of a space stays in `manifest.json` (section 2), so a list of spaces can be shown
  without opening each database. Renaming a space rewrites the manifest; it is not a setting.
- An absent key has the default above. A writer SHOULD NOT write a key only to store its
  default.
- A value a reader does not understand (an icon that is not one emoji, a `capture.enabled` that
  is not `on` or `off`) is treated as the default and kept as it is. For `capture.enabled`, any
  value other than `on` means the same as `off` for a writer of captured conversations: when in
  doubt, nothing is captured.
- Every change to a setting is recorded as a change (17.1) with action `settings.set`, the key
  as `subject` and `{"value": ...}` as `detail`.

### 15.2 folders

The folders on disk that feed the space: projects whose conversations are captured into it,
folders of notes it imports, and folders where an agent keeps its sessions.

| Column | Type | Meaning |
|---|---|---|
| `id` | TEXT, primary key | The folder's id, a ULID |
| `kind` | TEXT | What the folder is (below) |
| `path` | TEXT | Where it is, as recorded in 15.3 |
| `capture` | TEXT | `on` or `off`. Default `on` |
| `rules` | TEXT | A JSON object with the rules of 15.4. Default `{}` |
| `created_at` | INTEGER | When it was added, ms UTC |
| `created_by` | INTEGER | The provenance of the change that added it |
| `updated_at` | INTEGER | When it last changed, ms UTC |
| `updated_by` | INTEGER | The provenance of the last change |

`(kind, path)` is unique. Kinds follow the naming rule of node kinds (5.3). This version defines:

| Kind | Meaning | `capture = off` means |
|---|---|---|
| `project` | A folder people and agents work in, such as a code repository. Conversations held in it, or in any folder inside it, are captured into this space (15.5) | Nothing from conversations held in it is stored, in this space or any other |
| `notes` | A folder of Markdown files whose notes are imported into the space ([decision 0006](https://github.com/knodemd/handbook/blob/main/decisions/0006-markdown.md)) | The folder is not read |
| `agent` | A folder where an agent keeps its sessions, read by capture in addition to the agent's usual folders. `rules.agent` names the agent | The folder is not read |

- A reader MUST keep folders of kinds it does not know, and MUST NOT act on them.
- A folder that does not exist on the machine reading the space is kept and skipped, not an
  error. An app MAY show it as missing.
- Folders are only added, changed and removed by a person (16.6). Each such change is
  recorded as a change (17.1) with action `folder.set` or `folder.remove` and the folder's `id`
  as `subject`. Removing a folder deletes its row; nothing already captured or imported from it
  is removed.

### 15.3 Paths across machines

A space moves between machines (sync, a copy, a backup), and the same project usually sits under
a different home folder on each one. Paths are recorded so they still match there.

**Recording.** A path is first made absolute and normal: `/` separators (a Windows drive is
written `C:/...`), no `.` or `..` parts, no trailing `/`, Unicode NFC. Symbolic links are not
resolved: the path is the one the person chose. Then:

- a path inside the home folder of the person who adds it is stored relative to it, starting with
  `~/`, such as `~/Developer/knode`; the home folder itself is `~`;
- any other path is stored absolute, such as `/Volumes/Work/site`.

**Matching.** To find the folder a local path belongs to, the local path is recorded the same
way, with the local home folder. A folder matches when the recorded local path equals its `path`,
or starts with its `path` followed by `/`. The comparison is exact (case-sensitive) on the
recorded text. When several folders match, the one with the longest `path` wins.

For example, on a machine whose home is `/Users/ada`, a conversation held in
`/Users/ada/Developer/knode/core` is recorded as `~/Developer/knode/core` and belongs to the
`project` folder `~/Developer/knode`; on another machine whose home is `/home/ada`, the same
folder matches `/home/ada/Developer/knode/core`.

### 15.4 Rules

`folders.rules` is a JSON object. A reader MUST keep keys it does not know and ignore them. This
version defines:

| Key | Value | Kinds | When absent |
|---|---|---|---|
| `include` | Array of glob patterns | `notes` | `["**/*.md"]`: every Markdown file |
| `exclude` | Array of glob patterns | `project`, `notes` | `[]`. For `notes`, matching files and folders are not read. For `project`, a conversation held in a matching folder is treated as if the project's `capture` were `off` |
| `agent` | An agent name, as in provenance (`claude-code`, `codex`) | `agent` | Required: a folder of kind `agent` without it is skipped |

Patterns are matched against the path relative to the folder, with `/` separators: `*` matches
any characters except `/`, `**` matches any number of folders (including none), and `?` matches
one character. A folder `node_modules` and everything in it is excluded by `node_modules/**`.

### 15.5 Capture routing

When capture finds a conversation, it decides where it goes from the folder the conversation was
held in (for an agent session, its working directory) and the folders of every space the
implementation knows (for Knode Service, its list of spaces):

1. Find the `project` folders, in all those spaces, that match the conversation's folder (15.3),
   and keep the most specific ones: those with the longest `path`. The same path may be a
   project of more than one space.
2. If there are any, the conversation is written to each of their spaces where the folder's
   `capture` is `on`, the folder is not excluded by its `rules`, and the space's
   `capture.enabled` is `on`. If none of them qualifies, the conversation is **not stored
   anywhere**: capture off for a project means nothing of it is kept, not that it goes
   somewhere else.
3. If no `project` folder matches and the conversation was read from an `agent` folder of a
   space (with `capture` on), it goes to that space; otherwise to the default space of the
   implementation (for Knode, `default_space` in its configuration). In both cases only if that
   space's `capture.enabled` is `on`.

A conversation that is not stored MUST NOT leave anything in any space: no node, no property,
no history row. An implementation MAY keep, outside every space, how far it has read each
session, so it does not read it again.

## 16. Agents, keys and grants

Added in schema version 3. Each agent proves who it is with its own key, and each space says
what each agent may do in it: nothing, read, or write. The model and its reasons are in
[decision 0008](https://github.com/knodemd/handbook/blob/main/decisions/0008-permissions.md).

### 16.1 The model

- An **agent** is a program that reads and writes spaces through Knode Service, such as Claude
  Code connected with `knode mcp`. The same program connected twice (say, for work and for
  personal projects) is two agents, each with its own key.
- A **key** is a secret the agent presents when it connects. Spaces only hold its
  **fingerprint**; the key itself never goes into a package (16.2).
- A **grant** gives one agent one level in one space: `none`, `read` or `write`. The space is the
  one that holds the row. An agent without a grant in a space has `none` there.
- **People are not agents.** The app and the CLI act for the person who owns the spaces and are
  not limited by grants. Only a person adds agents, issues and revokes keys, and changes grants.

### 16.2 Keys

- **Making a key.** A key is 32 bytes from a cryptographically secure random generator, written
  as `knk_` followed by 64 lower-case hexadecimal digits. Knode Service makes one when a person
  connects an agent (for example with `knode connect claude-code`) and when a key is rotated.
- **Fingerprint.** The fingerprint of a key is `sha256:` followed by the lower-case hexadecimal
  SHA-256 of the key's text (UTF-8, including `knk_`). The key
  `knk_0000000000000000000000000000000000000000000000000000000000000000` has the fingerprint
  `sha256:2d8e7ee0d1fd2f7fa95d62fff7405c54a6c86833c447fe84c53f2de800dfb3a1`. A key has 256
  random bits, so a plain hash is enough to make the fingerprint useless for finding the key;
  no salt or slow hash is needed.
- **Where keys live.** A key is stored only on the machine where it was made, outside every
  package: in the system's credential store (the Keychain on macOS, the Secret Service on
  Linux), or, where there is none, in a file under `$KNODE_HOME` that only its owner can read.
  An implementation MUST NOT write a key into a package (its database, manifest, `blobs/`,
  `sync/` or any other file), into a log, or into an export. Packages are synced and shared;
  fingerprints are safe to share, keys are not.
- **Rotating a key.** A new key is made, its fingerprint added, and the old one revoked, in one
  change (17.1). The agent keeps its id, name and grants.
- **Revoking a key.** Its row gets `revoked_at` and `revoked_by`; the row is never deleted, and a
  revoked fingerprint never becomes valid again. Removing an agent is revoking all its keys and
  setting its grants to `none`; the agent's row stays, so its past changes still have a name.
- **Copying to spaces.** When an agent is given a grant in a space, Knode Service writes the
  agent's row and the fingerprints of its keys into that space. When a key is revoked, it writes
  the revocation into every space it knows that lists the key. A space that is synced or copied
  carries the revocation with it, so a key revoked on one machine is refused on every machine
  that opens the space.

### 16.3 agents

| Column | Type | Meaning |
|---|---|---|
| `id` | TEXT, primary key | The agent's id, a ULID. The same in every space the agent has a grant in |
| `name` | TEXT, unique | The agent's name, used as `provenance.actor` for its changes (16.7). Follows the naming rule of node kinds (5.3), up to 64 characters. Never changes |
| `kind` | TEXT | The program, as in provenance: `claude-code`, `codex`, `gemini-cli` |
| `created_at` | INTEGER | When it was added to this space, ms UTC |
| `created_by` | INTEGER | The provenance of the person who added it |

The first agent of a kind is usually named after it (`claude-code`); a second one gets another
name (`claude-code-work`). Rows are never deleted.

### 16.4 agent_keys

| Column | Type | Meaning |
|---|---|---|
| `fingerprint` | TEXT, primary key | The key's fingerprint (16.2) |
| `agent_id` | TEXT | The agent the key belongs to |
| `created_at` | INTEGER | When the key was made, ms UTC |
| `created_by` | INTEGER | The provenance of the person who connected the agent |
| `revoked_at` | INTEGER, or NULL | When the key was revoked, ms UTC; NULL while it is valid |
| `revoked_by` | INTEGER, or NULL | The provenance of the person who revoked it |

Indexed by `agent_id`. An agent may have more than one valid key, such as one per machine.

### 16.5 grants

| Column | Type | Meaning |
|---|---|---|
| `agent_id` | TEXT, primary key | The agent |
| `level` | TEXT | `none`, `read` or `write` |
| `updated_at` | INTEGER | When the level was set, ms UTC |
| `updated_by` | INTEGER | The provenance of the person who set it |

No row and `none` mean the same; a row with `none` records that access was taken away. `write`
includes `read`.

### 16.6 What Knode Service checks

**When an agent connects**, it presents its key. Knode Service computes the fingerprint and looks
for it among the agents of its machine. A connection with no key, or with a key it does not
know, is refused: the agent is told to connect with `knode connect`.

**On every call**, not once per connection, so that a revocation or a lower grant takes effect
on the next call, and for every space the call names, lists or searches:

1. The fingerprint MUST be in the space's `agent_keys`, for the same agent, with `revoked_at`
   NULL. Otherwise the agent's level in that space is `none`.
2. The level is the agent's row in `grants`, or `none` when there is none.
3. With `none`, the space is **invisible**: it is left out of every list, search and
   notification, and a call that names it fails exactly as for a space that does not exist,
   with the same error and message. An agent cannot tell a space it may not see from one that
   is not there.
4. With `read`, calls that read are allowed and calls that write fail with a permission error.
5. With `write`, calls that read and calls that write are allowed.

| Level | Allowed calls (socket methods of Knode Service, and the MCP tools on top of them) |
|---|---|
| `none` | Nothing. The space is invisible |
| `read` | `node.get`, `links.from`, `links.back`, `search`, `recent`, `history`, reading properties, and `subscribe` notifications for the space |
| `write` | Everything `read` allows, plus `node.create`, `node.update`, `node.delete`, `link.create`, `link.delete`, setting and removing properties, undo (17.4) and restore (17.5) |

Whatever its level, an agent MUST NOT be able to create a space, change settings (15.1) or
folders (15.2), add agents, issue or revoke keys, change grants, or purge history (17.7). These
are for people only.

### 16.7 Provenance of agents

A change made by an agent is recorded with `actor_kind = agent` and, as `actor`, the `name` of
the agent its key belongs to (16.3), never a name the client gives for itself. `session` and
`source` are filled as before (section 10). A provenance row whose `actor_kind` is `agent` and
whose `actor` equals an agent's `name` belongs to that agent, including rows written before the
agent had a key: an agent named `claude-code` owns the changes `claude-code` made before schema version 3.

### 16.8 What is logged

- **In the space:** every write is a change (17.1) with the provenance of whoever made it, and
  every change to agents, keys and grants is a change too: `agent.add` (subject: the agent id),
  `agent.key.add` and `agent.key.revoke` (subject: the fingerprint), and `grant.set` (subject:
  the agent id, detail: `{"level": "read", "previous": "none"}`).
- **On the machine, outside every space:** Knode Service keeps an access log under
  `$KNODE_HOME`, with one entry per call: the time, the agent's id, the space's id, the method,
  and the outcome (`allowed`, `denied` or `invisible`). The log MUST NOT contain keys, node
  content or search queries. Reads are not recorded in the space: it is synced and shared, and a
  record of every read would grow without end.

### 16.9 What grants do not do

Grants decide what an agent can do **through Knode Service**. They are not a sandbox: a program
that runs as the person can read the package directly, or run the `knode` CLI, which acts for the
person. Grants keep each agent to the spaces it was given and record what it did; keeping a
program away from the files is the operating system's job. Whoever can write to a package can
also change its grants; a package is protected by who can reach it, and, from v0.4, by
encryption.

## 17. Change history and undo

Added in schema version 3. Every change to a node or a link is kept, with who made it and when,
so any of them can be undone and traced. Undo never deletes history: it is one more change.

### 17.1 changes

One row per change: one write that a person or an agent asked for, made in one transaction.
Every row that a change adds to `history` and `link_history` points at it.

| Column | Type | Meaning |
|---|---|---|
| `id` | INTEGER, primary key, autoincrement | Order of the change in the space; it only grows |
| `changed_at` | INTEGER | When, ms UTC |
| `changed_by` | INTEGER | The provenance of the change |
| `action` | TEXT | What was done (below) |
| `subject` | TEXT, `''` for none | What it was done to: a node id, a folder id, an agent id, a setting key, a fingerprint |
| `undoes` | INTEGER, or NULL | For an `undo`, the change it undoes |
| `detail` | TEXT | A JSON object with details of the action. Default `{}` |

Indexed by `(subject, id)`, `undoes` and `changed_at`. Actions use the naming rule of kinds with
`.` between parts. This version defines:

| Action | Subject | Made when |
|---|---|---|
| `node.create` | The node | A node is created (with its properties, links and tags) |
| `node.update` | The node | A node's kind, title, body or tags change. A rename's rewritten bodies (7.4) are part of the same change |
| `node.delete` | The node | A node is deleted |
| `node.restore` | The node | A node is put back to an earlier version, or recovered after a delete (17.5) |
| `node.purge` | The node | A node and its history are erased (17.7) |
| `link.create` | The source node | A link is made by hand |
| `link.delete` | The source node | A link is removed by hand |
| `undo` | The subject of the undone change | A change is undone (17.4) |
| `import` | `''` | Many nodes are written at once, such as an import of Markdown files |
| `settings.set` | The key | 15.1 |
| `folder.set`, `folder.remove` | The folder | 15.2 |
| `agent.add`, `agent.key.add`, `agent.key.revoke`, `grant.set` | 16.8 | 16.8 |
| `migrate` | `''` | A migration rewrites data (section 13), with provenance `system`/`knode` |

A reader MUST keep and show actions it does not know. Setting or removing properties alone adds
no change (5.9): properties carry their own provenance, like links did before this version.

A writer at schema version 3 MUST add exactly one `changes` row for each write in this table, in
the same transaction as the write, and MUST point every `history` and `link_history` row the
write adds at it.

### 17.2 history, from schema version 3

`history` (5.7) gains two columns:

| Column | Type | Meaning |
|---|---|---|
| `change_id` | INTEGER | The change the row belongs to. NULL only in rows written before schema version 3, and migration 3 fills those |
| `restored_from` | INTEGER, or NULL | When the row puts back an earlier version (17.4, 17.5), the `seq` of the row whose content it restores |

Each row still holds the node **after** the change (or, for `delete`, just before it). The
**previous value** of a row is the row before it for the same node (the highest lower `seq` with
the same `node_id`); a `create` has none. `changed_at` and `changed_by` stay, equal to those of
the change.

### 17.3 link_history

Every link made or removed by hand, kept like the versions of nodes.

| Column | Type | Meaning |
|---|---|---|
| `seq` | INTEGER, primary key, autoincrement | Order of the row in the space |
| `change_id` | INTEGER | The change |
| `op` | TEXT | `create` or `delete` |
| `source_id` | TEXT | The link's source. Not a foreign key: history outlives the node |
| `kind` | TEXT | The link's kind |
| `target_id` | TEXT | The link's target. Not a foreign key |

Indexed by `change_id`, `(source_id, seq)` and `(target_id, seq)`.

- Only links of kinds other than `mentions` are recorded. `mentions` links come from bodies
  (section 7), and every body is already in `history`, so they can always be computed again.
- When a node is deleted, each of its links other than `mentions`, in either direction, gets a
  `delete` row in the same change, so the delete can be undone with its links.

### 17.4 Undo

To **undo** a change C is to make a new change U, with action `undo`, `undoes` set to C's `id`,
and the provenance of whoever asked for it, that puts back everything C touched as it was just
before C:

- **Each node C has a `history` row for.** Its version before C is its latest row with a lower
  `seq` than C's first row for it.
  - C created the node (no earlier row, or the earlier row is a `delete`): U deletes it.
  - C deleted the node: U creates it again, with the same id, from C's `delete` row.
  - Otherwise: U updates it to its version before C.
  - The rows U adds that put back a version have `restored_from` set to the row they put
    back; a `delete` row added by U has none.
- **Each `link_history` row of C**, the other way round: a link C created is deleted if it
  still exists; a link C deleted is made again if both nodes exist, with the provenance of U.
- **Nodes are written as in section 12**, with one exception: a title put back by an undo does not
  rewrite the bodies that link to it (7.4), because the bodies C rewrote are put back from their
  own versions. Links from bodies (7.1) and links waiting for a title (7.3) are worked out as
  for any write.

Rules:

- **Conflicts.** If a node or link that C touched has been changed by a later change, the undo
  would overwrite someone's later work. A writer MUST NOT apply it without the person's
  confirmation; to an agent it fails with an error that names the later changes. Restoring a
  version (17.5) is the way to go back past later changes on purpose.
- **Once.** A change that has an undo (a change whose `undoes` is its id) that has not itself
  been undone MUST NOT be undone again. Undoing the undo redoes the change.
- **What can be undone:** changes with actions `node.create`, `node.update`, `node.delete`,
  `node.restore`, `link.create`, `link.delete`, `undo` and `import`. Any other change MUST NOT
  be undone through undo; settings, folders, agents, keys and grants are simply changed again.
- **Nothing is removed.** Undo never deletes or changes rows of `changes`, `history` or
  `link_history`.

### 17.5 Restoring a version

To put a node back to an earlier version R (a row of its `history`) is to make a change with
action `node.restore` and the node as `subject`, which writes R's kind, title, body and tags as
an `update` row (a `create` row if the node is deleted now), with `restored_from` set to R's
`seq`. It is written as an ordinary update: a restored title renames, and rewrites the bodies
that link to the node (7.4).

**Recovering a deleted node** is restoring it to its `delete` row. The node keeps its id, so links
written by id and `[[links]]` waiting for its title come back (7.3). The links other than
`mentions` that the delete removed (the `delete` rows of `link_history` in the change that deleted
it) are made again when both nodes exist.

### 17.6 Who changed what

The history of a node is its `history` rows joined to `changes` and `provenance`, and, for an
agent, to `agents` by name (16.7). For example, Ada writes a note, `claude-code` adds a line
in a session, and Ada undoes it:

`changes`:

| id | changed_by | action | subject | undoes |
|---|---|---|---|---|
| 41 | person `Ada` | `node.create` | `01M5B425TNF334HAF0FXPS05AZ` | |
| 42 | agent `claude-code`, session `5f1c...` | `node.update` | `01M5B425TNF334HAF0FXPS05AZ` | |
| 43 | person `Ada` | `undo` | `01M5B425TNF334HAF0FXPS05AZ` | 42 |

`history` of the node (`kind`, `title` and `tags` left out):

| seq | change_id | op | body | restored_from |
|---|---|---|---|---|
| 57 | 41 | `create` | `Beds: tomato, basil.` | |
| 58 | 42 | `update` | `Beds: tomato, basil. Add mint.` | |
| 59 | 43 | `update` | `Beds: tomato, basil.` | 57 |

The previous value of row 58 is row 57; row 59 says it puts back row 57, as part of a change that
undoes change 42 and was made by Ada.

### 17.7 Retention

- **Everything is kept by default.** A writer MUST NOT delete or change rows of `changes`,
  `history` or `link_history`, except to purge.
- **Purge.** A person, never an agent, MAY erase a node and all of its versions for good, for
  example after pasting a secret into it. Purging deletes the node if it exists, its rows in
  `history`, and the rows of `link_history` where it is the source or the target, and adds a
  change with action `node.purge` and the node as `subject`, holding no content. The `changes`
  rows of earlier changes stay: they hold no content. A purged node cannot be restored, and an
  undo that would need its versions MUST be refused.
- **No compaction in this version.** Later versions may define how old versions are compacted;
  until then a space keeps every version of every node.

## Appendix: the schema

Schema version 3: migrations 1, 2 and 3, in order, exactly as written here.

Migration 1:

```sql

CREATE TABLE settings (
    key   TEXT PRIMARY KEY NOT NULL,
    value TEXT NOT NULL
) STRICT;

CREATE TABLE provenance (
    id         INTEGER PRIMARY KEY,
    actor_kind TEXT NOT NULL CHECK (actor_kind IN ('person', 'agent', 'system')),
    actor      TEXT NOT NULL,
    session    TEXT NOT NULL DEFAULT '',
    source     TEXT NOT NULL DEFAULT '',
    UNIQUE (actor_kind, actor, session, source)
) STRICT;

CREATE TABLE nodes (
    seq        INTEGER PRIMARY KEY,
    id         TEXT NOT NULL UNIQUE CHECK (length(id) = 26),
    kind       TEXT NOT NULL DEFAULT 'note',
    title      TEXT NOT NULL,
    title_key  TEXT NOT NULL,
    body       TEXT NOT NULL DEFAULT '',
    plain      TEXT NOT NULL DEFAULT '',
    created_at INTEGER NOT NULL,
    updated_at INTEGER NOT NULL,
    created_by INTEGER NOT NULL REFERENCES provenance (id),
    updated_by INTEGER NOT NULL REFERENCES provenance (id)
) STRICT;

CREATE INDEX nodes_title_key ON nodes (title_key);
CREATE INDEX nodes_updated_at ON nodes (updated_at);
CREATE INDEX nodes_kind ON nodes (kind, updated_at);

CREATE TABLE tags (
    node_id TEXT NOT NULL REFERENCES nodes (id) ON DELETE CASCADE,
    tag     TEXT NOT NULL,
    PRIMARY KEY (node_id, tag)
) STRICT, WITHOUT ROWID;

CREATE INDEX tags_tag ON tags (tag);

CREATE TABLE links (
    source_id  TEXT NOT NULL REFERENCES nodes (id) ON DELETE CASCADE,
    kind       TEXT NOT NULL,
    target_id  TEXT NOT NULL REFERENCES nodes (id) ON DELETE CASCADE,
    created_at INTEGER NOT NULL,
    created_by INTEGER NOT NULL REFERENCES provenance (id),
    PRIMARY KEY (source_id, kind, target_id)
) STRICT, WITHOUT ROWID;

CREATE INDEX links_target ON links (target_id, kind);

CREATE TABLE unresolved_links (
    source_id  TEXT NOT NULL REFERENCES nodes (id) ON DELETE CASCADE,
    target_key TEXT NOT NULL,
    target     TEXT NOT NULL,
    PRIMARY KEY (source_id, target_key)
) STRICT, WITHOUT ROWID;

CREATE INDEX unresolved_links_target ON unresolved_links (target_key);

CREATE TABLE history (
    seq        INTEGER PRIMARY KEY AUTOINCREMENT,
    node_id    TEXT NOT NULL,
    op         TEXT NOT NULL CHECK (op IN ('create', 'update', 'delete')),
    changed_at INTEGER NOT NULL,
    changed_by INTEGER NOT NULL REFERENCES provenance (id),
    kind       TEXT NOT NULL,
    title      TEXT NOT NULL,
    body       TEXT NOT NULL,
    tags       TEXT NOT NULL DEFAULT '[]'
) STRICT;

CREATE INDEX history_node ON history (node_id, seq);

CREATE VIRTUAL TABLE nodes_fts USING fts5 (
    title,
    plain,
    content = 'nodes',
    content_rowid = 'seq',
    tokenize = 'unicode61 remove_diacritics 2',
    prefix = '2 3'
);

-- Rank by BM25, a match in the title weighing five times one in the text.
INSERT INTO nodes_fts (nodes_fts, rank) VALUES ('rank', 'bm25(5.0, 1.0)');

CREATE TRIGGER nodes_fts_insert AFTER INSERT ON nodes BEGIN
    INSERT INTO nodes_fts (rowid, title, plain) VALUES (new.seq, new.title, new.plain);
END;

CREATE TRIGGER nodes_fts_delete AFTER DELETE ON nodes BEGIN
    INSERT INTO nodes_fts (nodes_fts, rowid, title, plain)
    VALUES ('delete', old.seq, old.title, old.plain);
END;

CREATE TRIGGER nodes_fts_update AFTER UPDATE OF title, plain ON nodes BEGIN
    INSERT INTO nodes_fts (nodes_fts, rowid, title, plain)
    VALUES ('delete', old.seq, old.title, old.plain);
    INSERT INTO nodes_fts (rowid, title, plain) VALUES (new.seq, new.title, new.plain);
END;
```

Migration 2:

```sql
CREATE TABLE properties (
    node_id    TEXT NOT NULL REFERENCES nodes (id) ON DELETE CASCADE,
    key        TEXT NOT NULL,
    value      TEXT NOT NULL,
    updated_at INTEGER NOT NULL,
    updated_by INTEGER NOT NULL REFERENCES provenance (id),
    PRIMARY KEY (node_id, key)
) STRICT, WITHOUT ROWID;

CREATE INDEX properties_key_value ON properties (key, value);
```

Migration 3:

```sql
-- Space settings and folders (section 15).
CREATE TABLE folders (
    id         TEXT PRIMARY KEY NOT NULL CHECK (length(id) = 26),
    kind       TEXT NOT NULL,
    path       TEXT NOT NULL,
    capture    TEXT NOT NULL DEFAULT 'on' CHECK (capture IN ('on', 'off')),
    rules      TEXT NOT NULL DEFAULT '{}',
    created_at INTEGER NOT NULL,
    created_by INTEGER NOT NULL REFERENCES provenance (id),
    updated_at INTEGER NOT NULL,
    updated_by INTEGER NOT NULL REFERENCES provenance (id),
    UNIQUE (kind, path)
) STRICT;

-- Agents, keys and grants (section 16). Keys themselves are never stored here.
CREATE TABLE agents (
    id         TEXT PRIMARY KEY NOT NULL CHECK (length(id) = 26),
    name       TEXT NOT NULL UNIQUE,
    kind       TEXT NOT NULL,
    created_at INTEGER NOT NULL,
    created_by INTEGER NOT NULL REFERENCES provenance (id)
) STRICT;

CREATE TABLE agent_keys (
    fingerprint TEXT PRIMARY KEY NOT NULL
                CHECK (length(fingerprint) = 71 AND substr(fingerprint, 1, 7) = 'sha256:'),
    agent_id    TEXT NOT NULL REFERENCES agents (id),
    created_at  INTEGER NOT NULL,
    created_by  INTEGER NOT NULL REFERENCES provenance (id),
    revoked_at  INTEGER,
    revoked_by  INTEGER REFERENCES provenance (id)
) STRICT;

CREATE INDEX agent_keys_agent ON agent_keys (agent_id);

CREATE TABLE grants (
    agent_id   TEXT PRIMARY KEY NOT NULL REFERENCES agents (id),
    level      TEXT NOT NULL CHECK (level IN ('none', 'read', 'write')),
    updated_at INTEGER NOT NULL,
    updated_by INTEGER NOT NULL REFERENCES provenance (id)
) STRICT;

-- Change history and undo (section 17).
CREATE TABLE changes (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    changed_at INTEGER NOT NULL,
    changed_by INTEGER NOT NULL REFERENCES provenance (id),
    action     TEXT NOT NULL,
    subject    TEXT NOT NULL DEFAULT '',
    undoes     INTEGER REFERENCES changes (id),
    detail     TEXT NOT NULL DEFAULT '{}'
) STRICT;

CREATE INDEX changes_subject ON changes (subject, id);
CREATE INDEX changes_undoes ON changes (undoes) WHERE undoes IS NOT NULL;
CREATE INDEX changes_changed_at ON changes (changed_at);

ALTER TABLE history ADD COLUMN change_id INTEGER REFERENCES changes (id);
ALTER TABLE history ADD COLUMN restored_from INTEGER REFERENCES history (seq);

CREATE INDEX history_change ON history (change_id);

CREATE TABLE link_history (
    seq       INTEGER PRIMARY KEY AUTOINCREMENT,
    change_id INTEGER NOT NULL REFERENCES changes (id),
    op        TEXT NOT NULL CHECK (op IN ('create', 'delete')),
    source_id TEXT NOT NULL,
    kind      TEXT NOT NULL,
    target_id TEXT NOT NULL
) STRICT;

CREATE INDEX link_history_change ON link_history (change_id);
CREATE INDEX link_history_source ON link_history (source_id, seq);
CREATE INDEX link_history_target ON link_history (target_id, seq);

-- Every version written before schema version 3 becomes a change of its own,
-- with the same number as its history row.
INSERT INTO changes (id, changed_at, changed_by, action, subject)
SELECT seq, changed_at, changed_by, 'node.' || op, node_id FROM history ORDER BY seq;

UPDATE history SET change_id = seq;
```
