# 吃透 Agent Loop 和 Tool Calling

这篇笔记配合 `packages/agent/src/agent-loop.ts` 阅读。目标不是记住每一行代码，而是能讲清楚：

1. Agent Loop 为什么不是简单的 `while true`。
2. 一条用户消息如何变成模型请求、工具调用、工具结果、下一次模型请求。
3. Runtime 如何处理工具错误、取消、截断输出、并发工具调用和上下文变换。

## 先看整体职责

`agent-loop.ts` 是 `packages/agent` 的核心循环。它不关心 CLI、TUI、Session 文件、AGENTS.md，也不关心具体工具是 read、write 还是 bash。

它只负责一件事：

```text
给定 context、tools、model、streamFn
持续执行：
  模型响应
  工具调用
  工具结果回写
  下一次模型响应
直到本轮结束
```

换句话说：

```text
coding-agent 负责把 Pi 做成产品。
agent-loop 负责把模型和工具跑成一个闭环。
```

## 核心入口

这个文件有两组入口。

第一组返回事件流：

```ts
agentLoop(...)
agentLoopContinue(...)
```

第二组直接执行，并返回本次产生的新消息：

```ts
runAgentLoop(...)
runAgentLoopContinue(...)
```

区别：

```text
agentLoop:
  新增 prompt，然后开始跑。

agentLoopContinue:
  不新增 prompt，从已有 context 继续跑。
  常见用途是 retry，或者 context 已经包含 user/toolResult，需要重新请求模型。
```

`agentLoopContinue` 明确拒绝从 assistant 消息继续：

```ts
if (context.messages[context.messages.length - 1].role === "assistant") {
  throw new Error("Cannot continue from message role: assistant");
}
```

原因是多数 provider 要求下一轮输入最后必须是 user 或 toolResult。assistant 后面直接接 assistant，会让 provider 认为对话结构非法。

## 三个最重要的状态变量

读 `runLoop` 时，先盯住这三个变量：

```ts
let currentContext = initialContext;
let lastCompletedTurn: PrepareNextTurnContext | undefined;
const newMessages: AgentMessage[] = ...
```

含义：

```text
currentContext.messages:
  当前完整运行上下文。
  下一次模型请求会从这里取消息。

newMessages:
  这一次 run 新产生的消息。
  用于返回给调用方，也用于 agent_end 事件。

lastCompletedTurn:
  上一次模型响应和工具结果的快照。
  prepareNextTurn 用它做压缩、切模型、切 thinking level 等处理。
```

这三个变量解决的是不同问题：

```text
currentContext 是“下一步要给模型看什么”。
newMessages 是“这次运行新增了什么”。
lastCompletedTurn 是“上一轮发生了什么，给运行时做决策”。
```

## 主流程图

一条普通用户消息的链路是：

```text
prompts
  ↓
declareToolChanges()
  ↓
写入 currentContext.messages
  ↓
prepareRequest()
  ↓
streamAssistantResponse()
  ↓
assistant message
  ↓
检查 toolCall
  ↓
executeToolCalls()
  ↓
ToolResultMessage
  ↓
写回 currentContext.messages
  ↓
finishTurn()
  ↓
如果还有工具结果要给模型看，继续下一轮请求
```

更具体一点：

```text
UserMessage
  ↓
LLM
  ↓
AssistantMessage(toolCall: read)
  ↓
Runtime 执行 read
  ↓
ToolResultMessage(read 的结果)
  ↓
LLM 再次读取 ToolResult
  ↓
AssistantMessage(最终回答)
```

## 两层循环分别做什么

`runLoop` 里有两层循环：

```ts
while (true) {
  while (hasMoreToolCalls || pendingMessages.length > 0) {
    ...
  }
}
```

内层循环负责模型请求：

```text
只要还有工具结果要喂给模型，
或者有 steering 消息插队，
就继续请求模型。
```

外层循环负责用户可感知的一轮：

