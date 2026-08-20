# Shell

Runs commands in the host system's shell and returns the combined output. One tool serves all supported shells (PowerShell, bash/sh, cmd), and the choice of a specific terminal is made inside the tool. Name aliases: `Bash`, `RunCommand`.

## Operations

The tool accepts an optional `operation` field; when the field is absent, the `run` operation is implied. Available operations: `run`, `set_default_shell`, `bash_output`, `kill`, `list_tasks`, `shell_open`, `shell_send`, `shell_read`, `shell_close`. An unknown `operation` value is an error.

### run

Executes a command in the chosen shell and returns the result. The primary operation; it is the one that accepts input of the form `{ "command": "..." }` without an `operation` field.

Parameters:

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| command | string | yes | — | The command to execute, written in the syntax of the chosen shell. |
| timeout | number or string | no | 60000 | The foreground blocking-wait limit in milliseconds. Maximum 60000; a larger value is an error, not a limit. Ignored for a background run. |
| description | string | no | — | A short 5–10 word description; used in the interface and as the label of the background task. |
| run_in_background | boolean or string | no | false | When `true`, the command is launched as a separate background task, and the call returns immediately with a task identifier. |
| shell | string | no | — | The name of the terminal for this call (one of the available shell names). When absent, the session default is used, then the system default. |
| raw | boolean or string | no | false | When `true`, returns uncompressed output; by default the output goes through compression. |
| monitor | object | no | — | Configuration of the proactive monitor for a background run. Requires `run_in_background: true`. |

Returns: combined `stdout` (`stderr` is appended to it if non-empty), `exit_code`, `interrupted`, the terminal name `shell`, the family `shell_kind`, the semantic interpretation of the exit code `return_code_interpretation`, `background_task_id` for a background run, `monitor_id` when a monitor is set, `persisted_output_path` when output spills to disk, `is_image`, `no_output_expected`, `compression` (compression report).

Limitations: `timeout` is limited to 60000 ms.

### set_default_shell

Switches the default terminal for the current session. Subsequent `run` calls without an explicit `shell` use the new default.

Parameters:

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| shell | string | yes | — | The name of the terminal that becomes the session default. Must be a known and available name. |

Returns: the new default `default_shell` and the previously switched default `previous_default` (empty if the session has not switched the default before).

Limitations: an unknown or unavailable name is an error; no substitution is performed.

### bash_output

Reads the new (since the last read) output of a background task.

Parameters:

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| task_id | string | yes | — | The `bg_<n>` identifier returned by the launching `run`. |
| block | boolean or string | no | false | Wait mode: `false`/omitted — an instant snapshot of the current delta; `true` — wait for the first new output or completion; `"completion"` — wait for completion and return the entire accumulated delta in one call. |
| timeout | number or string | no | 60000 | The wait limit for a blocking read in milliseconds. Maximum 60000. Ignored for an instant snapshot. |
| filter | string | no | — | A regular expression; the returned delta is reduced to the lines that match it. |
| raw | boolean or string | no | false | When `true`, returns the uncompressed delta. |

Returns: `task_id`, `new_output` (the delta), `status` (`running`/`exited`/`killed`), `exit_code` after completion, `compression`. After completion, a line of the form `[task bg_<n> exited with code N]` is added to the block text.

Limitations: a nonexistent `task_id` is an error; an invalid `filter` regular expression is an error.

### kill

Stops a background item by identifier type.

Parameters:

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| task_id | string | yes | — | The identifier `bg_<n>` (terminate the task), `mon_<n>` (disarm the monitor; the task keeps running) or `res_<n>` (stop the resident). |

Returns: `task_id` and `success`. For `bg_<n>` — `true` if the running task was terminated, `false` if it has already exited. For `mon_<n>`/`res_<n>` — `true` if a live item was stopped/disarmed, `false` if the identifier is already absent.

Limitations: a nonexistent `bg_<n>` is an error; an absent `mon_<n>`/`res_<n>` is a soft `success: false`.

### list_tasks

Lists all background tasks and the monitors attached to each, as well as registered residents.

Parameters: none.

Returns: a list of tasks (newest first) with fields `task_id`, `status`, `exit_code`, `description`, `output_bytes`, `persisted_output_path`, `monitors`, and a list of residents. An empty list is not an error; the block text is `(no background tasks)`.

