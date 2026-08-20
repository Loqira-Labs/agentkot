# Files

Reading and modifying files through a private virtual layer. The tool works with text, binary data, images, notebooks, code, and directories; changes first enter the virtual layer and are transferred to disk only by the sync operation.

## Operations

The operation is selected by the `operation` field. If the field is omitted, the operation is inferred from the set of parameters: the presence of any of `content`, `content_base64`, `autocreate` means `write`; otherwise the presence of any of `old_string`, `new_string`, `replace_all`, `cell_id`, `cell_type`, `cell_edit_mode` means `edit`; otherwise `read`. An unknown `operation` value returns an error.

### read

Reads a file or directory. The result format depends on the path type: text file, image, notebook, code, or a list of directory entries. A missing file is returned as a soft result (not an error), so existence does not need to be checked in advance.

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| file_path | string | yes | — | path to the file or directory; a relative path resolves against the session working directory |
| offset | integer or string | no | 1 | line number to start reading from (1-indexed); 0 means 1 |
| limit | integer or string | no | — | maximum number of lines to read; when set, disables the byte read limit |
| virtual | boolean or string | no | true | read the unsynced version from the virtual layer; false reads the real file on disk |
| revirtual | boolean or string | no | false | only with `virtual: false`; drops the unsynced virtual-layer entry for this path |
| depth | integer or string | no | 1 | recursion depth when reading a directory; maximum 64 |
| view | string | no | skeleton for code | full, skeleton, or symbols (for code files) |
| symbol | string | no | — | name of a code-file symbol to read in full |

A text file is returned with line numbering: each line is prefixed with a right-aligned number (width at least 6 characters) and a tab character. The result contains the path, the content, the number of lines in the slice, the number of the first line of the slice, and the total number of lines in the file. An empty file returns a message that the file is empty; an offset beyond the end of the file returns an out-of-range message.

Reading a code file without a line range returns the structure (skeleton) by default: signatures with bodies collapsed into `{ … }`, with line numbering. `view: "symbols"` returns a flat list of symbols, `view: "full"` returns verbatim content, `symbol` returns the verbatim content of a single symbol. When `offset`/`limit` are specified the file is read verbatim. Recognized extensions: rust, python, javascript, typescript, tsx, go, java, c, c++.

Reading a directory at `depth` 1 (default) returns one level: subdirectories first, then files, each sorted by name. At `depth > 1` a flat list across the whole subtree is returned, sorted by relative path. The result is limited to 1000 entries (a truncation flag is set when exceeded). Each entry carries the name, the directory flag, the size in bytes (absent for a directory), presence (on disk, only in the virtual layer, in both), the unsynced-change flag, the has-children flag, and (for a recursive list) the path relative to the directory.

Reading an image returns the content in base64 and the MIME type. The image side is limited to 2000 pixels: an excess is scaled down preserving proportions; an image with an aspect ratio of 3 or more is sliced into square tiles along the short side. The MIME type is determined from the data header, not from the extension. An empty image file (0 bytes) is rejected.

Reading a `.ipynb` notebook returns the cells as `<cell id="…" type="code|markdown">` tags. The notebook is read in full and is limited to 256 KB.

Reading a path marked for deletion returns the soft result "deletion pending"; reading with `virtual: false` shows the on-disk copy.

### write

Creates a new file or overwrites an existing one. The content enters the virtual layer, not the disk.

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| file_path | string | yes | — | path to write to |
| content | string | one of the two | — | complete text content |
| content_base64 | string | one of the two | — | base64-encoded binary content |
| autocreate | boolean or string | no | true | create missing parent directories |

`content` and `content_base64` are mutually exclusive; exactly one of them is required. Binary data is written byte-for-byte without conversion. Creating a new file does not require a prior read; overwriting an existing text file requires a prior full read (partial or structural read is rejected). Writing text normalizes line endings to LF; new files receive UTF-8 encoding without BOM and LF line endings.

### edit

Replaces an exact string in an existing file. For `.ipynb`, cell editing is performed.

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| file_path | string | yes | — | path to modify |
| old_string | string | yes (except .ipynb) | — | exact string to find and replace |
| new_string | string | yes | — | replacement string; must differ from old_string |
| replace_all | boolean or string | no | false | replace all occurrences |
| cell_id | string | no (yes for replace/delete in .ipynb) | — | cell identifier or the short form `cell-N` |
| cell_type | string | no (yes for insert) | — | `code` or `markdown` |
| cell_edit_mode | string | no | replace | `replace`, `insert`, or `delete` |

String matching is checked without regard to typographic quotes (a curly quote matches a straight quote). If no occurrence is found, an error is returned; if there are multiple occurrences without `replace_all`, an error is returned indicating the number of occurrences. `old_string == new_string` is rejected. An empty `old_string` creates a new file if the file does not exist, or inserts the content into an existing empty file; for a non-empty existing file an error is returned. The edit preserves the detected line endings of the file.

