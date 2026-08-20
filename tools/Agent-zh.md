# Agent

创建并管理子代理、队友和长期流水线。单一工具；动作由 `operation` 字段选择；别名 `Task`。

## Operations

### spawn

同步启动一个子代理：父会话等待其完成，并在同一轮中收到最终报告。适用于其结果需立即用于下一步的任务。

参数：

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| operation | string | yes | — | `"spawn"` |
| description | string | yes | — | 任务的简短（3–5 词）描述 |
| prompt | string | yes | — | 子代理的任务 |
| subagent_type | string | no | general-purpose | 代理类型（见 "Agent types"） |
| model | string | no | inherited | 模型覆盖：别名 `sonnet`/`opus`/`haiku` 或规范 id |
| provider | string | no | inherited | 提供方覆盖（注册名称） |
| effort | string | no | inherited | 推理强度：`off`/`low`/`medium`/`high`/`xhigh`/`max` |
| isolation | string | no | — | `"worktree"` 或 `"remote"`；`remote` 不可用 |
| cwd | string | no | inherited | 子代理工作目录的绝对路径 |
| seed_history | string | no | — | 来自 `History{fork}` 的一次性 `fork_id`；别名 `fork_from` |

返回 `completed` 结果：`agent_id`、`agent_type`、`content`（报告中的文本块）、`total_tool_use_count`、`total_duration_ms`、`total_tokens`、`usage`（四个令牌计数器）、`prompt`、`worktree_path`/`worktree_branch`（若保留了工作树）、`transcript_path`（子代理完整会话记录的路径）。若子代理未返回文本，则替换为无输出标记和会话记录路径。

约束：在子代理完成前阻塞父会话的轮次。子代理的中间工具调用不会出现在父会话的会话记录中。

### delegate

在后台启动一个子代理，并立即返回 `async_launched` 启动确认；最终结果随后以 `<task-notification>` 的形式在单独的轮次中到达。工作运行期间，可以给该委派代理发消息并按地址停止它。

参数 —— 与 `spawn` 相同，另加：

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| name | string | no | agent UUID | 显式的可寻址地址；在成功启动后注册 |
| until | object | no | — | 完成门禁规格（见 "The until gate"） |

返回 `async_launched`：`agent_id`、`address`（显式名称或 UUID）、`description`、`prompt`、`output_file`（计划的报告路径 `<sidechains>/<agent-uuid>.output.md`）、`can_read_output_file`。

约束：名称被占用是错误。普通委派代理会挂起，绝对存活时限为 12 小时。

### hire

创建一个持久可唤醒的队友：一个处理简报、进入等待、并由消息（`send`）唤醒的对等会话。存活至 `stop` 或主会话的关闭级联。

参数：

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| operation | string | yes | — | `"hire"` |
| description | string | yes | — | 队友的任务标签 |
| prompt | string | yes | — | 简报 —— 队友的第一轮 |
| subagent_type | string | no | general-purpose | 代理类型 |
| model | string | no | inherited | 队友整个生命期的模型覆盖 |
| provider | string | no | inherited | 队友整个生命期的提供方覆盖 |
| effort | string | no | inherited | 队友的推理强度 |
| name | string | yes | — | 可寻址名称（邮箱键） |
| team_name | string | no | no team | 队友加入的团队 |
| mode | string | no | — | 仅 `"plan"` —— 进入规划模式 |
| isolation | string | no | — | `"worktree"` 或 `"remote"`；`remote` 不可用 |
| seed_history | string | no | — | 一次性 `fork_id`；别名 `fork_from` |

返回 `teammate_spawned`：`agent_id`、`session_id`（队友的会话标识）、`name`、`team_name`。

约束：简报在后台运行；其失败会邮寄给主代理（从不静默丢失）。名称不能以 `team:` 开头，不能是 `orchestrator`，不能包含 `@`。

### send

向代理地址投递一条消息。消息被排入目标会话的重入队列，并在其下一个轮次边界被读取。对等操作：运行中的子代理也可以发送消息（回复主代理）。

参数：

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| operation | string | yes | — | `"send"` |
| to | string | yes | — | 收件人：一个受管的代理、`team:<name>`、`orchestrator`；跨域通过 `@<domain>` |
| message | string | yes | — | 消息正文；超过 30000 字符时被截断并加标记 |

返回 `message_sent`：`to`、`delivered`、`reason`（拒绝时）、`roster`（拒绝时的已知地址）、`truncated`、`delivered_to`（广播时的列表）、`unsynced_paths`（关于发送方未同步路径的警告）。

