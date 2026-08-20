# Shell

在宿主系统的 shell 中运行命令并返回合并后的输出。一个工具支持所有受支持的 shell（PowerShell、bash/sh、cmd），具体终端的选择在工具内部完成。名称别名：`Bash`、`RunCommand`。

## 操作

工具接受一个可选的 `operation` 字段；当该字段缺省时，隐含为 `run` 操作。可用操作：`run`、`set_default_shell`、`bash_output`、`kill`、`list_tasks`、`shell_open`、`shell_send`、`shell_read`、`shell_close`。未知的 `operation` 值是错误。

### run

在所选 shell 中执行命令并返回结果。这是主操作；也就是那个在不带 `operation` 字段时接受形如 `{ "command": "..." }` 输入的操作。

参数：

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| command | string | yes | — | 要执行的命令，按所选 shell 的语法书写。 |
| timeout | number or string | no | 60000 | 前台阻塞等待限制，单位毫秒。最大 60000；更大的值是错误，而非限制。后台运行时忽略。 |
| description | string | no | — | 简短的 5–10 词描述；用于界面并作为后台任务的标签。 |
| run_in_background | boolean or string | no | false | 当为 `true` 时，命令作为单独的后台任务启动，调用立即返回一个任务标识符。 |
| shell | string | no | — | 本次调用所用终端的名称（可用 shell 名称之一）。缺省时使用会话默认值，其次系统默认值。 |
| raw | boolean or string | no | false | 当为 `true` 时，返回未压缩的输出；默认情况下输出经过压缩。 |
| monitor | object | no | — | 后台运行的前瞻监视器配置。需要 `run_in_background: true`。 |

返回：合并后的 `stdout`（若 `stderr` 非空则追加其后）、`exit_code`、`interrupted`、终端名称 `shell`、所属系列 `shell_kind`、退出码的语义解释 `return_code_interpretation`、后台运行时的 `background_task_id`、设置监视器时的 `monitor_id`、输出落盘时的 `persisted_output_path`、`is_image`、`no_output_expected`、`compression`（压缩报告）。

限制：`timeout` 上限为 60000 毫秒。

### set_default_shell

切换当前会话的默认终端。之后不带显式 `shell` 的 `run` 调用使用新的默认值。

参数：

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| shell | string | yes | — | 成为会话默认值的终端名称。必须是已知且可用的名称。 |

返回：新的默认值 `default_shell` 以及先前切换的默认值 `previous_default`（若会话之前未切换过默认值则为空）。

限制：未知或不可用的名称是错误；不做任何替代。

### bash_output

读取后台任务的新输出（自上次读取以来）。

参数：

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| task_id | string | yes | — | 启动它的 `run` 返回的 `bg_<n>` 标识符。 |
| block | boolean or string | no | false | 等待模式：`false`/缺省 — 当前增量的即时快照；`true` — 等待第一条新输出或完成；`"completion"` — 等待完成并在一次调用中返回整个累积增量。 |
| timeout | number or string | no | 60000 | 阻塞读取的等待限制，单位毫秒。最大 60000。即时快照时忽略。 |
| filter | string | no | — | 正则表达式；返回的增量仅保留匹配它的行。 |
| raw | boolean or string | no | false | 当为 `true` 时，返回未压缩的增量。 |

返回：`task_id`、`new_output`（增量）、`status`（`running`/`exited`/`killed`）、完成后的 `exit_code`、`compression`。完成后，块文本中追加一行形如 `[task bg_<n> exited with code N]` 的内容。

限制：不存在的 `task_id` 是错误；无效的 `filter` 正则表达式是错误。

### kill

按标识符类型停止一个后台项。

参数：

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| task_id | string | yes | — | 标识符 `bg_<n>`（终止任务）、`mon_<n>`（解除监视器；任务继续运行）或 `res_<n>`（停止常驻进程）。 |

返回：`task_id` 和 `success`。对于 `bg_<n>` — 若运行中的任务被终止则为 `true`，若已退出则为 `false`。对于 `mon_<n>`/`res_<n>` — 若活项被停止/解除则为 `true`，若标识符已不存在则为 `false`。

限制：不存在的 `bg_<n>` 是错误；缺失的 `mon_<n>`/`res_<n>` 是软性的 `success: false`。

### list_tasks

列出所有后台任务及每个任务所挂接的监视器，以及已注册的常驻进程。

参数：无。

返回：任务列表（新的在前），含字段 `task_id`、`status`、`exit_code`、`description`、`output_bytes`、`persisted_output_path`、`monitors`，以及常驻进程列表。空列表不是错误；块文本为 `(no background tasks)`。

### shell_open