### shell_open

Opens a persistent interactive shell session — one live interpreter process whose state (working directory, environment and shell variables) is preserved between commands.

Parameters:

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| shell | string | no | — | The name of the terminal for the session (the same names as for `run`). When absent — the session default, then the system default. |
| cwd | string | no | session directory | The working directory in which the session starts. |
| description | string | no | — | A short 5–10 word description. |

Returns: `session_id` (`sh_<n>`) and `output` — the output captured in the startup window (prompt/banner).

Limitations: a nonexistent `cwd`, a startup failure, or exceeding the session limit (16 concurrent by default) is an error.

### shell_send

Sends a command or input to a live session and collects the output produced. The session state is preserved between sends.

Parameters:

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| session_id | string | yes | — | The `sh_<n>` identifier of the session. |
| input | string | yes | — | The command or input to send; `\r` (terminal Enter) is appended to it for execution. |
| timeout | number or string | no | 10000 | The output collection window limit in milliseconds. Maximum 60000. |

Returns: `session_id`, `output` (delta), `closed` (whether the session closed). The window also ends early on silence; the session command continues executing after the window closes.

Limitations: a missing or closed session is an error.

### shell_read

Incrementally reads the new output of a live session without sending input — for catching up on a long-running command.

Parameters:

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| session_id | string | yes | — | The `sh_<n>` identifier of the session. |
| timeout | number or string | no | 10000 | The output collection window limit in milliseconds. Maximum 60000. |

Returns: `session_id`, `output` (delta), `closed`.

### shell_close

Terminates the process of a live session and frees its slot.

Parameters:

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| session_id | string | yes | — | The `sh_<n>` identifier of the session to close. |

Returns: `session_id` and `success` (`true` if a running session was terminated).

Limitations: an absent identifier is an error; `success` is `false` when the session has already exited.

## Behavior and limitations

### Shell selection

The tool chooses the terminal internally by name. Default names: on Windows — `pwsh` (PowerShell) and `cmd` (cmd.exe); on POSIX — `bash` and `sh`.

Custom named shells are defined by the optional file `<config_directory>/shells.json` — a JSON array of records:

| field | type | required | meaning |
|---|---|---|---|
| name | string | yes | the name given in the `shell` parameter; unique case-insensitively; a record named after a system shell overrides it |
| kind | string | yes | the syntax family: `bash`, `sh`, `pwsh` or `cmd` — determines the launch arguments, output decoding and exit-code interpretation |
| program | string | no | the explicit path to the interpreter; checked as a path on disk, with no `PATH` search for it. Without this field, the interpreter is looked up in `PATH` by the family's canonical names. A specified path that is absent on disk is an error at selection time, with no substitution of another shell |
| edition | string | no | for the `pwsh` family: `desktop` (PowerShell 5.1) or `core` (PowerShell 7+); without it the edition is determined by the file name |
| default | boolean | no | make this shell the system default (only one); the session default from `set_default_shell` has priority |

```json
[
  { "name": "git-bash", "kind": "bash", "program": "C:/Program Files/Git/bin/bash.exe" },
  { "name": "ps5", "kind": "pwsh", "program": "C:/Windows/System32/WindowsPowerShell/v1.0/powershell.exe", "edition": "desktop" }
]
```

If the file is absent — only the system shells are available. A file with a parse error does not abort startup: a warning is printed, and the system shells remain.

Resolution priority for `run` and `shell_open`: the name in the call (`shell`) → the session default (switched via `set_default_shell`) → the system default. System default: on Windows — `pwsh` (lookup of `pwsh`, then `powershell`, then Linux fallback paths), on POSIX — `bash` with fallback to `sh`. `cmd` is reachable only by an explicit name and is never chosen automatically.

Selection is performed by probing without launching; the result is memoized per process. An absent interpreter, or a configured path that is absent on disk, is an error; no silent substitution of another shell occurs.

Command launch arguments: bash/sh — `-c <command>`; PowerShell (both editions) — `-NoProfile -NonInteractive -Command <command>`; cmd — `/C <command>`.

### Timeouts and exit codes

The hard model-managed blocking-wait limit is 60000 ms (1 minute) for: a foreground `run` (`timeout`), a blocking `bash_output` (when `block` resolves to waiting for output or completion), the `shell_send`/`shell_read` window. A value of exactly 60000 is allowed; a larger one is an error, never a limit. The limit is not configurable.

