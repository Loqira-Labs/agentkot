# History

用于读取已记录的聊天历史的工具：从会话记录中恢复过去的工具结果、列出它们、准备用于启动子代理的历史切片，并列出会话。返回的是执行时刻记录的字节，而不是已更改的当前文件。一个工具，五个操作，通过 `operation` 字段区分：`recover`、`query`、`fork`、`list_sessions`、`compact`。

作用域（`scope`）：`session`（仅当前会话）、`project`（项目的所有会话）和 `global`（机器上的所有项目）。默认值：主代理为 `project`，子代理为 `session`。`global` 是只读的，且仅限主代理；`fork` 拒绝 `global`。子代理显式使用 `project` 或 `global` 作用域时会收到一个错误。

## 操作

### recover

返回一个过去的工具结果（通常是较早的文件读取），或按调用标识符恢复它，并将其作为结果文本返回或重新注入会话记录。

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| file_path | string | 是，若无 tool_use_id | — | 要恢复其过去结果的绝对路径 |
| tool_use_id | string | 是，若无 file_path | — | 确切的调用标识符；可恢复任何过去的结果，包括错误 |
| mode | string | 否 | `return` | `return`——作为结果文本返回；`inject`——另外插入会话记录 |
| tool_name | string | 否 | `Files` | 要恢复的工具 |
| tool_operation | string | 否 | 若 tool_name = `Files` 则为 `read` | 被调用工具的操作 |
| strip_line_numbers | boolean | 否 | `Files` 和 `Read` 为 `true` | 移除 `NN→` 和 `NN\t` 形式的行号前缀 |
| scope | string | 否 | `project` / `session` | 历史范围 |

`file_path` 和 `tool_use_id` 互斥；二者必须恰好提供一个。按 `tool_use_id` 恢复时，字段 `tool_name` 和 `tool_operation` 会被忽略。

返回 `file_path`、`recovered`（是否找到结果）、`tool_use_id`、`captured_at`（会话记录中的时间）、`from_session`（`project`/`global` 作用域下的来源会话）、`from_externalized` 以及 `bytes`（移除编号后的文本长度）。未找到是 “否” 的软结果，而不是错误。纯图像（无文本）会返回带有明确消息和零字节数的结果。

`inject` 模式会向会话记录添加一条简短确认和恢复的文本作为元消息，使文本进入上下文并在未来的上下文压缩后仍保留。`return` 模式不会更改会话记录。

### query

按筛选条件列出过去的工具结果，从最新到最旧。只返回元数据而不返回正文；正文由 `recover` 操作返回。

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| tool_name | string | 否 | 不筛选 | 工具名称 |
| tool_operation | string | 否 | 不筛选 | 工具操作；这里没有隐含值，读取和编辑都会被列出 |
| file_path | string | 否 | 不筛选 | 调用所触及的路径 |
| limit | integer | 否 | 全部匹配 | 在 N 条匹配后停止；最小为 1 |
| include_errors | boolean | 否 | `false` | 是否包含错误结果 |
| session_id | string | 否 | 不筛选 | 限定到一个会话（uuid）；该会话必须在作用域内可达 |
| scope | string | 否 | `project` / `session` | 历史范围 |

筛选条件以逻辑 AND 组合。返回 `captures`——一个记录列表，每条包含 `tool_use_id`、`tool_name`、`tool_operation`、`file_path`、`is_error`、`captured_at`、`from_session`、`from_project`（仅 `global` 作用域）和 `bytes`。

### fork

将历史切片实体化为一组适合启动子代理的消息。生成一次性 `fork_id` 标识符，在启动子代理时通过 `seed_history` 传给 `Agent` 工具。该操作本身不会启动子代理。

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| range | object | 否 | 整个可见上下文 | 要切分哪段历史切片 |
| strip_line_numbers | boolean | 否 | `Files`/`Read` 的捕获为 `true` | 从捕获正文中移除行号 |
| scope | string | 否 | `project` / `session` | 历史范围；`global` 会被拒绝 |

`range` 字段是一个对象，包含键 `kind`，其值为以下类型之一：

| kind | 附加字段 | 含义 |
|---|---|---|
| whole_context | — | 整个当前可见上下文（默认） |
| last_n | `n`（整数，最小为 1） | 最后 N 条消息 |
| before_compact_boundary | — | 严格位于上下文压缩边界之前的所有内容 |
| capture | `tool_name`、`file_path`，可选的 `tool_operation` | 一个过去的结果，折叠为一条引导消息 |
| whole_session | `session_id`（uuid） | 一个完整会话 |

