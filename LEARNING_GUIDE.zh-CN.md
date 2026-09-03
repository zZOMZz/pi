# Pi 源码学习指南

本文是一份按功能机制组织的源码学习路线，适用于希望理解 Pi 如何从“调用一次模型”逐步发展成完整 coding agent 的读者。

本文不是 API 手册的中文翻译。它重点回答四类问题：

1. 一个用户请求经过哪些对象和函数？
2. 每一层拥有哪部分状态，承担什么职责？
3. Session、Skills、Extensions、权限控制等能力如何接入主链路？
4. 如何通过测试和小实验证明自己真的理解了代码？

> 本文基于撰写时的 `feature/learn-agent-loop` 分支。该分支同时包含当前 coding-agent 实现和实验中的新 Session / AgentHarness 架构。学习时应记录当前提交，避免后续代码变化造成定位偏差：`git rev-parse --short HEAD`。

## 1. 学习原则

### 1.1 先建立纵向主链，再横向学习功能

Pi 的核心调用链是：

```text
CLI / Interactive / Print / RPC
                │
                ▼
           AgentSession
                │
                ▼
              Agent
                │
                ▼
            agent-loop
                │
                ▼
              pi-ai
                │
                ▼
             Provider
```

横向能力围绕这条主链工作：

```text
Resources / Skills ──► system prompt 与用户输入
Extensions         ──► 输入、模型、工具、事件与 UI 钩子
Tools              ◄──► agent-loop
SessionManager     ◄──  AgentSession 产生的事件
Compaction         ──►  下一次模型调用的上下文
TUI / RPC          ◄──  AgentSession 事件
```

如果直接从 MCP、权限或 TUI 开始，会不断遇到尚未理解的 Agent、Tool、Event 和 Session 概念。正确顺序是先掌握主链，再学习横向能力。

### 1.2 每章使用同一种学习循环

每章固定完成以下步骤：

1. 阅读文档，明确这一层解决的问题。
2. 阅读类型定义，找出输入、输出和状态所有者。
3. 跟踪一条具体调用链，不要从头到尾浏览所有实现。
4. 阅读测试，把测试当作可执行规范。
5. 完成一个小实验，主动改变一个条件并预测结果。
6. 用本章验收问题进行复述。

建议为每章保存一页笔记，只记录：

```text
职责：这一层负责什么？
边界：这一层明确不负责什么？
状态：状态由谁持有？
输入：从哪里进入？
输出：通过返回值还是事件离开？
失败：错误、取消、重试怎样传播？
扩展点：外部代码在哪里介入？
```

### 1.3 阅读测试时避免真实模型调用

普通单元测试优先使用 faux provider，不需要 API key，也不会产生模型费用。不要直接运行完整 Vitest suite；仓库中包含可能启用真实端点的 e2e 测试。

运行某个 Vitest 文件：

```bash
cd packages/agent
node "$(git rev-parse --show-toplevel)/node_modules/vitest/dist/cli.js" --run test/agent-loop.test.ts
```

对 `packages/coding-agent/test/suite/`，重点阅读其 faux provider harness：

- [`packages/coding-agent/test/suite/harness.ts`](packages/coding-agent/test/suite/harness.ts)
- [`packages/coding-agent/test/suite/README.md`](packages/coding-agent/test/suite/README.md)

## 2. 仓库地图

| Package | 主要职责 | 建议学习阶段 |
|---|---|---|
| `pi-ai` | 模型、消息、流式事件、Provider、认证、Tool Schema | 第一阶段只学公共边界，后期再学 Provider 细节 |
| `pi-agent-core` | Agent 状态、agent-loop、工具执行、消息队列；同时包含实验 Harness | 最先深入 |
| `pi-coding-agent` | CLI 产品层、AgentSession、Session、Skills、Extensions、工具、TUI 模式 | 主体学习对象 |
| `pi-tui` | 终端渲染、组件、输入、Editor、Markdown、图片 | 核心机制之后 |
| `pi-telemetry` | 与厂商无关的遥测接口和参考实现 | 后期 |
| `pi-protocol` | 实验性远程协议的路由信封、CBOR 和 framing | 实验架构阶段 |
| `pi-client` | 实验性远程 Session client | 实验架构阶段 |
| `pi-server` | 实验性 Session 路由与 attachment 管理 | 实验架构阶段 |
| `session-backends/sqlite-node` | 新 Session 抽象的 SQLite backend | 实验架构阶段 |
| `pi-evals` | 使用真实模型评价端到端 agent 行为 | 最后学习 |

主要入口：

- 仓库介绍：[`README.md`](README.md)
- coding-agent 使用说明：[`packages/coding-agent/README.md`](packages/coding-agent/README.md)
- CLI 入口：[`packages/coding-agent/src/cli.ts`](packages/coding-agent/src/cli.ts)
- CLI 主流程：[`packages/coding-agent/src/main.ts`](packages/coding-agent/src/main.ts)
- SDK 组装入口：[`packages/coding-agent/src/core/sdk.ts`](packages/coding-agent/src/core/sdk.ts)
- 产品运行时：[`packages/coding-agent/src/core/agent-session.ts`](packages/coding-agent/src/core/agent-session.ts)
- 状态 Agent：[`packages/agent/src/agent.ts`](packages/agent/src/agent.ts)
- 最小循环：[`packages/agent/src/agent-loop.ts`](packages/agent/src/agent-loop.ts)

## 3. 必须先区分的术语

### 3.1 Turn、Run 与 Session

- **Turn**：一次 LLM 调用，加上该响应触发的工具执行。
- **Run**：从一次用户输入开始，经过零到多个 Turn，直到没有工具调用和排队消息。
- **Session**：可持久化、可恢复、可分支的长期对话历史。

示例：

```text
用户：读取 package.json 后告诉我版本

Run
├─ Turn 1
│  ├─ LLM 生成 read tool call
│  └─ 执行 read，得到 toolResult
└─ Turn 2
   └─ LLM 根据 toolResult 返回最终答案
```

### 3.2 Message、AgentMessage 与 SessionEntry

- `Message` 是模型能够理解的消息。
- `AgentMessage` 可以包含应用自定义消息；调用模型前必须转换成 `Message`。
- `SessionEntry` 是持久化记录，除消息外还可以表示模型切换、思考等级、压缩摘要、标签等。

不能把三者当作同一个结构。它们分别服务于模型协议、运行时状态和持久化历史。

### 3.3 Tool、Skill、Extension 与 MCP