打开一个持久的交互式 shell 会话 — 一个活的解释器进程，其状态（工作目录、环境变量和 shell 变量）在命令之间保留。

参数：

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| shell | string | no | — | 会话所用终端的名称（与 `run` 相同的名称）。缺省时 — 会话默认值，其次系统默认值。 |
| cwd | string | no | session directory | 会话启动时所在的工作目录。 |
| description | string | no | — | 简短的 5–10 词描述。 |

返回：`session_id`（`sh_<n>`）和 `output` — 启动窗口内捕获的输出（提示符/横幅）。

限制：不存在的 `cwd`、启动失败或超出会话上限（默认 16 个并发）是错误。

### shell_send

向活的会话发送命令或输入，并收集产生的输出。会话状态在两次发送之间保留。

参数：

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| session_id | string | yes | — | 会话的 `sh_<n>` 标识符。 |
| input | string | yes | — | 要发送的命令或输入；执行时向其追加 `\r`（终端回车）。 |
| timeout | number or string | no | 10000 | 输出收集窗口限制，单位毫秒。最大 60000。 |

返回：`session_id`、`output`（增量）、`closed`（会话是否已关闭）。窗口也会在静默时提前结束；窗口关闭后会话命令继续执行。

限制：缺失或已关闭的会话是错误。

### shell_read

在不发送输入的情况下增量读取活会话的新输出 — 用于追赶一个长时间运行的命令。

参数：

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| session_id | string | yes | — | 会话的 `sh_<n>` 标识符。 |
| timeout | number or string | no | 10000 | 输出收集窗口限制，单位毫秒。最大 60000。 |

返回：`session_id`、`output`（增量）、`closed`。

### shell_close

终止活会话的进程并释放其槽位。

参数：

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| session_id | string | yes | — | 要关闭的会话的 `sh_<n>` 标识符。 |

返回：`session_id` 和 `success`（若运行中的会话被终止则为 `true`）。

限制：缺失的标识符是错误；当会话已退出时 `success` 为 `false`。

## 行为与限制

### Shell 选择

工具在内部按名称选择终端。默认名称：Windows 上 — `pwsh`（PowerShell）和 `cmd`（cmd.exe）；POSIX 上 — `bash` 和 `sh`。

自定义命名的 shell 由可选文件 `<config_directory>/shells.json` 定义 — 一个 JSON 记录数组：

| 字段 | 类型 | 必填 | 含义 |
|---|---|---|---|
| name | string | yes | 在 `shell` 参数中给出的名称；大小写不敏感地唯一；以系统 shell 命名的记录会覆盖它 |
| kind | string | yes | 语法系列：`bash`、`sh`、`pwsh` 或 `cmd` — 决定启动参数、输出解码和退出码解释 |
| program | string | no | 解释器的显式路径；作为磁盘路径检查，不为其做 `PATH` 搜索。无此字段时，解释器按该系列的标准名称在 `PATH` 中查找。指定的路径在磁盘上不存在时，选择时即为错误，不会用其他 shell 替代 |
| edition | string | no | 用于 `pwsh` 系列：`desktop`（PowerShell 5.1）或 `core`（PowerShell 7+）；无此字段时按文件名确定版本 |
| default | boolean | no | 将此 shell 设为系统默认值（仅一个）；来自 `set_default_shell` 的会话默认值优先级更高 |

```json
[
  { "name": "git-bash", "kind": "bash", "program": "C:/Program Files/Git/bin/bash.exe" },
  { "name": "ps5", "kind": "pwsh", "program": "C:/Windows/System32/WindowsPowerShell/v1.0/powershell.exe", "edition": "desktop" }
]
```

若该文件缺失 — 只有系统 shell 可用。解析出错的文件不会中止启动：打印警告，系统 shell 仍然可用。

`run` 和 `shell_open` 的解析优先级：调用中的名称（`shell`）→ 会话默认值（通过 `set_default_shell` 切换）→ 系统默认值。系统默认值：Windows 上 — `pwsh`（依次查找 `pwsh`、`powershell`、再是 Linux 回退路径），POSIX 上 — `bash` 并回退到 `sh`。`cmd` 只能通过显式名称到达，绝不会被自动选中。

选择通过不启动进程的探测完成；结果按进程记忆化。解释器缺失，或配置的路径在磁盘上不存在，是错误；不会静默替换为其他 shell。

命令启动参数：bash/sh — `-c <command>`；PowerShell（两个版本）— `-NoProfile -NonInteractive -Command <command>`；cmd — `/C <command>`。

### 超时与退出码

