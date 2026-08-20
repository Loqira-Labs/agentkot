# Memory

用于存储代理持久笔记的工具，与项目文件和聊天历史不同：笔记在会话之间驻留在磁盘上，具有作用域和分类，可被索引以便搜索，并在每个会话开始时自动预加载。一个工具，五个操作，通过 `operation` 字段区分：`read`、`write`、`list`、`delete`、`search`。每个操作都接受 `scope` 字段。

作用域：

| 作用域 | 磁盘位置 | 含义 |
|---|---|---|
| project | `<project_root>/.kot/agent-memory` | 项目笔记；通过版本控制共享；默认使用 |
| user | `<config_home>/memory` | 机器级笔记，跨所有项目和会话共享 |
| local | `KOT_REMOTE_MEMORY_DIR`，或 `<project_root>/.kot/agent-memory-local` | 本项目在这台机器上的笔记；为避免进入版本控制，该目录会被加入仓库的忽略规则 |

## 操作

### read

按 slug 名称读取给定作用域中的一条笔记。

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| name | string | 是 | — | 笔记的 slug 名称（文件名主干，不含 `.md` 扩展名） |
| scope | string | 否 | `project` | 作用域 |

返回完整记录：`name`、`title`、可选的 `description`、`kind`、`body`（原始 markdown 正文）、`scope` 以及 `path`（磁盘上的路径）。不存在的笔记是一个 “未找到” 的软结果，而不是错误。

### write

创建或覆盖一条笔记及其在索引中的行。对已有 slug 的写入会覆盖该笔记。

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| name | string | 是 | — | 笔记的 slug 名称（文件名主干） |
| content | string | 是 | — | markdown 格式的笔记正文；至少 1 个非空字符 |
| description | string | 否 | 无 | 单行描述（索引和预加载的索引行） |
| kind | string | 否 | `other` | 分类：`user`、`feedback`、`project`、`reference`、`other` |
| scope | string | 否 | `project` | 作用域 |

返回 `name`、`scope`、`path`、`outcome`（新笔记为 `created`，覆盖时为 `updated`）以及 `index_updated`（始终为 `true`）。

slug 约束：名称必须是字符 `[A-Za-z0-9._-]` 构成的单一主干，不能为空，不能是 `.` 或 `..`，不能等于名称 `MEMORY`（保留的索引），并且不能包含路径分隔符、`:` 或会破坏索引解析的字符。无效的 slug 会在校验阶段被拒绝。

`description` 会被折叠为单行（任何空白字符都被替换为单个空格）；空或仅含空白的描述根本不会被写入。没有单独的显示标题——slug 本身就作为标题。

### list

列出给定作用域的索引（索引行列表）。

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| scope | string | 否 | `project` | 作用域 |

返回 `scope` 和 `entries`——一个索引行列表，每行包含字段 `title`、`file` 和 `hook`。

### delete

按 slug 删除一条笔记，并从索引中移除其行。

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| name | string | 是 | — | 要删除的笔记的 slug 名称 |
| scope | string | 否 | `project` | 作用域 |

返回 `name`、`scope`、`removed`（如果笔记不存在则为 `false`——空操作）以及 `index_updated`（索引是否发生变化）。

### search

按 BM25 相关度搜索给定作用域的笔记。

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| query | string | 是 | — | 自由文本查询；至少 1 个非空字符 |
| scope | string | 否 | `project` | 作用域 |
| max_results | integer 或 string | 否 | 100 | 返回匹配数量的上限 |

返回 `scope`、`query` 和 `hits`——按相关度降序排列的匹配。每条匹配包含 `name`、`title`、`kind`、`snippet`（匹配附近的正文片段）和 `score`（BM25 分数）。

## 行为与限制

- 每条笔记是一个文件 `<scope_dir>/<slug>.md`，包含 frontmatter 和 markdown 正文。frontmatter 存储字段 `name`（标题）、可选的 `description` 和 `metadata.type`（分类）。写入和覆盖都是原子的（临时文件 + 重命名）。
- 作用域索引是文件 `<scope_dir>/MEMORY.md`，其行格式为 `- [title](slug.md) — hook`。索引行（`hook`）在 `description` 已设置且非空时等于 `description`，否则等于标题。写入会添加或替换索引行；删除会移除它。读取索引时，与索引格式不匹配的行会被丢弃。
- BM25 搜索：词频饱和 `k1 = 1.2`，文档长度归一化 `b = 0.75`，元数据字段（标题 + 索引行）相对于正文的权重为 `2.0`。分词时，词项被切分为 Unicode 字母数字字符或 `_` 的最长连续序列，其余都是分隔符。西里尔字母和其他字母表都能处理；没有分隔符的连续 CJK 文本会形成单个词项序列（其内部子串不会匹配）。
- 排序：任意查询词匹配即可（OR 语义）；重复的查询词不会使分数翻倍。分数低于最佳结果分数 15% 的匹配会被丢弃；最佳结果始终保留。分数相同时保持索引顺序。返回匹配的硬性上限是 100。
- 记录按作用域目录以串行方式读取和覆盖：锁同时保护笔记文件和共享索引，包括跨不同的会话和处理同一作用域的进程。
- 预加载：会话开始时，笔记作为系统消息提供给模型，而不调用 `list`/`read` 操作。`user` 作用域以完整笔记正文提供，且没有容量限制。`project` 和 `local` 作用域仅提供索引行——这类笔记的正文由 `read` 操作按需读取。加载某一个作用域失败不会中断会话启动。
- 一次操作的最大结果大小为 25 000 个字符。已存储的笔记不会因此被修改。
- 内容规则：只存储值得带入未来会话的知识。禁止存储密钥、进度和状态日志、临时事实，以及可以从代码、git 或当前对话中廉价恢复的内容。对于 `user` 作用域——只存储适用于任何会话和项目的通用行为规则，并使用 `feedback` 分类。选择最窄的合适作用域，并根据当前来源核实回忆的事实。

## 示例

写入一条项目笔记：

```json
{"operation": "write", "scope": "project", "name": "build-commands", "description": "项目构建与门禁命令", "kind": "project", "content": "构建：cargo build。门禁：cargo test && cargo clippy。"}
```

读取一条笔记：

```json
{"operation": "read", "scope": "project", "name": "build-commands"}
```

列出作用域索引：

```json
{"operation": "list", "scope": "user"}
```

按查询搜索：

```json
{"operation": "search", "scope": "project", "query": "构建 门禁", "max_results": 5}
```

删除一条笔记：

```json
{"operation": "delete", "scope": "project", "name": "build-commands"}
```