约束：未知或非存活的收件人是软拒绝 `delivered:false` 并附地址列表；只有邮件路由器缺失才是错误。向团队广播会排除发送方自身。跨域投递不披露发送方的文件路径。

### stop

按地址停止一个代理。队友被解散：当前工作被中断、会话被关闭、名称被注销、临时工作树若无改动则移除、若有改动则保留。委派代理收到停止请求并发送 `aborted` 通知。

参数：

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| operation | string | yes | — | `"stop"` |
| name | string | yes | — | 要停止的代理的地址 |

返回 `stopped`：`name`、`stopped`、`kind`（`"teammate"`/`"delegate"`）、`reason`（拒绝时）、`roster`。

约束：未知地址是软拒绝并附地址列表。受限的外部地址（`name@domain`）在所有权上是软拒绝：生命期不跨域。

### workflow

运行、检查或停止一个已声明的 TOML 流水线。一次运行把流水线铺展到持久任务板上（每个步骤一个任务）；就绪的步骤并发运行；最终结果以 `<task-notification>` 的形式到达。

参数：

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| operation | string | yes | — | `"workflow"` |
| action | string | no | `"run"` | `"run"`/`"status"`/`"stop"` |
| name | string | for run | — | 声明文件的文件名主干 `<cwd>/.kot/workflows/<name>.toml` |
| definition | string | for run | — | 内联 TOML 声明文本 |
| resume | string | for run | — | 已有运行的 id `wf-<8hex>` |
| run_id | string | for status/stop | — | 要检查/停止的运行的 id |

对 `run`，`name`/`definition`/`resume` 三者必须恰好提供一个。返回 `workflow_started`（`run_id`、`name`、`list_id`、`steps`、`resumed`）、`workflow_status`（`run_id`、`name`、`state`、`steps`、`usage`、`reason`）或 `workflow_stopped`（`run_id`、`stopped`、`reason`）。

声明格式（TOML）：

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

模式规则：`version` 为必填且等于 1；未知字段和未知版本是错误。一个步骤在 `agent`/`team`/`workflow`/`ask` 中恰好有一个执行器。带 `agent` 或 `team` 执行器的步骤必须携带非空 `prompt`；`workflow` 或 `ask` 步骤则完全不能携带 `prompt`。`model` 和 `provider` 仅允许出现在 `agent` 步骤上 —— 在团队中，由每个成员各自携带。步骤名为 `[A-Za-z0-9_-]+`，禁止重复。`depends_on` 构成无环图（未知引用、自依赖、环均为错误）；`inputs` 是 `depends_on` 的子集。团队中角色唯一，且不允许空团队。`transactional` 适用于 `agent` 和 `team` 步骤（对带 `until` 门禁的步骤默认为 true，否则为 false）。`until` 门禁不适用于 `ask` 步骤。流水线嵌套深度至多 3。`until` 的值按运行限制校验（仅向下），监督代理必须是已知的代理类型。`on_error.retry` 是额外尝试次数（默认 0）；`on_error.action` 为 `stop`（运行失败，级联停止）或 `escalate`（暂停并邮寄所有者，可恢复）。

约束：声明在产生任何副作用前被完整校验。运行状态（`running`/`paused`/`completed`/`failed`/`aborted`，或软未命中时的 `unknown`）保存在任务板上和 `run.json` 中；已完成的运行即使在进程重启后也可检查。`resume` 时，已完成的步骤被跳过，其余步骤以全新的重试预算重新启动。停止是级联的 —— 整个节点子树被取消；暂停的运行被直接终结。

### list_providers

列出可用于跨提供方子代理启动的提供方。无参数。

返回 `providers_listed` 及列表：`name`、`is_default`（当前会话的提供方）、`spawnable`（该名称能否传入 `provider`）、`model_count`。会话自身的提供方排在最前。

### list_models

列出一个提供方的模型（或所有提供方的模型）及其能力。

参数：

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| operation | string | yes | — | `"list_models"` |
| provider | string | no | all providers | 按注册的提供方名称过滤 |

返回 `models_listed`：`provider`、`providers`（每个提供方：`name`、`is_default`、`spawnable`、`models`）、`reason`/`known_providers`（来自未知过滤器的软未命中时）。模型卡：`id`、`display_name`、`kind`、`for_chat`、`context_window`、`max_output_tokens`、`family`、`input_modalities`、`output_modalities`、`reasoning`、`tools`、`streaming`、`structured_outputs`、`preview`、`notes`。

只有 `for_chat: true` 的模型能驱动代理会话；`for_chat: false` 的卡是媒体/嵌入行，通过 Media 工具生成。

