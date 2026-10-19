# The .knode format - version 0

**Status:** draft. Version 0 can still change; it is frozen as 1.0 before Knode's public beta.
The reference implementation is
[knode-format](https://github.com/knodemd/knode/tree/main/core/crates/knode-format), with
[knode-md](https://github.com/knodemd/knode/tree/main/core/crates/knode-md) for the Markdown rules.

A `.knode` file is one **space**: a local memory shared by a person and their AI agents. It holds
**nodes** (notes, ideas, decisions, captured conversations), the **links** between them, who wrote
each one and from where (**provenance**), every earlier version (**history**), and a full-text
index to search them.

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
15. [Appendix: the schema](#appendix-the-schema)

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
`1` for the schema in this document. See section 13.

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

The full SQL is in the [appendix](#appendix-the-schema). A writer MUST create exactly that schema
for version 1. A reader MUST NOT depend on the order of columns; it reads them by name.

### 5.1 settings

| Column | Type | Meaning |
|---|---|---|
| `key` | TEXT, primary key | A dotted name |
| `value` | TEXT | The value |

Defined keys:

| Key | Value |
|---|---|
| `space.id` | The `id` from `manifest.json`. Written when the space is created; never changes |

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

**Metadata is never in the body.** Dates, authors, sources, tags and permissions live in columns
and tables. A body MUST NOT be given YAML frontmatter by a writer; frontmatter is only produced
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

Indexed by `(node_id, seq)`. See section 11.

### 5.8 nodes_fts

An FTS5 table indexing `nodes.title` and `nodes.plain`, stored as an external-content table
over `nodes` (rowid `nodes.seq`), so the text is not stored twice. Triggers on `nodes` keep it
in step. See section 9.

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

## 11. History

Every change to a node adds one row to `history`, in the same transaction as the change:

- creating a node adds a `create` row with the new node;
- changing its kind, title, body or tags adds an `update` row with the node after the change,
  including bodies rewritten by a rename (7.4); a change that leaves every field as it was adds
  nothing;
- deleting it adds a `delete` row with the node as it was.

History rows are never changed. The history of a deleted node stays. Link changes made by hand
are not part of the history in this version; links carry their own provenance.

## 12. Writing a node, step by step

Each write is one SQLite transaction, so a failed write changes nothing.

**Create.**

1. Check the kind, title and tags (section 5). Make a new ULID.
2. Read the body: plain text (section 8) and `[[links]]` (section 6).
3. Find or add the provenance row.
4. Insert into `nodes` with `created_at = updated_at = now` and
   `created_by = updated_by = provenance`; insert its tags.
5. Apply its links (7.1), then resolve links waiting for its title (7.3).
6. Add a `create` row to `history`.

**Update.** Compare each new field with the current one and stop if nothing changes. Write the
changed fields (a new title also writes `title_key`; a new body also writes `plain` and applies
its links), set `updated_at` and `updated_by`, add an `update` row to `history`, and on a new
title rewrite the nodes that link here (7.4) and resolve links waiting for it (7.3).

**Delete.** Turn the mentions of the node into unresolved links (7.5), add a `delete` row to
`history`, then delete the row from `nodes`.

## 13. Versions and migrations

Two numbers describe a space:

- **The format version**, `format` in `manifest.json`: the major version of this
  specification. It changes only when a reader of the old version could not safely read the
  new one. This document is format `0`.
- **The schema version**, `PRAGMA user_version` of `knode.db`: how many migrations have run.
  This document describes schema version `1`.

**Migrations** are numbered from 1. Migration *n* takes the database from schema version *n − 1*
to *n*. Version 0 is an empty database; migration 1 creates the schema in the appendix.

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

These are part of the design but not of version 0. Readers of version 0 MUST leave them as they
are.

| What | Where | Planned |
|---|---|---|
| Attachments by hash | `blobs/` | v0.3 |
| Sync as Loro updates | `sync/` | v0.4 |
| Embeddings for semantic search (sqlite-vec) | new tables in `knode.db` | v0.3 |
| Folders that feed a space; capture off per project | new tables and settings | v0.2 |
| Permissions per agent: read, write or nothing | new tables | v0.2 |
| Saved positions on the map and canvas | new tables | v0.3 |
| Encryption metadata | `manifest.json` and settings | v0.4 |

## Appendix: the schema

Schema version 1, exactly as migration 1 creates it.

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