模型管理的硬性阻塞等待上限为 60000 毫秒（1 分钟），适用于：前台 `run`（`timeout`）、阻塞式 `bash_output`（当 `block` 解析为等待输出或完成时）、`shell_send`/`shell_read` 窗口。恰好 60000 的值是允许的；更大的值是错误，而非限制。该上限不可配置。

前台 `run` 的默认超时为 60000 毫秒。超时到期时，命令被整体终止（整个进程树），工具返回超时错误。

退出码是子进程的码。若命令被用户中断（`interrupted: true`），码为 130（128+SIGINT 惯例）。`stdout` 与 `stderr` 合并：若 `stderr` 非空则追加到 `stdout` 之后。

退出码的解释取决于 shell。bash：`grep`/`rg` 返回码 1 — “未找到匹配”（不是错误），码 2 及以上 — 错误；`diff` 返回码 1 — “文件不同”（不是错误），码 2 及以上 — 错误。PowerShell：`grep`/`rg`/`findstr` 返回码 1 — “未找到匹配”；`robocopy` — 位字段（0 — “已同步”，1–7 — “文件已复制”，8 及以上 — 错误）。cmd：只有默认规则 — 码 0 为成功。在所有其他情况下，仅码 0 算作成功。

输出解码取决于 shell：PowerShell Desktop（5.1）产生 UTF-16 LE（通过 BOM 或 NUL 字节的存在识别）并将 `\r\n` 规范化为 `\n`；PowerShell Core（7+）以及 bash/sh/cmd — UTF-8，对无效字节有损。

### 后台任务

使用 `run_in_background: true` 时，命令被分离为后台任务，调用立即返回标识符 `bg_<n>` 和文本 “命令正在后台运行，ID 为 bg_<n>”。标识符从 `bg_0` 起单调递增。stdout 和 stderr 持续合并到一个缓冲区；任务状态为 `running`、`exited`（带码）、`killed`。

缓冲区在内存中累积至 64 MB 阈值；超过后，输出落盘到文件 `<runtime>/shell-bg/<id>.out`，内存中保留 1 MB 尾部。文件路径可从 `persisted_output_path` 获得。

输出按 UTF-8 有损解码；读取边界处不完整的多字节序列会保留到下一次读取，以免一个字符被拆到两次读取之间。

后台任务列表在代理进程内共享；会话结束时，其任务连同落盘文件一起被终止并移除。之前运行遗留的孤立落盘文件在启动时被移除。

`bash_output` 的 `block` 模式：`false`/缺省 — 即时快照；`true` — 等待第一条新输出或完成；字符串 `"completion"`、`"exit"`、`"until_exit"`、`"done"` — 等待完成并在一次调用中返回整个增量。`"completion"` 读取仅在进程停止且两个流都完全排空后才返回。

### 监视器

监视器是一个前瞻的观察者，当存在 `monitor` 字段时随后台 `run` 一起启动。它从自己的游标开始监视任务（从字节 0 起，因此启动到建立之间的输出不会丢失），并在触发时无需轮询即向会话投递通知。返回标识符 `mon_<n>`。

`on` 字段（等待什么）：`completion`（默认）— 任务完成或被终止；`output` — 一条新输出行匹配 `pattern`；`idle` — `idle_secs` 秒内没有新输出；`heartbeat` — 任务运行期间每 `interval_secs` 秒触发一次。

监视器字段规则：

| 字段 | 必填性 | 约束 |
|---|---|---|
| on | no（默认 `completion`）| `completion`/`output`/`idle`/`heartbeat` 之一 |
| pattern | 当 `on:"output"` 时必填 | 非空正则表达式；仅与 `output` 一起使用 |
| idle_secs | 当 `on:"idle"` 时必填 | 整数 ≥ 1；仅与 `idle` 一起使用 |
| interval_secs | 当 `on:"heartbeat"` 时必填 | 整数 ≥ 1；仅与 `heartbeat` 一起使用 |
| recurring | no | 见下文 |
| throttle_secs | 当 `on:"output"` + `recurring:true` 时必填 | 整数 ≥ 1；仅在此组合下使用 |
| raw | no | 当为 `true` 时，通知摘录为未经压缩的原始尾部 |

重复：`completion` — 单次（终结性），不允许与 `recurring: true` 一起使用。`output`/`idle` — 默认单次；`recurring: true` 使它们持续触发（对 `output`，需要 `throttle_secs` — 抑制通知风暴的合并窗口；对 `idle` — 每次静默时段触发一次，新活动会重新武装它）。`heartbeat` — 本质上就是重复的，不允许与 `recurring: false` 一起使用。

每个数字型的监视器时长字段上限为 100 年（定时器可表示范围）；更大的值是错误。`output` 匹配仅针对完整行；超过 64 KB 且无终止符的行会导致监视器响亮关闭并给出错误消息（不截断、不做前缀匹配）。通知摘录为最多 8 KB 的尾部。

