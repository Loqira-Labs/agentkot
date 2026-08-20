# Tools

一个元工具，用于管理当前会话中智能体的当前工具集：添加、移除、列出、搜索、检查和调用工具。这是智能体在工作过程中扩充或缩减自身能力集合的唯一入口。

## 操作

该工具通过 `operation` 字段区分操作。允许的值：`add`、`remove`、`list`、`search`、`inspect`、`invoke`。

### add

根据给定配置，将由预注册能力生成的工具添加到当前会话工具集中。在发布版本中恰好有一种能力可用——`cli-wrapper`。

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| `operation` | string | yes | — | 值 `"add"` |
| `capability` | string | yes | — | 能力名称；本版本中唯一允许的值是 `"cli-wrapper"` |
| `config` | object/value | 对 `cli-wrapper` 必填 | — | 能力配置；可以是内联对象，或 `{"config_path": "<path>"}` 以从文件加载。对 `cli-wrapper` 必填，且必须至少包含 `name`、`script` 和 `commands` |
| `activate` | string | no | `"immediate"` | 工具何时变为可调用：`"immediate"`（当前回合）或 `"deferred"`（下一回合） |

`config_path` 相对于会话工作目录解析：路径会被规范化（展开 `~`、规范化分隔符、折叠 `.` 和 `..`），相对路径会拼接到会话目录。文件被读取并解析为 JSON。`config_path` 之外的其他键会覆盖在已加载对象之上——内联值优先。

返回：已添加工具的规范名称列表（`added`）、新的工具集版本号（`epoch`）、`immediate` 标志、正在启动的常驻进程列表（`pending`）、激活错误列表（`activation_failed`）、事件标识符（`event_id`）、`metadata_pending` 和 `delivery_pending` 标志。

错误：未知能力返回错误 `unknown capability '<name>'; available: [...]` 并附带可用列表；无效配置返回 `invalid config for capability '<name>': <reason>`；未产生任何工具的能力返回 `capability '<name>' produced no tools`；包内或与已存在工具的名称/别名冲突返回单独的错误。提供商服务器工具无法动态安装。

### remove

按名称从当前工具集中移除工具。在工具集的下一次快照时生效。

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| `operation` | string | yes | — | 值 `"remove"` |
| `name` | string | yes | — | 要移除工具的规范名称 |

返回：名称（`name`）、`was_present` 标志（当名称不在工具集中时为 false——幂等操作，无错误）、`event_id`、`metadata_pending`、`delivery_pending`、`deactivation_failed` 列表、新的 `epoch`。

核心工具 `Tools` 和 `Agent` 无法移除——尝试移除会返回错误。缺失的名称不是错误，而是 `was_present: false` 的结果。动态添加的工具可通过以相同能力和配置重复 `add` 来恢复；对于内置工具没有恢复操作。

### list

显示当前工具集，可选地附带相对于给定版本号的差异。

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| `operation` | string | yes | — | 值 `"list"` |
| `since_epoch` | integer/string | no | 无 | 若设置，还会返回 `delta`——自该版本号以来发生的变化；`0` = “自第一次变更起” |
| `kind` | string | no | 无 | 按工具种类（例如 `"builtin"`、`"mcp"`）、按来源标签（例如 `"base"`、`"agent"`、`"mcp:<server>"`）或按调用方式（`"direct"`、`"via_tools"`）过滤；比较不区分大小写 |

返回：`epoch`、`tools` 列表（每项：`name`、`aliases`、`kind`、`source`、`call_mode`、`availability`、`interface_digest`、`decorated`、`deferred`），并且仅在设置了 `since_epoch` 时返回 `delta` 字段。

无法识别的 `kind` 值会给出空工具列表，而不是错误。

### search

揭示延迟工具——那些仅在提示中按名称出现、其完整模式在请求时才返回的工具。

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| `operation` | string | yes | — | 值 `"search"` |
| `query` | string | yes | — | `select:<name1>,<name2>` 用于按精确名称直接揭示，或使用关键字；单词前的 `+` 前缀使其成为必选 |
| `max_results` | integer/string | no | `5` | 关键字结果的最大数量；在 `select:` 路径下被忽略 |