The default timeout for a foreground `run` is 60000 ms. On timeout expiry, the command is killed entirely (the whole process tree) and the tool returns a timeout error.

The exit code is the child process's code. If the command is interrupted by the user (`interrupted: true`), the code is 130 (the 128+SIGINT convention). `stdout` and `stderr` are merged: `stderr` is appended to `stdout` if non-empty.

Exit-code interpretation depends on the shell. bash: `grep`/`rg` with code 1 — "No matches found" (not an error), code 2 and above — an error; `diff` with code 1 — "Files differ" (not an error), code 2 and above — an error. PowerShell: `grep`/`rg`/`findstr` with code 1 — "No matches found"; `robocopy` — a bit field (0 — "Already in sync", 1–7 — "Files copied", 8 and above — an error). cmd: only the default rule — code 0 is success. In all other cases only code 0 counts as success.

Output decoding depends on the shell: PowerShell Desktop (5.1) produces UTF-16 LE (recognized by a BOM or by the presence of NUL bytes) and normalizes `\r\n` to `\n`; PowerShell Core (7+) and bash/sh/cmd — UTF-8 with loss on invalid bytes.

### Background tasks

With `run_in_background: true`, the command is detached into a background task, and the call returns immediately with the identifier `bg_<n>` and the text "Command running in background with ID bg_<n>". Identifiers increase monotonically from `bg_0`. stdout and stderr are continuously merged into one buffer; the task status is `running`, `exited` (with a code), `killed`.

The buffer accumulates in memory up to a 64 MB threshold; after it is exceeded, the output spills to the file `<runtime>/shell-bg/<id>.out`, and a 1 MB tail remains in memory. The file path is available in `persisted_output_path`.

Output is decoded as UTF-8 with loss; an incomplete multi-byte sequence at a read boundary is held until the next read so that a character is not split across reads.

The list of background tasks is shared for the agent process; at session end, its tasks are killed and removed together with their spill files. Orphaned spill files from previous runs are removed at startup.

The `block` mode for `bash_output`: `false`/omitted — an instant snapshot; `true` — wait for the first new output or completion; the strings `"completion"`, `"exit"`, `"until_exit"`, `"done"` — wait for completion and return the whole delta in one call. A `"completion"` read returns only when the process has stopped and both streams are fully drained.

### Monitors

A monitor is a proactive observer that starts together with a background `run` when the `monitor` field is present. It watches the task from its own cursor (starting at byte 0, so output between launch and setup is not lost) and, on triggering, delivers a notification to the session without polling. The identifier `mon_<n>` is returned.

The `on` field (what to wait for): `completion` (default) — task completion or kill; `output` — a new output line matches `pattern`; `idle` — no new output for `idle_secs` seconds; `heartbeat` — fires every `interval_secs` seconds while the task runs.

Monitor field rules:

| field | requiredness | constraint |
|---|---|---|
| on | no (default `completion`) | one of `completion`/`output`/`idle`/`heartbeat` |
| pattern | required for `on:"output"` | a non-empty regular expression; only with `output` |
| idle_secs | required for `on:"idle"` | integer ≥ 1; only with `idle` |
| interval_secs | required for `on:"heartbeat"` | integer ≥ 1; only with `heartbeat` |
| recurring | no | see below |
| throttle_secs | required for `on:"output"` + `recurring:true` | integer ≥ 1; only in this combination |
| raw | no | when `true`, the notification excerpt is the raw tail without compression |

Repetition: `completion` — one-shot (terminal), `recurring: true` is not allowed with it. `output`/`idle` — one-shot by default; `recurring: true` makes them keep firing (for `output`, `throttle_secs` is required — the coalescing window that suppresses a notification storm; for `idle` — one firing per silence episode, new activity re-arms it). `heartbeat` — recurring by nature, `recurring: false` is not allowed with it.

Every numeric monitor duration field is limited to 100 years (timer representability); a larger value is an error. An `output` match is made only on complete lines; a line longer than 64 KB without termination causes a loud monitor shutdown with an error message (no truncation or prefix matching). The notification excerpt is a tail of up to 8 KB.

