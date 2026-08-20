# Tools

A meta-tool that manages the live tool set of the agent in the current session: adds, removes, lists, searches, inspects, and invokes tools. This is the only surface through which the agent expands or reduces its own set of capabilities during work.

## Operations

The tool distinguishes operations by the `operation` field. Allowed values: `add`, `remove`, `list`, `search`, `inspect`, `invoke`.

### add

Adds to the live session tools produced by a pre-registered capability from the given configuration. In the release build exactly one capability is available — `cli-wrapper`.

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| `operation` | string | yes | — | the value `"add"` |
| `capability` | string | yes | — | the capability name; the only allowed value in the build is `"cli-wrapper"` |
| `config` | object/value | yes for `cli-wrapper` | — | the capability configuration; either an inline object, or `{"config_path": "<path>"}` to load from a file. Required for `cli-wrapper` and must contain at least `name`, `script`, and `commands` |
| `activate` | string | no | `"immediate"` | when the tools become callable: `"immediate"` (in the current turn) or `"deferred"` (in the next turn) |

`config_path` is resolved relative to the session working directory: the path is normalized (`~` is expanded, separators are normalized, `.` and `..` are collapsed), a relative path is joined to the session directory. The file is read and parsed as JSON. Other keys next to `config_path` are overlaid on top of the loaded object — inline values take priority.

Returns: the list of canonical names of the added tools (`added`), the new tool-set epoch number (`epoch`), the `immediate` flag, the list of starting residents (`pending`), the list of activation errors (`activation_failed`), the event identifier (`event_id`), the `metadata_pending` and `delivery_pending` flags.

Errors: an unknown capability returns the error `unknown capability '<name>'; available: [...]` with the list of available ones; an invalid configuration returns `invalid config for capability '<name>': <reason>`; a capability that produced no tools returns `capability '<name>' produced no tools`; a name/alias conflict within the package or with an already existing tool returns a separate error. A provider-server tool cannot be installed dynamically.

### remove

Removes a tool from the live set by name. Takes effect on the next snapshot of the set.

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| `operation` | string | yes | — | the value `"remove"` |
| `name` | string | yes | — | the canonical name of the tool to remove |

Returns: the name (`name`), the `was_present` flag (false when the name was not in the set — an idempotent operation without error), `event_id`, `metadata_pending`, `delivery_pending`, the `deactivation_failed` list, the new `epoch`.

The core tools `Tools` and `Agent` cannot be removed — an attempt returns an error. A missing name is not an error but a result with `was_present: false`. A dynamically added tool is restored by a repeated `add` with the same capability and configuration; for a built-in tool there is no restore operation.

### list

Shows the live tool set, optionally with the difference relative to the given epoch.

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| `operation` | string | yes | — | the value `"list"` |
| `since_epoch` | integer/string | no | absent | if set, `delta` is also returned — what changed since this epoch; `0` = "since the very first mutation" |
| `kind` | string | no | absent | filter by tool kind (for example `"builtin"`, `"mcp"`), or by source label (for example `"base"`, `"agent"`, `"mcp:<server>"`), or by call mode (`"direct"`, `"via_tools"`); case-insensitive comparison |

Returns: `epoch`, the `tools` list (for each: `name`, `aliases`, `kind`, `source`, `call_mode`, `availability`, `interface_digest`, `decorated`, `deferred`) and, only when `since_epoch` is set, the `delta` field.

An unrecognized `kind` value gives an empty tool list, not an error.

### search

Reveals deferred tools — those present in the hint only by name, whose full schema is returned on request.

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| `operation` | string | yes | — | the value `"search"` |
| `query` | string | yes | — | `select:<name1>,<name2>` for direct revealing by exact names, or keywords; the `+` prefix before a word makes it mandatory |
| `max_results` | integer/string | no | `5` | maximum keyword results; ignored on the `select:` path |

