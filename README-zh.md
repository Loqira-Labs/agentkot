![KOT](assets/title.jpg)

# KOT — 就是最好的 AI 智能体框架

<h3 align="center">
  <a href="README.md">English</a> &nbsp;·&nbsp; <a href="README-ru.md">Русский</a> &nbsp;·&nbsp; <b>简体中文</b>
</h3>

**一个文件，足矣。**

这是你能装到自己机器上最好的可用智能体。一个可执行文件：它读取代码和文档、搜索项目、编辑文件、运行命令和长驻进程、维护计划与任务看板、在会话之间记住重要之事、雇佣其他智能体并在它们之间分配工作——并且可同时跨越多个提供商。

它不请求许可，不装出关心的样子，也不去适配坏掉的流程。它是工具。把它当工具用。

## 立场

**没有安全机制，也不会有。**智能体以启动它的用户权限运行命令、编辑文件。没有沙箱，没有允许/拒绝/询问规则，没有逐步确认。这是有意为之的决定：一个每隔一步就要征求许可的智能体做不完工作，它把整个会话花在关于意图的对话上。如果你需要隔离，拿现成的——容器、虚拟机、独立用户账户、成百上千种智能体沙箱中的任意一个——在其中运行 `kot`。隔离是环境的属性，不是智能体的属性。

**不需要技能（skills）。**技能就是一段带指令的文本，被人包进了一个带注册表、安装器和版本的单独机制。你有一个会读文件、遵循指令的智能体。想按技能工作——把文档交给智能体，让它照做。这里不会出现一个专门读文本的子系统。

**不需要 MCP。**MCP 是给智能体没有的工具做传输用的。这里这些工具是内置的：shell、files、search、memory、history、媒体生成、编排。偶尔需要一个外部服务——告诉智能体用命令调用它。长期需要——通过 `Tools{add: "cli-wrapper"}` 把它的 CLI 包成动态工具，智能体就能获得带参数校验的强类型操作，无需中间协议、额外进程及其协议限制。

**它不会去适配你的流程。**说清任务、给出上下文、检查结果。如果你想要一个对表述不清的任务礼貌附和、去适配坏掉管线的智能体——那就用 Claude Code、Codex 以及其他那些。

## 核心优势：跨提供商的团队

单个模型上的单个智能体，要么慢、要么浅、要么成为瓶颈。这里角色分散在各个提供商之间，每个参与者都运行在适合其工作的模型上。

这种分工的一个例子——提供商和模型由你根据自己的任务和已有订阅来选择：

| 角色 | 示例提供商 | 为什么 |
|---|---|---|
| 主导智能体 | anthropic | 掌控整个任务、写代码、做决策 |
| 架构师、审查者 | openai | 对计划和差异的另一流派独立视角 |
| 研究员、代码侦察 | deepseek | 在轻量模型上进行数十个并行读取 |
| 安全审查、第二双眼睛 | zai | 在独立配额上再做一轮独立检查 |
| `Search{smart}` 内的语义搜索 | 任意轻量提供商 | 搜索不应占用主模型 |

如何启用：

- `providers.json` 中的每个 key 自动成为可用的启动目标——无需单独配置；
- `Agent{spawn|delegate|hire, provider, model, effort}` ——子智能体在所选的提供商上启动，父级的模型绝不会跨提供商泄露；
- `Agent{set_model}` ——在两轮之间把活跃的队友切换到另一个提供商和模型；
- 流水线步骤和团队步骤成员各自携带自己的 `provider` 和 `model`；
- `kot web --search-provider … --search-model …` ——语义搜索迁移到一个独立的轻量提供商。

每个提供商当前的模型标识符由智能体自己通过 `Agent{list_models}` 读取，在选择模型时会在 Web 界面中可见。

结果：一个并行的团队，最强的模型只做真正需要它的事，而日常工作和侦察则跑在轻量模型上。

## 基准测试

