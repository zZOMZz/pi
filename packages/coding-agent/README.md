<p align="center">
  <a href="https://pi.dev">
    <img alt="pi 标志" src="https://pi.dev/logo-auto.svg" width="128">
  </a>
</p>
<p align="center">
  <a href="https://discord.com/invite/3cU7Bz4UPx"><img alt="Discord" src="https://img.shields.io/badge/discord-community-5865F2?style=flat-square&logo=discord&logoColor=white" /></a>
  <a href="https://www.npmjs.com/package/@earendil-works/pi-coding-agent"><img alt="npm" src="https://img.shields.io/npm/v/@earendil-works/pi-coding-agent?style=flat-square" /></a>
</p>

> 新贡献者新建的 issue 和 PR 默认会被自动关闭。维护者每天都会检查这些自动关闭的 issue。详见 [CONTRIBUTING.md](../../CONTRIBUTING.md)。

---

Pi 是一个轻量的终端编程工具。你可以让 pi 适应自己的工作流，而不是反过来；无需 fork，也不用改 pi 的内部实现。它支持用 TypeScript [扩展](#extensions)、[技能](#skills)、[提示词模板](#prompt-templates)和[主题](#themes)来扩展。把这些内容打包成 [Pi Package](#pi-packages)，就能通过 npm 或 git 分享给其他人。

Pi 自带的默认能力已经很实用，但没有内置子 agent、计划模式这类功能。你可以直接让 pi 帮你做出来，或者安装符合自己工作流的第三方 Pi Package。

Pi 有四种运行方式：交互模式、打印或 JSON 输出模式、用于进程集成的 RPC 模式，以及可嵌入自己应用的 SDK。

<a id="share-your-oss-coding-agent-sessions"></a>

## 分享你的开源编程 agent 会话

如果你用 pi 做开源项目，欢迎分享你的编程 agent 会话记录。

公开的开源会话数据能用真实开发工作流，帮助改进模型、提示词、工具和评估。

完整说明见 [X 上的这篇帖子](https://x.com/badlogicgames/status/2037811643774652911)。

要发布会话，请使用 [`badlogic/pi-share-hf`](https://github.com/badlogic/pi-share-hf)。安装方式请看它的 README.md。你只需要一个 Hugging Face 账号、Hugging Face CLI 和 `pi-share-hf`。

也可以看看[这个视频](https://x.com/badlogicgames/status/2041151967695634619)，里面演示了我如何发布 `pi-mono` 的会话记录。

我会定期在这里发布自己的 `pi-mono` 工作会话：

- [Hugging Face 上的 badlogicgames/pi-mono](https://huggingface.co/datasets/badlogicgames/pi-mono)

## 目录

- [快速开始](#quick-start)
- [Provider 和模型](#providers--models)
- [交互模式](#interactive-mode)
  - [编辑器](#editor)
  - [命令](#commands)
  - [键盘快捷键](#keyboard-shortcuts)
  - [消息队列](#message-queue)
- [会话](#sessions)
  - [分支](#branching)
  - [压缩](#compaction)
- [设置](#settings)
- [上下文文件](#context-files)
- [自定义](#customization)
  - [提示词模板](#prompt-templates)
  - [技能](#skills)
  - [扩展](#extensions)
  - [主题](#themes)
  - [Pi Package](#pi-packages)
- [以编程方式使用](#programmatic-usage)
- [设计理念](#philosophy)
- [CLI 参考](#cli-reference)

---

<a id="quick-start"></a>

## 快速开始

```bash
npm install -g --ignore-scripts @earendil-works/pi-coding-agent
```

`--ignore-scripts` 会在安装时禁用依赖的生命周期脚本。正常通过 npm 安装 Pi 不需要执行安装脚本。

也可以用安装脚本：

```bash
curl -fsSL https://pi.dev/install.sh | sh
```

使用 API key 登录：

```bash
export ANTHROPIC_API_KEY=sk-ant-...
pi
```

或者使用已有订阅：

```bash
pi
/login  # 然后选择 Provider
```

接下来直接和 pi 对话即可。默认情况下，pi 会给模型四个工具：`read`、`write`、`edit` 和 `bash`，模型会用它们完成你的请求。还可以通过[技能](#skills)、[提示词模板](#prompt-templates)、[扩展](#extensions)或 [Pi Package](#pi-packages)增加能力。

**各平台说明：** [Windows](docs/windows.md) | [Termux（Android）](docs/termux.md) | [tmux](docs/tmux.md) | [终端设置](docs/terminal-setup.md) | [Shell 别名](docs/shell-aliases.md)

---

<a id="providers--models"></a>

## Provider 和模型

Pi 会为每个内置 Provider 维护一份支持调用工具的模型列表。已配置 Provider 的模型目录会自动刷新；如需立刻刷新，运行 `pi update --models`。你可以通过订阅（`/login`）或 API key 登录，然后用 `/model`（或 Ctrl+L）选择该 Provider 的任意模型。在模型选择器中按 Ctrl+S，可以把当前高亮的模型保存为启动默认值。

**订阅：**
- Anthropic Claude Pro/Max
- OpenAI ChatGPT Plus/Pro (Codex)
- GitHub Copilot

**API key：**
- Anthropic
- Ant Ling
- OpenAI
- Azure OpenAI
- DeepSeek
- NVIDIA NIM
- Google Gemini
- Google Vertex
- Amazon Bedrock
- Mistral
- Groq
- Cerebras
- Cloudflare AI Gateway
- Cloudflare Workers AI
- xAI
- OpenRouter
- Vercel AI Gateway
- ZAI Coding Plan (Global)
- ZAI Coding Plan (China)
- OpenCode Zen
- OpenCode Go
- Hugging Face
- Fireworks
- Together AI
- Baseten
- Kimi For Coding
- MiniMax
- Xiaomi MiMo
- Xiaomi MiMo Token Plan (China)
- Xiaomi MiMo Token Plan (Amsterdam)
- Xiaomi MiMo Token Plan (Singapore)

Pi 也支持 llama.cpp router server。用 `/login llama.cpp` 配置它，用 `/llama` 管理下载和已加载的模型，再通过 `/model` 选择已加载的模型。安装和使用说明见 [docs/llama-cpp.md](docs/llama-cpp.md)。

其他 Provider 的配置方式见 [docs/providers.md](docs/providers.md)。

**自定义 Provider 和模型：** 如果它们兼容受支持的 API（OpenAI、Anthropic、Google），可通过 `~/.pi/agent/models.json` 添加 Provider。自定义 API 或 OAuth 请使用扩展。详见 [docs/models.md](docs/models.md) 和 [docs/custom-provider.md](docs/custom-provider.md)。

---

<a id="interactive-mode"></a>

## 交互模式

<p align="center"><img src="docs/images/interactive-mode.png" alt="交互模式" width="600"></p>

界面从上到下依次是：

- **启动信息栏**：显示快捷键（完整列表见 `/hotkeys`）、已加载的 AGENTS.md、提示词模板、技能和扩展
- **消息区**：你的消息、助手回复、工具调用及结果、通知、错误和扩展 UI
- **编辑器**：输入内容的地方；边框颜色表示当前思考等级
- **底栏**：工作目录、会话名称、累计 token/缓存用量（`↑` 输入、`↓` 输出、`R` 缓存读取、`W` 缓存写入、`CH` 最近一次缓存命中率）、费用、上下文用量和当前模型。累计值包含助手回复、工具上报的用量及摘要生成。

编辑器可以临时换成其他 UI，例如内置的 `/settings`，或由扩展提供的自定义 UI（比如让用户以结构化方式回答模型问题的问答工具）。[扩展](#extensions)还可以替换编辑器、在其上下添加组件、添加状态栏、自定义底栏或覆盖层。

<a id="editor"></a>

### 编辑器

| 功能 | 操作 |
|---------|-----|
| 引用文件 | 输入 `@`，模糊搜索项目文件 |
| 路径补全 | 按 Tab 补全路径 |
| 多行输入 | Shift+Enter（Windows Terminal 中为 Ctrl+Enter） |
| 外部编辑器 | Ctrl+G 会打开 `externalEditor`、`$VISUAL`、`$EDITOR`；Windows 上用 Notepad，其他系统用 `nano` |
| 剪贴板 | Ctrl+V 粘贴图片或文本（Windows 为 Alt+V），也可把图片拖进终端 |
| Bash 命令 | `!command` 执行并把输出发送给 LLM，`!!command` 只执行、不发送 |

删除单词、撤销等操作使用标准编辑快捷键。详见 [docs/keybindings.md](docs/keybindings.md)。

<a id="commands"></a>

### 命令

在编辑器里输入 `/` 可唤起命令。[扩展](#extensions)可以注册自定义命令，[技能](#skills)以 `/skill:name` 的形式使用，[提示词模板](#prompt-templates)则通过 `/templatename` 展开。

| 命令 | 说明 |
|---------|-------------|
| `/login`、`/logout` | 管理 Provider 凭据 |
| [`/llama`](docs/llama-cpp.md) | 下载、加载和卸载 llama.cpp router 模型 |
| `/model` | 切换模型；在选择器中按 Ctrl+S 保存启动默认值 |
| `/thinking` | 切换思考等级；在选择器中按 Ctrl+S 保存启动默认值 |
| `/scoped-models` | 启用或禁用 Ctrl+P 循环切换时使用的模型 |
| `/settings` | 设置主题、消息发送方式、传输方式等偏好 |
| `/resume` | 从历史会话中选择一个继续 |
| `/new` | 新建会话 |
| `/name <name>` | 设置会话显示名称 |
| `/session` | 显示会话信息（文件、ID、消息数、token、费用） |
| `/tree` | 跳到会话中的任意位置，并从那里继续 |
| `/trust` | 保存项目的信任决定，供后续会话使用（需要重启） |
| `/fork` | 从之前的一条用户消息新建会话 |
| `/clone` | 将当前活动分支复制到新会话 |
| `/compact [prompt]` | 手动压缩上下文，也可附加自定义指令 |
| `/copy` | 将最后一条助手消息复制到剪贴板 |
| `/export [file]` | 将会话导出为 HTML 或 JSONL 文件 |
| `/import <file>` | 从 JSONL 文件导入并恢复会话 |
| `/share` | 上传为私有 GitHub gist，并生成可分享的 HTML 链接 |
| `/reload` | 重新加载快捷键、扩展、技能、提示词、主题和上下文文件 |
| `/hotkeys` | 显示全部键盘快捷键 |
| `/changelog` | 显示版本历史 |
| `/quit` | 退出 pi |

<a id="keyboard-shortcuts"></a>

### 键盘快捷键

完整列表见 `/hotkeys`。可以通过 `~/.pi/agent/keybindings.json` 自定义。详见 [docs/keybindings.md](docs/keybindings.md)。

**常用快捷键：**

| 按键 | 操作 |
|-----|--------|
| Ctrl+C | 清空编辑器 |
| 连按 Ctrl+C 两次 | 退出 |
| Escape | 取消或中止 |
| 连按 Escape 两次 | 打开 `/tree` |
| Ctrl+L | 打开模型选择器 |
| Ctrl+P / Shift+Ctrl+P | 向前/向后循环切换已设范围的模型 |
| Shift+Tab | 循环切换思考等级 |
| Ctrl+O | 折叠/展开工具输出 |
| Ctrl+T | 折叠/展开思考块 |
| Ctrl+X | 复制最后一条助手消息；若全屏模式关闭了选中即复制，则复制当前选中的文本 |

<a id="message-queue"></a>

### 消息队列

agent 工作时也可以继续提交消息：

- **Enter**：将消息加入 *steering* 队列；当前助手回合执行完工具调用后发送
- **Alt+Enter**：将消息加入 *follow-up* 队列；仅在 agent 完成所有工作后发送
- **Escape**：中止当前工作，并把排队消息放回编辑器
- **Alt+Up**：将排队消息取回编辑器

Windows Terminal 默认把 `Alt+Enter` 用于全屏。请按 [docs/terminal-setup.md](docs/terminal-setup.md) 的说明重新映射它，让 pi 能收到 follow-up 快捷键。

可在[设置](docs/settings.md)中配置发送方式：`steeringMode` 和 `followUpMode` 可设为 `"one-at-a-time"`（默认，每条等待响应）或 `"all"`（一次发送所有排队消息）。对支持多种传输方式的 Provider，`transport` 用于选择偏好：`"sse"`、`"websocket"` 或 `"auto"`。

---

<a id="sessions"></a>

## 会话

会话以带树结构的 JSONL 文件保存。每一项都有 `id` 和 `parentId`，所以可以直接在同一文件里分支，无需创建新文件。文件格式见 [docs/session-format.md](docs/session-format.md)。

### 管理

会话会按工作目录自动保存到 `~/.pi/agent/sessions/`。

```bash
pi -c                  # 继续最近的会话
pi -r                  # 浏览并选择历史会话
pi --no-session        # 临时模式（不保存）
pi --name "my task"    # 启动时设置会话显示名称
pi --session <path|id> # 使用指定会话文件或 ID
pi --fork <path|id>    # 从指定会话文件或 ID fork 出新会话
```

在交互模式中使用 `/session` 查看当前会话 ID，再通过 `--session <id>` 或 `--fork <id>` 复用它。

<a id="branching"></a>

### 分支

**`/tree`**：直接浏览会话树。选择任意历史节点后从那里继续，也可以在不同分支间切换。所有历史都保留在同一个文件里。

<p align="center"><img src="docs/images/tree-view.png" alt="会话树视图" width="600"></p>

- 输入文字即可搜索；用 Ctrl+←/Ctrl+→ 或 Alt+←/Alt+→ 折叠、展开和跳转分支；用 ←/→ 翻页
- 过滤模式（Ctrl+O）：默认 → 无工具 → 仅用户 → 仅已标记 → 全部
- 按 Ctrl+X 复制选中的消息
- 按 Shift+L 为条目添加书签标签，按 Shift+T 切换是否显示标签时间戳

**`/fork`**：从活动分支中过去的一条用户消息创建新会话文件。它会打开选择器，复制到该位置为止的活动路径，并把选中的提示词放入编辑器供你修改。

**`/clone`**：在当前位置把当前活动分支复制为新会话文件。新会话会保留完整的活动路径历史，并以空编辑器打开。

**`--fork <path|id>`**：直接通过 CLI fork 已有会话文件或部分会话 UUID。这会把完整的源会话复制到当前项目中的新会话文件。

<a id="compaction"></a>

### 压缩

会话太长时可能耗尽上下文窗口。压缩会概括较早的消息，同时保留最近的消息。

**手动：** `/compact` 或 `/compact <custom instructions>`

**自动：** 默认启用。上下文溢出时会恢复并重试，接近上限时则会提前触发。可通过 `/settings` 或 `settings.json` 配置。

压缩会丢失部分细节。完整历史仍保留在 JSONL 文件中，可通过 `/tree` 回看。可使用[扩展](#extensions)自定义压缩行为；内部机制见 [docs/compaction.md](docs/compaction.md)。

---

<a id="settings"></a>

## 设置

可用 `/settings` 修改常用选项，也可以直接编辑 JSON 文件：

| 位置 | 生效范围 |
|----------|-------|
| `~/.pi/agent/settings.json` | 全局（所有项目） |
| `.pi/settings.json` | 项目级（覆盖全局设置） |

全部选项见 [docs/settings.md](docs/settings.md)。

### 项目信任

交互模式启动时，如果项目目录中包含项目级设置、资源或项目 `.agents/skills`，且该目录及其父目录在 `~/.pi/agent/trust.json` 中没有已保存的决定，pi 会先询问是否信任该项目。信任后，pi 可以加载 `.pi/settings.json` 和 `.pi` 资源、安装缺失的项目包，并执行项目扩展。

在做出信任决定前，pi 只加载上下文文件、用户/全局扩展和通过 CLI `-e` 指定的扩展，以便它们处理 `project_trust` 事件。项目级扩展、由项目包管理的扩展和项目设置，只有在项目被信任后才会加载。切换到另一个 cwd 的会话时，如果该 cwd 的信任状态尚未在当前进程中确定，也会遵循这一规则。

非交互模式（`-p`、`--mode json` 和 `--mode rpc`）不会显示信任提示。没有适用的已保存决定时，它们会使用全局设置中的 `defaultProjectTrust`：`ask`（默认）和 `never` 会忽略这些项目资源，`always` 则会信任它们。可传入 `--approve`/`-a` 或 `--no-approve`/`-na`，只覆盖本次运行的项目信任设置。

如果没有扩展或已保存的决定适用，则由 `defaultProjectTrust` 决定回退行为。可在 `~/.pi/agent/settings.json` 中将它设为 `"ask"`、`"always"` 或 `"never"`，也可以通过 `/settings` 修改。

`pi config` 和包管理命令使用相同的项目信任流程，但 `pi update` 永远不会提示。对单条命令，可传入 `--approve` 以信任项目级设置，或传入 `--no-approve` 忽略它们。

在交互模式中使用 `/trust`，可保存当前项目及其直接父目录的信任决定，供后续会话使用。它只会写入 `~/.pi/agent/trust.json`；当前会话不会重新加载，因此需要重启 pi 才会生效。

### 遥测与更新检查

Pi 启动时有两个彼此独立的功能：

- **更新检查：** 请求 `https://pi.dev/api/latest-version`，检查是否有新版 Pi。设定 `PI_SKIP_VERSION_CHECK=1` 可禁用。禁用更新检查只会关闭这一项。
- **安装/更新遥测：** 首次安装后，或检测到 changelog 更新后，会向 `https://pi.dev/api/report-install` 发送匿名版本 ping。这个设置也控制 OpenRouter、Cloudflare 和直连 NVIDIA NIM 请求中可选的 Provider 归因请求头。要退出，可在 `settings.json` 中将 `enableInstallTelemetry` 设为 `false`，或设定 `PI_TELEMETRY=0`。这不会禁用更新检查；除非关闭更新检查或启用离线模式，否则 Pi 仍可能联系 `pi.dev` 获取最新版本。

使用 `--offline` 或 `PI_OFFLINE=1`，可禁用这里提到的全部启动网络操作，包括更新检查、包更新检查以及安装/更新遥测。

---

<a id="context-files"></a>

## 上下文文件

Pi 启动时会从以下位置加载 `AGENTS.md`（或 `CLAUDE.md`）：
- `~/.pi/agent/AGENTS.md` (global)
- 父目录（从 cwd 向上逐级查找）
- 当前目录

如果一个目录中有 `AGENTS.override.md`，Pi 会加载它，而不加载该目录中的 `AGENTS.md` 或 `CLAUDE.md`。其他目录中的上下文文件仍会被拼接起来。

可用来写项目说明（`AGENTS.md`/`CLAUDE.md`）、约定和常用命令。所有匹配的文件都会被拼接。

使用 `--no-context-files`（或 `-nc`）禁用上下文文件加载。

### 系统提示词（System Prompt）

用 `.pi/SYSTEM.md`（项目级）或 `~/.pi/agent/SYSTEM.md`（全局）替换默认 System Prompt。若只想追加、不替换，请使用 `APPEND_SYSTEM.md`。

---

<a id="customization"></a>

## 自定义

<a id="prompt-templates"></a>

### 提示词模板

将可复用提示词写成 Markdown 文件。输入 `/name` 即可展开。

```markdown
<!-- ~/.pi/agent/prompts/review.md -->
检查这段代码中的 bug、安全问题和性能问题。
重点关注：{{focus}}
```

放在 `~/.pi/agent/prompts/`、`.pi/prompts/` 或 [Pi Package](#pi-packages) 中，即可分享给其他人。详见 [docs/prompt-templates.md](docs/prompt-templates.md)。

<a id="skills"></a>

### 技能

遵循 [Agent Skills 标准](https://agentskills.io)的按需能力包。可以通过 `/skill:name` 调用，也可以让 agent 自动加载。

```markdown
<!-- ~/.pi/agent/skills/my-skill/SKILL.md -->
# 我的技能
当用户询问 X 时使用此技能。

## 步骤
1. 先做这个
2. 再做那个
```

放在 `~/.pi/agent/skills/`、`~/.agents/skills/`、`.pi/skills/` 或 `.agents/skills/`（从 `cwd` 到各级父目录）中，或放入 [Pi Package](#pi-packages) 后分享给其他人。详见 [docs/skills.md](docs/skills.md)。

<a id="extensions"></a>

### 扩展

<p align="center"><img src="docs/images/doom-extension.png" alt="Doom 扩展" width="600"></p>

通过 TypeScript 模块为 pi 加入自定义工具、命令、键盘快捷键、事件处理器和 UI 组件。

```typescript
export default function (pi: ExtensionAPI) {
  pi.registerTool({ name: "deploy", ... });
  pi.registerCommand("stats", { ... });
  pi.on("tool_call", async (event, ctx) => { ... });
}
```

默认导出也可以是 `async`。pi 会等待异步扩展工厂完成后才继续启动，适合在调用 `pi.registerProvider()` 前获取远程模型列表等一次性初始化工作。

**可以做什么：**

- 自定义工具（也可完全替换内置工具）
- 子 agent 和计划模式
- 自定义压缩和摘要
- 权限关卡和路径保护
- 自定义编辑器和 UI 组件
- 状态栏、启动信息栏和底栏
- Git checkpoint 和自动提交
- SSH 和沙箱执行
- 集成 MCP server
- 把 pi 做成 Claude Code 的样子
- 等待时玩游戏（没错，能运行 Doom）
- ……想到什么都可以做

放在 `~/.pi/agent/extensions/`、`.pi/extensions/` 或 [Pi Package](#pi-packages) 中，即可分享给其他人。详见 [docs/extensions.md](docs/extensions.md) 和 [examples/extensions/](examples/extensions/)。

<a id="themes"></a>

### 主题

内置主题：`dark`、`light`。主题支持热重载：修改正在使用的主题文件后，pi 会立即应用变更。

放在 `~/.pi/agent/themes/`、`.pi/themes/` 或 [Pi Package](#pi-packages) 中，即可分享给其他人。详见 [docs/themes.md](docs/themes.md)。

<a id="pi-packages"></a>

### Pi Package

通过 npm 或 git 打包并分享扩展、技能、提示词和主题。可在 [npmjs.com](https://www.npmjs.com/search?q=keywords%3Api-package) 或 [Discord](https://discord.com/channels/1456806362351669492/1457744485428629628) 查找 Package。

> **安全提示：** Pi Package 拥有完整系统访问权限。扩展可以执行任意代码，技能可以指示模型执行包括运行可执行文件在内的任何操作。安装第三方 Package 前，请先审查源代码。

```bash
pi install npm:@foo/pi-tools
pi install npm:@foo/pi-tools@1.2.3      # 固定版本
pi install git:github.com/user/repo
pi install git:github.com/user/repo@v1  # tag 或 commit
pi install git:git@github.com:user/repo
pi install git:git@github.com:user/repo@v1  # tag 或 commit
pi install https://github.com/user/repo
pi install https://github.com/user/repo@v1      # tag 或 commit
pi install ssh://git@github.com/user/repo
pi install ssh://git@github.com/user/repo@v1    # tag 或 commit
pi remove npm:@foo/pi-tools
pi uninstall npm:@foo/pi-tools          # remove 的别名
pi list
pi update                               # 只更新 pi
pi update --all                         # 更新 pi 和 Package
pi update --extensions                  # 只更新 Package
pi update --models                      # 只刷新模型目录
pi update --self                        # 只更新 pi
pi update --self --force                # 即使已是当前版本也重新安装 pi
pi update npm:@foo/pi-tools             # 更新一个 Package
pi config                               # 启用/禁用扩展、技能、提示词、主题
```

Package 会安装到 `~/.pi/agent/git/`（git）或 `~/.pi/agent/npm/`（npm）。使用 `-l` 可安装到项目本地（`.pi/git/`、`.pi/npm/`）。Git 的 `@ref` 是固定的 tag 或 commit；固定版本的 Package 会被 `pi update --extensions` 和 `pi update --all` 跳过。如需把已有 Package 移到新 ref，请使用 `pi install git:host/user/repo@new-ref`。默认情况下，Git Package 使用 `npm install --omit=dev` 安装依赖，所以运行时依赖必须写在 `dependencies` 中；如果配置了 `npmCommand`，Git Package 会使用普通的 `install`，以兼容包装器。如果你使用 Node 版本管理器，并希望 Package 安装复用稳定的 npm 环境，可在 `settings.json` 里设置 `npmCommand`，例如 `["mise", "exec", "node@20", "--", "npm"]`。

在 `package.json` 中添加 `pi` 字段即可创建 Package：

```json
{
  "name": "my-pi-package",
  "keywords": ["pi-package"],
  "pi": {
    "extensions": ["./extensions"],
    "skills": ["./skills"],
    "prompts": ["./prompts"],
    "themes": ["./themes"]
  }
}
```

没有 `pi` manifest 时，pi 会从约定目录（`extensions/`、`skills/`、`prompts/`、`themes/`）自动发现资源。

详见 [docs/packages.md](docs/packages.md)。

---

<a id="programmatic-usage"></a>

## 以编程方式使用

### SDK

```typescript
import { createAgentSession, ModelRuntime, SessionManager } from "@earendil-works/pi-coding-agent";

const modelRuntime = await ModelRuntime.create();
const { session } = await createAgentSession({
  sessionManager: SessionManager.inMemory(),
  modelRuntime,
});

await session.prompt("当前目录中有哪些文件？");
```

需要进行高级的多会话 runtime 替换时，请使用 `createAgentSessionRuntime()` 和 `AgentSessionRuntime`。

详见 [docs/sdk.md](docs/sdk.md) 和 [examples/sdk/](examples/sdk/)。

### RPC 模式

要集成非 Node.js 环境，可通过 stdin/stdout 使用 RPC 模式：

```bash
pi --mode rpc
```

RPC 模式严格使用 LF 分隔的 JSONL framing。客户端只能按 `\n` 切分记录。不要使用 Node `readline` 这类通用行读取器，因为它们也会按 JSON payload 内的 Unicode 分隔符切分。

协议说明见 [docs/rpc.md](docs/rpc.md)。

---

<a id="philosophy"></a>

## 设计理念

Pi 的可扩展性很强，因此不需要规定你该怎么工作。其他工具内置的功能，可以通过[扩展](#extensions)、[技能](#skills)自己实现，或从第三方 [Pi Package](#pi-packages) 安装。这样核心保持轻量，同时你又能把 pi 调整成适合自己工作方式的样子。

**没有内置 MCP。** 用带 README 的 CLI 工具（参见[技能](#skills)），或编写一个支持 MCP 的扩展。[为什么？](https://mariozechner.at/posts/2025-11-02-what-if-you-dont-need-mcp/)

**没有内置子 agent。** 实现方式很多：通过 tmux 启动多个 pi 实例、用[扩展](#extensions)自己实现，或者安装符合你需求的 Package。

**没有权限弹窗。** 可以在容器中运行，或根据自己的环境和安全要求，用[扩展](#extensions)实现确认流程。

**没有计划模式。** 可以把计划写进文件、用[扩展](#extensions)实现，或者安装一个 Package。

**没有内置待办事项。** 它们容易让模型混乱。可以使用 TODO.md 文件，或用[扩展](#extensions)自己实现。

**没有后台 bash。** 请用 tmux。过程完全可见，也能直接交互。

完整理由见[这篇博客](https://mariozechner.at/posts/2025-11-30-pi-coding-agent/)。

---

<a id="cli-reference"></a>

## CLI 参考

```bash
pi [options] [--] [@files...] [messages...]
```

### Package 命令

```bash
pi install <source> [-l]     # 安装 Package，-l 表示项目本地安装
pi remove <source> [-l]      # 移除 Package
pi uninstall <source> [-l]   # remove 的别名
pi update [source|self|pi]   # 只更新 pi，或更新一个 Package 来源
pi update --all              # 更新 pi 和 Package
pi update --extensions       # 只更新 Package
pi update --models           # 只刷新模型目录
pi update --self             # 只更新 pi
pi update --self --force     # 即使已是当前版本也重新安装 pi
pi update --extension <src>  # 更新一个 Package
pi list                      # 列出已安装的 Package
pi config                    # 启用/禁用 Package 资源
```

`pi config` 和项目 Package 命令接受 `--approve`/`--no-approve`，用于对单条命令信任或忽略项目级设置。`pi update` 永远不会询问项目信任。

### 模式

| 标记 | 说明 |
|------|-------------|
| （默认） | 交互模式 |
| `-p`, `--print` | 打印响应后退出 |
| `--mode json` | 以 JSON Lines 输出所有事件（见 [docs/json.md](docs/json.md)） |
| `--mode rpc` | 用于进程集成的 RPC 模式（见 [docs/rpc.md](docs/rpc.md)） |
| `--export <in> [out]` | 将会话导出为 HTML |

在打印模式下，pi 也会读取管道传入的 stdin，并将其合并进初始提示词：

```bash
cat README.md | pi -p "总结这段文本"
```

### 模型选项

| 选项 | 说明 |
|--------|-------------|
| `--provider <name>` | Provider（anthropic、openai、google 等） |
| `--model <pattern>` | 模型匹配模式或 ID（支持 `provider/id` 和可选的 `:<thinking>`） |
| `--api-key <key>` | API key（覆盖环境变量） |
| `--thinking <level>` | `off`、`minimal`、`low`、`medium`、`high`、`xhigh`、`max` |
| `--models <patterns>` | 供 Ctrl+P 循环切换的逗号分隔匹配模式 |
| `--list-models [search]` | 列出可用模型 |

### 会话选项

| 选项 | 说明 |
|--------|-------------|
| `-c`, `--continue` | 继续最近的会话 |
| `-r`, `--resume` | 浏览并选择会话 |
| `--session <path\|id>` | 使用指定会话文件或部分 UUID |
| `--fork <path\|id>` | 从指定会话文件或部分 UUID fork 出新会话 |
| `--session-dir <dir>` | 自定义会话存储目录 |
| `--no-session` | 临时模式（不保存） |
| `--name <name>`, `-n <name>` | 启动时设置会话显示名称 |

### 工具选项

| 选项 | 说明 |
|--------|-------------|
| `--tools <list>`, `-t <list>` | 将内置、扩展和自定义工具中的指定工具名加入允许列表 |
| `--exclude-tools <list>`, `-xt <list>` | 在内置、扩展和自定义工具中禁用指定工具名 |
| `--no-builtin-tools`, `-nbt` | 默认禁用内置工具，但保持扩展/自定义工具启用 |
| `--no-tools`, `-nt` | 默认禁用所有工具 |

可用的内置工具：`read`、`bash`、`powershell`（Windows）、`edit`、`write`、`grep`、`find`、`ls`

### 资源选项

| 选项 | 说明 |
|--------|-------------|
| `-e`, `--extension <source>` | 从路径、npm 或 git 加载扩展（可重复指定） |
| `--no-extensions` | 禁用扩展发现 |
| `--skill <path>` | 加载技能（可重复指定） |
| `--no-skills` | 禁用技能发现 |
| `--prompt-template <path>` | 加载提示词模板（可重复指定） |
| `--no-prompt-templates` | 禁用提示词模板发现 |
| `--theme <path>` | 加载主题（可重复指定） |
| `--no-themes` | 禁用主题发现 |
| `--no-context-files`, `-nc` | 禁用 AGENTS.md 和 CLAUDE.md 上下文文件发现 |

将 `--no-*` 与显式标记组合使用，可忽略 settings.json，只加载需要的资源（例如 `--no-extensions -e ./my-ext.ts`）。

### 其他选项

| 选项 | 说明 |
|--------|-------------|
| `--system-prompt <text>` | 替换默认提示词（仍会追加上下文文件和技能） |
| `--append-system-prompt <text>` | 追加到 System Prompt |
| `--tui-mode <mode>` | TUI 模式：`regular`（默认）或实验性的 `fullscreen` |
| `--use-theme <name[/name]>` | 不修改设置，只为本次运行设置初始交互主题 |
| `--verbose` | 强制输出详细启动信息 |
| `-a`, `--approve` | 本次运行信任项目级文件 |
| `-na`, `--no-approve` | 本次运行忽略项目级文件 |
| `--` | 停止解析选项；剩余参数作为提示词或 `@file` 输入 |
| `-h`, `--help` | 显示帮助 |
| `-v`, `--version` | 显示版本 |

### 文件参数

在文件前加 `@`，即可将文件内容放入消息：

```bash
pi @prompt.md "回答这个问题"
pi -p @screenshot.png "这张图片里是什么？"
pi @code.ts @test.ts "检查这些文件"
```

### 示例

```bash
# 交互模式并提供初始提示词
pi "列出 src/ 下所有 .ts 文件"

# 非交互模式
pi -p "总结这个代码库"

# 以连字符开头的提示词
pi -p -- "- 总结这些要点"

# 非交互模式，使用管道 stdin
cat README.md | pi -p "总结这段文本"

# 命名的一次性会话
pi --name "release audit" -p "审查这个仓库"

# 使用不同模型
pi --provider openai --model gpt-4o "帮我重构"

# 带 Provider 前缀的模型（无需 --provider）
pi --model openai/gpt-4o "帮我重构"

# 带思考等级简写的模型
pi --model sonnet:high "解决这个复杂问题"

# 限制循环切换的模型
pi --models "claude-*,gpt-4o"

# 只读模式
pi --tools read,grep,find,ls -p "审查代码"

# 禁用一个扩展或内置工具，其余工具仍可用
pi --exclude-tools ask_question

# 高思考等级
pi --thinking high "解决这个复杂问题"
```

### 环境变量

| 变量 | 说明 |
|----------|-------------|
| `AI_AGENT` | CLI 和 RPC 入口会设为 `pi`，让通用工具能将子进程归属到 Pi |
| `PI_CODING_AGENT` | CLI 和 RPC 入口会设为 `true`，让子进程识别自己正运行在 Pi 中 |
| `PI_CODING_AGENT_DIR` | 覆盖配置目录（默认：`~/.pi/agent`） |
| `PI_CODING_AGENT_SESSION_DIR` | 覆盖会话存储目录（会被 `--session-dir` 覆盖） |
| `PI_PACKAGE_DIR` | 覆盖 Package 目录（适用于 Nix/Guix 中不便 token 化的 store path） |
| `PI_SERVER_DIR` | 覆盖实验性 server profile 和 socket 目录（默认：`~/.pi/server`） |
| `PI_SERVER_ID` | 省略 `--server-id` 时选择逻辑上的实验性 server ID |
| `PI_OFFLINE` | 禁用启动网络操作，包括更新检查、Package 更新检查和安装/更新遥测 |
| `PI_SKIP_VERSION_CHECK` | 跳过启动时的 Pi 版本更新检查，阻止对 `pi.dev` 最新版本接口的请求 |
| `PI_TELEMETRY` | 覆盖安装/更新遥测和 Provider 归因请求头。用 `1`/`true`/`yes` 启用，`0`/`false`/`no` 禁用。这不会禁用更新检查 |
| `PI_CACHE_RETENTION` | 设为 `long` 可延长提示词缓存（Anthropic：1 小时；OpenAI：24 小时） |
| `VISUAL`, `EDITOR` | 未设置 `externalEditor` 时，作为 Ctrl+G 的外部编辑器后备；Windows 默认 Notepad，其他系统默认 `nano` |

由 LLM 调用的 `bash` 和 `powershell` 工具执行命令时，也会收到当前会话元数据：

| 变量 | 说明 |
|----------|-------------|
| `PI_SESSION_ID` | 当前会话 ID |
| `PI_SESSION_FILE` | 会话 JSONL 的绝对路径；临时会话中未设置 |
| `PI_PROVIDER` | 当前选中模型的 Provider |
| `PI_MODEL` | 当前选中模型的 ID |
| `PI_REASONING_LEVEL` | 当前实际生效的推理等级 |

这些值会在每条命令启动时解析。有关语义、示例及自定义工具如何选择不接收它们，见[环境变量](docs/environment-variables.md#shell-tool-session-environment)。

---

## 贡献与开发

贡献指南见 [CONTRIBUTING.md](../../CONTRIBUTING.md)，安装、fork 和调试说明见 [docs/development.md](docs/development.md)。

## 许可证

MIT

## 相关项目

- [@earendil-works/pi-ai](https://www.npmjs.com/package/@earendil-works/pi-ai)：核心 LLM 工具包
- [@earendil-works/pi-agent-core](https://www.npmjs.com/package/@earendil-works/pi-agent-core)：Agent 框架
- [@earendil-works/pi-tui](https://www.npmjs.com/package/@earendil-works/pi-tui)：终端 UI 组件

<p align="center">
  <a href="https://pi.dev">pi.dev</a> 域名由以下机构慷慨捐赠
  <br /><br />
  <a href="https://exe.dev"><img src="docs/images/exy.png" alt="Exy 吉祥物" width="48" /><br />exe.dev</a>
</p>
