# cli-wrapper

一种通过 `Tools{add}` 添加的能力：将任意 CLI 脚本转化为类型化的智能体工具。配置指定工具名称、解释器和脚本，以及一组带类型化参数的命名命令；每个命令都成为该工具的一个操作。

## 配置

该能力通过调用 `Tools{operation:"add", capability:"cli-wrapper", config:{...}}` 添加。配置字段：

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| `name` | string | yes | — | 模型调用的工具名称；非空 |
| `description` | string | no | 空 | 工具描述，显示在提示和模式中 |
| `interpreter` | string | no | 空 | 解释器程序（例如 `"python"`、`"node"`）；为空时直接运行 `script` |
| `script` | string | yes | — | 脚本的路径；非空 |
| `script_args` | array | no | 空 | 位于脚本和命令名称之间的附加参数 |
| `timeout_ms` | integer | no | `60000` | 单次调用的时间限制，以毫秒为单位；必须在 `1..=60000` 范围内 |
| `read_only` | boolean | no | `false` | 声明该工具为只读且可安全并行启动 |
| `commands` | object | yes | — | 命名命令（至少一个）；键是操作名称 |
| `setup` | object | no | 无 | 包装常驻进程的工具的自动设置 |

每个命令（`commands.<name>`）按如下方式描述：

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| `description` | string | no | 无 | 命令描述 |
| `params` | array | no | 空 | 参数声明的有序列表 |
| `required` | array | no | 空 | 必填参数的名称 |
| `read_only` | boolean | no | 继承工具的 `read_only` | 针对此命令的只读覆盖 |
| `resident_stop` | boolean | no | `false` | 将该命令标记为规范的常驻进程停止命令 |

参数声明（`params[]`）：

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| `name` | string | yes | — | 参数名称——也是模型填写的输入字段 |
| `type` | string | yes | — | JSON Schema 类型：`string`、`integer`、`number`、`boolean`、`array` |
| `description` | string | no | 无 | 参数描述 |
| `style` | string | no | `"positional"` | 传递方式：`positional` 或 `flag` |
| `flag` | string | no | 由 `name` 中的 `_` 替换为 `-` 得到的名称 | 对 `style: "flag"` 的标志名称覆盖 |

`setup` 块（常驻进程）：

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| `probe` | array | yes（激活时非空） | — | 就绪探测的 argv 尾部（退出码 0 = 就绪） |
| `start` | array | no | — | 常驻进程启动的 argv 尾部；以分离方式启动 |
| `ready_timeout_ms` | integer | no | `10000` | 启动后等待常驻进程就绪的时长，以毫秒为单位 |
| `ready_hint_ms` | integer | no | 无 | 预计就绪时间，显示给智能体 |
| `log_file` | string | no | `<script directory>/<name>.serve.log` | 常驻进程 stdout/stderr 的写入位置；支持 `{session_id}` 和 `{cwd}` |
| `resident_key` | string | no | 无 | 逻辑上的常驻进程标识符；字面量，不含模板 |
| `scope` | string | no | `"shared"` | 常驻进程所有权的范围；仅支持 `shared` |
| `skip` | boolean | no | `false` | 跳过整个常驻进程激活并立即将其视为就绪 |

## 命令如何成为操作

`commands` 中的每个名称都是该工具 `operation` 字段的允许值。调用输入是一个扁平对象：`operation` 加上命令参数。同一命令中使用相同 `name` 的参数会被收集到单个模式属性中；含义与特定操作绑定的参数，其描述会用这些操作的列表进行标注。

## 命令行构建

调用按如下顺序组装 argv：

```
[interpreter?] script script_args... operation [positional...] [flags...]
```

- 位置参数按声明顺序以裸值形式添加。
- 标志参数在其后按声明顺序以 `--<flag> <value>` 形式添加；布尔值 `true` 只给出 `--<flag>`，`false` 省略该标志；数组为每个元素重复 `--<flag> <value>`。
- 位置数组会展开为多个裸值。
- 字符串值原样传递；数字和布尔值按其文本表示传递（`42`、`3.14`、`true`/`false`）。

