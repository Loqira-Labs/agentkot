# Plan

A tool for maintaining the agent's plan and tasks: replaces the current session's checklist, keeps a persistent task graph on disk with owners and dependencies, and moves the session into a read-only planning mode and back. One tool with six operations distinguished by the `operation` field: `plan`, `task_add`, `task_update`, `task_list`, `enter_planning`, `finish_planning`.

## Operations

### plan

Replaces the session's checklist entirely with a single call. The checklist lives in memory and belongs to a single session; it does not survive an agent restart.

The `todos` parameter is an array of items. Each item:

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| content | string | yes | — | the text of the checklist item; minimum 1 non-empty character |
| status | string | yes | — | the item state: `pending`, `in_progress`, or `completed` |
| activeForm | string | yes | — | the spinner label (present continuous, e.g. "Creating tests"); minimum 1 non-empty character |

Returns the previous checklist (`old_todos`) and the submitted list as-is (`new_todos`).

Clearing behavior: if all submitted items have the `completed` status (an empty list `[]` also satisfies this condition), the stored checklist is cleared, and the response still returns the original submitted list. In all other cases the stored checklist becomes equal to the submitted list.

### task_add

Creates a single task in the persistent task graph.

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| subject | string | yes | — | a brief task title; minimum 1 non-empty character |
| description | string | yes | — | what needs to be done |
| activeForm | string | no | absent | the spinner label |
| owner | string | no | absent (the owner is the agent itself) | the name of the executing agent, the delegation target |
| metadata | object | no | empty | arbitrary attached metadata |

Returns the created task: its identifier (`id`, a decimal string) and title (`subject`). A new task is assigned the `pending` status.

### task_update

Changes a single task: fields, status, dependencies, owner, metadata.

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| task_id | string | yes | — | the task identifier; minimum 1 non-empty character |
| subject | string | no | unchanged | the new title |
| description | string | no | unchanged | the new description |
| activeForm | string | no | unchanged | the new spinner label |
| status | string | no | unchanged | `pending`, `in_progress`, `completed`, or `deleted` |
| addBlocks | array of strings | no | empty | identifiers of tasks that this task blocks |
| addBlockedBy | array of strings | no | empty | identifiers of tasks that block this task |
| owner | string | no | unchanged | the new owner |
| metadata | object | no | unchanged | a patch of metadata by keys; the value `null` removes a key |

Returns `success`, `task_id`, a list of actually changed fields (`updated_fields`), an optional `error`, and an optional `status_change` with the transition `from`/`to`.

The value `status: "deleted"` is a pseudo-status: the task is deleted, and its identifier is cascadingly removed from the `blocks` and `blocked_by` lists of all other tasks. Updating a non-existent task is not an error: `success: false` is returned with `error: "Task not found"`.

Metadata is merged by keys: missing keys are added, and a key with the JSON value `null` is removed.

### task_list

Lists the tasks of the current task board.

Takes no parameters except `operation`.

Returns a list of visible tasks sorted by numeric identifier. Tasks that have a truthy `_internal` key in their metadata are hidden. From each task's `blocked_by` list, completed (`completed`) blocking tasks are excluded — only open blockers remain. Each record has `id`, `subject`, `status`, an optional `owner`, and `blocked_by`.

### enter_planning

Moves the session into planning mode — a read-only mode, for research.

Takes no parameters except `operation`.

Returns a confirmation message. The operation is idempotent: a repeated call in planning mode is allowed. Planning mode is a property of the main session; a sub-agent cannot enter it and receives an error.

### finish_planning

Ends planning mode: saves the plan text, restores the session's previous mode, and ends the turn for the user to review.

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| plan | string | no | the existing plan file on disk is used | the finished plan text in markdown |

Returns `plan` (the text from the parameter or read from disk), an optional `file_path` (the path to the plan file), and `restored_mode` (the restored mode, e.g. `default`).

The plan text is written to the file `<plans_root>/<session-id>.md`. If the `plan` parameter is not provided, the contents of the already existing plan file are substituted into the response. A plan-file write error does not interrupt the operation. After the call the turn ends for the user to review.

Calling `finish_planning` outside planning mode is rejected at the validation stage and in the call itself.

## Behavior and limitations

- The `task_list` and `enter_planning` operations are read-only. The `plan` operation is not safe for concurrent execution (a non-atomic checklist update). Operations on tasks on disk are serialized by a lock.
- The `plan` operation's checklist is stored in memory and belongs to a single session; checklists of different sessions are not mixed, and the checklist does not survive a restart. The persistent task graph is stored on disk.
- The persistent task graph is stored as one JSON file per task in the directory `<tasks_root>/<list-id>/<id>.json`. The board identifier (`list-id`) defaults to the session identifier; for an active teammate hired into a team, it is the team's shared board `team-<owner8>-<name>`. The environment variable `KOT_TASK_LIST_ID` overrides the board identifier.
- Task identifiers are decimal strings allocated as "maximum existing + 1". Deleting the task with the largest identifier does not return it for reuse within the process.
- Task mutations are serialized by an exclusive lock on the file `<list-id>/.lock`: 30 attempts, 5–100 ms backoff. Task files are written atomically (a temporary file + rename); reads do not require the lock.
- Dependency links are maintained symmetrically: the record "A blocks B" adds B to A's `blocks` and A to B's `blocked_by`. Adding the same link repeatedly is idempotent. A reference to a non-existent task creates no link and is not an error.
- When a task is created, the `TaskCreated` event fires: a blocking decision rolls back the creation and returns an error. When a task moves to `completed`, `TaskCompleted` fires: a blocking decision leaves the status unchanged and returns `success: false` without an error.
- A corrupted task file (existing but not parseable) gives an explicit error with the file name; foreign `.json` files in the board directory whose stem is not a number are skipped and not counted as tasks.
- The maximum result size of an operation is 100 000 characters.

## Examples

Replacing the checklist:

```json
{"operation": "plan", "todos": [
  {"content": "Study the code", "status": "completed", "activeForm": "Studying the code"},
  {"content": "Write the documentation", "status": "in_progress", "activeForm": "Writing the documentation"}
]}
```

Clearing the checklist:

```json
{"operation": "plan", "todos": []}
```

Creating a task with an owner:

```json
{"operation": "task_add", "subject": "Add a cache", "description": "Implement a request cache", "owner": "worker-1"}
```

Creating a blocker: task 2 waits for task 1 to complete:

```json
{"operation": "task_update", "task_id": "2", "addBlockedBy": ["1"]}
```

Moving a task to the completed state:

```json
{"operation": "task_update", "task_id": "1", "status": "completed"}
```

Deleting a task:

```json
{"operation": "task_update", "task_id": "3", "status": "deleted"}
```

Finishing planning with a plan passed in:

```json
{"operation": "finish_planning", "plan": "# Plan\n1. Do A\n2. Do B"}
```
