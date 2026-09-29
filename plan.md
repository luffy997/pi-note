# Pi 源码学习与面试准备 7 天细化教程

目标不是把所有源码背完，而是能围绕当前仓库讲清楚：

1. Pi 的分层架构是什么。
2. 用户输入后，Agent Loop 如何推进。
3. Tool Call 如何校验、执行、回写上下文。
4. Session Tree、Context、Compaction 的关系。
5. 项目指令、Skills、Extensions 如何进入运行时。
6. 如果自己设计 Coding Agent，会借鉴和调整哪些设计。

学习方式：每天按“阅读文件 → 追踪主链路 → 做小实验 → 准备面试表达”推进。不要一开始逐行读大文件，先抓调用链和数据结构。

## 当前仓库源码地图

先记住这个仓库不是只有 `agent/ai/coding-agent/tui`，核心包如下：

| 包 | 作用 | 面试优先级 |
|---|---|---:|
| `packages/coding-agent` | CLI、交互模式、会话管理、工具、配置、资源加载 | 最高 |
| `packages/agent` | Agent Loop、工具调用、运行时 Harness、Compaction 基础能力 | 最高 |
| `packages/ai` | 多 Provider 模型 API、消息类型、流式响应、模型目录 | 高 |
| `packages/durable` | 持久化会话/文档/事务抽象，实验性运行时会用到 | 中 |
| `packages/tui` | 终端 UI 组件、输入、渲染、键盘处理 | 中低 |
| `packages/client` / `packages/server` / `packages/protocol` | 实验性客户端/服务端/RPC 协议 | 中低 |
| `packages/chord` | 插件和服务组合运行时 | 低，最后了解 |
| `packages/telemetry` | 遥测事件契约 | 低 |

建议先画依赖主线：

```text
coding-agent
  ├─ agent
  │   └─ ai
  ├─ tui
  ├─ durable / client / server / protocol 主要用于实验性服务化路径
  └─ chord 主要用于插件和服务组合
```

## Day 1：建立源码地图和启动链路

目标：知道“Pi 是什么”和“代码从哪里进来”。

阅读顺序：

1. `README.md`
2. `package.json`
3. `packages/coding-agent/package.json`
4. `packages/coding-agent/src/main.ts`
5. `packages/coding-agent/src/cli.ts`
6. `packages/coding-agent/src/cli/args.ts`
7. `packages/coding-agent/src/modes/interactive/interactive-mode.ts`
8. `packages/coding-agent/src/modes/print-mode.ts`
9. `packages/coding-agent/src/modes/rpc/rpc-mode.ts`

要回答的问题：

- CLI 入口在哪里。
- 交互模式、print 模式、RPC 模式分别解决什么问题。
- `coding-agent` 和 `agent` 的边界是什么。
- 为什么核心 Agent Loop 没有直接放在 CLI 里。

建议用命令定位入口：

```bash
rg -n "function main|export async function|new AgentSession|interactive|print" packages/coding-agent/src
```

当天产出：

```text
Pi 是一个 Coding Agent Harness。coding-agent 负责产品运行时和 CLI/TUI，agent 负责模型-工具循环，ai 屏蔽不同模型 Provider 的消息和流式接口差异。
```

面试表达模板：

```text
我读 Pi 时先按包拆分。它不是一个单文件 ReAct Demo，而是分成产品层、Agent Runtime 层、模型 Provider 层和 UI/服务化层。这样 CLI、TUI、RPC 可以复用同一个 Agent 能力。
```

## Day 2：吃透 Agent Loop 和 Tool Calling

目标：能讲清楚一次用户请求如何变成多轮模型调用和工具调用。

重点文件：

1. `packages/agent/src/agent-loop.ts`
2. `packages/agent/src/agent.ts`
3. `packages/agent/src/types.ts`
4. `packages/agent/test/agent-loop.test.ts`
5. `packages/agent/test/agent.test.ts`

先读主函数：