In a notebook, `insert` requires `cell_type`; `replace` and `delete` require `cell_id` (an identifier or `cell-N`). For `insert` without `cell_id` the new cell is inserted at position 0, otherwise after the specified cell. `replace` with `cell-N` where N equals the number of cells appends a cell at the end. A new cell receives a 13-character identifier; a code cell receives `execution_count: null` and an empty `outputs` list. The notebook is serialized with 1-space indentation. Notebook errors (invalid JSON, cell not found) are returned as a result with an error flag.

### delete

Marks a file for deletion in the virtual layer. The file is not removed from disk until sync.

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| file_path | string | yes | — | path of the file to delete |

Deleting a directory is rejected. The result reports what happened: marked for deletion (the file will be removed at sync), a new-file entry reset (the file did not exist on disk), or the file absent everywhere (a soft result).

### sync

Transfers virtual-layer changes to disk.

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| file_path | string | no | — | path to sync; an absent path syncs all modified paths |

Sync applies the "first syncer wins" principle: if the file on disk changed since the agent read it, sync does not overwrite someone else's change. Non-overlapping changes on disk are merged automatically; overlapping changes are returned as a conflict, and the virtual layer receives the file with the conflict markers `<<<<<<< staged / ======= / >>>>>>> disk`, which must be read, resolved, and synced again. A binary file is not merged: on divergence a conflict is returned instructing to overwrite the file. The result contains the list of synced paths (with create/update/delete kind, created directories, and the auto-merge flag) and the list of conflicts. With `autocreate: false`, a missing parent directory is returned as a conflict.

### diff

Shows the difference between the virtual layer and the disk. Read-only.

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| file_path | string | no | — | path to compare; an absent path shows all modified paths |

The difference is returned as displayable unified-diff hunks; it is not meant to be applied as a patch. A file marked for deletion is shown as the deletion of its base content; a binary file shows an empty hunk.

### discard

Removes virtual-layer entries, returning the state to disk. The disk is not modified.

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| file_path | string | no | — | path to reset; an absent path resets the whole virtual layer |

### status

Shows the state of the virtual layer. Read-only.

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| file_path | string | no | — | path to check; an absent path shows all paths |

For each path, the unsynced-change flag, the disk-divergence flag, and the deletion-marked flag are returned. Status shows both modified entries and clean read-cache entries.

## Behavior and limitations

The virtual layer belongs to one agent (session). Writes and edits enter it before sync; other agents and external processes do not see it. Reading by default sees the virtual version on top of the disk.

External processes (shell, git, editors) work directly with the disk and see only synced content. Before an external process must see changes, sync is performed. After an external write, the agent's read cache can become stale: reading with `virtual: false, revirtual: true` drops the stale virtual-layer entry. The `revirtual` parameter is dangerous in that it discards unsynced work for that path — it is applied only to a path without its own unsynced edits.

A file change on disk between the check and the edit is returned as a "file unexpectedly changed" error, not as a silent write. For an edit, touching the file (changing the time without changing the content) is allowed; for an overwrite via write it is not.

Divergence checking at sync relies on the content hash, not on modification time. Syncing a single path is serialized within the process; an edit that lands between the snapshot and the write is not lost.

The virtual layer is kept in memory and disappears on restart. At session start the layer is restored from the log: successful write, edit, and delete operations after the last reset point are reapplied. Paths whose final content already matches the disk are not restored. An operation that cannot be applied after restart is declared skipped, and the path is marked for a forced conflict at the next sync. Two entries of a path with different case that coincide on the given disk are not merged: the later one is dropped and reported, and the path refuses to sync until resolved.

An external change to a file present in the agent's memory is delivered to the agent as a file-changed notification after 250 ms debouncing. The agent's own sync does not produce such a notification.

Read limits: a full text read and a notebook read are limited to 256 KB (an explicit `limit` disables the limit for text); a structural code read is limited to 4 MB; the final read result is limited to 262144 characters with truncation at a line boundary. Editing a file larger than 1 GiB is rejected. Reading binary files is rejected by a list of extensions, and for unlisted formats a NUL-byte check is performed. Reading PDF is rejected. Device paths and UNC paths are handled without filesystem access.

## Examples

Reading a text file, lines 10 through 29:

```json
{"operation": "read", "file_path": "src/main.rs", "offset": 10, "limit": 20}
```

Recursive directory read:

```json
{"operation": "read", "file_path": "rust-agent", "depth": 3}
```

Creating a new file:

```json
{"operation": "write", "file_path": "notes/todo.md", "content": "# Tasks\n"}
```

Point edit of an existing file:

```json
{"operation": "edit", "file_path": "README.md", "old_string": "version 1.0", "new_string": "version 1.1"}
```

Syncing all modified files to disk:

```json
{"operation": "sync"}
```
