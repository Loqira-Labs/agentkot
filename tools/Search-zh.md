# Search

以三种方式搜索项目：按文件名、按带正则表达式的文件内容、按语义。所有模式均为只读。

## 模式

模式由 `mode` 字段选择，取值为 `files`、`content` 或 `smart`。缺失或未知的值会返回错误并指出允许的取值。

### files

按名称模式列出文件。

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| pattern | string | 是 | — | 名称模式，例如 `**/*.rs` 或 `*.{ts,tsx}` |
| path | string | 否 | 会话工作目录 | 要搜索的目录；必须存在且为目录 |

搜索递归执行，遵循隐藏文件和忽略目录的规则，按修改时间排序。它返回相对于会话工作目录的路径列表（其外的路径以绝对路径返回），以及文件数量和截断标记。结果限制为 100 个文件；截断会在结果中标记。环境变量 `KOT_GLOB_NO_IGNORE` 和 `KOT_GLOB_HIDDEN`（默认均开启）控制对忽略文件和隐藏文件的处理，`KOT_GLOB_MAX_RESULTS` 则取代 100 来设置限制。

### content

用正则表达式搜索文件内容。

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| pattern | string | 是 | — | 正则表达式；开头的 `-` 会包在 `-e` 中 |
| path | string | 否 | 会话工作目录 | 要搜索的文件或目录 |
| glob | string | 否 | — | 名称过滤器；按空格和逗号切分，含 `{` 和 `}` 的片段保持完整 |
| output_mode | string | 否 | files_with_matches | `files_with_matches`、`content` 或 `count` |
| -B | integer | 否 | — | 匹配之前的内容行数 |
| -A | integer | 否 | — | 匹配之后的内容行数 |
| -C | integer | 否 | — | 匹配之前和之后的内容行数 |
| context | integer | 否 | — | 匹配之前和之后的内容行数；优先级 context > -C > -B/-A |
| -n | boolean | 否 | true | 显示行号 |
| -i | boolean | 否 | false | 忽略大小写搜索 |
| type | string | 否 | — | 文件类型过滤器，例如 `rust`、`js`、`py` |
| head_limit | integer | 否 | 250 | 最大结果数；0 表示无限制 |
| offset | integer | 否 | 0 | 在 head_limit 之前跳过前 N 个结果 |
| multiline | boolean | 否 | false | 点号匹配换行符，模式可跨行 |

版本控制目录（`.git`、`.svn`、`.hg`、`.bzr`、`.jj`、`.sl`）在搜索中被排除，隐藏文件被包含，长行在 500 列处截断。路径既可接受目录，也可接受单个文件。

`files_with_matches` 模式只返回文件列表，按从新到旧排序。`content` 模式返回匹配的行，带行号和相对于工作目录的路径。`count` 模式返回每个文件的匹配数以及所有文件的总数。`head_limit` 限制在搜索之后应用；实际发生截断时会设置标志，并在文本中附加使用新的 `offset` 继续的说明。

### smart

用自然语言按语义搜索。

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| query | string | 是 | — | 搜索意图，例如 "where JWT is validated" |
| max_results | integer | 否 | — | 结果数量的软限制 |

查询由一个受限的子代理执行，该子代理只读工作，且只能访问 Files（read）和 Search 工具。子代理的中间调用被隐藏。返回严格的 `hits` 列表，其中每个元素包含路径、可选的行号，以及该元素为何相关的强制说明。子代理限制为 8 轮，必须以给定结构返回答案；无效的答案作为错误返回。如果当前配置下无法启动子代理，则该模式返回错误。

## 行为与限制

所有模式均为只读，不破坏状态，在并发执行下是安全的。相对 `path` 相对于会话工作目录解析，而不是相对于进程工作目录。绝对路径按原样使用。对于 `files` 模式，带 `~` 的路径会展开为主目录；在 `content` 模式中路径按原样返回。

`files` 和 `content` 模式由 ripgrep 命令执行，按名称从 PATH 调用。缺少 ripgrep 会返回安装错误。默认超时为 20 秒（`KOT_GLOB_TIMEOUT_SECONDS` 变量）。无结果的超时返回超时错误；有部分结果的超时将其作为正常结果返回：在 `files` 模式中它们带截断标记，而在 `content` 模式中对于任何 `output_mode` 值（`files_with_matches`、`content`、`count`）都没有单独的标志。退出码 1 表示无匹配（不是错误）；码 2 表示正则表达式错误或参数无效；其他码表示失败。EAGAIN 错误会用单线程重试一次。输出限制为 20 MB（stderr 为 1 MB）；超出时丢弃末尾不完整的行。

`content` 模式下的 UNC 路径会立即被拒绝；在 `files` 模式下，检查被推迟，不访问文件系统。不存在的路径返回错误；在 `files` 模式下，不是目录的路径也返回错误。

## 示例

列出所有 Rust 源文件：

```json
{"mode": "files", "pattern": "**/*.rs"}
```

包含某个函数调用的文件列表：

```json
{"mode": "content", "pattern": "fn validate_input", "glob": "*.rs", "output_mode": "files_with_matches"}
```

匹配的行，前后各带两行上下文：

```json
{"mode": "content", "pattern": "MAX_READ_SIZE", "output_mode": "content", "context": 2, "-n": true}
```

从下一页继续输出：

```json
{"mode": "content", "pattern": "TODO", "output_mode": "content", "head_limit": 250, "offset": 250}
```

对某个实现位置进行语义搜索：

```json
{"mode": "smart", "query": "where JWT token validity is checked", "max_results": 5}
```