An incompatible field combination is an error before the task launches; the task is not spawned. If monitor setup unexpectedly fails after launch, the task is killed and an error is returned. A monitor lives longer than the turn that set it up; it is disarmed by an explicit `kill mon_<n>` (the task keeps running) or at session end.

### Interactive sessions

A session is one live interpreter process in a pseudo-terminal (ConPTY on Windows, pty on POSIX), sized 50 lines × 200 columns. Launch: bash/sh — `-i`, PowerShell — `-NoLogo`, cmd — without flags. Up to 16 concurrent sessions are allowed by default.

State (working directory, environment and shell variables, activated virtual environment) is preserved between commands — sessions exist precisely for this.

`shell_send` appends `\r` (terminal Enter; `\n` is not Enter) to the input. The collection window ends on silence (no new bytes for 250 ms) or on the overall `timeout`; the session command continues executing after the window closes. The `shell_open` startup window is 2 seconds.

Session output is decoded as UTF-8 with loss, and ANSI control sequences (CSI, OSC, single control bytes) are removed from it; printable text, `\n` and `\t` are preserved. The session reader answers terminal queries (DSR/DA) so that the PowerShell interpreter does not hang.

`shell_close` terminates the process, waits for the remaining output to be read (at most 1.5 s) and removes the session from the list. At agent session end, all its interactive sessions are closed.

### Output compression

The output of a foreground `run` and of `bash_output` deltas, as well as monitor notification excerpts, goes through compression. The `raw: true` field disables compression for the call.

Compression picks a named filter by the command string; when there is no match, a universal safe transform is applied: ANSI sequence removal → collapsing empty lines → deduplication of consecutive identical lines → line limiting (the first 160 and the last 200 lines are kept, with an absolute limit of 400 lines; the omitted middle is marked with a marker).

Default named filters: `cargo build/check`, `cargo clippy`, `cargo test`, `git status`, `git log`, `git diff/show`, `git push`, `rg/grep`. Each keeps the signal (errors, warnings, failed tests, summaries) and discards the noise (build progress lines, hints, empty separators), applies the line limit, and on an empty result emits a message of the form "ok …".

The rewritten output is accepted only if it is strictly shorter than the original and actually reduces the token count (BPE count); otherwise the original text is returned. The compression report (`compression`) contains the original and compressed sizes, the token estimate, the savings percentage, the filter name and (when a tee file is written) the path.

On a nonzero exit code and actual compression, the full uncompressed output is written to the file `<temp>/kot-shell-raw/shell-raw-*.log`, and a hint with the path is added to the output. A write failure degrades silently (no file).

The tool result is limited to 30000 characters.

### Misc

- The command executes with the privileges of the user who launched the agent.
- If stdout is entirely an `image/*` data URI, it is decoded; when the larger side exceeds 2000 pixels it is downscaled preserving proportions (Lanczos3), re-encoded as PNG and returned as an image block. The size limit is 20 MB; an unreadable or still-too-large image degrades to text.
- The `no_output_expected` field is `true` when the command semantically produces no output (for example, a lone `cd`/`Set-Location`).
- Before launch, the process group is set up so that a kill takes the whole descendant tree (cargo→rustc→linker), not only the direct shell.

## Examples

Running a command in the default shell:

```json
{ "command": "cargo test --workspace", "description": "Run workspace tests" }
```

Running in the background with a completion monitor:

```json
{
  "operation": "run",
  "command": "python train.py",
  "description": "Model training",
  "run_in_background": true,
  "monitor": { "on": "completion" }
}
```

Running in the background with an output-line monitor:

```json
{
  "operation": "run",
  "command": "cargo test --workspace",
  "run_in_background": true,
  "monitor": { "on": "output", "pattern": "FAILED", "recurring": true, "throttle_secs": 30 }
}
```

Reading a background task's output to completion:

```json
{ "operation": "bash_output", "task_id": "bg_3", "block": "completion", "timeout": 30000 }
```

Opening an interactive session, sending a command and reading the result:

```json
{ "operation": "shell_open", "shell": "bash", "cwd": "/home/user/project" }
```

```json
{ "operation": "shell_send", "session_id": "sh_0", "input": "export KOTX=42 && echo $KOTX" }
```

Stopping a background task and disarming a monitor:

```json
{ "operation": "kill", "task_id": "bg_3" }
```

```json
{ "operation": "kill", "task_id": "mon_1" }
```