- `agentLoop`
- `agentLoopContinue`
- `runAgentLoop`
- `runAgentLoopContinue`
- `runLoop`
- `streamAssistantResponse`
- `executeToolCalls`

主链路：

```text
用户消息
  ↓
agentLoop()
  ↓
declareToolChanges()
  ↓
prepareRequest()
  ↓
streamAssistantResponse()
  ↓
模型返回 AssistantMessage
  ↓
检查 toolCall
  ↓
validateToolArguments()
  ↓
beforeToolCall()
  ↓
tool.execute()
  ↓
afterToolCall()
  ↓
ToolResultMessage 写回 context
  ↓
继续下一轮 LLM 请求
```

重点理解两个循环：

```text
外层循环：处理 follow-up message，让 Agent 在自然结束后还能接收后续用户输入。
内层循环：处理 tool call、steering message、显式 continue。
```

必须看懂的设计点：

- `AgentMessage[]` 只在 LLM 边界转换成 `Message[]`。
- `declareToolChanges()` 用系统消息声明工具集合变化。
- 工具调用支持并行和顺序执行。
- 模型输出被截断时，不执行可能参数不完整的工具调用，而是生成错误 ToolResult。
- `finishTurn()` 可以决定继续或结束。
- `prepareRequest()` 和 `prepareNextTurn()` 是插入上下文压缩、模型切换、请求前处理的关键钩子。

小实验：

```bash
node "$(git rev-parse --show-toplevel)/node_modules/vitest/dist/cli.js" --run packages/agent/test/agent-loop.test.ts
```

如果依赖未安装，先只阅读测试，不需要强行安装。

当天产出：

```text
Agent Loop 不是 while true。它要处理终止条件、工具错误、截断输出、取消、排队消息、turn 生命周期、工具集合变化和运行时钩子。
```

面试高频题：

```text
问：为什么 Tool Call 要作为 ToolResult 回写给模型？
答：模型只负责决策，不直接拥有外部世界状态。Runtime 执行工具后，必须把结构化结果放回上下文，模型才能基于真实执行结果决定重试、换方案或结束。
```

## Day 3：AgentSession 和产品运行时

目标：知道 `coding-agent` 如何把核心 Agent Loop 包装成可用的 Coding Agent。

重点文件：

1. `packages/coding-agent/src/core/agent-session.ts`
2. `packages/coding-agent/src/core/agent-session-runtime.ts`
3. `packages/coding-agent/src/core/agent-session-services.ts`
4. `packages/coding-agent/src/core/event-bus.ts`
5. `packages/coding-agent/src/core/model-runtime.ts`
6. `packages/coding-agent/src/core/model-resolver.ts`
7. `packages/coding-agent/src/core/defaults.ts`

`agent-session.ts` 很大，不要第一遍逐行读。按这些关键词定位：

```bash
rg -n "class AgentSession|new Agent\\(|prepareRequest|prepareNextTurn|finishTurn|append|prompt|compact|abort|retry" packages/coding-agent/src/core/agent-session.ts
```

要理解的职责：

- 创建和持有 `Agent`。
- 管理模型、thinking level、工具集合。
- 连接 SessionManager，把消息持久化。
- 在请求前构建上下文。
- 触发自动或手动 compaction。
- 向 TUI/JSON/RPC 模式发布事件。
- 处理 abort、retry、follow-up、steering。

建议画图：

```text
CLI/TUI
  ↓
AgentSession
  ├─ Agent
  ├─ SessionManager
  ├─ ModelRuntime / ModelResolver
  ├─ ResourceLoader
  ├─ Tool Definitions
  └─ EventBus
```

当天产出：

```text
AgentSession 是产品级 Runtime，不是 Agent Loop 本身。Agent Loop 负责模型和工具的循环；AgentSession 负责把循环接到会话、模型配置、资源、事件、压缩和 UI。
```

## Day 4：Session Tree、投影和持久化