| 概念 | 本质 | 是否直接执行代码 | 何时进入上下文 |
|---|---|---:|---|
| Tool | 模型可调用的结构化函数 | 是 | Tool schema 通常随模型请求发送 |
| Skill | 按需加载的说明和配套资源 | 可包含脚本，但由 agent 按说明调用 | 元数据进入 system prompt；正文按需读取或显式展开 |
| Extension | 运行在 Pi 进程中的 TypeScript 插件 | 是 | 通过注册和事件钩子介入运行时 |
| MCP | 外部工具/资源协议 | 取决于 MCP server | Pi 核心不内置，需要 Extension 或外部适配器 |

### 3.4 当前实现与实验架构

当前 coding-agent 的主要可运行链路是：

```text
AgentSession → Agent → agent-loop
     │
     └─ SessionManager（JSONL tree）
```

`packages/agent/src/harness/`、`packages/protocol`、`packages/client`、`packages/server` 和 `packages/coding-agent/src/experimental/` 描述另一套正在演进的 durable Session / AgentHarness / remote presentation 架构。

学习时先理解当前产品链路，再比较新架构。不要把两套 Session API 混在一张调用图中。

---

# 第一阶段：最小 Agent 内核

## 第 1 章：`pi-ai`——模型调用边界

### 本章目标

理解 Agent 最终向模型层提交什么，以及模型层如何把不同 Provider 的差异归一化。

本章不要求掌握所有 Provider。第一遍只学公共类型、流式事件和 faux provider。

### 问题、示例、解决方案

问题：OpenAI、Anthropic、Google 的请求格式、思考内容、Tool Call 和结束原因并不一致。

具体例子：Agent 只想知道模型输出了文本、思考内容还是工具调用，不应该在 agent-loop 中判断每个厂商的原始事件格式。

解决方案：`pi-ai` 对上提供统一的 `Model`、`Context`、`Message`、`Tool`、`AssistantMessageEvent` 和 `StopReason`；Provider adapter 在边界处完成转换。

### 阅读顺序

1. [`packages/ai/src/types.ts`](packages/ai/src/types.ts)
2. [`packages/ai/src/models.ts`](packages/ai/src/models.ts)
3. [`packages/ai/src/utils/event-stream.ts`](packages/ai/src/utils/event-stream.ts)
4. [`packages/ai/src/providers/faux.ts`](packages/ai/src/providers/faux.ts)
5. 选择一个熟悉的 Provider：
   - OpenAI：[`packages/ai/src/api/openai-responses.ts`](packages/ai/src/api/openai-responses.ts)
   - Anthropic：[`packages/ai/src/api/anthropic-messages.ts`](packages/ai/src/api/anthropic-messages.ts)

第一遍重点定位：

- `Model` 如何描述 provider、context window、max tokens 和 reasoning 能力。
- `Context` 如何携带 system prompt、messages 和 tools。
- `AssistantMessage` 如何同时容纳 text、thinking 和 toolCall。
- `EventStream` 如何同时提供增量事件和最终结果。
- `stopReason` 与错误、abort、长度截断之间的关系。

### 小实验

使用 faux provider 构造一次纯文本响应和一次 Tool Call 响应。写下它们产生的事件顺序，不接入 `Agent`。

### 代表测试

- [`packages/ai/test/faux-provider.test.ts`](packages/ai/test/faux-provider.test.ts)
- [`packages/ai/test/stream.test.ts`](packages/ai/test/stream.test.ts)
- [`packages/ai/test/validation.test.ts`](packages/ai/test/validation.test.ts)
- [`packages/ai/test/tool-call-without-result.test.ts`](packages/ai/test/tool-call-without-result.test.ts)

### 验收问题

- 为什么 agent-loop 不应直接处理 Provider 的原始 SSE 事件？
- 流式 partial message 和最终 message 的职责有什么区别？
- Tool 参数验证应发生在 Provider 层还是 Agent 执行层？

## 第 2 章：`agent-loop`——最小执行循环

### 本章目标

理解 Pi 最核心的控制流：模型响应、工具执行、Tool Result 回填以及下一次模型调用。

### 核心结构

入口位于 [`packages/agent/src/agent-loop.ts`](packages/agent/src/agent-loop.ts)：

- `agentLoop()`：从新消息启动。
- `agentLoopContinue()`：从已有上下文继续。
- `runLoop()`：控制 Turn、工具链和消息队列。
- `streamAssistantResponse()`：调用模型并转发流式事件。
- `executeToolCalls()`：选择并行或串行工具执行。

循环可以简化为：

```text
加入用户消息
    │
    ▼
调用 LLM ───────────────┐
    │                    │
    ▼                    │
是否包含 toolCall？      │
    │                    │
  否│      是            │
    │       ▼            │
    │   校验并执行工具    │
    │       │            │
    │       ▼            │
    │   加入 toolResult ─┘
    ▼
处理 follow-up 后结束
```

### 必须跟踪的路径

1. 无工具：user → assistant → `agent_end`。
2. 单工具：user → assistant/toolCall → toolResult → assistant。
3. 多工具并行：preflight 顺序执行，实际工具并发，Tool Result 保持原始调用顺序。
4. 工具参数非法：不执行工具，产生错误 Tool Result。
5. LLM 因长度停止：不执行可能被截断的工具参数。
6. abort：模型流和工具如何观察同一个 `AbortSignal`。
7. steer 与 follow-up：分别在什么时机注入下一条消息。

### 关键设计

`agent-loop` 只维护一次运行所需的局部上下文，不负责：

- 把消息写入磁盘。
- 构建完整 coding-agent system prompt。
- 弹出 UI。
- 加载 Skills 或 Extensions。
- 自动压缩长期历史。

这些职责被留给更高层，保证循环可以独立测试和复用。

### 小实验

给 faux provider 配置以下脚本响应：

1. 第一次返回 `calculate` Tool Call。
2. 工具返回计算结果。
3. 第二次返回最终文本。

订阅所有事件，验证事件顺序和最终 `newMessages`。

### 代表测试

- [`packages/agent/test/agent-loop.test.ts`](packages/agent/test/agent-loop.test.ts)
- [`packages/agent/test/utils/calculate.ts`](packages/agent/test/utils/calculate.ts)

### 验收问题

- 为什么代码需要内层工具循环和外层 follow-up 循环？
- 为什么并发完成的工具仍按原始 Tool Call 顺序生成 Tool Result？
- `terminate: true` 为什么必须在整批结果都要求终止时才提前停止？

## 第 3 章：`Agent`——状态、队列和生命周期

### 本章目标

理解 `Agent` 为什么不是另一个 agent-loop，而是 agent-loop 的有状态包装。

### 问题、示例、解决方案

问题：裸 agent-loop 适合一次运行，但交互式应用需要保存消息、切换模型、排队新消息、订阅事件和取消当前运行。

