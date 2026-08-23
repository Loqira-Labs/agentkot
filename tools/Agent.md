# Agent

Creates and manages sub-agents, teammates, and long-lived pipelines. A single tool; the action is selected by the `operation` field; alias `Task`.

## Operations

### spawn

Launches a sub-agent synchronously: the parent session waits for completion and receives the final report in the same turn. Suitable for tasks whose result is needed immediately for the next step.

Parameters:

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| operation | string | yes | — | `"spawn"` |
| description | string | yes | — | short (3–5 word) description of the task |
| prompt | string | yes | — | the task for the sub-agent |
| subagent_type | string | no | general-purpose | agent type (see "Agent types") |
| model | string | no | inherited | model override: alias `sonnet`/`opus`/`haiku` or a canonical id |
| provider | string | no | inherited | provider override (registered name) |
| effort | string | no | inherited | reasoning effort: `off`/`low`/`medium`/`high`/`xhigh`/`max` |
| isolation | string | no | — | `"worktree"` or `"remote"`; `remote` is unavailable |
| cwd | string | no | inherited | absolute path of the child's working directory |
| seed_history | string | no | — | one-shot `fork_id` from `History{fork}`; alias `fork_from` |

Returns a `completed` result: `agent_id`, `agent_type`, `content` (the text blocks of the report), `total_tool_use_count`, `total_duration_ms`, `total_tokens`, `usage` (four token counters), `prompt`, `worktree_path`/`worktree_branch` (if a worktree is kept), `transcript_path` (path to the child's full transcript). If the child returns no text, a no-output marker and the transcript path are substituted.

Constraints: blocks the parent's turn until the child completes. The child's intermediate tool calls do not appear in the parent's transcript.

### delegate

Launches a sub-agent in the background and immediately returns the `async_launched` launch confirmation; the terminal result arrives later as a `<task-notification>` in a separate turn. While the work runs, you can message the delegate and stop it by address.

Parameters — the same as for `spawn`, plus:

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| name | string | no | agent UUID | explicit addressable address; registered after a successful launch |
| until | object | no | — | completion-gate specification (see "The until gate") |

Returns `async_launched`: `agent_id`, `address` (the explicit name or UUID), `description`, `prompt`, `output_file` (the planned report path `<sidechains>/<agent-uuid>.output.md`), `can_read_output_file`.

Constraints: a taken name is an error. An ordinary delegate parks and has an absolute lifetime limit of 12 hours.

### hire

Creates a persistent wakeable teammate: a peer session that processes the briefing, goes to wait, and is woken by a message (`send`). Lives until `stop` or the closing cascade of the lead session.

Parameters:

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| operation | string | yes | — | `"hire"` |
| description | string | yes | — | the teammate's task label |
| prompt | string | yes | — | the briefing — the teammate's first turn |
| subagent_type | string | no | general-purpose | agent type |
| model | string | no | inherited | model override for the teammate's whole life |
| provider | string | no | inherited | provider override for the teammate's whole life |
| effort | string | no | inherited | the teammate's reasoning effort |
| name | string | yes | — | addressable name (mailbox key) |
| team_name | string | no | no team | the team the teammate joins |
| mode | string | no | — | only `"plan"` — enter planning mode |
| isolation | string | no | — | `"worktree"` or `"remote"`; `remote` is unavailable |
| seed_history | string | no | — | one-shot `fork_id`; alias `fork_from` |

Returns `teammate_spawned`: `agent_id`, `session_id` (the teammate's session identifier), `name`, `team_name`.

Constraints: the briefing runs in the background; its failure is mailed to the lead (never lost silently). The name cannot start with `team:`, cannot be `orchestrator`, and cannot contain `@`.

### send

Delivers a message to an agent address. The message is queued into the target session's re-entry and read at its next turn boundary. A peer-to-peer operation: a running sub-agent can also send a message (replying to the lead).

Parameters:

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| operation | string | yes | — | `"send"` |
| to | string | yes | — | recipient: an owned agent, `team:<name>`, `orchestrator`; cross-domain via `@<domain>` |
| message | string | yes | — | message body; beyond 30000 characters it is truncated with a marker |

Returns `message_sent`: `to`, `delivered`, `reason` (on refusal), `roster` (known addresses on refusal), `truncated`, `delivered_to` (a list on broadcast), `unsynced_paths` (a warning about the sender's unsynced paths).

Constraints: an unknown or non-live recipient is a soft refusal `delivered:false` with a list of addresses; only the absence of the mail router is an error. Broadcasting to a team excludes the sender itself. Cross-domain delivery does not disclose the sender's file paths.

### stop

Stops an agent by address. A teammate is dissolved: the current work is interrupted, the session is closed, the name is deregistered, the temporary worktree is removed if it has no changes and kept if it does. A delegate receives a stop request and sends an `aborted` notification.

Parameters:

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| operation | string | yes | — | `"stop"` |
| name | string | yes | — | address of the agent to stop |

Returns `stopped`: `name`, `stopped`, `kind` (`"teammate"`/`"delegate"`), `reason` (on refusal), `roster`.

Constraints: an unknown address is a soft refusal with a list of addresses. A qualified foreign address (`name@domain`) is a soft refusal on ownership rights: the lifecycle does not cross domains.

### workflow

Runs, inspects, or stops a declared TOML pipeline. A run lays the pipeline out onto a durable board (one task per step); ready steps run concurrently; the terminal result arrives as a `<task-notification>`.

Parameters:

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| operation | string | yes | — | `"workflow"` |
| action | string | no | `"run"` | `"run"`/`"status"`/`"stop"` |
| name | string | for run | — | stem of the declaration file `<cwd>/.kot/workflows/<name>.toml` |
| definition | string | for run | — | inline TOML declaration text |
| resume | string | for run | — | id of an existing run `wf-<8hex>` |
| run_id | string | for status/stop | — | id of the run to inspect/stop |

For `run`, exactly one of `name`/`definition`/`resume` is required. Returns `workflow_started` (`run_id`, `name`, `list_id`, `steps`, `resumed`), `workflow_status` (`run_id`, `name`, `state`, `steps`, `usage`, `reason`), or `workflow_stopped` (`run_id`, `stopped`, `reason`).

Declaration format (TOML):

```toml
version = 1

[limits]
timeout = 43200000        # ms, default and ceiling 43200000 (12 h)
max_iterations = 100      # default and ceiling 100

[[steps]]
name = "spec"             # required, [A-Za-z0-9_-]+
prompt = "write a specification from the requirements"
agent = "general-purpose" # default general-purpose

[[steps]]
name = "build"
prompt = "implement per the specification"
agent = "general-purpose"
model = "claude-opus-5"   # alias or canonical id
provider = "anthropic"    # cross-provider for an agent step/team members
depends_on = ["spec"]     # DAG dependencies by step names
inputs = ["spec"]         # subset of depends_on
transactional = true      # worktree transactionality
[steps.until]
check = "npm test"
max_iterations = 50
timeout = 3600000
supervisor = { agent = "arbiter", model = "haiku" }
[steps.on_error]
retry = 2
action = "escalate"       # "stop" (default) | "escalate"

[[steps]]
name = "review"
prompt = "review the result"
depends_on = ["build"]
[steps.team]
[[steps.team.members]]
role = "reviewer"         # required, unique within the step
agent = "arbiter"
model = "haiku"

[[steps]]
name = "approve"
ask = "accept the result?"   # ask gate: question to the owner
depends_on = ["review"]

[[steps]]
name = "deploy"
workflow = "deploy-pipeline"    # nested pipeline by file name
depends_on = ["approve"]
```

Schema rules: `version` is required and equals 1; unknown fields and an unknown version are errors. A step has exactly one executor out of `agent`/`team`/`workflow`/`ask`. A step with an `agent` or `team` executor must carry a non-empty `prompt`; a `workflow` or `ask` step must not carry a `prompt` at all. `model` and `provider` are allowed only on an `agent` step — in a team, each member carries them. A step name is `[A-Za-z0-9_-]+`, duplicates are forbidden. `depends_on` forms an acyclic graph (unknown references, self-dependency, cycles are errors); `inputs` is a subset of `depends_on`. In a team, roles are unique and an empty team is not allowed. `transactional` applies to `agent` and `team` steps (defaults to true for a step with an `until` gate, otherwise false). The `until` gate does not apply to an `ask` step. Pipeline nesting depth is at most 3. `until` values are checked against the run limits (downward only), and the supervisor must be a known agent type. `on_error.retry` is additional attempts (default 0); `on_error.action` is `stop` (run failure, cascading stop) or `escalate` (pause and mail the owner, resumable).

Constraints: the declaration is validated in full before any side effects. The run state (`running`/`paused`/`completed`/`failed`/`aborted`, or `unknown` on a soft miss) lives on the board and in `run.json`; a finished run can be inspected even after a process restart. On `resume`, completed steps are skipped and the rest are restarted with a fresh retry budget. Stopping is cascading — the entire node subtree is cancelled; a paused run is finalized directly.

### list_providers

Lists the providers available for cross-provider sub-agent launch. No parameters.

Returns `providers_listed` with the list: `name`, `is_default` (the current session's provider), `spawnable` (whether the name can be passed in `provider`), `model_count`. The session's own provider comes first.

### list_models

Lists a provider's models (or all providers' models) with their capabilities.

Parameters:

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| operation | string | yes | — | `"list_models"` |
| provider | string | no | all providers | filter by a registered provider name |

Returns `models_listed`: `provider`, `providers` (per provider: `name`, `is_default`, `spawnable`, `models`), `reason`/`known_providers` (on a soft miss from an unknown filter). The model card: `id`, `display_name`, `kind`, `for_chat`, `context_window`, `max_output_tokens`, `family`, `input_modalities`, `output_modalities`, `reasoning`, `tools`, `streaming`, `structured_outputs`, `preview`, `notes`.

Only a model with `for_chat: true` can drive an agent session; a card with `for_chat: false` is a media/embedding row, generated through the Media tool.

### current_model

Reads the calling session's current model state. No parameters.

Returns `current_model`: `provider`, `model`, `effort`, `routing_version`, `capabilities` (context window, max output tokens, input modalities, presence of reasoning), `frozen_head_format_version`, `frozen_head_digest`, `coordinator_generation`, `coordinator_next_seq`, `pending` (ordered pending changes).

Constraints: the operation is available when the session publishes its state; otherwise an error is returned.

### current_session

Reads the calling session's current working directory, persistent context, and mode. No parameters.

Returns `current_session`: `session_id`, `cwd`, `persistent_context`, `planning_mode`, `routing_version`, `frozen_head_digest`, `coordinator_generation`, `coordinator_next_seq`, `pending`.

Constraints: as for `current_model`.

### status

Lists the agents of the calling session's own domain: teammates (by name) and delegates (in registration order), with lifecycle and live state. No parameters.

Returns `status` with rows: `name`, `address` (only for an unnamed delegate, while it runs), `kind`, `team`, `agent_id`, `session_id`, `lifecycle`, `live_state` (`running`/`idle`/`closed`, or absent), `model`, `provider`, `pending_mail`, `last_wake_error`.

Constraints: the live state comes from the state probe; for a non-live session the probe fields are absent and model/provider come from the launch record. The state is not invented.

### set_model

Switches the provider/model of a live teammate in the own domain. Applied from the next turn; the first turn on the new pair re-reads the full history (the old pair's prompt cache is invalidated).

Parameters:

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| operation | string | yes | — | `"set_model"` |
| name | string | yes | — | the teammate's mail name |
| model | string | yes | — | target model id from the target provider's catalog |
| provider | string | no | current provider | target provider |

Returns `set_model`: `applied`, `name`, `session_id`, `old_provider`/`old_model`, `new_provider`/`new_model`, `event_id`, `publication_pending`, `metadata_pending`, `delivery_pending`, `reason`/`roster` (on refusal).

Constraints: an unknown name is a soft refusal with a list of addresses. A delegate name or a qualified foreign name is a refusal with an explanation. A miss against the model catalog is an error.

## Behavior and constraints

### Agent types

Built-in definitions: `general-purpose`, `Explore`, `Planner`, `arbiter`. An unknown `subagent_type` is an error with the list of available types, without silent substitution.

- `general-purpose` — full tool set, autonomous execution of multi-step tasks; not one-shot, not read-only.
- `Explore` — a one-shot read-only researcher; tool set `Files`, `Memory`, `Search`, `History`, `Shell`; returns one report.
- `Planner` — a one-shot read-only planner; the same tool set; returns one report.
- `arbiter` — the judge role; tool set `Files`, `Memory`, `Agent`, `Plan`; not one-shot, not read-only; receives a judging briefing at launch.

One-shot types: `Explore` and `Planner`. For `arbiter`, a judging instruction is added to the briefing at launch; other tasks are passed through unchanged.

### Isolation

- `isolation:"worktree"` — a temporary git worktree over the git CLI from the session `cwd` repository's HEAD; directory `<temp>/kot-agent-worktrees/agent-wt-<id8>`, branch `agent/<id8>`; this directory becomes the child's working directory. On completion the cleanup is idempotent: with no changes the worktree is removed, with changes it is kept and reported in the output.
- `isolation:"remote"` — unavailable; the call returns an error.
- `cwd` — an absolute override of the child's working directory; mutually exclusive with `worktree`.

Every child has an isolated session and an isolated virtual files layer. The child's edits are synced to disk before it finishes; your own edits are synced before launching the child if it must see them.

### Mail addressing

The `to` recipient is an owned agent (an explicit name or an unnamed delegate's UUID), `team:<name>` (broadcast to active members except the sender), `orchestrator` (one's own lead), or cross-domain via `@<domain>`. The `team:` prefix and the `orchestrator` address are reserved; a name cannot contain `@`.

### Provider/model/reasoning-effort inheritance and override

- `model`: with no value, the parent's model is inherited. Aliases `sonnet`→`claude-sonnet-5`, `opus`→`claude-opus-5`, `haiku`→`claude-haiku-4-5-20251001` (case-insensitive); any other non-empty id is passed through as-is, without silent substitution. When the child's provider publishes a model catalog, an id outside that catalog and a model that is not chat-eligible are rejected before the child starts; a provider without a catalog receives the id as-is.
- `provider`: with no value, the parent's provider is inherited. On override, the entire child launch runs on this provider; an unregistered name is an error. With a `provider` set, the passed `model` or that provider's default is used; the parent's model does not leak across providers.
- `effort`: an explicit value wins over inheritance; cross-provider children do not inherit the parent's level; `off` disables reasoning where the provider supports it.

### seed_history (fork_from)

A one-shot `fork_id` from `History{fork}` (alias `fork_from`; pass only one). Redemption is checked against the owning session; a miss is an error. A session that itself carries the fork-gate marker cannot fork again. For `hire`, if the failure occurs before the teammate actually starts, the fork is returned — only a teammate that actually started consumes it.

### The until gate (delegate only)

The `until` object runs a "worker → check" loop: after each worker turn the `check` command runs in its working directory; exit code 0 means done; otherwise the failure output is fed into the worker's next turn.

| field | type | required | default | meaning |
|---|---|---|---|---|
| check | string | yes | — | the check command; code 0 = done |
| max_iterations | integer | no | 100 | the worker-iteration ceiling (minimum 1, hard ceiling 100) |
| timeout | integer | no | 43200000 | total loop budget in milliseconds (hard ceiling 43200000 = 12 h) |
| supervisor | string or object | no | — | an agent type name or `{agent, model?, provider?}` |

Two consecutive identical (normalized) gate outputs are a "no progress" escalation. On a passing gate, the supervisor issues a verdict as the first line `VERDICT: accept|continue|escalate` (case-insensitive): `accept` completes, `continue` feeds specifics to the worker, `escalate` fails the loop. A missing verdict or an unknown token is a loud failure, without silent acceptance. Values above the ceilings are an error, without silent truncation.

### Orchestration constraints for children and teammates

- Children and teammates do not orchestrate: arbitrary `spawn`/`delegate`/`hire`/`workflow` are forbidden. Only the orchestrating (main) session can spawn sub-agents.
- A session with a restricted research grant (declared in its first message) can synchronously spawn one-shot read-only `Explore`/`Planner`; a teammate can additionally use `Search{mode:"smart"}`.
- `send` is available to a running child as well: it can reply to the lead at any point during the work.
- Teammates in one team share a common `Plan` board.

### Limits and defaults

- The maximum tool result size is 100000 characters.
- The `send` message body is 30000 characters (truncation with a marker).
- The delegate and workflow-run `<task-notification>` summary is 2000 characters.
- The ordinary (parked) delegate's lifetime limit is 12 hours.
- The `until` iteration ceiling is 100; the `until` budget is 43200000 ms (12 h).
- The workflow schema version is 1; pipeline nesting depth is 3.
- The `send` warning names at most 10 unsynced sender paths.

### Error handling

- The child's intermediate tool calls are hidden from the parent: only the final report enters the parent's turn.
- A field of a foreign operation (for example, `name` in `spawn`) is an error, never silently swallowed.
- Soft refusals (`delivered:false`, `stopped:false`, `applied:false`, an unknown `run_id`, an unknown `provider` filter) are returned as an ordinary result with `reason` and a list of addresses/names — the model corrects the recipient and retries.
- Interruption: a synchronous `spawn` links the parent's abort to the child; `delegate` does not link it — the child survives the parent's interruption.

## Examples

Synchronous code researcher:

```json
{"operation": "spawn", "description": "find token validation", "prompt": "Find where JWT is validated in the project and return the path and line number", "subagent_type": "Explore"}
```

Background delegate with a name and a completion gate:

```json
{"operation": "delegate", "description": "fix the build", "prompt": "Fix the compilation errors and get a green build", "name": "builder-1", "until": {"check": "npm run build", "max_iterations": 50}}
```

Hiring a reviewer teammate into a team:

```json
{"operation": "hire", "description": "reviewer", "prompt": "Review the changes against the project rules and send verdicts by mail", "name": "reviewer", "team_name": "alpha"}
```

Messaging a teammate and stopping it:

```json
{"operation": "send", "to": "reviewer", "message": "also check the error handling in the new module"}
```

```json
{"operation": "stop", "name": "reviewer"}
```

Switching a teammate's model and inspecting agents:

```json
{"operation": "set_model", "name": "reviewer", "provider": "anthropic", "model": "claude-opus-5"}
```

```json
{"operation": "status"}
```