目标：讲清楚 Session、History、Context 的区别。

重点文件：

1. `packages/coding-agent/src/core/session-manager.ts`
2. `packages/coding-agent/src/core/messages.ts`
3. `packages/coding-agent/src/core/session-export.ts`
4. `packages/coding-agent/docs/session-format.md`
5. `packages/coding-agent/test/session-manager.test.ts`

先看这些类型：

- `SessionHeader`
- `SessionEntryBase`
- `SessionMessageEntry`
- `CompactionEntry`
- `BranchSummaryEntry`
- `CustomEntry`
- `CustomMessageEntry`
- `ContextEditEntry`
- `SessionProjection`
- `SessionContext`

核心数据结构：

```text
SessionEntry
  ├─ id
  ├─ parentId
  ├─ type
  └─ timestamp
```

这意味着 Session 不是简单数组，而是树：

```text
root
  ↓
user1
  ↓
assistant1
  ├─ branch A
  └─ branch B
       ↓
     active leaf
```

重点函数：

- `buildSessionPath()`
- `buildContextEntries()`
- `sessionEntryToContextMessages()`
- `buildSessionProjection()`
- `buildSessionContext()`
- `appendEntry()`
- `setLeafId()`
- `getBranch()`
- `getTree()`

必须掌握的概念：

```text
Session：完整、可持久化的历史事实，包含分支、状态变化、压缩记录和自定义条目。
Active leaf：当前对话分支的末端。
Context：当前这次请求投影给模型看的消息序列。
Projection：从 Session Tree 到模型可见消息的转换结果。
```

小实验：

1. 找一个 session 文件，通常在 `~/.pi/agent/sessions/`。
2. 观察第一行 `type: "session"`。
3. 观察后续 entry 的 `id` 和 `parentId`。
4. 手动画出 active branch。
5. 找 `compaction` entry，理解它如何影响 context，但不删除原始历史。

当天产出：

```text
Session 不等于 Context。Session 是事实源，Context 是从当前 active leaf 投影出来、经过压缩和编辑后的模型输入。
```

面试高频题：

```text
问：为什么不用简单 messages[]？
答：Coding Agent 需要恢复、重试、分支、导航、压缩和扩展状态。线性 messages[] 很难表达分叉历史和 append-only 的状态演进，Session Tree 更适合持久化真实操作历史。
```

## Day 5：Context Management 和 Compaction

目标：能讲清楚长上下文超限时 Pi 怎么处理。

当前仓库有两套相关路径：稳定 coding-agent 路径和 agent harness 路径。面试优先看稳定路径，再了解 harness 抽象。

重点文件：

1. `packages/coding-agent/src/core/compaction/compaction.ts`
2. `packages/coding-agent/src/core/compaction/branch-summarization.ts`
3. `packages/coding-agent/src/core/compaction/utils.ts`
4. `packages/agent/src/harness/compaction/compaction.ts`
5. `packages/agent/src/harness/compaction/branch-summarization.ts`
6. `packages/agent/src/harness/messages.ts`
7. `packages/ai/src/utils/overflow.ts`
8. `packages/ai/src/utils/estimate.ts`

阅读主线：

```text
Session Tree
  ↓
active branch
  ↓
buildContextEntries()
  ↓
估算 token
  ↓
超过阈值
  ↓
选择需要总结的前缀
  ↓
调用模型生成 summary
  ↓
写入 compaction entry
  ↓
Context = summary + retained recent messages
```

重点理解：

- Compaction 不删除历史 entry。
- Compaction entry 本身成为上下文中的 summary。
- `firstKeptEntryId` 决定哪些原始 entry 继续保留在 context。
- 近期消息保留，较早消息总结。
- 自动 compaction 和手动 `/compact` 都应该走结构化路径。
- compaction 失败不能破坏 session，需要作为可恢复错误处理。

建议用这个例子讲：