不兼容的字段组合在任务启动前即为错误；任务不会被创建。若监视器建立后意外失败，任务被终止并返回错误。监视器的存活时间长于设置它的那个回合；它通过显式的 `kill mon_<n>`（任务继续运行）或在会话结束时被解除。

### 交互式会话

会话是伪终端（Windows 上为 ConPTY，POSIX 上为 pty）中的一个活的解释器进程，尺寸为 50 行 × 200 列。启动：bash/sh — `-i`，PowerShell — `-NoLogo`，cmd — 无标志。默认最多允许 16 个并发会话。

状态（工作目录、环境变量和 shell 变量、已激活的虚拟环境）在命令之间保留 — 会话正是为此而存在。

`shell_send` 向输入追加 `\r`（终端回车；`\n` 不是回车）。收集窗口在静默（250 毫秒无新字节）或总 `timeout` 时结束；窗口关闭后会话命令继续执行。`shell_open` 的启动窗口为 2 秒。

会话输出按 UTF-8 有损解码，并从中移除 ANSI 控制序列（CSI、OSC、单个控制字节）；可打印文本、`\n` 和 `\t` 被保留。会话读取器应答终端查询（DSR/DA），使 PowerShell 解释器不会挂起。

`shell_close` 终止进程，等待剩余输出被读取（最多 1.5 秒），并从列表中移除会话。代理会话结束时，其所有交互式会话都被关闭。

### 输出压缩

前台 `run` 的输出和 `bash_output` 增量，以及监视器通知摘录，都经过压缩。`raw: true` 字段为该调用禁用压缩。

压缩根据命令字符串挑选一个命名过滤器；无匹配时，应用通用安全变换：移除 ANSI 序列 → 折叠空行 → 去除连续重复行 → 行数限制（保留前 160 行和后 200 行，绝对上限 400 行；被省略的中间部分以标记注明）。

默认命名过滤器：`cargo build/check`、`cargo clippy`、`cargo test`、`git status`、`git log`、`git diff/show`、`git push`、`rg/grep`。每个都保留信号（错误、警告、失败的测试、摘要）并丢弃噪声（构建进度行、提示、空分隔符），应用行数限制，并在结果为空时输出一条形如 “ok …” 的消息。

重写后的输出只有在严格短于原文且确实减少了 token 数（BPE 计数）时才被采用；否则返回原文。压缩报告（`compression`）包含原始和压缩后的大小、token 估计、节省百分比、过滤器名称，以及（当写出 tee 文件时）路径。

当退出码非零且发生了实际压缩时，完整的未压缩输出被写入文件 `<temp>/kot-shell-raw/shell-raw-*.log`，输出中追加一条带路径的提示。写入失败时静默降级（无文件）。

工具结果限制为 30000 个字符。

### 其他

- 命令以启动代理的用户的权限执行。
- 若 stdout 完全是 `image/*` 数据 URI，则将其解码；当较长边超过 2000 像素时，按比例缩小（Lanczos3），重新编码为 PNG 并作为图像块返回。大小限制为 20 MB；无法读取或仍然过大的图像降级为文本。
- 当命令在语义上不产生输出时（例如单独的 `cd`/`Set-Location`），`no_output_expected` 字段为 `true`。
- 启动前会建立进程组，使 kill 作用于整个后代树（cargo→rustc→linker），而不仅是直接的 shell。

## 示例

在默认 shell 中运行命令：

```json
{ "command": "cargo test --workspace", "description": "运行工作区测试" }
```

带完成监视器在后台运行：

```json
{
  "operation": "run",
  "command": "python train.py",
  "description": "模型训练",
  "run_in_background": true,
  "monitor": { "on": "completion" }
}
```

带输出行监视器在后台运行：

```json
{
  "operation": "run",
  "command": "cargo test --workspace",
  "run_in_background": true,
  "monitor": { "on": "output", "pattern": "FAILED", "recurring": true, "throttle_secs": 30 }
}
```

读取后台任务的输出直至完成：

```json
{ "operation": "bash_output", "task_id": "bg_3", "block": "completion", "timeout": 30000 }
```

打开交互式会话、发送命令并读取结果：

```json
{ "operation": "shell_open", "shell": "bash", "cwd": "/home/user/project" }
```

```json
{ "operation": "shell_send", "session_id": "sh_0", "input": "export KOTX=42 && echo $KOTX" }
```

停止后台任务并解除监视器：

```json
{ "operation": "kill", "task_id": "bg_3" }
```

```json
{ "operation": "kill", "task_id": "mon_1" }
```
