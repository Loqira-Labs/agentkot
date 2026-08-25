![KOT](assets/title.jpg)

# KOT — simply the best AI harness

<h3 align="center">
  <b>English</b> &nbsp;·&nbsp; <a href="README-ru.md">Русский</a> &nbsp;·&nbsp; <a href="README-zh.md">简体中文</a>
</h3>

**One file is all you need.**

The best working agent you can install on your own machine. One executable: it reads code and documents, searches the project, edits files, runs commands and long-lived processes, keeps a plan and a task board, remembers what matters between sessions, hires other agents and splits the work between them — across several providers at once.

It does not ask for permission, does not perform concern, and does not adapt to a broken process. It is a tool. Treat it as a tool.

## Position

**There is no security, and there will not be.** The agent runs commands and edits files with the rights of the user who started it. No sandbox, no allow/deny/ask rules, no confirmation on every step. This is a deliberate decision: an agent that asks for permission every other step does not finish the job, it burns the session on a conversation about intentions. If you need isolation, take a ready-made one — a container, a virtual machine, a separate user account, any of the hundreds of agent sandboxes — and run `kot` inside it. Isolation is a property of the environment, not of the agent.

**Skills are not needed.** A skill is a text with instructions that someone wrapped into a separate mechanism with a registry, an installer and versions. You have an agent that reads files and follows instructions. Want to work by a skill — give the agent the document and tell it to follow it. A separate subsystem for reading text will not appear here.

**MCP is not needed.** MCP is a transport for tools an agent does not have. Here they are built in: shell, files, search, memory, history, media generation, orchestration. Need an external service once — tell the agent to call it with a command. Need it permanently — wrap its CLI into a dynamic tool through `Tools{add: "cli-wrapper"}`, and the agent gets typed operations with parameter validation, without an intermediate protocol, an extra process and its protocol limits.

**It will not adapt to your process.** State the task, give context, check the result. If you want an agent that politely agrees with a badly stated task and adapts to a broken pipeline — take Claude Code, Codex and the rest.

## The core strength: a team across providers

One agent on one model is either slow, or shallow, or a bottleneck. Here roles are spread across providers, and every participant runs on the model that fits its job.

An example of such a split — the set of providers and models is yours to choose, by your tasks and available subscriptions:

| Role | Example provider | Why |
|---|---|---|
| Lead agent | anthropic | holds the whole task, writes code, makes decisions |
| Architect, reviewer | openai | an independent view from another school on plans and diffs |
| Researchers, code reconnaissance | deepseek | dozens of parallel reads on a light model |
| Security review, a second pair of eyes | zai | one more independent pass on a separate quota |
| Semantic search inside `Search{smart}` | any light provider | search must not occupy the main model |

How it is enabled:

- every key in `providers.json` automatically becomes an available launch target — no separate setup;
- `Agent{spawn|delegate|hire, provider, model, effort}` — the child starts on the chosen provider, and the parent's model never leaks across providers;
- `Agent{set_model}` — switches a live teammate to another provider and model between turns;
- a pipeline step and a team-step member carry their own `provider` and `model`;
- `kot web --search-provider … --search-model …` — semantic search moves to a separate, light provider.

Current model identifiers of every provider are read by the agent itself through `Agent{list_models}`, and they are visible in the web interface when picking a model.

The result: a parallel team where the strongest model does what actually requires it, while routine work and reconnaissance run on light ones.

## Benchmarks