```text
原始 Session:
m1 m2 m3 m4 m5 m6 m7 m8 m9 m10

压缩后 Context:
summary(m1-m6)
m7 m8 m9 m10

Session 文件:
m1 m2 m3 m4 m5 m6 m7 m8 m9 m10 compaction
```

查找入口：

```bash
rg -n "compact|Compaction|reserveTokens|keepRecent|contextWindow|overflow" packages/coding-agent/src packages/agent/src packages/ai/src
```

当天产出：

```text
History 是事实源，Context 是动态投影，Compaction 是投影优化。它牺牲早期细节，保留任务状态和近期操作，从而让 Agent 在有限上下文内继续工作。
```

## Day 6：Tools、ResourceLoader、项目知识和扩展

目标：知道 Coding Agent 如何接入外部世界和项目指令。

Tools 重点文件：

1. `packages/coding-agent/src/core/tools/index.ts`
2. `packages/coding-agent/src/core/tools/read.ts`
3. `packages/coding-agent/src/core/tools/write.ts`
4. `packages/coding-agent/src/core/tools/edit.ts`
5. `packages/coding-agent/src/core/tools/bash.ts`
6. `packages/coding-agent/src/core/tools/grep.ts`
7. `packages/coding-agent/src/core/tools/find.ts`
8. `packages/coding-agent/src/core/tools/ls.ts`
9. `packages/coding-agent/src/core/tools/tool-definition-wrapper.ts`
10. `packages/coding-agent/src/core/bash-executor.ts`

Tool 需要看：

- schema 如何定义。
- 参数如何校验。
- 路径如何规范化。
- 文件修改如何排队或保护。
- bash 如何执行、截断输出、处理取消。
- 错误如何返回给模型，而不是直接崩溃。

ResourceLoader 重点文件：

1. `packages/coding-agent/src/core/resource-loader.ts`
2. `packages/coding-agent/src/core/skills.ts`
3. `packages/coding-agent/src/core/prompt-templates.ts`
4. `packages/coding-agent/src/core/system-prompt.ts`
5. `packages/coding-agent/src/core/extensions/loader.ts`
6. `packages/coding-agent/src/core/extensions/types.ts`
7. `packages/coding-agent/examples/extensions`

项目指令加载逻辑：

```text
agentDir 下的全局上下文文件
  ↓
从 cwd 向上查找 AGENTS.override.md / AGENTS.md / CLAUDE.md
  ↓
去重和处理 worktree shadow
  ↓
进入系统提示或追加系统提示
```

要区分四类“记忆”：

```text
Working Memory：当前 Context。
Episodic Memory：Session 历史。
Project Memory：AGENTS.md、Skills、Prompt Templates。
External Retrieval Memory：向量库、BM25、数据库等外部检索，本仓库不是主线。
```

当天产出：

```text
Pi 没有把所有记忆都塞进一个概念。项目指令、技能、会话历史和当前上下文分别由不同模块管理，最后由运行时按需投影给模型。
```

面试高频题：

```text
问：Coding Agent 的工具失败怎么办？
答：Runtime 把失败包装成结构化 ToolResult 放回上下文，模型据此重试或换策略。同时 Runtime 还要负责超时、取消、输出截断、路径边界、日志和状态持久化。
```

## Day 7：服务化、TUI 和面试总复盘

目标：补齐工程化视角，然后把源码学习转成可讲的项目经验。

选择性阅读：

1. `packages/tui/src/tui.ts`
2. `packages/tui/src/components/input.ts`
3. `packages/tui/src/components/markdown.ts`
4. `packages/coding-agent/src/modes/interactive/tui-renderer.ts`
5. `packages/coding-agent/src/experimental/services/README.md`
6. `packages/coding-agent/src/experimental/services/agent-controller.ts`
7. `packages/coding-agent/src/experimental/session-worker.ts`
8. `packages/client/src/client.ts`
9. `packages/server/src/server.ts`
10. `packages/protocol/src/protocol.ts`
11. `packages/durable/src/session/session.ts`