```text
当内层已经没有工具和 steering 消息时，
Agent 本来要结束。
但如果这时有 follow-up 消息，就重新打开内层循环。
```

可以这样区分：

```text
steering:
  Agent 正在跑时，用户插队影响当前运行。

follow-up:
  Agent 刚要停下时，用户追加下一条任务。
```

## prepareNextTurn 和 prepareRequest 的区别

这两个钩子很容易混。

`prepareNextTurn` 在上一轮完成之后运行：

```ts
const nextTurnSnapshot = await config.prepareNextTurn?.(lastCompletedTurn);
```

适合做：

```text
上下文压缩 compaction
根据上一轮结果切换模型
根据上一轮结果切换 thinking level
插入运行时准备的消息
```

它看到的是“上一轮模型响应 + 工具结果”。

`prepareRequest` 在真正请求模型前运行：

```ts
const requestUpdate = await config.prepareRequest?.({
  context: currentContext,
  model: config.model,
  thinkingLevel: config.reasoning ?? "off",
});
```

适合做：

```text
请求前最后一次 context 调整
模型选择确认
provider 前置处理
运行时上下文变换
```

一句话：

```text
prepareNextTurn 是 turn 之间的整理。
prepareRequest 是 provider 请求前的最后加工。
```

## provider 边界在哪里

文件开头就说了：

```text
Agent loop that works with AgentMessage throughout.
Transforms to Message[] only at the LLM call boundary.
```

真正转换发生在：

```ts
const llmMessages = await config.convertToLlm(messages);
```

也就是 `streamAssistantResponse()` 里。

为什么这样设计：

```text
AgentMessage 是 runtime 内部消息格式。
Message 是 provider 能理解的模型消息格式。

runtime 内部需要保留更多控制信息，
例如 toolResult、custom message、tool loadout change。

只有到调用模型前，才把这些消息投影成 provider payload。
```

这和 Session/Context 的关系类似：

```text
内部事实表示更丰富。
对外请求表示更受 provider 约束。
```

## 流式响应如何进入 context

`streamAssistantResponse()` 的核心逻辑：

```text
start:
  context.messages.push(partial assistant)

delta:
  替换 context 里最后一条 partial assistant

done/error:
  用 final assistant 替换 partial assistant
```

这样做有两个好处：

```text
UI 可以实时渲染。
context 里始终只有一条正在生成的 assistant 消息。
```

如果 provider 没有发 start，也会在 done/error 时补一条 final message。

## 工具集合为什么要写进 system message

工具可用性不是只存在内存里，还要告诉模型。

`declareToolChanges()` 做的事：

```text
读取 transcript 当前声明过的工具
读取 runtime 当前实际可执行工具
计算差异
把 toolsAdded/toolsRemoved 写到 system message
```

关键代码：

```ts
const changes = getToolStateChanges(
  getCurrentTools([...context.messages, ...baseline]),
  (context.tools ?? []).map(toToolDeclaration),
);
```

为什么不每次都重新塞完整工具列表：

```text
因为 transcript 是可重放的。
工具集合变化可以作为增量历史记录。
session replay 时，看到 toolsAdded/toolsRemoved 就能恢复当时模型可见的工具状态。
```

## Tool Calling 的完整链路

工具执行入口：

```ts
executeToolCalls(...)
```

它先决定顺序执行还是并发执行：

```ts
if (config.toolExecution === "sequential" || hasSequentialToolCall) {
  return executeToolCallsSequential(...);
}
return executeToolCallsParallel(...);
```

只要批次里有一个工具要求顺序执行，整个批次就顺序执行。

原因：

```text
edit/write/bash 这类工具有副作用。
如果它们并发跑，可能互相踩文件状态、工作目录状态或外部进程状态。
```

单个工具调用的生命周期是：