[Harness-Bench](https://github.com/Qihoo360/harness-bench) 衡量的正是 harness 本身：106 个沙箱离线智能体任务，覆盖 8 个工作流类别，模型固定，确定性的 oracle 校验器。

同一模型 —— DeepSeek V4 Flash；决定因素在于 harness：

| Harness | 得分 |
|---|---|
| **KOT** | **79.1%** |
| Hermes | 76.2% |

在同一模型上，KOT 以廉价快速模型超越 Hermes —— 在全部 106 个任务上领先 2.9 个百分点。

**Terminal-Bench 3.0**（[frontierbench.ai](https://www.frontierbench.ai/)）——真正的终端工作：构建、调试、运维。对整个任务套件的完整单次运行（k=1）——固定数据集的 66 个任务加上先前注册表修订中的 6 个，共 72 个 trials，每个任务由其专属验证器评分。KOT 驱动 [DeepSeek V4 Flash](https://api-docs.deepseek.com/)，推理强度为 `high`。完整运行记录：[Harbor Hub 上的 72 个 trials](https://hub.harborframework.com/jobs/8f27a07f-4a88-41ce-b06c-6a713961c791)。

| 模型（effort） | 智能体 | 解决率 | Token | 成本 |
|---|---|---|---|---|
| **[DeepSeek V4 Flash](https://api-docs.deepseek.com/)（high）** | **KOT** | **17.0%** | 3.3B | **$71.75** |
| [Grok 4.5](https://docs.x.ai/developers/models/grok-4.5)（xhigh） | [Cursor CLI](https://cursor.com/docs/cli/overview) | 15.7% ± 1.5% | 1.2B | $766.02 |
| [Sonnet 5](https://www.anthropic.com/news/claude-sonnet-5)（max） | [Claude Code](https://claude.com/claude-code) | 14.6% ± 1.5% | 17.9B | $6.9k |
| [GPT-5.6 Luna](https://developers.openai.com/api/docs/models/gpt-5.6-luna)（max） | [Codex](https://openai.com/codex/) | 14.3% ± 1.3% | 11.9B | $1.6k |
| [GLM 5.2](https://z.ai/blog/glm-5.2)（max） | [Claude Code](https://claude.com/claude-code) | 4.6% ± 1.0% | 3.3B | $3.4k |

下面四行来自 Terminal-Bench 3.0 官方排行榜（k=5）；KOT 一行是我们对同一任务套件的自主运行。廉价模型配上强大的 harness，解决率超过下面每一个参赛者——而整个 72 任务的运行成本不到 $72，比整个排行榜中任何一行都低一个数量级。完整解决 12 个任务，平均 reward 17.0%。

## 交付内容

预编译的二进制文件就在本文件旁边：

| 平台 | 文件 |
|---|---|
| Linux (x86-64) | `releases/linux/kot` |
| macOS (Apple Silicon) | `releases/macos/kot.bin` |
| Windows (x86-64) | `releases/windows/kot.exe` |

Web 界面（页面、脚本、样式）、图标、分词器和输出压缩规则都内嵌在二进制里——旁边无需放任何东西。把文件放进 `PATH` 上的任意目录即可。

在 macOS 和 Linux 上需要可执行权限位；在 macOS 上如果文件是经由网络下载的，还需要移除隔离标志：

```
mv kot.bin kot && chmod +x kot        # macOS
xattr -d com.apple.quarantine kot     # macOS, if the file was downloaded by a browser
chmod +x kot                          # Linux
```

### 系统中必须具备的东西

两个外部程序，两者都必须能在 `PATH` 上找到：

- **`rg` (ripgrep)** ——文件和内容搜索；
- **`git`** ——项目根目录探测和仓库操作。

没有它们，智能体无法正常运行：`kot doctor` 会把它们报告为缺失的必要依赖。

| 系统 | 安装 |
|---|---|
| Windows | `winget install BurntSushi.ripgrep.MSVC` 和 `winget install Git.Git` |
| macOS | `brew install ripgrep git` |
| Debian, Ubuntu | `sudo apt install ripgrep git` |
| Fedora, RHEL | `sudo dnf install ripgrep git` |
| Arch | `sudo pacman -S ripgrep git` |
| Alpine | `apk add ripgrep git` |

安装后检查：`kot doctor` ——它会打印两个程序被检测到的版本以及配置状态。

## 运行

```
kot                       # same as kot web with default settings
kot web --addr 127.0.0.1:8080 --workspace C:\projects\my-app
kot web --new             # prints the URL that opens a fresh session
```

`kot web` 在 `127.0.0.1:8080` 上启动一个 HTTP 服务器并打印 URL。会话工作目录由 `--workspace` 设置（默认为当前目录）。一次性覆盖项 `--provider`、`--model`、`--search-provider`、`--search-model` 只对本次运行生效，绝不会改动设置文件。

命令：

| 命令 | 用途 |
|---|---|
| `kot web` | Web 界面（默认命令） |
| `kot login [--provider anthropic\|anthropic-oauth\|openai-codex]` | 通过 OAuth 进行订阅登录；Claude 订阅使用 `anthropic-oauth` 提供商 |
| `kot logout [--provider <name>]` | 删除已存储的订阅令牌 |
| `kot config show` | 显示生效配置（密钥已掩码） |
| `kot config set provider <name> [--base-url …]` | 固定默认提供商 |
| `kot config set model <id>` | 固定默认模型 |
| `kot config set key <provider>` | 存储一个 API key（隐藏输入） |
| `kot doctor [--live]` | 检查环境、配置和凭据；`--live` 会对提供商执行一次最小化的真实请求 |

普通登录会在同一次运行中打印授权 URL，并从成功页接收授权码。在没有浏览器的机器上，登录需要两次调用：

```
kot login --headless            # prints the authorization URL and exits
kot login --code <CODE>         # completes the login with the code from the success page
```

全局选项 `--config-dir`（变量 `KOT_CONFIG_DIR`）用于移动整个配置目录。

在没有设置的情况下首次运行时，智能体会引导你选择提供商、输入凭据、挑选模型，然后写入 `settings.json`。

## 提供商与模型

支持：`anthropic`、`anthropic-oauth`、`openai`、`openai-codex`、`gemini`、`deepseek`、`grok`、`openrouter`、`zai`、`moonshotai`、`together`、`fireworks`、`lmstudio`、`lmstudio-native`、`ollama`、`openai-generic`。

凭据有两种接入方式：

- OAuth 订阅——Claude Pro/Max 使用 `kot login --provider anthropic-oauth`，ChatGPT Plus/Pro 使用 `kot login --provider openai-codex`；令牌存储在 `auth/` 中；
- API key——`kot config set key <provider>`；`anthropic` 是仅 API key 的 Anthropic 路由，key 写入 `providers.json`。

`lmstudio`、`lmstudio-native` 和 `ollama` 无需 key 即可在本地运行。`openai-generic` 需要显式的 base URL，并接受可选的 key。运行时的 key 解析顺序：环境变量 → `providers.json` → 外部 `api_key_helper`。

模型目录内嵌在二进制里：每个模型都带有标识符、上下文窗口、输出上限、输入与输出模态、推理和工具支持，以及一个聊天资格标志。只有具备聊天资格的模型才能驱动会话；图像、视频和音频生成模型可供 Media 工具使用。完整目录（含媒体模型）由智能体通过 `Agent{list_models}` 读取；Web 界面在选择模型时列出聊天模型。

**内置目录不认识的模型由你自己添加**——写在 `<config-home>/models.json` 里：`kot models add --provider <名称> --id <id> --for-chat true …`、`kot models list`、`kot models remove`。同一条记录也可以逐字段 **覆盖** 内置模型——修正后的上下文窗口、另一个显示名称、该模型实际接受的推理级别；`kot models add --clear <字段>` 只撤销其中一项覆盖，其余保持不变。在 Web 界面里，这些由模型表单完成：选择器中任意模型旁的铅笔图标编辑该行，最后一行「+ Add model…」新建一条，删除则移除你的记录（用户模型从列表中消失，被覆盖的内置模型回到它自带的描述）。留空的字段表示「按程序解析」，其后的灰色文本显示解析结果。命令行写入的记录会被运行中的 `kot web` 在下次刷新提供商列表时接管——无需重启。

推理强度在会话创建时设置，并可在活跃会话上切换：`off`、`low`、`medium`、`high`、`xhigh`、`max`。

## Web 界面

- 项目与会话列表：创建、恢复、回退到更早的消息、关闭和删除会话。
- 工作目录、提供商、模型和推理强度的选择——创建会话时和活跃会话上均可。
- 实时回答流式输出、正在运行工具的实时卡片、可在发送前编辑和删除的消息队列。
- 硬性回合中断和可撤销的软性中断。
- 后台任务面板：输出、监视器、停止。
- 子智能体和队友面板：子智能体的工作历史、给它发邮件、停止、重启后恢复队友。
- 智能体未同步文件的列表，一键写入磁盘。
- 每个会话的 token 统计：输入、缓存写入、缓存读取、输出，以及其中的推理子计数。

## 智能体工具

| 工具 | 作用 |
|---|---|
| [Agent](tools/Agent-zh.md) | 生成子智能体、雇佣队友、智能体之间通信、持久流水线、选择子智能体的提供商和模型 |
| [Shell](tools/Shell-zh.md) | 在主机 shell 中执行命令、带监视器的后台任务、持久交互式 shell 会话 |
| [Search](tools/Search-zh.md) | 按模式搜索文件、按正则表达式搜索内容、语义搜索 |
| [Files](tools/Files-zh.md) | 通过私有虚拟层读写文件、结构化代码读取、目录、notebook 和图像 |
| [Plan](tools/Plan-zh.md) | 会话清单、带负责人和依赖的持久任务图、规划模式 |
| [Memory](tools/Memory-zh.md) | 带作用域的持久笔记、搜索以及会话开始时的自动预取 |
| [History](tools/History-zh.md) | 读取已记录的历史、恢复过往结果、用于启动子智能体的历史切片、上下文压缩 |
| [Media](tools/Media-zh.md) | 通过已配置模型生成图像、视频、音乐和语音 |
| [Tools](tools/Tools-zh.md) | 管理智能体自己的工具集：添加、移除、搜索、调用 |
| [cli-wrapper](tools/cli-wrapper-zh.md) | 把任意 CLI 脚本变成强类型智能体工具，包括由常驻进程支撑的 |

每一行都指向完整描述：操作、参数、行为、限制和调用示例。

## 编排

智能体把工作分配给其他智能体：

- **同步子智能体**——启动、完成任务并在同一回合内返回报告；
- **委托（delegate）**——在后台工作，通过通知报告完成；你可以给它发消息并停止它；硬性存活上限为 12 小时；
- **队友（teammate）**——由邮件唤醒的常驻参与者，存活直到被显式停止；同一团队的队友共享一个任务看板；
- **完成门**——一个委托不断重复其回合，直到给定的检查命令以零退出码结束；监督智能体可以接受结果、打回返工或停止循环；
- **流水线（pipeline）**——用 TOML 声明的步骤图，带依赖、团队步骤、嵌套流水线和向负责人提问；一次运行在重启后仍可存活并恢复。

子智能体会获得一个隔离会话和它自己的虚拟文件层；必要时，还能获得独立的工作目录或临时的 git 工作树。子智能体的提供商、模型和推理强度要么继承，要么显式设置，包括指定不同的提供商。

## 记忆、历史与上下文

- **Memory** ——三种作用域下的精选笔记：project（通过版本控制共享）、user（在机器上所有项目间共享）、local（此机器上的本项目）。每次会话开始时，笔记会自动提供给智能体。
- **History** ——磁盘上每个会话的完整记录。智能体从中恢复过往结果、列出它们，并把一段历史切片带入新的子智能体会话。
- **上下文压缩** ——较早的前缀被折叠成摘要，而最近的消息保持原样。折叠之前，完整历史会被写入会话旁边的归档，记录本身保持不截断。对子智能体，自动压缩是开启的；对主会话则是关闭的，按需运行。
- **虚拟文件层** ——智能体的编辑与磁盘分离累积，在同步时才落到磁盘。重启后该层会从记录中恢复。

## 磁盘上的数据

配置目录是 `~/.kot`（可被 `KOT_CONFIG_DIR` 覆盖）。

```
~/.kot/
├── settings.json                  # settings
├── providers.json                 # API keys
├── shells.json                    # named shells (optional)
├── auth/                          # subscription tokens
├── memory/                        # notes of the user scope
└── projects/<project-key>/
    ├── <session>.jsonl            # session transcript
    ├── <session>.media/           # externalized attachments
    ├── compact-archives/          # history archives taken before compaction
    └── sidechains/<agent>.jsonl   # sub-agent transcripts
```

项目 key 由仓库根目录（无仓库时则为该目录）派生而来，因此会话绑定到项目而非启动目录。运行时根目录和数据根目录由 `KOT_RUNTIME_DIR` 和 `KOT_DATA_DIR` 设置。

project 和 local 作用域的记忆笔记存放在项目内部：`.kot/agent-memory` 和 `.kot/agent-memory-local`。

## 设置

配置目录中的 `settings.json` 用于设置默认提供商、模型、登录方式、子进程的环境变量、上下文压缩参数，以及守护进程和客户端参数。

有用的环境变量：

| 变量 | 作用 |
|---|---|
| `KOT_CONFIG_DIR`, `KOT_RUNTIME_DIR`, `KOT_DATA_DIR` | 配置、运行时和数据根目录的位置 |
| `KOT_WEB_ADDR`, `KOT_WEB_WORKSPACE`, `KOT_WEB_PROVIDER`, `KOT_WEB_MODEL`, `KOT_WEB_NEW` | `kot web` 各选项的值 |
| `KOT_GLOB_MAX_RESULTS`, `KOT_GLOB_TIMEOUT_SECONDS`, `KOT_GLOB_NO_IGNORE`, `KOT_GLOB_HIDDEN` | 文件搜索的限制和模式 |
| `KOT_TASK_LIST_ID` | 会话所使用的任务看板 |
| `ENABLE_TOOL_SEARCH` | 延迟工具模式 |

## 文档集

- [tools/Agent-zh.md](tools/Agent-zh.md) — 编排：子智能体、队友、邮件、流水线
- [tools/Shell-zh.md](tools/Shell-zh.md) — 命令、后台任务、监视器、交互式会话
- [tools/Search-zh.md](tools/Search-zh.md) — 按名称、按内容和按语义搜索
- [tools/Files-zh.md](tools/Files-zh.md) — 读写文件、虚拟层
- [tools/Plan-zh.md](tools/Plan-zh.md) — 清单、任务图、规划模式
- [tools/Memory-zh.md](tools/Memory-zh.md) — 持久笔记及其作用域
- [tools/History-zh.md](tools/History-zh.md) — 会话历史、恢复与分叉、上下文压缩
- [tools/Media-zh.md](tools/Media-zh.md) — 图像、视频、音乐、语音
- [tools/Tools-zh.md](tools/Tools-zh.md) — 管理工具集
- [tools/cli-wrapper-zh.md](tools/cli-wrapper-zh.md) — 把你自己的 CLI 变成智能体工具
