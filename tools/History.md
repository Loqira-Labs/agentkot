# History

A tool for reading the recorded chat history: recovers past tool results from the transcript, lists them, prepares a history slice for launching a child, and lists sessions. Returns the bytes recorded at the moment of execution, not the changed current file. One tool with five operations distinguished by the `operation` field: `recover`, `query`, `fork`, `list_sessions`, `compact`.

Scopes (`scope`): `session` (only the current session), `project` (all sessions of the project), and `global` (all projects on the machine). Defaults: `project` for the main agent, `session` for a sub-agent. `global` is read-only and main-agent-only; `fork` rejects `global`. A sub-agent with the explicit `project` or `global` scope receives an error.

## Operations

### recover

Returns one past tool result (usually an earlier file read) or recovers it by call identifier, and either returns it as result text or re-injects it into the transcript.

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| file_path | string | yes, if no tool_use_id | — | the absolute path whose past result to recover |
| tool_use_id | string | yes, if no file_path | — | the exact call identifier; recovers any past result, including errors |
| mode | string | no | `return` | `return` — return as result text; `inject` — additionally insert into the transcript |
| tool_name | string | no | `Files` | which tool to recover |
| tool_operation | string | no | `read` if tool_name = `Files` | the operation of the called tool |
| strip_line_numbers | boolean | no | `true` for `Files` and `Read` | remove line-number prefixes of the form `NN→` and `NN\t` |
| scope | string | no | `project` / `session` | the history horizon |

`file_path` and `tool_use_id` are mutually exclusive; exactly one of them is required. When recovering by `tool_use_id`, the fields `tool_name` and `tool_operation` are ignored.

Returns `file_path`, `recovered` (whether the result was found), `tool_use_id`, `captured_at` (the time in the transcript), `from_session` (the source session for the `project`/`global` scope), `from_externalized`, and `bytes` (the text length after numbering removal). Not found is a soft "no" result, not an error. A pure image (with no text) is returned with a clear message and a zero byte count.

The `inject` mode adds a short confirmation and the recovered text as a meta-message to the transcript, so that the text enters the context and survives a future compaction. The `return` mode does not change the transcript.

### query

Lists past tool results by filters, from newest to oldest. Returns only metadata without bodies; bodies are returned by the `recover` operation.

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| tool_name | string | no | no filter | the tool name |
| tool_operation | string | no | no filter | the tool operation; there is no implied value here, both reads and edits are listed |
| file_path | string | no | no filter | the path that the call touched |
| limit | integer | no | all matches | stop after N matches; minimum 1 |
| include_errors | boolean | no | `false` | whether to include error results |
| session_id | string | no | no filter | confine to one session (uuid); the session must be reachable in the scope |
| scope | string | no | `project` / `session` | the history horizon |

Filters are combined with logical AND. Returns `captures` — a list of records, each with `tool_use_id`, `tool_name`, `tool_operation`, `file_path`, `is_error`, `captured_at`, `from_session`, `from_project` (only for `global`), and `bytes`.

### fork

Materializes a history slice into a set of messages suitable for launching a child. Emits a one-time `fork_id` identifier, which is passed to the `Agent` tool via `seed_history` when launching the child. The operation itself does not launch a child.

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| range | object | no | the whole visible context | which history slice to fork |
| strip_line_numbers | boolean | no | `true` for `Files`/`Read` captures | remove line numbering from the capture body |
| scope | string | no | `project` / `session` | the history horizon; `global` is rejected |

The `range` field is an object with the key `kind` and a value of one of the kinds:

| kind | extra fields | meaning |
|---|---|---|
| whole_context | — | the whole current visible context (default) |
| last_n | `n` (integer, minimum 1) | the last N messages |
| before_compact_boundary | — | everything strictly before the context compaction boundary |
| capture | `tool_name`, `file_path`, optional `tool_operation` | one past result, folded into a leading message |
| whole_session | `session_id` (uuid) | one entire session |

Returns `fork_id`, `message_count` (the number of messages in the slice; `0` means "nothing to fork"), `scope`, `range_kind`, and `sessions_scanned` (the number of sessions that contributed messages). The identifier is one-time: reuse yields the error "not found or already consumed". The slice ends with a synthetic guard message.

### list_sessions

Lists the sessions of the scope, from most recent activity to oldest, to pick one for `query` or `fork`.

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| scope | string | no | `project` / `session` | the history horizon |
| include_sidechains | boolean | no | `false` | whether to include sub-agent transcripts |
| limit | integer | no | all sessions | keep the first N sessions of the list |
| active_since | string | no | no filter | an RFC3339 instant; sessions with activity not earlier than it remain |
| text | string | no | no filter | a case-insensitive substring over the title or the first prompt |

Filters are combined with logical AND (time cutoff → text → limit). An invalid `active_since` is an error. Returns `sessions` — a list of records with `session_id`, `project_key` (only for `global`), `started_at`, `last_activity_at`, `message_count_estimate` (an approximate estimate), `first_user_text`, `title`, `is_current`, and `is_sidechain`.

### compact

Schedules context compaction: before the next response, folds the older prefix of the current session into a brief summary, keeping the most recent messages. Takes no parameters; always acts on the live session. The call always schedules the compaction and returns `scheduled: true`. The folding itself happens outside the tool, before the next request to the model. Use only when the context is genuinely full (around 90%).

## Behavior and limitations

- The `recover`, `query`, `fork`, and `list_sessions` operations only read history. `compact` is not a read (it requests a context rewrite), but it is not destructive: the full transcript on disk is not changed.
- When the history-reading mechanism is absent, a call to an operation that requires reading returns the error "history reader unavailable"; there is no fallback path to the live filesystem.
- History reading is serialized against the cancellation flag: when the session is cancelled, an interruption error is returned.
- Line-number removal: the prefix `^\s*\d+[→\t]` at the start of each line is removed, so that the recovered body is ready to write. By default it applies to results of the `Files` and `Read` tools; for other tools it is not applied by default; an explicit value always wins.
- Path resolution on match is lexical: separators are normalized to `/`, `.` and `..` are resolved, the Windows drive letter is uppercased, and names are compared verbatim without case folding and without touching the filesystem.
- Results marked as deleted (tombstone) do not surface in `query`, in `fork`, or in ranges.
- A result body spilled out of budget (spill) is restored on read from the session directory; a missing spill file is an error, not a truncated preview.
- The tool never spills its own result out of budget: the recovered body cannot go to spill and loop.
- The `fork` slice is canonicalized and repaired before delivery: broken "tool call ↔ result" pairs are restored with a synthetic error result, and dangling results are dropped. The slice is re-linked linearly so that the child's first request satisfies the provider contract.
- `compact` does not enforce the context-fullness check; the "around 90%" recommendation is a hint, not a gate.

## Examples

Recover a past file read:

```json
{"operation": "recover", "file_path": "C:/work/src/lib.rs"}
```

Recover a call result by identifier with insertion into the context:

```json
{"operation": "recover", "tool_use_id": "toolu_0123", "mode": "inject"}
```

List past reads of a file:

```json
{"operation": "query", "tool_name": "Files", "tool_operation": "read", "file_path": "C:/work/src/lib.rs", "limit": 10}
```

Prepare a slice of the last 5 messages for a child:

```json
{"operation": "fork", "range": {"kind": "last_n", "n": 5}, "scope": "session"}
```

List project sessions with a substring in the title:

```json
{"operation": "list_sessions", "scope": "project", "text": "parser", "limit": 20}
```

Compact the current session's context:

```json
{"operation": "compact"}
```