具体例子：用户在工具执行期间输入“先别改文件，先解释原因”。应用必须保存该消息，并在当前 Turn 完整结束后把它注入下一次模型请求。

解决方案：[`Agent`](packages/agent/src/agent.ts) 持有 `AgentState`、运行状态、`AbortController`、steering queue、follow-up queue 和事件订阅者。

### 阅读顺序

1. [`packages/agent/src/types.ts`](packages/agent/src/types.ts) 中的：
   - `AgentState`
   - `AgentContext`
   - `AgentEvent`
   - `AgentTool`
   - `AgentLoopConfig`
2. [`packages/agent/src/agent.ts`](packages/agent/src/agent.ts)
3. [`packages/agent/README.md`](packages/agent/README.md) 的 Event Flow 与 Steering and Follow-up。

### 状态所有权

`Agent` 主要持有：

- system prompt
- model 与 thinking level
- tools
- 当前内存 messages
- 是否正在运行
- 当前 partial assistant message
- pending tool calls
- steering/follow-up queues

`Agent` 不持有磁盘 Session 树。磁盘持久化由 coding-agent 的 `AgentSession` 和 `SessionManager` 完成。

### 事件屏障

`Agent.subscribe()` 支持异步 listener，并按注册顺序等待。高层可以在 `message_end` 时完成持久化，再允许后续阶段继续。这与直接消费裸 `agentLoop()` 的观察型流不同。

### 小实验

在一个耗时工具执行期间分别调用 `steer()` 和 `followUp()`，记录它们进入模型上下文的 Turn。然后把 queue mode 从 `one-at-a-time` 改成 `all`，预测上下文变化。

### 代表测试

- [`packages/agent/test/agent.test.ts`](packages/agent/test/agent.test.ts)

### 验收问题

- `AgentState.messages` 与持久化 Session 有什么区别？
- `steer` 为什么不是立即中止正在执行的工具？
- `agent_end` 已发出时，为什么 `prompt()` 仍可能尚未 resolve？

---

# 第二阶段：从 Agent 到 Coding Agent

## 第 4 章：Tools——让模型作用于真实环境

### 本章目标

理解 Tool 从 schema 暴露、参数生成、参数验证到执行结果回填的完整生命周期。

### 两层 Tool 表示

`pi-agent-core` 使用 `AgentTool`，关心模型和执行：

```text
name + description + parameters + execute
```

`pi-coding-agent` 使用更丰富的 `ToolDefinition`，还要支持：

- UI label 和结果渲染。
- Extensions 动态注册或覆盖。
- 运行时 cwd。
- system prompt contribution。
- Tool Call 和 Tool Result 钩子。

两者通过 wrapper 连接：

- [`packages/coding-agent/src/core/tools/tool-definition-wrapper.ts`](packages/coding-agent/src/core/tools/tool-definition-wrapper.ts)
- [`packages/coding-agent/src/core/extensions/index.ts`](packages/coding-agent/src/core/extensions/index.ts)

### 阅读顺序

1. `AgentTool`：[`packages/agent/src/types.ts`](packages/agent/src/types.ts)
2. 参数验证和调度：[`packages/agent/src/agent-loop.ts`](packages/agent/src/agent-loop.ts)
3. ToolDefinition：[`packages/coding-agent/src/core/extensions/types.ts`](packages/coding-agent/src/core/extensions/types.ts)
4. 内置工具目录：[`packages/coding-agent/src/core/tools`](packages/coding-agent/src/core/tools)
5. 文件修改队列：[`packages/agent/src/harness/tools/file-mutation-queue.ts`](packages/agent/src/harness/tools/file-mutation-queue.ts)

重点比较：

- `read`：读取结果、图片和文本截断。
- `bash`：子进程、增量输出、abort、超时。
- `edit`：精确替换和 diff。
- `write`：创建或覆盖文件。

### 工具执行路径

```text
Provider 生成 toolCall
  → 查找同名 AgentTool
  → prepareArguments
  → TypeBox 参数验证
  → beforeToolCall / extension tool_call
  → execute(signal, onUpdate)
  → afterToolCall / extension tool_result
  → tool_execution_end
  → ToolResultMessage
```

### 小实验

实现一个只读的 `repo_stats` Tool，返回文件数量和 TypeScript 文件数量：

- 使用 TypeBox 定义参数。
- 支持 `AbortSignal`。
- 正常失败时抛出异常，不把错误伪装成成功文本。
- 为 details 保留结构化数据，为 content 返回给模型的简洁文本。

### 代表测试

- [`packages/agent/test/harness/tools.test.ts`](packages/agent/test/harness/tools.test.ts)
- [`packages/coding-agent/test/tools.test.ts`](packages/coding-agent/test/tools.test.ts)
- [`packages/coding-agent/test/file-mutation-queue.test.ts`](packages/coding-agent/test/file-mutation-queue.test.ts)

### 验收问题

- Tool 的 `content` 和 `details` 分别服务于谁？
- 为什么失败应抛出异常，而不是返回“执行失败”文本？
- Tool schema、参数验证和权限判断为什么是三个不同阶段？

## 第 5 章：`AgentSession`——产品级编排中心

### 本章目标

理解 coding-agent 如何把 Agent、模型、工具、Session、Skills、Extensions、Compaction 和重试组合成一个运行时。

### 先看对象是如何创建的

构造链路：

```text
main.ts
  → createAgentSessionServices()
      ├─ SettingsManager
      ├─ ModelRuntime
      └─ ResourceLoader
  → createAgentSessionFromServices()
  → createAgentSession()
      ├─ new Agent(...)
      └─ new AgentSession(...)
  → createAgentSessionRuntime()
```

对应文件：

- [`packages/coding-agent/src/main.ts`](packages/coding-agent/src/main.ts)
- [`packages/coding-agent/src/core/agent-session-services.ts`](packages/coding-agent/src/core/agent-session-services.ts)
- [`packages/coding-agent/src/core/sdk.ts`](packages/coding-agent/src/core/sdk.ts)
- [`packages/coding-agent/src/core/agent-session-runtime.ts`](packages/coding-agent/src/core/agent-session-runtime.ts)

### 再跟踪 `prompt()`

不要顺序阅读三千多行的 `agent-session.ts`。先只跟踪 [`AgentSession.prompt()`](packages/coding-agent/src/core/agent-session.ts)：

```text
输入文本
  → Extension command 检查
  → input event 拦截或转换
  → Skill command / prompt template 展开
  → 检查是否已有活动 Run
  → 验证 model 与 auth
  → 必要时预压缩
  → 构造 user AgentMessage
  → before_agent_start
  → Agent.prompt()
  → 处理 retry / overflow / compaction / queued messages
```