### current_model

读取调用会话当前的模型状态。无参数。

返回 `current_model`：`provider`、`model`、`effort`、`routing_version`、`capabilities`（上下文窗口、最大输出令牌、输入模态、是否存在推理）、`frozen_head_format_version`、`frozen_head_digest`、`fallback_lease`、`coordinator_generation`、`coordinator_next_seq`、`pending`（有序的待定变更）。

约束：该操作在会话发布其状态时可用；否则返回错误。

### current_session

读取调用会话当前的工作目录、持久上下文和模式。无参数。

返回 `current_session`：`session_id`、`cwd`、`persistent_context`、`planning_mode`、`routing_version`、`frozen_head_digest`、`coordinator_generation`、`coordinator_next_seq`、`pending`。

约束：同 `current_model`。

### status

列出调用会话自身域内的代理：队友（按名称）和委派代理（按注册顺序），附生命期和存活状态。无参数。

返回 `status` 及行：`name`、`address`（仅对无名称的委派代理、在其运行期间）、`kind`、`team`、`agent_id`、`session_id`、`lifecycle`、`live_state`（`running`/`idle`/`closed`，或缺省）、`model`、`provider`、`pending_mail`、`last_wake_error`。

约束：存活状态来自状态探测；对非存活会话，探测字段缺省，model/provider 来自启动记录。状态不凭空捏造。

### set_model

切换自身域内一个存活队友的提供方/模型。从下一轮起生效；在新配对上第一轮会重新读取完整历史（旧配对的提示缓存被作废）。

参数：

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| operation | string | yes | — | `"set_model"` |
| name | string | yes | — | 队友的邮件名称 |
| model | string | yes | — | 来自目标提供方目录的目标模型 id |
| provider | string | no | current provider | 目标提供方 |

返回 `set_model`：`applied`、`name`、`session_id`、`old_provider`/`old_model`、`new_provider`/`new_model`、`event_id`、`publication_pending`、`metadata_pending`、`delivery_pending`、`reason`/`roster`（拒绝时）。

约束：未知名称是软拒绝并附地址列表。委派代理名称或受限的外部名称是带说明的拒绝。对模型目录的未命中是错误。

## Behavior and constraints

### Agent types

内置定义：`general-purpose`、`Explore`、`Planner`、`arbiter`。未知的 `subagent_type` 是错误并附可用类型列表，不做静默替换。

- `general-purpose` —— 完整工具集，自主执行多步任务；非一次性，非只读。
- `Explore` —— 一次性只读研究员；工具集 `Files`、`Memory`、`Search`、`History`、`Shell`；返回一份报告。
- `Planner` —— 一次性只读规划者；同样的工具集；返回一份报告。
- `arbiter` —— 裁判角色；工具集 `Files`、`Memory`、`Agent`、`Plan`；非一次性，非只读；启动时接收裁判简报。

一次性类型：`Explore` 和 `Planner`。对 `arbiter`，启动时在简报中加入裁判指令；其他任务原样传递。

### Isolation

- `isolation:"worktree"` —— 基于会话 `cwd` 仓库 HEAD 的 git CLI 之上的临时 git 工作树；目录 `<temp>/kot-agent-worktrees/agent-wt-<id8>`，分支 `agent/<id8>`；该目录成为子代理的工作目录。完成时清理是幂等的：无改动则移除工作树，有改动则保留并在输出中报告。
- `isolation:"remote"` —— 不可用；调用返回错误。
- `cwd` —— 子代理工作目录的绝对覆盖；与 `worktree` 互斥。

每个子代理都有隔离的会话和隔离的虚拟文件层。子代理的编辑在其完成前同步到磁盘；你自己的编辑在启动子代理前同步，如果它需要看到这些编辑。

### Mail addressing

`to` 收件人是一个受管的代理（显式名称或无名称委派代理的 UUID）、`team:<name>`（广播给除发送方外的活跃成员）、`orchestrator`（自己的主代理），或通过 `@<domain>` 跨域。`team:` 前缀和 `orchestrator` 地址被保留；名称不能包含 `@`。

### Provider/model/reasoning-effort inheritance and override

