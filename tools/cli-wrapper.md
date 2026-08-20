# cli-wrapper

A capability added through `Tools{add}`: turns any CLI script into a typed agent tool. The configuration specifies the tool name, interpreter, and script, plus a set of named commands with typed parameters; each command becomes an operation of the tool.

## Configuration

The capability is added by the call `Tools{operation:"add", capability:"cli-wrapper", config:{...}}`. Configuration fields:

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| `name` | string | yes | — | the tool name the model calls; non-empty |
| `description` | string | no | empty | the tool description, shown in the hint and the schema |
| `interpreter` | string | no | empty | the interpreter program (for example `"python"`, `"node"`); when empty, `script` is run directly |
| `script` | string | yes | — | the path to the script; non-empty |
| `script_args` | array | no | empty | additional arguments between the script and the command name |
| `timeout_ms` | integer | no | `60000` | the time limit of one call in milliseconds; must lie within `1..=60000` |
| `read_only` | boolean | no | `false` | declares the tool read-only and safe for parallel launch |
| `commands` | object | yes | — | named commands (at least one); keys are operation names |
| `setup` | object | no | absent | the auto-setup of a tool wrapping a resident process |

Each command (`commands.<name>`) is described as follows:

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| `description` | string | no | absent | the command description |
| `params` | array | no | empty | an ordered list of parameter declarations |
| `required` | array | no | empty | the names of required parameters |
| `read_only` | boolean | no | inherits the tool's `read_only` | read-only override for this command |
| `resident_stop` | boolean | no | `false` | marks the command as the canonical resident stop |

Parameter declaration (`params[]`):

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| `name` | string | yes | — | the parameter name — also the input field the model fills |
| `type` | string | yes | — | the JSON Schema type: `string`, `integer`, `number`, `boolean`, `array` |
| `description` | string | no | absent | the parameter description |
| `style` | string | no | `"positional"` | the way of passing: `positional` or `flag` |
| `flag` | string | no | the name from `name` with `_` replaced by `-` | override of the flag name for `style: "flag"` |

The `setup` block (resident):

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| `probe` | array | yes (non-empty on activation) | — | the argv tail of the readiness probe (exit 0 = ready) |
| `start` | array | no | — | the argv tail of the resident launch; launched detached |
| `ready_timeout_ms` | integer | no | `10000` | how long to wait for resident readiness after launch, in milliseconds |
| `ready_hint_ms` | integer | no | absent | the estimated time to readiness, shown to the agent |
| `log_file` | string | no | `<script directory>/<name>.serve.log` | where the resident stdout/stderr are written; supports `{session_id}` and `{cwd}` |
| `resident_key` | string | no | absent | the logical resident identifier; literal, without templates |
| `scope` | string | no | `"shared"` | the scope of resident ownership; only `shared` is supported |
| `skip` | boolean | no | `false` | skip the entire resident activation and immediately treat it as ready |

## How commands become operations

Each name in `commands` is an allowed value of the tool's `operation` field. The call input is a flat object: `operation` plus the command parameters. Parameters of one command using the same `name` are collected into a single schema property; the description of a parameter whose meaning is tied to a specific operation is annotated with the list of these operations.

## Command-line construction

The call assembles argv in the order:

```
[interpreter?] script script_args... operation [positional...] [flags...]
```

- Positional parameters are added as a bare value in declaration order.
- Flag parameters are added after, in declaration order, as `--<flag> <value>`; a boolean `true` gives only `--<flag>`, `false` omits the flag; an array repeats `--<flag> <value>` for each element.
- A positional array is expanded into several bare values.
- A string value is passed as is; numbers and booleans are passed by their textual representation (`42`, `3.14`, `true`/`false`).

## Exit code and output

- The process stdout and stderr are read concurrently and merged: stderr is appended after stdout when non-empty.
- `exit_code` is the process exit code; when absent — `-1`.
- A non-zero exit code makes the result erroneous (`is_error`).
- Exceeding `timeout_ms` returns a timeout error; a user interruption returns a result with `interrupted: true` and `exit_code: 130`.
- For a resident tool, a failed call while the resident is starting or has crashed is supplemented with an explanatory note; a successful call does not imply resident readiness — readiness is confirmed only by the canonical `probe`.