### 事件与持久化

`AgentSession` 订阅 `Agent` 事件。在 `message_end` 时：

- 先向 Extensions 发出对应事件。
- 再通知 UI/RPC listener。
- 最后把消息追加到 `SessionManager`。

阅读时检查具体代码顺序，不只看注释。顺序会影响扩展能否替换消息、UI 看到什么，以及最终保存什么。

### 小实验

使用 coding-agent test suite harness 完成一轮纯文本响应，再完成一轮 Tool Call。分别记录：

- Agent 内存消息。
- AgentSession 事件。
- Session JSONL entries。

证明它们不是同一个数据层。

### 代表测试

- [`packages/coding-agent/test/suite/agent-session-prompt.test.ts`](packages/coding-agent/test/suite/agent-session-prompt.test.ts)
- [`packages/coding-agent/test/agent-session-concurrent.test.ts`](packages/coding-agent/test/agent-session-concurrent.test.ts)
- [`packages/coding-agent/test/agent-session-retry.test.ts`](packages/coding-agent/test/agent-session-retry.test.ts)

### 验收问题

- `AgentSession` 比 `Agent` 多负责了哪些产品行为？
- 为什么 auth 验证不应放在 agent-loop？
- Extension 替换 `message_end` 消息时，怎样保证内存状态和持久化结果一致？

## 第 6 章：Session——持久化、树与恢复

### 本章目标

理解当前 coding-agent JSONL Session 的树结构，以及“历史记录”和“发送给模型的上下文”为什么不同。

### 数据模型

当前 Session 文件由一个 header 和一系列 append-only entries 组成。常见 Entry 包括：

- message
- model change
- thinking level change
- compaction
- branch summary
- custom entry / custom message
- label
- session info

每个树节点通过 `id` 与 `parentId` 连接。`leafId` 指向当前活动分支的末端。

```text
root user
└─ assistant
   ├─ user A
   │  └─ assistant A  ← leaf
   └─ user B
      └─ assistant B
```

从 A 切换到 B 不需要删除 A。它只是改变活动 leaf，并可能生成离开分支的摘要。

### 阅读顺序

1. [`packages/coding-agent/docs/sessions.md`](packages/coding-agent/docs/sessions.md)
2. [`packages/coding-agent/docs/session-format.md`](packages/coding-agent/docs/session-format.md)
3. [`packages/coding-agent/src/core/session-manager.ts`](packages/coding-agent/src/core/session-manager.ts)
4. 优先定位：
   - create / open / in-memory
   - append entry
   - set leaf
   - get branch
   - build session context

### 关键区别

```text
完整 Session tree
      │ 选择 active leaf 的祖先链
      ▼
当前 branch entries
      │ 应用 compaction / branch summary / custom entry 转换
      ▼
AgentMessage[]
      │ transformContext + convertToLlm
      ▼
Message[] 发给模型
```

### 小实验

使用 `SessionManager.inMemory()`：

1. 追加 user A、assistant A。
2. 回到 user A 之前的位置。
3. 追加 user B、assistant B。
4. 分别切换 leaf，调用 context 构建逻辑。
5. 验证磁盘/内存中存在两个分支，但模型一次只看到活动分支。

### 代表测试

- [`packages/coding-agent/test/session-manager/tree-traversal.test.ts`](packages/coding-agent/test/session-manager/tree-traversal.test.ts)
- [`packages/coding-agent/test/session-manager/build-context.test.ts`](packages/coding-agent/test/session-manager/build-context.test.ts)
- [`packages/coding-agent/test/agent-session-branching.test.ts`](packages/coding-agent/test/agent-session-branching.test.ts)
- [`packages/coding-agent/test/agent-session-tree-navigation.test.ts`](packages/coding-agent/test/agent-session-tree-navigation.test.ts)

### 验收问题

- 为什么 Session 用树而不是普通消息数组？
- `leafId` 改变后，哪些数据改变，哪些历史数据不变？
- 为什么恢复 Session 时还要重放 model change 和 thinking level change？

## 第 7 章：Context 与 Compaction——控制长期对话

### 本章目标

理解持久化历史如何变成有限 context window 中的模型输入，以及压缩为何不会简单删除旧记录。

### 问题、示例、解决方案

问题：Session 可以无限增长，模型 context window 不可以。

具体例子：历史共有 120k tokens，而模型只允许 128k；还必须为下一次回答和工具结果预留空间。

解决方案：选择一个 cut point，把较老内容总结为 compaction summary，同时保留最近消息。原始 entries 仍留在 Session 中，模型上下文使用摘要投影。

### 阅读顺序

1. [`packages/coding-agent/docs/compaction.md`](packages/coding-agent/docs/compaction.md)
2. [`packages/coding-agent/src/core/compaction`](packages/coding-agent/src/core/compaction)
3. `AgentSession` 中 `_checkCompaction()`、`compact()` 和 overflow recovery。
4. `SessionManager` 的 context building。

### 两种摘要

| 类型 | 触发场景 | 作用 |
|---|---|---|
| Compaction summary | 当前分支持续变长 | 用摘要替换较老上下文 |
| Branch summary | 从一个分支切换到另一个分支 | 把离开分支的重要工作带到新位置 |

### 必须关注的边界

- cut point 不能破坏 assistant toolCall 与对应 toolResult 的配对。
- 最近消息需要保留，而不是全部总结。
- 摘要生成本身也可能失败、取消或溢出。
- 压缩改变模型上下文投影，不应销毁 Session 的原始历史。
- Extensions 可以自定义或拦截 summarization。

### 小实验

构造一个包含多轮 Tool Call 的 Session，降低 compaction 阈值，观察：

- 哪个 entry 被选为 cut point。
- summary entry 的 parentId。
- 压缩前后 `buildSessionContext()` 的差异。
- 原始树是否仍可导航。

### 代表测试

- [`packages/coding-agent/test/compaction.test.ts`](packages/coding-agent/test/compaction.test.ts)
- [`packages/coding-agent/test/agent-session-compaction.test.ts`](packages/coding-agent/test/agent-session-compaction.test.ts)
- [`packages/coding-agent/test/branch-summarization.test.ts`](packages/coding-agent/test/branch-summarization.test.ts)
- [`packages/coding-agent/test/compaction-serialization.test.ts`](packages/coding-agent/test/compaction-serialization.test.ts)

### 验收问题

- Compaction 为什么是“上下文投影”问题，而不是“删除历史”问题？
- 为什么不能在 assistant toolCall 与 toolResult 之间切断？
- Branch summary 和 compaction summary 分别解决什么问题？