## 退出码与输出

- 进程的 stdout 和 stderr 被并发读取并合并：stderr 非空时追加在 stdout 之后。
- `exit_code` 是进程退出码；缺失时为 `-1`。
- 非零退出码会使结果成为错误（`is_error`）。
- 超过 `timeout_ms` 返回超时错误；用户中断返回带 `interrupted: true` 和 `exit_code: 130` 的结果。
- 对于常驻进程工具，在常驻进程启动中或已崩溃时的失败调用会附带说明性提示；成功的调用并不意味着常驻进程就绪——就绪仅由规范的 `probe` 确认。

## 常驻进程

- 激活时，工具首先运行 `probe`。如果探测已经通过，则不运行 `start`——不会杀死或重启一个存活的常驻进程；存在的事实被登记，工具即就绪。
- 否则，通过允许的启动 argv（以及设置时的 `resident_key`）执行原子的常驻进程接管：胜者以分离方式启动常驻进程（使用独立的进程组），记录 pid，并生成一个后台就绪监视器；败者等待另一个监视器。激活立即返回“正在预热”状态，不会阻塞添加。
- 监视器轮询 `probe`（探测之间暂停 500 ms，探测上限 5000 ms），直到就绪或 `ready_timeout_ms` 到期；到期时常驻进程被标记为失败，并通知会话。
- 带 `resident_stop: true` 的命令在退出码为 0 后，等待 `probe` 停止通过（最多检查 10 000 ms，探测窗口 2000 ms），并删除其常驻进程记录。结果记录在 `lifecycle` 中：`confirmed`、`unconfirmed`、`interrupted` 或 `unbound`；除 `confirmed` 之外的任何结果都会将结果标记为错误，并保留记录。

## 行为与限制

- 输出裁剪：带整数 N（最小为 1）的 `_head` 参数保留合并输出的前 N 行；裁剪时会添加一行说明保留行数。它不影响命令的状态和参数。
- 工具结果的最大大小为 30 000 个字符。
- 以 `_` 开头的参数名称是保留的，配置中禁止使用。
- `{session_id}` 和 `{cwd}` 模板仅在 `probe`、`start` 和 `log_file` 中替换；在 `script_args` 中模板变量被禁止，会导致配置错误。
- `{port}` 模板和值 `scope: "per_agent"` 不受支持——指定其中任何一个都会导致配置错误。
- `read_only` 标志是一种声明：它将调用标记为读取且可安全并行启动。它不检查脚本的实际行为，也不限制进程权限；默认情况下该工具被视为会修改状态。
- 没有 `commands`、`name` 或 `script` 为空、`timeout_ms` 超出 `1..=60000`、带 `resident_stop` 的常驻进程命令没有 `setup` 块，或此类命令多于一个的配置会被拒绝并返回配置错误。

## 示例

基于系统命令的工具：命令名称成为操作，参数成为其参数（`git status --short`）：

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
      },
      "log": {
        "description": "最近的提交",
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

带解释器和常量脚本参数的工具：

```json
{
  "operation": "add",
  "capability": "cli-wrapper",
  "config": {
    "name": "Index",
    "description": "在项目索引中搜索",
    "interpreter": "python",
    "script": "tools/index.py",
    "script_args": ["--root", "."],
    "timeout_ms": 30000,
    "commands": {
      "lookup": {
        "description": "按标识符查找条目",
        "params": [{ "name": "id", "type": "string", "style": "positional" }],
        "required": ["id"],
        "read_only": true
      }
    }
  }
}
```

基于常驻进程的工具：

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
        "description": "按标识符查找条目",
        "params": [{ "name": "id", "type": "string", "style": "positional" }],
        "required": ["id"]
      },
      "stop": { "description": "停止常驻进程", "params": [], "required": [], "resident_stop": true }
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

调用已添加的工具：

```json
{ "operation": "invoke", "name": "Index", "input": { "operation": "lookup", "id": "42" } }
```

将长命令的输出裁剪为前几行：

```json
{ "operation": "invoke", "name": "Git", "input": { "operation": "log", "max_count": 100, "_head": 40 } }
```