```text
prepareToolCall()
  ↓
executePreparedToolCall()
  ↓
finalizeExecutedToolCall()
  ↓
createToolResultMessage()
  ↓
emitToolResultMessage()
```

## prepareToolCall：执行前阶段

`prepareToolCall()` 做四件事：

1. 查找工具是否存在。
2. 调用 `tool.prepareArguments()` 做参数兼容或归一化。
3. 调用 `validateToolArguments()` 做 schema 校验。
4. 调用 `beforeToolCall()` 给运行时审批或拦截机会。

如果工具不存在：

```ts
result: createErrorToolResult(`Tool ${toolCall.name} not found`)
```

如果参数非法：

```text
catch error → createErrorToolResult(error.message)
```

如果 `beforeToolCall` 阻止：

```text
返回 error ToolResult
可选 terminate = true
```

注意：这些失败都不是直接 throw 到外层，而是变成 ToolResult。

原因：

```text
模型需要看到工具失败的事实。
然后模型才能决定重试、换工具、换方案或结束。
```

这就是 Coding Agent 和普通函数调用的区别。

## executePreparedToolCall：真正执行工具

真正执行在：

```ts
prepared.tool.execute(
  prepared.toolCall.id,
  prepared.args as never,
  signal,
  updateCallback,
)
```

第四个参数是进度回调：

```ts
(partialResult) => {
  emit({ type: "tool_execution_update", partialResult })
}
```

这里没有每次 update 都直接阻塞工具执行，而是收集 promise：

```ts
updateEvents.push(Promise.resolve(emit(...)));
```

工具执行结束后再：

```ts
await Promise.all(updateEvents);
```

这样可以避免 UI 或事件消费者慢，反过来拖慢工具本身。

## finalizeExecutedToolCall：执行后阶段

执行后会进入：

```ts
config.afterToolCall(...)
```

这个钩子可以改写：

```text
content
details
usage
terminate
isError
```

典型用途：

```text
压缩过长工具输出
补充 usage
把技术成功但策略失败的结果标记为错误
隐藏不该直接给模型的细节
```

## ToolResultMessage 为什么重要

工具执行结果最后会变成：

```ts
{
  role: "toolResult",
  toolCallId,
  toolName,
  content,
  details,
  usage,
  isError,
  timestamp,
}
```

它会被写入：

```text
currentContext.messages
newMessages
事件流
```

这一步非常关键。

模型不能直接知道工具执行结果。Runtime 必须把结果作为消息写回上下文：

```text
AssistantMessage(toolCall)
  ↓
Runtime 执行工具
  ↓
ToolResultMessage
  ↓
下一次模型请求
```

所以面试里可以说：

```text
Tool Calling 的闭环不是“模型调用工具”结束，而是 ToolResult 回写后，再让模型基于真实结果继续推理。
```

## 为什么模型输出被截断时不执行工具

代码里有这个分支：

```ts
message.stopReason === "length"
  ? await failToolCallsFromTruncatedMessage(...)
  : await executeToolCalls(...)
```

意思是：如果 assistant message 因输出 token 限制被截断，那么里面的 tool call 参数可能是不完整的。

即使 JSON 被 salvage parser 勉强解析出来，也可能语义上缺字段。

所以这里选择：

```text
不执行工具。
为每个 toolCall 生成错误 ToolResult。
让模型重新发完整 toolCall。
```

这是一个很好的工程化细节：

```text
宁愿让模型重试，也不要执行可能参数不完整的危险操作。
```

## 顺序工具和并发工具

顺序执行：

```text
tool1 prepare → execute → finalize → emit result
tool2 prepare → execute → finalize → emit result
```

并发执行：

```text
先逐个 prepare
再 Promise.all 执行可执行项
最后按原始 toolCall 顺序 emit ToolResult
```

并发模式里，虽然执行可能乱序完成，但结果写回仍保持原顺序：

```text
assistant toolCall 顺序稳定
ToolResultMessage 顺序稳定
transcript replay 稳定
下一次模型输入稳定
```