---

# 第三阶段：资源、扩展与安全

## 第 8 章：Resources 与 Skills——按需注入能力说明

### 本章目标

理解 Skill 的发现、验证、去重、提示词曝光和显式调用过程。

### Skill 的渐进式加载

Skill 的关键思想不是把所有 `SKILL.md` 正文都塞进 system prompt，而是渐进式披露：

```text
启动时发现 Skill
  → 读取 frontmatter
  → 把 name / description / location 放入 system prompt
  → 模型判断任务是否匹配
  → 模型使用 read 打开 SKILL.md
  → 再按说明读取 references、运行 scripts
```

显式 `/skill:name args` 是另一条路径：Pi 直接读取正文，构造成 skill block，并作为用户消息的一部分提交。

### 阅读顺序

1. [`packages/coding-agent/docs/skills.md`](packages/coding-agent/docs/skills.md)
2. [`packages/coding-agent/src/core/skills.ts`](packages/coding-agent/src/core/skills.ts)
3. [`packages/coding-agent/src/core/resource-loader.ts`](packages/coding-agent/src/core/resource-loader.ts)
4. [`packages/coding-agent/src/core/package-manager.ts`](packages/coding-agent/src/core/package-manager.ts)
5. [`packages/coding-agent/src/core/system-prompt.ts`](packages/coding-agent/src/core/system-prompt.ts)
6. `AgentSession._expandSkillCommand()`。

### 资源来源

Skill 可能来自：

- 用户级 `~/.pi/agent/skills/`。
- 用户级 `~/.agents/skills/`。
- 项目 `.pi/skills/`。
- 项目或祖先目录 `.agents/skills/`。
- Pi package。
- settings 中显式路径。
- CLI `--skill`。
- Extension 动态发现结果。

学习时为每个来源标出：是否需要 project trust、优先级、去重键以及 source metadata。

### Skill 与 system prompt 的依赖

Skills 默认依赖 `read` 工具进行按需加载。阅读 `buildSystemPrompt()` 时检查：如果没有可用 read tool，Skill 元数据是否仍会注入，以及为什么。

### 小实验

创建一个临时 Skill：

```text
demo-skill/
├─ SKILL.md
├─ references/
│  └─ protocol.md
└─ scripts/
   └─ inspect.mjs
```

分别通过自动匹配和 `/skill:demo-skill` 使用它，比较最终 user message 与 system prompt。

### 代表测试

- [`packages/coding-agent/test/skills.test.ts`](packages/coding-agent/test/skills.test.ts)
- [`packages/coding-agent/test/resource-loader.test.ts`](packages/coding-agent/test/resource-loader.test.ts)
- [`packages/coding-agent/test/sdk-skills.test.ts`](packages/coding-agent/test/sdk-skills.test.ts)
- [`packages/coding-agent/test/system-prompt.test.ts`](packages/coding-agent/test/system-prompt.test.ts)

### 验收问题

- 为什么 system prompt 只放 Skill 元数据，而不是所有正文？
- 自动 Skill 调用和 `/skill:name` 的消息路径有什么不同？
- Skill 来源冲突时，由谁决定最终启用的资源？

## 第 9 章：Extensions——可执行的运行时扩展

### 本章目标

理解 Extensions 如何在不修改核心代码的情况下增加工具、命令、事件处理、Provider 和 UI。

### 问题、示例、解决方案

问题：不同用户需要不同的权限确认、远程执行、状态栏、工作流命令和工具集合，把这些全部内置会使核心复杂并强制用户接受同一种工作流。

具体例子：有人希望每次执行危险 bash 前确认，有人则始终在容器中运行，不需要确认。

解决方案：Pi 保持核心较小，通过 Extension API 注册工具和命令，并在运行生命周期的明确位置触发事件。

### 阅读顺序

1. [`packages/coding-agent/docs/extensions.md`](packages/coding-agent/docs/extensions.md) 的以下部分：
   - Quick Start
   - Lifecycle Overview
   - Agent Events
   - Tool Events
   - ExtensionAPI Methods
   - Custom Tools
   - Error Handling
2. [`packages/coding-agent/src/core/extensions/types.ts`](packages/coding-agent/src/core/extensions/types.ts)
3. [`packages/coding-agent/src/core/extensions/loader.ts`](packages/coding-agent/src/core/extensions/loader.ts)
4. [`packages/coding-agent/src/core/extensions/runner.ts`](packages/coding-agent/src/core/extensions/runner.ts)
5. [`packages/coding-agent/examples/extensions/hello.ts`](packages/coding-agent/examples/extensions/hello.ts)
6. [`packages/coding-agent/examples/extensions/dynamic-tools.ts`](packages/coding-agent/examples/extensions/dynamic-tools.ts)

### 生命周期观察点

按一次 prompt 的时间顺序整理以下事件：

```text
input
before_agent_start
agent_start
turn_start
message_start / update / end
tool_execution_start
tool_call
tool_result
tool_execution_end
turn_end
agent_end
agent_settled
```

需要分清：

- 通知型事件：观察状态，不改变执行结果。
- 变换型事件：可以替换输入、消息、headers 或工具结果。
- 控制型事件：可以阻止 Tool Call、切换 Session 或请求退出。
- UI API：只在有 UI 的 mode 中可用。

### 小实验

实现一个审计 Extension：

- 记录每个 Tool Call 的名字和耗时。
- 禁止修改指定目录。
- 注册 `/audit` 命令显示统计。
- 在 print mode 中不能依赖交互式确认。
- 在 shutdown 时释放资源。

### 代表测试

- [`packages/coding-agent/test/extensions-runner.test.ts`](packages/coding-agent/test/extensions-runner.test.ts)
- [`packages/coding-agent/test/extensions-input-event.test.ts`](packages/coding-agent/test/extensions-input-event.test.ts)
- [`packages/coding-agent/test/agent-session-dynamic-tools.test.ts`](packages/coding-agent/test/agent-session-dynamic-tools.test.ts)

### 验收问题

- Skill 和 Extension 的根本区别是什么？
- `agent_end` 与 `agent_settled` 为什么都需要？
- Extension 覆盖内置 Tool 时，schema、执行和 UI 渲染分别来自哪里？

## 第 10 章：Permission 与安全边界

### 本章目标

理解 Pi 不提供统一内置 permission popup 的原因，并准确区分资源信任、工具策略和真正的隔离边界。

### 三层模型

#### 第一层：Project Trust

Project Trust 决定是否加载项目提供的配置和可执行扩展，例如：

- `.pi/settings.json`
- `.pi/extensions/`
- `.pi/skills/`
- `.pi/SYSTEM.md`
- 项目 package resources

