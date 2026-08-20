# Memory

A tool for storing the agent's persistent notes, distinct from project files and from the chat history: notes live on disk between sessions, have a scope and a classification, are indexed for search, and are automatically prefetched into the start of every session. One tool with five operations distinguished by the `operation` field: `read`, `write`, `list`, `delete`, `search`. Every operation takes the `scope` field.

Scopes:

| scope | location on disk | meaning |
|---|---|---|
| project | `<project_root>/.kot/agent-memory` | project notes; shared through version control; used by default |
| user | `<config_home>/memory` | machine-wide notes, shared across all projects and sessions |
| local | `KOT_REMOTE_MEMORY_DIR`, or `<project_root>/.kot/agent-memory-local` | notes of this project on this machine; so that they do not enter version control, the directory is added to the repository ignore rules |

## Operations

### read

Reads a single note by its slug name in the given scope.

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| name | string | yes | — | the slug name of the note (the file stem, without the `.md` extension) |
| scope | string | no | `project` | the scope |

Returns the full record: `name`, `title`, an optional `description`, `kind`, `body` (the raw markdown body), `scope`, and `path` (the path on disk). A missing note is a soft "not found" result, not an error.

### write

Creates or overwrites a note and its line in the index. A write to an existing slug overwrites the note.

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| name | string | yes | — | the slug name of the note (the file stem) |
| content | string | yes | — | the note body in markdown; minimum 1 non-empty character |
| description | string | no | absent | a one-line description (the hook for the index and the prefetch) |
| kind | string | no | `other` | the classification: `user`, `feedback`, `project`, `reference`, `other` |
| scope | string | no | `project` | the scope |

Returns `name`, `scope`, `path`, `outcome` (`created` for a new note, `updated` on overwrite), and `index_updated` (always `true`).

Slug constraint: the name must be a single stem of the characters `[A-Za-z0-9._-]`, must not be empty, must not be `.` or `..`, must not equal the name `MEMORY` (the reserved index), and must not contain path separators, `:`, or characters that break the index parsing. An invalid slug is rejected at the validation stage.

`description` is collapsed into a single line (any whitespace characters are replaced with a single space); an empty or whitespace-only description is not written at all. There is no separate display title — the slug itself serves as the title.

### list

Lists the index (the list of hook lines) of the given scope.

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| scope | string | no | `project` | the scope |

Returns `scope` and `entries` — a list of index lines, each with the fields `title`, `file`, and `hook`.

### delete

Deletes a note by slug and removes its line from the index.

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| name | string | yes | — | the slug name of the note to delete |
| scope | string | no | `project` | the scope |

Returns `name`, `scope`, `removed` (`false` if the note did not exist — a no-op), and `index_updated` (whether the index changed).

### search

Searches the notes of the given scope by BM25 relevance.

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| query | string | yes | — | a free-text query; minimum 1 non-empty character |
| scope | string | no | `project` | the scope |
| max_results | integer or string | no | 100 | the limit on the number of returned matches |

Returns `scope`, `query`, and `hits` — matches in descending order of relevance. Each match carries `name`, `title`, `kind`, `snippet` (a fragment of the body around the match), and `score` (the BM25 score).

## Behavior and limitations

- Each note is a file `<scope_dir>/<slug>.md` with frontmatter and a markdown body. The frontmatter stores the fields `name` (the title), an optional `description`, and `metadata.type` (the classification). Writes and overwrites are atomic (a temporary file + rename).
- The scope index is the file `<scope_dir>/MEMORY.md` with lines of the form `- [title](slug.md) — hook`. The hook (`hook`) is the `description` if it is set and non-empty, otherwise the title. A write adds or replaces the index line; a delete removes it. Lines that do not match the index format are dropped when the index is read.
- BM25 search: term frequency saturation `k1 = 1.2`, document length normalization `b = 0.75`, the weight of the metadata field (title + hook) relative to the body is `2.0`. Terms are tokenized as maximal sequences of Unicode alphanumeric characters or `_`, everything else is a separator. Cyrillic and other alphabets are handled; continuous CJK text without separators forms a single term-sequence (a substring within it does not match).
- Ranking: any query term matches (OR semantics); repeated query terms do not multiply the score. Matches with a score below 15% of the best result's score are dropped; the best result is always kept. On equal scores the index order is preserved. The hard cap on returned matches is 100.
- Records are read and overwritten in a serialized manner per scope directory: a lock protects both the note files and the shared index, including across different sessions and processes working with the same scope.
- Prefetch: at session start, the notes are fed to the model as a system message, without calling the `list`/`read` operations. The `user` scope is fed with full note bodies and without a volume limit. The `project` and `local` scopes are fed only with hook lines — the body of such a note is read on demand by the `read` operation. A failure to load one scope does not interrupt session start.
- The maximum result size of an operation is 25 000 characters. The stored notes are not modified by this.
- Content rules: store only knowledge worth carrying into a future session. It is forbidden to store secrets, progress and status logs, transient facts, and what is cheaply recoverable from code, git, or the current conversation. For the `user` scope — only universal behavioral rules that apply in any session and project, with the `feedback` classification. Pick the narrowest suitable scope and verify recalled facts against current sources.

## Examples

Writing a project note:

```json
{"operation": "write", "scope": "project", "name": "build-commands", "description": "Project build and gate commands", "kind": "project", "content": "Build: cargo build. Gate: cargo test && cargo clippy."}
```

Reading a note:

```json
{"operation": "read", "scope": "project", "name": "build-commands"}
```

Listing the scope index:

```json
{"operation": "list", "scope": "user"}
```

Searching by query:

```json
{"operation": "search", "scope": "project", "query": "build gate", "max_results": 5}
```

Deleting a note:

```json
{"operation": "delete", "scope": "project", "name": "build-commands"}
```