这一部分不用深挖所有实现，重点理解为什么要服务化：

- UI 和 Agent 运行时解耦。
- 多客户端可以连接同一个 session worker。
- Transcript 可以作为可复制状态发布。
- Slash Commands、Models、Plugins 通过服务接口组合。
- Durable 层提供更通用的会话和事务抽象。

最终架构图：

```text
User
  ↓
CLI / TUI / RPC
  ↓
AgentSession
  ├─ Agent
  │   ├─ runLoop
  │   ├─ streamAssistantResponse
  │   └─ executeToolCalls
  ├─ SessionManager
  │   ├─ Session Tree
  │   ├─ Active Leaf
  │   └─ JSONL
  ├─ ResourceLoader
  │   ├─ AGENTS.md
  │   ├─ Skills
  │   ├─ Prompt Templates
  │   └─ Extensions
  ├─ ModelRuntime
  ├─ Tools
  │   ├─ read / write / edit
  │   ├─ grep / find / ls
  │   └─ bash
  └─ Compaction
      └─ summary + retained recent messages
  ↓
pi-ai
  ↓
LLM Provider
```

准备 5 分钟讲稿：

```text
1. Pi 是什么：Coding Agent Harness。
2. 架构：coding-agent、agent、ai 三层主线。
3. Agent Loop：模型输出、工具执行、ToolResult 回写。
4. Session Tree：append-only 历史和 active leaf。
5. Context：从 Session 投影出来的模型输入。
6. Compaction：summary + recent messages，不删除历史。
7. ResourceLoader：项目指令、技能、扩展进入运行时。
8. 工程化：取消、重试、错误、截断、事件、持久化。
9. 我的设计思考：哪些设计我会复用，哪些会按项目简化。
```

面试对比表：

| Pi | 自己的 Agent 项目可以对应 |
|---|---|
| `Agent` / `runLoop` | 主循环 |
| `AgentSession` | 产品运行时 |
| `SessionManager` | 会话持久化 |
| Session Tree | 分支、重试、恢复 |
| `buildSessionProjection()` | 上下文构建 |
| Compaction | 长上下文压缩 |
| Tools | 外部执行系统 |
| ResourceLoader | 项目知识加载 |
| Extensions | 插件机制 |
| TUI / RPC | 多交互入口 |

最终回答模板：

```text
我研究 Pi 时没有只看工具调用 Demo，而是重点看了它如何把 Agent Loop 产品化。它的核心思路是：Session 记录完整事实，Context 是当前推理投影，Compaction 优化投影大小，Tool Runtime 保证外部执行边界。这个拆分让我在设计自己的 Coding Agent 时，会优先把模型决策、工具执行、会话持久化和上下文管理分层。
```

## 7 天优先级

时间不够时按这个顺序砍：

| 模块 | 重要度 | 要求 |
|---|---:|---|
| Agent Loop | 5 | 能讲源码链路 |
| Tool Calling | 5 | 能讲校验、执行、结果回写 |
| Session Tree | 5 | 能画 active leaf |
| Context Projection | 5 | 能区分 Session 和 Context |
| Compaction | 5 | 能讲 summary + retained tail |
| AgentSession | 4 | 理解产品运行时职责 |
| ResourceLoader / AGENTS.md | 4 | 能讲项目知识加载 |
| Tool Runtime 工程化 | 4 | 能讲失败、取消、截断、恢复 |
| ai Provider 层 | 3 | 理解消息和流式接口 |
| TUI | 2 | 知道职责即可 |
| 服务化 experimental | 2 | 了解方向即可 |
| chord / telemetry | 1 | 最后扫一遍 |

每天最后 30 分钟不要继续看源码，只做口述复盘：

```text
这个模块解决什么问题？
如果不用这个设计会怎样？
核心数据结构是什么？
一次真实请求如何经过它？
我自己的项目会怎么借鉴或简化？
```