它不限制模型启动后的 read、write、edit 或 bash。

入口：

- [`packages/coding-agent/src/core/trust-manager.ts`](packages/coding-agent/src/core/trust-manager.ts)
- [`packages/coding-agent/src/cli/project-trust.ts`](packages/coding-agent/src/cli/project-trust.ts)
- [`packages/coding-agent/examples/extensions/project-trust.ts`](packages/coding-agent/examples/extensions/project-trust.ts)

#### 第二层：Tool Policy

Extension 可以在工具调用前检查参数并允许、拒绝或请求用户确认：

- [`packages/coding-agent/examples/extensions/permission-gate.ts`](packages/coding-agent/examples/extensions/permission-gate.ts)
- [`packages/coding-agent/examples/extensions/protected-paths.ts`](packages/coding-agent/examples/extensions/protected-paths.ts)
- [`packages/coding-agent/examples/extensions/confirm-destructive.ts`](packages/coding-agent/examples/extensions/confirm-destructive.ts)

这一层适合工作流约束和误操作防护，但 Extension 本身仍与 Pi 进程拥有相同系统权限。

#### 第三层：OS / VM / Container Isolation

真正的安全边界必须由操作系统、容器或虚拟机提供：

- 把整个 Pi 放进 Docker/OpenShell。
- 保留 Pi 在宿主机，把工具路由到 Gondolin micro-VM。
- 只挂载所需目录。
- 只传入必要凭证。
- 不需要网络时关闭网络。

文档：

- [`packages/coding-agent/docs/security.md`](packages/coding-agent/docs/security.md)
- [`packages/coding-agent/docs/containerization.md`](packages/coding-agent/docs/containerization.md)

### 具体威胁分析练习

对以下场景分别指出三层机制能否防护：

1. 仓库中的恶意 `.pi/extensions/evil.ts`。
2. README 中的 prompt injection 要求上传 SSH key。
3. 模型误执行 `rm`。
4. 已获信任 Extension 主动读取环境变量。
5. Docker 中将宿主工作区以读写方式挂载。

### 代表测试

- [`packages/coding-agent/test/trust-manager.test.ts`](packages/coding-agent/test/trust-manager.test.ts)
- [`packages/coding-agent/test/trust-selector.test.ts`](packages/coding-agent/test/trust-selector.test.ts)

### 验收问题

- Project Trust 为什么不是 sandbox？
- Permission Extension 为什么不是强安全边界？
- 在非交互模式中，依赖 UI confirm 的权限策略会出现什么问题？

## 第 11 章：MCP——作为 Extension 综合项目

### 本章目标

不是寻找不存在的“Pi MCP 核心模块”，而是使用已掌握的 Tool 和 Extension 机制设计一个 MCP adapter。

Pi 明确不内置 MCP。仓库中的 MCP 相关文字主要说明它可以通过 Extension 或 package 集成，不构成 MCP client 实现。

### 最小适配架构

```text
Pi Extension
├─ 读取 MCP server 配置
├─ 建立 stdio / HTTP transport
├─ initialize
├─ tools/list
├─ MCP Tool → Pi ToolDefinition
├─ tools/call → AgentToolResult
└─ shutdown 时关闭连接与子进程
```

### 映射问题

实现前必须明确：

| MCP 概念 | Pi 对应物 | 需要处理的差异 |
|---|---|---|
| tool name | `ToolDefinition.name` | 冲突、非法字符、server namespace |
| description | Tool description | 长度和上下文成本 |
| inputSchema | TypeBox/JSON Schema | schema 方言和 unsupported keywords |
| text content | Tool result text block | 基本直接转换 |
| image content | image block | MIME、尺寸、base64 规范化 |
| resource | Tool 或 Skill/上下文 | Pi 没有一一对应的内置 MCP Resource 层 |
| prompt | command/template/skill | 需要选择产品语义 |
| server error | thrown error / error Tool Result | 是否可重试和如何暴露给模型 |

### 生命周期与安全问题

- Extension reload 时旧进程必须退出。
- 一个 server 断开不能让整个 Pi 卡死。
- Tool 名称必须稳定，避免同名 server 相互覆盖。
- MCP server 与普通 Extension 一样拥有其进程获得的系统权限。
- stdio stderr 不能污染 Pi 的 RPC stdout。
- Tool Result 图片需要经过 coding-agent 的图像规范化。
- schema 更新后要决定动态刷新还是要求 reload。

### 分阶段实验

1. 只连接一个固定 MCP server，列出 tools。
2. 把一个纯文本 Tool 映射为 Pi Tool。
3. 增加多 server namespace。
4. 支持 abort、timeout 和断线错误。
5. 支持图片结果。
6. 增加配置验证、reload 和 shutdown。

### 验收标准

- faux MCP server 能稳定完成 initialize、list 和 call。
- MCP 进程退出后 Tool Call 返回明确错误，不挂起 Agent。
- 同名 tools 不会静默覆盖。
- Extension reload 后没有遗留子进程。
- 不使用真实外部服务也能完成自动化测试。

### 验收问题

- 为什么 MCP 应该在 Tools 和 Extensions 之后学习？
- MCP Resource 是否应该强行映射成 Tool？还有哪些选择？
- MCP server 的权限边界由 Pi、Extension 还是操作系统决定？

---

# 第四阶段：交互界面与实验架构

## 第 12 章：TUI、Print 与 RPC

### 本章目标

理解同一个 `AgentSession` 如何被不同 I/O 外壳使用，并区分当前 RPC mode 与实验性远程 protocol。

### 三种当前运行模式

| 模式 | 输入输出 | 适用场景 |
|---|---|---|
| Interactive | 终端 TUI | 人机交互 |
| Print / JSON | 单次输入和 stdout | shell 管道、脚本 |
| RPC | stdin/stdout JSONL | 外部进程控制 Pi |

入口：

- [`packages/coding-agent/src/modes/interactive`](packages/coding-agent/src/modes/interactive)
- [`packages/coding-agent/src/modes/print-mode.ts`](packages/coding-agent/src/modes/print-mode.ts)
- [`packages/coding-agent/src/modes/rpc`](packages/coding-agent/src/modes/rpc)
- [`packages/coding-agent/docs/rpc.md`](packages/coding-agent/docs/rpc.md)

### TUI 阅读顺序

1. [`packages/tui/README.md`](packages/tui/README.md) 的 Core API。
2. [`packages/tui/src/tui.ts`](packages/tui/src/tui.ts)
3. main-screen 与 alt-screen renderer。
4. component interface、focus 与 input。
5. [`packages/coding-agent/src/modes/interactive/interactive-mode.ts`](packages/coding-agent/src/modes/interactive/interactive-mode.ts) 如何订阅 AgentSession。