## Resident process

- On activation the tool first runs `probe`. If the probe already passes, `start` is not run — a live resident is not killed or restarted; the fact of presence is registered and the tool is ready.
- Otherwise an atomic resident takeover is performed by the allowed launch argv (and by `resident_key` when set): the winner launches the resident detached (with a separate process group), records the pid, and spawns a background readiness monitor; the loser waits for the other monitor. Activation returns immediately with the "warming up" status, without blocking the addition.
- The monitor polls `probe` (500 ms pause between probes, 5000 ms probe limit) until ready or until `ready_timeout_ms` expires; on expiry the resident is marked failed and the session is notified.
- A command with `resident_stop: true`, after exit code 0, waits until `probe` stops passing (check up to 10 000 ms, probe window 2000 ms) and deletes its resident record. The outcome is recorded in `lifecycle`: `confirmed`, `unconfirmed`, `interrupted`, or `unbound`; any outcome other than `confirmed` marks the result erroneous, and the record is kept.

## Behavior and limits

- Output trimming: the `_head` parameter with an integer N (minimum 1) keeps the first N lines of the merged output; when trimmed, a line with the number of kept lines is added. It does not affect the status and arguments of the command.
- The maximum size of a tool result is 30 000 characters.
- Parameter names starting with `_` are reserved and forbidden in the configuration.
- The `{session_id}` and `{cwd}` templates are substituted only in `probe`, `start`, and `log_file`; in `script_args` template variables are forbidden and cause a configuration error.
- The `{port}` template and the value `scope: "per_agent"` are not supported — specifying either gives a configuration error.
- The `read_only` flag is a declaration: it marks the call as reading and safe for parallel launch. It does not check the actual behavior of the script and does not limit the process rights; by default the tool is considered state-mutating.
- A configuration without `commands`, with an empty `name` or `script`, with `timeout_ms` outside `1..=60000`, with a resident `resident_stop` command without a `setup` block, or with more than one such command is rejected with a configuration error.

## Examples

A tool over a system command: the command name becomes an operation, and parameters become its arguments (`git status --short`):

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
      },
      "log": {
        "description": "recent commits",
        "params": [
          { "name": "max_count", "type": "integer", "style": "flag", "flag": "max-count" },
          { "name": "path", "type": "string", "style": "positional" }
        ],
        "required": []
      }
    }
  }
}
```

A tool with an interpreter and constant script arguments:

```json
{
  "operation": "add",
  "capability": "cli-wrapper",
  "config": {
    "name": "Index",
    "description": "search over the project index",
    "interpreter": "python",
    "script": "tools/index.py",
    "script_args": ["--root", "."],
    "timeout_ms": 30000,
    "commands": {
      "lookup": {
        "description": "find an entry by identifier",
        "params": [{ "name": "id", "type": "string", "style": "positional" }],
        "required": ["id"],
        "read_only": true
      }
    }
  }
}
```

A tool over a resident process:

```json
{
  "operation": "add",
  "capability": "cli-wrapper",
  "config": {
    "name": "Index",
    "interpreter": "python",
    "script": "tools/index.py",
    "commands": {
      "lookup": {
        "description": "find an entry by identifier",
        "params": [{ "name": "id", "type": "string", "style": "positional" }],
        "required": ["id"]
      },
      "stop": { "description": "stop the resident", "params": [], "required": [], "resident_stop": true }
    },
    "setup": {
      "probe": ["status", "--brief"],
      "start": ["serve", "--port", "47777"],
      "ready_timeout_ms": 20000,
      "ready_hint_ms": 3000,
      "resident_key": "index-runtime"
    }
  }
}
```

Invoke an added tool:

```json
{ "operation": "invoke", "name": "Index", "input": { "operation": "lookup", "id": "42" } }
```

Trim the output of a long command to the first lines:

```json
{ "operation": "invoke", "name": "Git", "input": { "operation": "log", "max_count": 100, "_head": 40 } }
```
