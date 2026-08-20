# Plan

用于维护代理的计划和任务的工具：替换当前会话的清单，在磁盘上保存带负责人和依赖的持久化任务图，并使会话进入只读的规划模式并退出。一个工具，六个操作，通过 `operation` 字段区分：`plan`、`task_add`、`task_update`、`task_list`、`enter_planning`、`finish_planning`。

## 操作

### plan

通过一次调用完整替换会话的清单。清单存放在内存中，归属于单个会话；它不会在代理重启后保留。

`todos` 参数是一个条目数组。每个条目：

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| content | string | 是 | — | 清单条目的文本；至少 1 个非空字符 |
| status | string | 是 | — | 条目状态：`pending`、`in_progress` 或 `completed` |
| activeForm | string | 是 | — | 加载指示标签（现在进行时，例如 “正在创建测试”）；至少 1 个非空字符 |

返回之前的清单（`old_todos`）和原样提交的列表（`new_todos`）。

清空行为：如果所有提交的条目都具有 `completed` 状态（空列表 `[]` 也满足此条件），则清空已存储的清单，而响应仍返回原始提交的列表。在所有其他情况下，已存储的清单会变为与提交的列表一致。

### task_add

在持久化任务图中创建一个任务。

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| subject | string | 是 | — | 任务的简短标题；至少 1 个非空字符 |
| description | string | 是 | — | 需要完成的内容 |
| activeForm | string | 否 | 无 | 加载指示标签 |
| owner | string | 否 | 无（负责人为代理本身） | 执行代理的名称，即委派目标 |
| metadata | object | 否 | 空 | 任意附加元数据 |

返回创建的任务：其标识符（`id`，一个十进制字符串）和标题（`subject`）。新任务被赋予 `pending` 状态。

### task_update

修改一个任务：字段、状态、依赖、负责人、元数据。

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| task_id | string | 是 | — | 任务标识符；至少 1 个非空字符 |
| subject | string | 否 | 不变 | 新标题 |
| description | string | 否 | 不变 | 新描述 |
| activeForm | string | 否 | 不变 | 新的加载指示标签 |
| status | string | 否 | 不变 | `pending`、`in_progress`、`completed` 或 `deleted` |
| addBlocks | string 数组 | 否 | 空 | 此任务所阻塞的任务的标识符 |
| addBlockedBy | string 数组 | 否 | 空 | 阻塞此任务的任务的标识符 |
| owner | string | 否 | 不变 | 新负责人 |
| metadata | object | 否 | 不变 | 按键合并的元数据补丁；值为 `null` 时移除该键 |

返回 `success`、`task_id`、实际变更的字段列表（`updated_fields`）、可选的 `error`，以及可选的 `status_change`，其中包含 `from`/`to` 的转换。

`status: "deleted"` 的值是一个伪状态：任务被删除，其标识符会从所有其他任务的 `blocks` 和 `blocked_by` 列表中级联移除。更新不存在的任务不是错误：返回 `success: false` 并带有 `error: "Task not found"`。

元数据按键合并：缺失的键会被添加，JSON 值为 `null` 的键会被移除。

### task_list

列出当前任务板的任务。

除 `operation` 外不接受任何参数。

返回按数字标识符排序的可见任务列表。元数据中具有真值 `_internal` 键的任务会被隐藏。从每个任务的 `blocked_by` 列表中排除已完成（`completed`）的阻塞任务——仅保留未完成的阻塞项。每条记录包含 `id`、`subject`、`status`、可选的 `owner` 和 `blocked_by`。

### enter_planning

使会话进入规划模式——一种只读模式，用于研究。

除 `operation` 外不接受任何参数。

返回确认消息。该操作是幂等的：允许在规划模式下重复调用。规划模式是主会话的属性；子代理无法进入，并会收到一个错误。

### finish_planning

结束规划模式：保存计划文本，恢复会话之前的模式，并结束本轮以供用户审阅。

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| plan | string | 否 | 使用磁盘上已有的计划文件 | 完成的计划文本，markdown 格式 |

返回 `plan`（来自参数的文本或从磁盘读取的文本）、可选的 `file_path`（计划文件的路径）以及 `restored_mode`（恢复的模式，例如 `default`）。

计划文本被写入文件 `<plans_root>/<session-id>.md`。如果未提供 `plan` 参数，则将已存在的计划文件的内容代入响应。计划文件写入错误不会中断该操作。调用结束后，本轮结束以供用户审阅。

在规划模式之外调用 `finish_planning` 会在校验阶段以及调用本身中被拒绝。

## 行为与限制

- `task_list` 和 `enter_planning` 操作是只读的。`plan` 操作不适合并发执行（清单更新不是原子的）。磁盘上任务的操作通过锁来串行化。
- `plan` 操作的清单存储在内存中，归属于单个会话；不同会话的清单不会混淆，清单也不会在重启后保留。持久化任务图存储在磁盘上。
- 持久化任务图在目录 `<tasks_root>/<list-id>/<id>.json` 中每个任务存储为一个 JSON 文件。任务板标识符（`list-id`）默认等于会话标识符；对于加入团队的活动队友，则是团队的共享任务板 `team-<owner8>-<name>`。环境变量 `KOT_TASK_LIST_ID` 会覆盖任务板标识符。
- 任务标识符是十进制字符串，按 “现有最大值 + 1” 分配。删除具有最大标识符的任务后，该标识符不会在进程内被重新使用。
- 任务修改通过对文件 `<list-id>/.lock` 的独占锁来串行化：30 次尝试，5–100 毫秒退避。任务文件以原子方式写入（临时文件 + 重命名）；读取不需要锁。
- 依赖关系对称维护：“A 阻塞 B” 这条记录会把 B 加入 A 的 `blocks`，并把 A 加入 B 的 `blocked_by`。重复添加同一条关系是幂等的。引用不存在的任务不会创建任何关系，也不是错误。
- 创建任务时会触发 `TaskCreated` 事件：阻塞性判定会回滚创建并返回错误。当任务变为 `completed` 时会触发 `TaskCompleted`：阻塞性判定会保持状态不变，并返回 `success: false` 而不带错误。
- 损坏的任务文件（存在但无法解析）会给出带有文件名的明确错误；任务板目录中文件名主干不是数字的无关 `.json` 文件会被跳过，不计入任务。
- 一次操作的最大结果大小为 100 000 个字符。

## 示例

替换清单：

```json
{"operation": "plan", "todos": [
  {"content": "研究代码", "status": "completed", "activeForm": "正在研究代码"},
  {"content": "编写文档", "status": "in_progress", "activeForm": "正在编写文档"}
]}
```

清空清单：

```json
{"operation": "plan", "todos": []}
```

创建带负责人的任务：

```json
{"operation": "task_add", "subject": "添加缓存", "description": "实现一个请求缓存", "owner": "worker-1"}
```

创建阻塞关系：任务 2 等待任务 1 完成：

```json
{"operation": "task_update", "task_id": "2", "addBlockedBy": ["1"]}
```

将任务移动到已完成状态：

```json
{"operation": "task_update", "task_id": "1", "status": "completed"}
```

删除任务：

```json
{"operation": "task_update", "task_id": "3", "status": "deleted"}
```

结束规划并传入计划：

```json
{"operation": "finish_planning", "plan": "# 计划\n1. 执行 A\n2. 执行 B"}
```