不要一开始研究颜色、Markdown 或图片。先找清楚：

```text
terminal input
  → editor/command
  → AgentSession.prompt()
  → AgentSessionEvent
  → component state
  → render
  → terminal output
```

### 两种“RPC/协议”不要混淆

- `packages/coding-agent/src/modes/rpc/`：当前 CLI 的 stdin/stdout JSONL 控制协议。
- `packages/protocol`、`client`、`server`：实验性 CBOR framed remote Session 协议。

前者直接控制一个 coding-agent 进程；后者面向远程 Session 路由、attachment 和多 presentation。

### 小实验

写一个最小 RPC client：

1. 启动 `pi --mode rpc`。
2. 发送 prompt。
3. 消费 `message_update`。
4. 等待 `agent_settled`，而不是只等待 `agent_end`。
5. 在运行中发送 steer。

### 代表测试

- [`packages/coding-agent/test/rpc-prompt-response-semantics.test.ts`](packages/coding-agent/test/rpc-prompt-response-semantics.test.ts)
- [`packages/coding-agent/test/rpc-jsonl.test.ts`](packages/coding-agent/test/rpc-jsonl.test.ts)
- [`packages/coding-agent/test/interactive-tui.test.ts`](packages/coding-agent/test/interactive-tui.test.ts)

### 验收问题

- 为什么 UI 应订阅事件，而不是轮询 Agent 状态？
- `agent_end` 和 `agent_settled` 对 RPC client 有何区别？
- 当前 JSONL RPC 与实验性 CBOR protocol 的边界是什么？

## 第 13 章：实验性 Session / AgentHarness / Remote 架构

### 本章目标

在理解当前实现后，研究 `research` 分支正在解决的持久化、并发、恢复和远程 presentation 问题。

### 为什么需要另一套架构

当前 `AgentSession + SessionManager` 主要围绕一个本地 coding-agent 进程设计。更一般的 Harness 需要处理：

- Session 与 UI 生命周期解耦。
- durable operation log。
- 进程重启后的恢复或悬挂操作识别。
- 多个 presentation attachment。
- memory、JSONL、SQLite 等可替换 backend。
- 远程 client/server 路由。
- 同一 Session 的 writer ownership 和并发控制。

### 阅读顺序

不要先读网络协议。按下面顺序：

1. Session 接口和记录类型：
   - [`packages/agent/src/harness/session/types.ts`](packages/agent/src/harness/session/types.ts)
   - [`packages/agent/src/harness/session/session.ts`](packages/agent/src/harness/session/session.ts)
2. 内存实现和 conformance tests：
   - [`packages/agent/src/harness/session/memory.ts`](packages/agent/src/harness/session/memory.ts)
   - [`packages/agent/src/harness/session/testing`](packages/agent/src/harness/session/testing)
3. JSONL 实现：
   - [`packages/agent/src/harness/session/jsonl`](packages/agent/src/harness/session/jsonl)
4. SQLite backend：
   - [`packages/session-backends/sqlite-node`](packages/session-backends/sqlite-node)
5. Harness 公共接口：
   - [`packages/agent/src/harness/agent-harness.ts`](packages/agent/src/harness/agent-harness.ts)
   - [`packages/agent/src/harness/types.ts`](packages/agent/src/harness/types.ts)
6. 远程 transport：
   - [`packages/protocol`](packages/protocol)
   - [`packages/client`](packages/client)
   - [`packages/server`](packages/server)
7. 产品实验集成：
   - [`packages/coding-agent/src/experimental`](packages/coding-agent/src/experimental)

### 当前状态注意事项

撰写本文时，`AgentHarness` 的一部分配置读取接口已经存在，但若干关键运行方法仍通过 `HarnessNotImplemented` 表示 scaffold。遇到这种代码时：

1. 先以类型和测试推断目标契约。
2. 再确认当前实现是否真的可调用。
3. 不要把未来设计当作当前产品行为。

### Session 日志的两类记录

新 Session 抽象区分：

- Entry：对话树中可被导航和投影为上下文的持久内容。
- Lane record：运行操作、Tool 执行、队列、abort、usage 等操作日志。

这使“对话内容”和“运行过程”可以独立表达。

### Remote 路由心智模型

```text
Client connection
  → serverId
  → sessionId
  → attachmentId
  → Session service
```

三段身份分别防止：

- 连接到了错误的逻辑 server。
- 请求被路由到错误 Session。
- Session 重新 attach 后，旧延迟消息污染新 presentation。

`pi-protocol` 只验证路由信封、严格 JSON 边界、CBOR 和 framing；业务 payload 由上层 service contract 验证。

### 小实验

1. 对 Memory、JSONL 和 SQLite backend 运行相同 conformance 测试。
2. 创建带两个分支的 Session，验证不同 backend 返回同样的 tree/context。
3. 模拟断开的 client attachment，验证旧 `attachmentId` 请求被拒绝。
4. 找出一个 `HarnessNotImplemented` 方法，写出它需要满足的状态迁移表，但暂不实现。

### 代表测试

- [`packages/agent/test/harness/memory-conformance.test.ts`](packages/agent/test/harness/memory-conformance.test.ts)
- [`packages/agent/test/harness/jsonl-session-repo-conformance.test.ts`](packages/agent/test/harness/jsonl-session-repo-conformance.test.ts)
- [`packages/session-backends/sqlite-node/test/repo-conformance.test.ts`](packages/session-backends/sqlite-node/test/repo-conformance.test.ts)
- [`packages/protocol/test/protocol.test.ts`](packages/protocol/test/protocol.test.ts)
- [`packages/client/test/client.test.ts`](packages/client/test/client.test.ts)
- [`packages/server/test/protocol.test.ts`](packages/server/test/protocol.test.ts)

### 验收问题

- 新 Session 抽象为什么同时需要 Entry 和 Lane record？
- `sessionId` 与 `attachmentId` 为什么不能合并？
- conformance tests 对可替换 backend 有什么价值？
- 哪些行为是当前实现，哪些仍是接口意图？

---

# 第五阶段：质量、可观测性与深化

## 第 14 章：Testing、Telemetry 与 Evals

### 本章目标

理解 deterministic unit test、runtime conformance test 和 model-backed eval 各自能证明什么。

### 三种验证方式

| 类型 | 是否调用真实模型 | 适合验证 | 不适合验证 |
|---|---:|---|---|
| 单元/集成测试 + faux provider | 否 | 事件顺序、状态迁移、错误边界、序列化 | 模型实际理解能力 |
| Conformance test | 否 | 多 backend/adapter 是否满足同一契约 | 具体实现性能和模型质量 |
| Eval | 是 | Prompt、Skill、Tool 对任务成功率的影响 | 完全确定性的回归保证 |