这是为了避免同一批工具结果在不同运行中顺序不同，导致模型看到的上下文不稳定。

## terminate 的含义

工具结果里可以带：

```ts
terminate: true
```

但这批工具只有在所有工具都要求 terminate 时才停止继续请求模型：

```ts
finalizedCalls.every((finalized) => finalized.result.terminate === true)
```

为什么不是任意一个 terminate 就停：

```text
一个批次可能有多个 toolCall。
如果只要一个工具 terminate 就停止，其他工具结果可能没机会被模型读取。
这里要求全体 terminate，语义更保守。
```

如果没有 terminate，工具结果会写回 context，然后继续下一次 LLM 请求。

## 错误处理思路

这个文件的错误处理有一个统一原则：

```text
工具层错误尽量转成 ToolResult。
模型层错误转成 AssistantMessage(stopReason = error)。
Loop 层通过事件和 finishTurn 给上层收口。
```

例子：

```text
工具不存在 → error ToolResult
参数校验失败 → error ToolResult
beforeToolCall 阻止 → error ToolResult
tool.execute throw → error ToolResult
afterToolCall throw → error ToolResult
模型请求失败 → assistant stopReason error
用户取消 → assistant stopReason aborted 或工具 Operation aborted
```

这保证了：

```text
失败也是上下文事实。
模型能看到失败，并决定下一步。
上层 UI/session 也能收到事件并持久化。
```

## 事件流在做什么

这个文件不断 emit：

```text
agent_start
turn_start
message_start
message_update
message_end
tool_execution_start
tool_execution_update
tool_execution_end
turn_end
agent_end
```

事件不是装饰，它连接了：

```text
TUI 实时渲染
print/json 输出
session 持久化
测试断言
上层 runtime 状态更新
```

可以把事件理解成：

```text
Agent Loop 对外公布自己的状态变化。
```

## 一次完整 trace

假设用户说：

```text
读取 README.md，总结项目做什么。
```

可能 trace：

```text
runAgentLoop
  emit agent_start
  emit turn_start
  append UserMessage

runLoop
  prepareRequest
  streamAssistantResponse
    AssistantMessage(toolCall: read README.md)

  executeToolCalls
    prepareToolCall
      find read tool
      validate args
      beforeToolCall
    executePreparedToolCall
      read README.md
    finalizeExecutedToolCall
      afterToolCall
    create ToolResultMessage

  append ToolResultMessage
  finishTurn

  next model request
    AssistantMessage(text: 项目总结)

  finishTurn
  emit agent_end
```

## 面试怎么讲

可以这样回答：

```text
Pi 的 Agent Loop 核心在 packages/agent/src/agent-loop.ts。
它把模型响应和工具执行做成一个事件驱动的状态机，而不是简单 while true。

每一轮会先准备上下文，然后在 provider 边界把 AgentMessage 转成模型 Message。
模型如果返回 toolCall，runtime 会查找工具、归一化参数、做 schema 校验，
再经过 beforeToolCall 审批，执行工具，afterToolCall 修正结果，
最后把 ToolResultMessage 写回 context。

下一次模型请求会读取这个 ToolResult，再决定继续调用工具还是给最终回答。
```

如果面试官追问“工程化在哪里”，可以补：

```text
它处理了 streaming partial message、工具集合增量声明、顺序/并发工具执行、
截断 toolCall 不执行、取消信号、工具错误转 ToolResult、finishTurn 收口、
steering/follow-up 队列，以及 prepareNextTurn/prepareRequest 两层上下文钩子。
```

## 读源码时最该背下来的句子

```text
Agent Loop 的本质不是模型调用工具，而是：
模型提出外部动作，Runtime 受控执行，结果作为 ToolResult 回写上下文，
再让模型基于真实结果继续推理。
```

这句话能把 Tool Calling、Runtime 边界、Context 回写三个重点串起来。