- `model`：无值时，继承父会话的模型。别名 `sonnet`→`claude-sonnet-5`、`opus`→`claude-opus-5`、`haiku`→`claude-haiku-4-5-20251001`（不区分大小写）；任何其他非空 id 原样传递，不做静默替换。当子代理的提供方发布模型目录时，目录之外的 id 和不具备聊天资格的模型会在子代理启动前被拒绝；没有目录的提供方原样接收该 id。
- `provider`：无值时，继承父会话的提供方。覆盖时，整个子代理启动都运行在该提供方上；未注册名称是错误。设置了 `provider` 时，使用传入的 `model` 或该提供方的默认模型；父会话的模型不会跨提供方泄漏。
- `effort`：显式值优先于继承；跨提供方的子代理不继承父会话的级别；`off` 在提供方支持时禁用推理。

### seed_history (fork_from)

来自 `History{fork}` 的一次性 `fork_id`（别名 `fork_from`；只能传一个）。赎回针对所属会话校验；未命中是错误。自身携带 fork 门禁标记的会话不能再 fork。对 `hire`，若失败发生在队友实际启动之前，则 fork 被退回 —— 只有实际启动的队友才消耗它。

### The until gate (delegate only)

`until` 对象运行一个「worker → check」循环：在每次 worker 轮次后，`check` 命令在其工作目录中运行；退出码 0 表示完成；否则失败输出被喂入 worker 的下一轮。

| field | type | required | default | meaning |
|---|---|---|---|---|
| check | string | yes | — | 检查命令；码 0 = 完成 |
| max_iterations | integer | no | 100 | worker 迭代上限（最小 1，硬上限 100） |
| timeout | integer | no | 43200000 | 总循环预算，毫秒（硬上限 43200000 = 12 小时） |
| supervisor | string or object | no | — | 代理类型名称或 `{agent, model?, provider?}` |

连续两次相同（规范化后）的门禁输出是「无进展」升级。门禁通过时，监督代理以第一行 `VERDICT: accept|continue|escalate`（不区分大小写）给出裁决：`accept` 完成，`continue` 把具体细节喂给 worker，`escalate` 使循环失败。缺失裁决或未知令牌是响亮失败，不做静默接受。高于上限的值是错误，不做静默截断。

### Orchestration constraints for children and teammates

- 子代理和队友不做编排：任意的 `spawn`/`delegate`/`hire`/`workflow` 被禁止。只有编排（主）会话能启动子代理。
- 具有受限研究授权的会话（在其首条消息中声明）能同步启动一次性只读的 `Explore`/`Planner`；队友还能额外使用 `Search{mode:"smart"}`。
- `send` 对运行中的子代理也可用：它可以在工作期间随时回复主代理。
- 同一团队的队友共享一个公共 `Plan` 任务板。

### Limits and defaults

- 最大工具结果大小为 100000 字符。
- `send` 消息正文为 30000 字符（截断加标记）。
- 委派代理和流水线运行的 `<task-notification>` 摘要为 2000 字符。
- 普通（挂起）委派代理的存活时限为 12 小时。
- `until` 迭代上限为 100；`until` 预算为 43200000 毫秒（12 小时）。
- 流水线模式版本为 1；流水线嵌套深度为 3。
- `send` 警告至多列出 10 个发送方未同步路径。

### Error handling

- 子代理的中间工具调用对父会话隐藏：只有最终报告进入父会话的轮次。
- 外部操作中的字段（例如 `spawn` 中的 `name`）是错误，从不静默吞掉。
- 软拒绝（`delivered:false`、`stopped:false`、`applied:false`、未知 `run_id`、未知 `provider` 过滤器）作为普通结果返回，附 `reason` 和地址/名称列表 —— 模型更正收件人并重试。
- 中断：同步 `spawn` 把父会话的中止链接到子代理；`delegate` 不链接 —— 子代理在父会话中断后存活。

## Examples

同步代码研究员：

```json
{"operation": "spawn", "description": "find token validation", "prompt": "Find where JWT is validated in the project and return the path and line number", "subagent_type": "Explore"}
```

带名称和完成门禁的后台委派代理：

```json
{"operation": "delegate", "description": "fix the build", "prompt": "Fix the compilation errors and get a green build", "name": "builder-1", "until": {"check": "npm run build", "max_iterations": 50}}
```

将一个审查队友招入团队：

```json
{"operation": "hire", "description": "reviewer", "prompt": "Review the changes against the project rules and send verdicts by mail", "name": "reviewer", "team_name": "alpha"}
```

向队友发消息并停止它：

```json
{"operation": "send", "to": "reviewer", "message": "also check the error handling in the new module"}
```

```json
{"operation": "stop", "name": "reviewer"}
```

切换队友的模型并检查代理：

```json
{"operation": "set_model", "name": "reviewer", "provider": "anthropic", "model": "claude-opus-5"}
```

```json
{"operation": "status"}
```