### Testing Harness

注意“test harness”和“AgentHarness”不是同一个概念：

- `test/suite/harness.ts` 是测试夹具，用 faux provider 构造可控 AgentSession。
- `AgentHarness` 是实验性的产品运行时接口。

优先阅读：

- [`packages/coding-agent/test/suite/harness.ts`](packages/coding-agent/test/suite/harness.ts)
- [`packages/ai/src/providers/faux.ts`](packages/ai/src/providers/faux.ts)
- [`packages/agent/src/harness/session/testing`](packages/agent/src/harness/session/testing)

### Telemetry

Telemetry 层保持 vendor-neutral：业务代码记录 operation、属性和结果，adapter 决定保存到内存、无操作或外部系统。

阅读：

- [`packages/telemetry/README.md`](packages/telemetry/README.md)
- [`packages/telemetry/src/index.ts`](packages/telemetry/src/index.ts)
- [`packages/telemetry/src/memory.ts`](packages/telemetry/src/memory.ts)
- [`packages/telemetry/src/testing/conformance.ts`](packages/telemetry/src/testing/conformance.ts)

重点检查敏感数据边界：prompt、源码、工具输出和凭证不应因为“加了 telemetry”就被默认暴露。

### Evals

Evals 使用真实模型回答“某个系统设计是否提高任务成功率”。例如比较：

```text
baseline：不加载目标 Skill
candidate：加载目标 Skill
repetitions：多次重复
judge：按任务结果评分
```

阅读：

- [`packages/evals/README.md`](packages/evals/README.md)
- [`packages/evals/src/pi-harness.ts`](packages/evals/src/pi-harness.ts)
- [`packages/evals/src/vitest-evals/harness-table.ts`](packages/evals/src/vitest-evals/harness-table.ts)

### 小实验

1. 用 faux provider 为 Tool Call 顺序写确定性测试。
2. 为一个 memory adapter 运行 conformance tests。
3. 设计一个 Skill 的 baseline/candidate eval，但除非明确需要，不运行真实模型。
4. 说明一次 eval 失败究竟是产品回归、模型方差还是 judge 问题。

### 验收问题

- 为什么 agent-loop 回归测试不应该依赖真实模型？
- Conformance test 与普通单元测试的区别是什么？
- 为什么 eval 需要重复运行并区分 correctness、token、latency 和 cost？

## 15. 推荐学习节奏

### 第一轮：建立主干

完成第 1～5 章。

输出物：

- 一张 `AgentSession → Agent → agent-loop → pi-ai` 调用图。
- 一张完整 Tool Call 事件时序图。
- 一个 faux provider 驱动的小 Tool。

### 第二轮：掌握上下文与持久化

完成第 6～8 章。

输出物：

- 一个包含分支和 compaction 的 Session tree。
- 一份 `SessionEntry → AgentMessage → Message` 转换说明。
- 一个自定义 Skill，并说明自动调用和显式调用的区别。

### 第三轮：掌握扩展与安全

完成第 9～11 章。

输出物：

- 一个 Tool 审计/权限 Extension。
- 一份三层安全边界威胁表。
- 一个最小 MCP adapter 设计和 faux server 测试。

### 第四轮：掌握外壳与未来架构

完成第 12～14 章。

输出物：

- 一个最小 RPC client。
- 当前 SessionManager 与新 Session abstraction 的对比表。
- 一份测试、conformance 和 eval 的分层策略。

## 16. 一条请求的最终复盘模板

完成全部章节后，选择一次真实交互，按下面模板复盘：

```text
1. 输入从哪个 mode 进入？
2. 哪一层处理命令、Skill 和模板展开？
3. system prompt 在哪里构建？
4. Agent 收到哪些 AgentMessage？
5. transformContext 和 convertToLlm 分别做了什么？
6. 哪个 Provider 接收最终 Context？
7. 流式事件怎样到达 UI？
8. Tool Call 在哪里验证、拦截和执行？
9. Tool Result 怎样回到下一次模型调用？
10. 哪些消息何时写入 Session？
11. 是否触发 retry、overflow recovery 或 compaction？
12. abort 时每一层如何收尾？
13. Extension 能在哪些位置改变结果？
14. 安全边界由哪一层提供？
```

如果可以不看代码完整回答这十四个问题，就已经建立了 Pi 的整体心智模型。之后再深入某个 Provider、TUI component 或远程协议，不会失去方向。

## 17. 常见误区

### 误区一：先读 `main.ts` 全部代码

`main.ts` 包含大量 CLI 分支、迁移和启动处理。第一遍只跟踪服务创建、Session 创建和 mode 分发。

### 误区二：把 `AgentSession` 当成 Session 文件

`AgentSession` 是运行时编排对象；`SessionManager` 管理持久化树。名字相似，生命周期不同。

### 误区三：把 Skill 当成 Tool

Skill 主要提供说明和资源；Tool 是结构化可调用函数。Skill 可以指导 Agent 调用 Tool，但二者不在同一层。

### 误区四：把 Project Trust 当成 Tool Permission

Project Trust 只控制项目资源加载。项目获信任后，内置工具仍以 Pi 进程权限运行。

### 误区五：认为仓库有内置 MCP 模块

Pi 的设计选择是通过 Extensions/packages 集成 MCP。学习重点应是适配边界，而不是搜索不存在的核心目录。

### 误区六：把当前 RPC 和实验 protocol 混为一谈

当前 RPC 是 CLI stdin/stdout JSONL；实验 protocol 是 routed envelope + CBOR framing，两者解决的问题不同。

### 误区七：把 scaffold 当成已完成行为

`research` 分支包含接口先行的代码。必须同时查看实现和测试，确认方法是否可运行。

## 18. 完成标准

完成这份路线不等于读过所有文件。满足以下条件即可认为完成第一轮源码学习：

- 能画出无工具和有工具时的完整 Run 时序。
- 能解释 `pi-ai`、`Agent`、`AgentSession` 和 `SessionManager` 的边界。
- 能解释 AgentMessage、Message 和 SessionEntry 的转换关系。
- 能写一个 Tool、一个 Skill 和一个 Extension，并说明各自适用场景。
- 能解释 steer、follow-up、retry、abort 和 compaction 的时机。
- 能区分 Project Trust、Tool Policy 和 OS isolation。
- 能说明如何通过 Extension 接入 MCP。
- 能区分当前产品链路与实验性 AgentHarness/remote 架构。
- 能为关键行为选择单元测试、conformance test 或 eval。