[Harness-Bench](https://github.com/Qihoo360/harness-bench) measures the harness itself: 106 sandboxed offline agent tasks across 8 workflow categories, fixed model, deterministic oracle validators.

Same model, DeepSeek V4 Flash — the harness decides:

| Harness | Score |
|---|---|
| **KOT** | **79.1%** |
| Hermes | 76.2% |

KOT on a cheap fast model outperforms Hermes on the same model — by 2.9 points on the full 106-task suite.

## What ships

Prebuilt binaries sit next to this file:

| Platform | File |
|---|---|
| Linux (x86-64) | `releases/linux/kot` |
| macOS (Apple Silicon) | `releases/macos/kot.bin` |
| Windows (x86-64) | `releases/windows/kot.exe` |

The web interface (page, script, styles), icons, tokenizer and output-compression rules are embedded in the binary — nothing has to be placed next to it. Put the file into any directory on `PATH`.

On macOS and Linux it needs the executable bit, and on macOS also the quarantine flag removed if the file arrived over the network:

```
mv kot.bin kot && chmod +x kot        # macOS
xattr -d com.apple.quarantine kot     # macOS, if the file was downloaded by a browser
chmod +x kot                          # Linux
```

### What must be present in the system

Two external programs, both must be available on `PATH`:

- **`rg` (ripgrep)** — file and content search;
- **`git`** — project-root detection and repository work.

Without them the agent does not run properly: `kot doctor` reports them as missing required dependencies.

| System | Install |
|---|---|
| Windows | `winget install BurntSushi.ripgrep.MSVC` and `winget install Git.Git` |
| macOS | `brew install ripgrep git` |
| Debian, Ubuntu | `sudo apt install ripgrep git` |
| Fedora, RHEL | `sudo dnf install ripgrep git` |
| Arch | `sudo pacman -S ripgrep git` |
| Alpine | `apk add ripgrep git` |

Check after installing: `kot doctor` — it prints the detected versions of both programs and the state of the configuration.

## Running

```
kot                       # same as kot web with default settings
kot web --addr 127.0.0.1:8080 --workspace C:\projects\my-app
kot web --new             # prints the URL that opens a fresh session
```

`kot web` starts an HTTP server on `127.0.0.1:8080` and prints the URL. The session working directory is set by `--workspace` (the current directory by default). One-off overrides `--provider`, `--model`, `--search-provider`, `--search-model` apply to this run only and never touch the settings file.

Commands:

| Command | Purpose |
|---|---|
| `kot web` | web interface (the default command) |
| `kot login [--provider anthropic\|anthropic-oauth\|openai-codex]` | subscription login over OAuth; Claude subscriptions use provider `anthropic-oauth` |
| `kot logout [--provider <name>]` | remove a stored subscription token |
| `kot config show` | show the effective configuration (secrets masked) |
| `kot config set provider <name> [--base-url …]` | pin the default provider |
| `kot config set model <id>` | pin the default model |
| `kot config set key <provider>` | store an API key (hidden input) |
| `kot doctor [--live]` | check the environment, configuration and credentials; `--live` performs a minimal real request to the provider |

The normal login prints the authorization URL and accepts the code from the success page in the same run. On a machine without a browser the login takes two calls:

```
kot login --headless            # prints the authorization URL and exits
kot login --code <CODE>         # completes the login with the code from the success page
```

The global option `--config-dir` (variable `KOT_CONFIG_DIR`) moves the whole configuration directory.

On the first run without settings the agent walks through choosing a provider, entering credentials and picking a model, then writes `settings.json`.

## Providers and models

Supported: `anthropic`, `anthropic-oauth`, `openai`, `openai-codex`, `gemini`, `deepseek`, `grok`, `openrouter`, `zai`, `moonshotai`, `together`, `fireworks`, `lmstudio`, `lmstudio-native`, `ollama`, `openai-generic`.

Credentials are connected in two ways:

- OAuth subscription — `kot login --provider anthropic-oauth` for Claude Pro/Max and `kot login --provider openai-codex` for ChatGPT Plus/Pro; tokens are stored in `auth/`;
- API key — `kot config set key <provider>`; `anthropic` is the Anthropic API-key-only route and the key is written to `providers.json`.

`lmstudio`, `lmstudio-native` and `ollama` run locally without a key. `openai-generic` requires an explicit base URL and accepts an optional key. Key resolution order at runtime: environment variable → `providers.json` → the external `api_key_helper`.

The model catalog is embedded in the binary: every model carries an identifier, a context window, an output limit, input and output modalities, reasoning and tool support, and a chat-eligibility flag. Only a chat-eligible model can drive a session; image, video and audio generation models are available to the Media tool. The full catalog, media models included, is read by the agent through `Agent{list_models}`; the web interface lists chat models when picking one.

**A model the built-in catalog does not know is added by you**, in `<config-home>/models.json`: `kot models add --provider <name> --id <id> --for-chat true …`, `kot models list`, `kot models remove`. The same record can OVERRIDE a built-in model field by field — a corrected context window, another display name, the reasoning levels the model really accepts — and `kot models add --clear <field>` drops one override back to the shipped value without deleting the rest. In the web interface the same thing is done by the model form: the pencil next to any model of the selector edits that row, the last row "+ Add model…" creates a new one, and deletion removes your record (a user model leaves the list, an overridden built-in one returns to its shipped description). A field left empty means "as the program resolves it", and the grey text behind it shows what that resolves to. A running `kot web` picks up a record written by the CLI on its next provider refresh — no restart.

The reasoning effort is set when a session is created and switched on a live session: `off`, `low`, `medium`, `high`, `xhigh`, `max`.

## Web interface

- The list of projects and sessions: create, resume, rewind to an earlier message, close and delete a session.
- Choice of working directory, provider, model and reasoning effort — both when creating a session and on a live one.
- Real-time answer streaming, a live card of the running tool, a message queue with editing and removal before delivery.
- Hard turn interruption and a revocable soft interruption.
- Background task panel: output, monitors, stop.
- Sub-agent and teammate panel: the child's work history, mail to it, stop, restoring teammates after a restart.
- The list of the agent's unsynced files and writing them to disk with one button.
- Token accounting per session: input, cache write, cache read, output, and the reasoning sub-count.

## Agent tools

| Tool | What it does |
|---|---|
| [Agent](tools/Agent.md) | spawning sub-agents, hiring teammates, mail between agents, durable pipelines, choosing the child's provider and model |
| [Shell](tools/Shell.md) | commands in the host shell, background tasks with monitors, persistent interactive shell sessions |
| [Search](tools/Search.md) | file search by pattern, content search by regular expression, semantic search |
| [Files](tools/Files.md) | reading and editing files through a private virtual layer, structural code reads, directories, notebooks and images |
| [Plan](tools/Plan.md) | the session checklist, a durable task graph with owners and dependencies, planning mode |
| [Memory](tools/Memory.md) | durable notes with scopes, search and automatic prefetch at session start |
| [History](tools/History.md) | reading the recorded history, recovering past results, a history slice for launching a child, context compaction |
| [Media](tools/Media.md) | image, video, music and speech generation through the configured models |
| [Tools](tools/Tools.md) | managing the agent's own tool set: add, remove, search, invoke |
| [cli-wrapper](tools/cli-wrapper.md) | turning any CLI script into a typed agent tool, including one backed by a resident process |

Every row leads to the full description: operations, parameters, behaviour, limits and call examples.

## Orchestration

The agent distributes work across other agents:

- **synchronous sub-agent** — starts, does the job and returns its report within the same turn;
- **delegate** — works in the background and reports completion with a notification; you can message it and stop it; the hard lifetime limit is 12 hours;
- **teammate** — a permanent participant woken by mail and alive until it is stopped explicitly; teammates of one team share a task board;
- **completion gate** — a delegate repeats its turn until the given check command exits with zero; a supervisor agent can accept the result, send it back for rework, or stop the loop;
- **pipeline** — a TOML-declared graph of steps with dependencies, team steps, nested pipelines and questions to the owner; a run survives a restart and resumes.

A child gets an isolated session and its own virtual files layer; when needed, a separate working directory or a temporary git worktree. The child's provider, model and reasoning effort are either inherited or set explicitly, including a different provider.

## Memory, history and context

- **Memory** — curated notes in three scopes: project (shared through version control), user (shared across all projects on the machine), local (this project on this machine). At the start of every session the notes are supplied to the agent automatically.
- **History** — a full transcript of every session on disk. The agent recovers past results from it, lists them, and carries a history slice into a new child session.
- **Context compaction** — the older prefix is folded into a summary while the most recent messages stay verbatim. Before folding, the full history is written to an archive next to the session, and the transcript itself stays untruncated. For sub-agents automatic compaction is on, for the main session it is off and runs on demand.
- **Virtual files layer** — the agent's edits accumulate separately from disk and reach it on sync. The layer is restored from the transcript after a restart.

## Data on disk

The configuration directory is `~/.kot` (overridden by `KOT_CONFIG_DIR`).

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

The project key is derived from the repository root (or the directory when there is no repository), so sessions are bound to the project rather than to the directory of launch. The runtime and data roots are set by `KOT_RUNTIME_DIR` and `KOT_DATA_DIR`.

Project and local memory notes live inside the project itself: `.kot/agent-memory` and `.kot/agent-memory-local`.

## Settings

`settings.json` in the configuration directory sets the default provider, the model, the login method, environment variables for child processes, context-compaction parameters, and daemon and client parameters.

Useful environment variables:

| Variable | Effect |
|---|---|
| `KOT_CONFIG_DIR`, `KOT_RUNTIME_DIR`, `KOT_DATA_DIR` | locations of the configuration, runtime and data roots |
| `KOT_WEB_ADDR`, `KOT_WEB_WORKSPACE`, `KOT_WEB_PROVIDER`, `KOT_WEB_MODEL`, `KOT_WEB_NEW` | values of the `kot web` options |
| `KOT_GLOB_MAX_RESULTS`, `KOT_GLOB_TIMEOUT_SECONDS`, `KOT_GLOB_NO_IGNORE`, `KOT_GLOB_HIDDEN` | limits and mode of the file search |
| `KOT_TASK_LIST_ID` | the task board the session works with |
| `ENABLE_TOOL_SEARCH` | deferred-tools mode |

## Documentation set

- [tools/Agent.md](tools/Agent.md) — orchestration: sub-agents, teammates, mail, pipelines
- [tools/Shell.md](tools/Shell.md) — commands, background tasks, monitors, interactive sessions
- [tools/Search.md](tools/Search.md) — search by name, by content and by meaning
- [tools/Files.md](tools/Files.md) — reading and editing files, the virtual layer
- [tools/Plan.md](tools/Plan.md) — checklist, task graph, planning mode
- [tools/Memory.md](tools/Memory.md) — durable notes and their scopes
- [tools/History.md](tools/History.md) — session history, recovery and forking, context compaction
- [tools/Media.md](tools/Media.md) — images, video, music, speech
- [tools/Tools.md](tools/Tools.md) — managing the tool set
- [tools/cli-wrapper.md](tools/cli-wrapper.md) — your own CLI as an agent tool