返回：`matches`（名称列表）、`query`（请求的回显）、`total_deferred_tools`（池中延迟工具的总数）、`pending_mcp_servers`（仅在结果为零时）。

关键字搜索有快速路径：精确名称匹配（不区分大小写）和 `mcp__` 前缀。必选词（`+word`）会过滤掉不匹配的工具；然后工具按名称部分、描述和提示中的匹配程度排序，按相关度降序返回，数量不超过 `max_results`。

### inspect

返回工具接口的完整描述及其在当前工具集中的可用性。

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| `operation` | string | yes | — | 值 `"inspect"` |
| `name` | string | yes | — | 规范名称或别名 |

返回：`requested_name`、`descriptor`（完整接口）、`availability`、`epoch`、`reason`（不可用时的说明，最多 2048 个字符）。

空名称是错误 `Tools{inspect}.name must not be blank`；不存在的工具是错误 `tool binding not found: <name>`。

### invoke

按名称调用工具。输入原样传递给目标工具，`Tools` 一侧不进行解析。

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| `operation` | string | yes | — | 值 `"invoke"` |
| `name` | string | yes | — | 目标的规范名称或别名 |
| `input` | value | yes | — | 目标工具的输入 |

该操作会在工具主体之前被拦截并路由到目标工具；带着此操作到达包装器主体是一种错误。

## 行为与限制

- `Tools` 工具始终存在且无法移除；它与 `Agent` 一起属于核心工具，核心工具禁止移除。
- `Tools` 无法通过 `invoke` 调用自身。
- 添加的工具不会在模型的工具调用接口中作为单独的函数出现。其接口通过 `Tools{inspect}` 获取，调用通过 `Tools{invoke}` 执行。“直接调用与 `via_tools`”的解析每个工具只发生一次。
- 仅当来源和接口指纹完全匹配时才允许重复添加（`add`）。内容发生变化（指纹不同）需要不同的名称——否则返回冲突错误。
- `activate: immediate`（默认值）使添加的工具在当前回合即可调用；`activate: deferred` 将调用推迟到下一回合，并保持提示缓存为热状态。
- 对于常驻进程仍在启动中的工具，`add` 返回一个 `pending` 条目，包含名称、预计就绪时间（`hint_ms`）和常驻进程日志路径；在就绪之前调用返回拒绝，而不是错误。
- 工具结果的最大大小为 100 000 个字符；实际上只有 `search` 在揭示多个工具时才会达到该限制。
- `search` 操作仅在启用延迟工具模式时可用。该模式由环境变量 `ENABLE_TOOL_SEARCH` 启用（真值或 `auto`/`auto:N`），而 `KOT_DISABLE_EXPERIMENTAL_BETAS` 变量为真值时会强制禁用它。当模式被禁用时，该操作返回错误。
- 每次工具集变更（add/remove）都会使版本号递增；`list` 返回当前版本号，`since_epoch` 允许获取相对于过去状态的差异。

## 示例

以内联配置添加 `cli-wrapper` 能力：

```json
{
  "operation": "add",
  "capability": "cli-wrapper",
  "config": {
    "name": "Git",
    "description": "仓库状态与历史",
    "script": "git",
    "read_only": true,
    "commands": {
      "status": {
        "description": "工作树状态",
        "params": [{ "name": "short", "type": "boolean", "style": "flag", "flag": "short" }],
        "required": []
      }
    }
  }
}
```

通过从文件加载配置来添加 `cli-wrapper` 能力：

```json
{
  "operation": "add",
  "capability": "cli-wrapper",
  "config": { "config_path": "tools/git-tool.json" }
}
```

显示当前工具集中的内置工具：

```json
{ "operation": "list", "kind": "builtin" }
```

按名称揭示延迟工具，然后按关键字：

```json
{ "operation": "search", "query": "select:Git" }
```

```json
{ "operation": "search", "query": "+git history", "max_results": 3 }
```

获取已添加工具的接口，然后调用它：

```json
{ "operation": "inspect", "name": "Git" }
```

```json
{ "operation": "invoke", "name": "Git", "input": { "operation": "status", "short": true } }
```