返回 `fork_id`、`message_count`（切片中的消息数；`0` 表示 “无可切分内容”）、`scope`、`range_kind` 和 `sessions_scanned`（贡献消息的会话数）。该标识符是一次性的：重复使用会返回 “not found or already consumed” 错误。切片以一条合成的防护消息结尾。

### list_sessions

列出作用域内的会话，从最近活动到最旧，以便为 `query` 或 `fork` 选择一个会话。

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| scope | string | 否 | `project` / `session` | 历史范围 |
| include_sidechains | boolean | 否 | `false` | 是否包含子代理的会话记录 |
| limit | integer | 否 | 全部会话 | 保留列表中的前 N 个会话 |
| active_since | string | 否 | 不筛选 | 一个 RFC3339 时间点；活动不早于该时间的会话会被保留 |
| text | string | 否 | 不筛选 | 对标题或第一条提示做的大小写不敏感的子串匹配 |

筛选条件以逻辑 AND 组合（时间截断 → 文本 → 数量）。无效的 `active_since` 是一个错误。返回 `sessions`——一个记录列表，包含 `session_id`、`project_key`（仅 `global` 作用域）、`started_at`、`last_activity_at`、`message_count_estimate`（近似估计值）、`first_user_text`、`title`、`is_current` 和 `is_sidechain`。

### compact

安排上下文压缩：在下一次响应之前，把当前会话较旧的前缀折叠为简短摘要，保留最近的消息。不接受任何参数；始终作用于活动会话。该调用总是安排压缩并返回 `scheduled: true`。折叠本身发生在工具之外，在下一次向模型发出请求之前。仅在上下文确实已满（约 90%）时使用。

## 行为与限制

- `recover`、`query`、`fork` 和 `list_sessions` 操作只读取历史。`compact` 不是读取（它请求重写上下文），但也不是破坏性的：磁盘上的完整会话记录不会被更改。
- 当历史读取机制不可用时，对需要读取的操作的调用会返回 “history reader unavailable” 错误；没有回退到实时文件系统的路径。
- 历史读取与取消标志串行化：当会话被取消时，会返回一个中断错误。
- 行号移除：每行开头的前缀 `^\s*\d+[→\t]` 会被移除，以便恢复的正文可直接写入。默认情况下它应用于 `Files` 和 `Read` 工具的结果；对于其他工具默认不应用；显式指定的值始终优先。
- 匹配时的路径解析是词法性的：分隔符被归一化为 `/`，`.` 和 `..` 会被解析，Windows 驱动器号会被大写，名称逐字比较，不做大小写折叠，也不接触文件系统。
- 被标记为已删除（墓碑）的结果不会出现在 `query`、`fork` 或范围中。
- 超出预算而溢出的结果正文会在读取时从会话目录恢复；缺失的 spill 文件是一个错误，而不是被截断的预览。
- 该工具永远不会让自己的结果超出预算而溢出：恢复的正文无法进入 spill 并循环。
- `fork` 切片在交付前会被规范化并修复：损坏的 “工具调用 ↔ 结果” 对会用合成的错误结果恢复，悬空的结果会被丢弃。切片会线性重新链接，以便子代理的第一个请求满足提供方契约。
- `compact` 不会强制执行上下文已满检查；“约 90%” 的建议是提示，而不是门禁。

## 示例

恢复过去的一次文件读取：

```json
{"operation": "recover", "file_path": "C:/work/src/lib.rs"}
```

按标识符恢复调用结果并插入上下文：

```json
{"operation": "recover", "tool_use_id": "toolu_0123", "mode": "inject"}
```

列出文件过去的读取记录：

```json
{"operation": "query", "tool_name": "Files", "tool_operation": "read", "file_path": "C:/work/src/lib.rs", "limit": 10}
```

为子代理准备最后 5 条消息的切片：

```json
{"operation": "fork", "range": {"kind": "last_n", "n": 5}, "scope": "session"}
```

列出标题中含某子串的项目会话：

```json
{"operation": "list_sessions", "scope": "project", "text": "解析器", "limit": 20}
```

压缩当前会话的上下文：

```json
{"operation": "compact"}
```