Returns: `matches` (list of names), `query` (the echo of the request), `total_deferred_tools` (how many deferred tools are in the pool in total), `pending_mcp_servers` (only on a zero result).

Keyword search has fast paths: an exact name match (case-insensitive) and the `mcp__` prefix. Mandatory words (`+word`) filter out non-matching tools; then tools are ranked by match in name parts, description, and hint, and returned in descending order of relevance in an amount not exceeding `max_results`.

### inspect

Returns the full description of the tool interface together with its availability in the current set.

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| `operation` | string | yes | — | the value `"inspect"` |
| `name` | string | yes | — | the canonical name or alias |

Returns: `requested_name`, `descriptor` (the full interface), `availability`, `epoch`, `reason` (an explanation when unavailable, limited to 2048 characters).

An empty name is the error `Tools{inspect}.name must not be blank`; a non-existent tool is the error `tool binding not found: <name>`.

### invoke

Invokes a tool by name. The input is passed to the target tool as is, without parsing on the `Tools` side.

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| `operation` | string | yes | — | the value `"invoke"` |
| `name` | string | yes | — | the canonical name or alias of the target |
| `input` | value | yes | — | the input of the target tool |

The operation is intercepted before the tool body and routed to the target tool; reaching the wrapper body with this operation is an error.

## Behavior and limits

- The `Tools` tool is always present and cannot be removed; together with `Agent` it belongs to the core tools, whose removal is forbidden.
- `Tools` cannot invoke itself through `invoke`.
- An added tool does not appear as a separate function in the model's tool-call interface. Its interface is obtained through `Tools{inspect}`, and calls are performed through `Tools{invoke}`. The "direct call versus `via_tools`" resolution happens once per tool.
- A repeated addition (`add`) is allowed only when the source and the interface fingerprint fully match. Changed content (a different fingerprint) requires a different name — otherwise a conflict error.
- `activate: immediate` (the default) makes the added tools callable in the current turn; `activate: deferred` moves the call to the next turn and keeps the prompt cache warm.
- For a tool with a resident that is still starting, `add` returns a `pending` entry with the name, the estimated readiness time (`hint_ms`), and the path to the resident log; a call before readiness returns a refusal, not an error.
- The maximum size of a tool result is 100 000 characters; the practical limit is reached only by `search` when revealing several tools.
- The `search` operation works only when the deferred-tools mode is enabled. The mode is enabled by the `ENABLE_TOOL_SEARCH` environment variable (a true value or `auto`/`auto:N`), while the `KOT_DISABLE_EXPERIMENTAL_BETAS` variable with a true value forcibly disables it. When the mode is disabled, the operation returns an error.
- Every set mutation (add/remove) increments the epoch number; `list` returns the current epoch, `since_epoch` allows obtaining the difference relative to a past state.

## Examples

Add the `cli-wrapper` capability with an inline configuration:

```json
{
  "operation": "add",
  "capability": "cli-wrapper",
  "config": {
    "name": "Git",
    "description": "repository state and history",
    "script": "git",
    "read_only": true,
    "commands": {
      "status": {
        "description": "working tree state",
        "params": [{ "name": "short", "type": "boolean", "style": "flag", "flag": "short" }],
        "required": []
      }
    }
  }
}
```

Add the `cli-wrapper` capability by loading the configuration from a file:

```json
{
  "operation": "add",
  "capability": "cli-wrapper",
  "config": { "config_path": "tools/git-tool.json" }
}
```

Show the built-in tools of the live set:

```json
{ "operation": "list", "kind": "builtin" }
```

Reveal deferred tools by names, then by keywords:

```json
{ "operation": "search", "query": "select:Git" }
```

```json
{ "operation": "search", "query": "+git history", "max_results": 3 }
```

Get the interface of an added tool, then invoke it:

```json
{ "operation": "inspect", "name": "Git" }
```

```json
{ "operation": "invoke", "name": "Git", "input": { "operation": "status", "short": true } }
```
