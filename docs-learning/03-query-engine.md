# Claude Code 源码学习：QueryEngine (`src/QueryEngine.ts`)

> QueryEngine 是连接"用户输入"与"底层 Agent Loop"的桥梁。它负责把一次 prompt 提交转化为完整的 API 请求生命周期，并在过程中管理状态、持久化、预算控制和 SDK 事件流。

---

## 一、QueryEngine 的定位

如果把 Claude Code 比作一艘飞船：

- **`query.ts`** = 曲速引擎（核心 Agent Loop，最难、最底层）
- **`QueryEngine.ts`** = 舰桥（编排一次完整的航行任务，管理船员和物资）
- **`main.tsx / REPL.tsx`** = 船壳和驾驶舱（UI、CLI 入口）

```ts
export class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]
  private abortController: AbortController
  private permissionDenials: SDKPermissionDenial[]
  private totalUsage: NonNullableUsage
  private readFileState: FileStateCache
  // ...
}
```

一个 `QueryEngine` 实例对应**一次会话（conversation）**。每次调用 `submitMessage()` 就是会话中的一个新 turn。

---

## 二、核心入口：`submitMessage()` 全流程

```ts
async *submitMessage(
  prompt: string | ContentBlockParam[],
  options?: { uuid?: string; isMeta?: boolean },
): AsyncGenerator<SDKMessage, void, unknown> {
```

这是一个 `async generator`，它 `yield` 出的事件会被 SDK 消费者（如 Claude Desktop、cowork）或 REPL UI 接收。

### 阶段 1：初始化与上下文准备

```ts
this.discoveredSkillNames.clear()
setCwd(cwd)
const persistSession = !isSessionPersistenceDisabled()
const startTime = Date.now()
```

#### 1.1 包装 `canUseTool`

```ts
const wrappedCanUseTool: CanUseToolFn = async (tool, input, toolUseContext, assistantMessage, toolUseID, forceDecision) => {
  const result = await canUseTool(tool, input, toolUseContext, assistantMessage, toolUseID, forceDecision)
  
  // 记录权限拒绝事件，供 SDK 上报
  if (result.behavior !== 'allow') {
    this.permissionDenials.push({
      tool_name: sdkCompatToolName(tool.name),
      tool_use_id: toolUseID,
      tool_input: input,
    })
  }
  
  return result
}
```

#### 1.2 获取 System Prompt

```ts
const { defaultSystemPrompt, userContext: baseUserContext, systemContext } = 
  await fetchSystemPromptParts({
    tools,
    mainLoopModel: initialMainLoopModel,
    additionalWorkingDirectories: Array.from(
      initialAppState.toolPermissionContext.additionalWorkingDirectories.keys()
    ),
    mcpClients,
    customSystemPrompt: customPrompt,
  })
```

`fetchSystemPromptParts()` 是整个系统 prompt 的工厂，它会：
- 根据当前工具集生成工具描述
- 注入 CLAUDE.md 记忆
- 注入 MCP 工具描述
- 注入 Agent 定义
- 注入 custom / append system prompt

#### 1.3 组装最终 systemPrompt

```ts
const systemPrompt = asSystemPrompt([
  ...(customPrompt !== undefined ? [customPrompt] : defaultSystemPrompt),
  ...(memoryMechanicsPrompt ? [memoryMechanicsPrompt] : []),
  ...(appendSystemPrompt ? [appendSystemPrompt] : []),
])
```

### 阶段 2：处理用户输入 `processUserInput`

这是最关键的分支点。用户的输入可能是：
1. **普通文本 prompt**
2. **斜杠命令**（如 `/compact`, `/cost`）
3. **Bash 模式命令**
4. **Bridge 远程输入**
5. **Ultraplan 关键词触发**

```ts
const { messages: messagesFromUserInput, shouldQuery, allowedTools, model: modelFromUserInput, resultText } = 
  await processUserInput({
    input: prompt,
    mode: 'prompt',
    context: processUserInputContext,
    messages: this.mutableMessages,
    uuid: options?.uuid,
    isMeta: options?.isMeta,
    querySource: 'sdk',
  })
```

#### 2.1 processUserInput 内部流程（`src/utils/processUserInput/processUserInput.ts`）

```
输入字符串/图片
    │
    ├─→ 图片处理：resize、downsample、存储到磁盘
    │
    ├─→ 检测是否以 `/` 开头
    │   ├─ YES → 解析 slash command → 执行本地命令
    │   │         ├─ `shouldQuery = false`（多数命令不调用 API）
    │   │         └─ 返回命令输出作为 messages
    │   └─ NO  → 继续
    │
    ├─→ 检测 Ultraplan 关键词（如 "帮我制定一个计划"）
    │   └─ YES → 自动路由到 `/ultraplan`
    │
    ├─→ 提取附件（Attachments）
    │   ├─ IDE 选中的代码片段
    │   ├─ 当前工作目录上下文
    │   ├─ 文件变更通知
    │   └─ Agent Mention（`@agent-explore`）
    │
    ├─→ 执行 UserPromptSubmit Hooks
    │   └─ 可能阻塞、追加上下文、或阻止继续
    │
    └─→ 组装为 UserMessage
```

**如果 `shouldQuery === false`**（如 `/cost` 命令直接输出本地信息），`submitMessage` 会直接返回结果，**不进入 API 调用**。

### 阶段 3：持久化 Transcript

```ts
this.mutableMessages.push(...messagesFromUserInput)
const messages = [...this.mutableMessages]

// 在进 API 前先把用户消息写入磁盘
if (persistSession && messagesFromUserInput.length > 0) {
  const transcriptPromise = recordTranscript(messages)
  if (isBareMode()) {
    void transcriptPromise  // 脚本模式：fire-and-forget
  } else {
    await transcriptPromise  // 交互模式：阻塞确保可恢复
    if (isEnvTruthy(process.env.CLAUDE_CODE_EAGER_FLUSH)) {
      await flushSessionStorage()
    }
  }
}
```

**为什么要在 API 响应前就写 transcript？**
> 如果用户在发送消息后立刻杀掉进程（如在 Desktop app 中点 Stop），而 API 还没返回任何 assistant 消息，那么 transcript 里只有 queue-operation 条目。`getLastSessionLog` 会过滤掉这些并返回 null，导致 `--resume` 失败。提前写入让会话总是可以从"用户消息已接受"的状态恢复。

### 阶段 4：进入 Agent Loop

```ts
for await (const message of query({
  messages,
  systemPrompt,
  userContext,
  systemContext,
  canUseTool: wrappedCanUseTool,
  toolUseContext: processUserInputContext,
  fallbackModel,
  querySource: 'sdk',
  maxTurns,
  taskBudget,
})) {
  // 消费 query() 生成器产出的各种消息...
}
```

这就是连接第 1 章（Agent Loop）的地方。`QueryEngine` 作为"消费者"，把 `query()` 的原始事件翻译成 SDK 可以理解的事件。

### 阶段 5：消息类型路由与状态同步

`query()` 会 `yield` 多种类型的消息。`QueryEngine` 用一个大 `switch` 处理它们：

#### `assistant`

```ts
case 'assistant':
  if (message.message.stop_reason != null) {
    lastStopReason = message.message.stop_reason
  }
  this.mutableMessages.push(message)
  yield* normalizeMessage(message)  // 转成 SDKMessage 格式 yield 出去
  break
```

#### `progress`（工具执行进度）

```ts
case 'progress':
  this.mutableMessages.push(message)
  if (persistSession) {
    messages.push(message)
    void recordTranscript(messages)  // 进度也要写 transcript
  }
  yield* normalizeMessage(message)
  break
```

#### `user`（工具结果、中断消息）

```ts
case 'user':
  this.mutableMessages.push(message)
  yield* normalizeMessage(message)
  turnCount++
  break
```

#### `stream_event`（API 流事件）

用于内部追踪 token 使用：

```ts
case 'stream_event':
  if (message.event.type === 'message_start') {
    currentMessageUsage = updateUsage(currentMessageUsage, message.event.message.usage)
  }
  if (message.event.type === 'message_delta') {
    currentMessageUsage = updateUsage(currentMessageUsage, message.event.usage)
    if (message.event.delta.stop_reason != null) {
      lastStopReason = message.event.delta.stop_reason
    }
  }
  if (message.event.type === 'message_stop') {
    this.totalUsage = accumulateUsage(this.totalUsage, currentMessageUsage)
  }
  if (includePartialMessages) {
    yield { type: 'stream_event', event: message.event, ... }
  }
  break
```

#### `attachment`（附件消息）

```ts
case 'attachment':
  this.mutableMessages.push(message)
  if (persistSession) {
    messages.push(message)
    void recordTranscript(messages)
  }
  
  // 子类型：structured_output / max_turns_reached / queued_command
  if (message.attachment.type === 'structured_output') {
    structuredOutputFromTool = message.attachment.data
  } else if (message.attachment.type === 'max_turns_reached') {
    // 直接 yield error_max_turns result 并 return
  }
  break
```

#### `system`（compact_boundary / api_error / snip boundary）

```ts
case 'system':
  // 处理 snipReplay（HISTORY_SNIP 特性）
  const snipResult = this.config.snipReplay?.(message, this.mutableMessages)
  if (snipResult !== undefined) {
    this.mutableMessages = snipResult.messages
    break
  }
  
  this.mutableMessages.push(message)
  
  if (message.subtype === 'compact_boundary') {
    // 释放压缩前的消息，让 GC 回收
    this.mutableMessages.splice(0, mutableBoundaryIdx)
    messages.splice(0, localBoundaryIdx)
    
    yield { type: 'system', subtype: 'compact_boundary', compact_metadata: ... }
  }
  if (message.subtype === 'api_error') {
    yield { type: 'system', subtype: 'api_retry', attempt: ..., error: ... }
  }
  break
```

### 阶段 6：终止条件检查

在消费 `query()` 的过程中，QueryEngine 会实时监控几个终止条件：

#### 预算超限

```ts
if (maxBudgetUsd !== undefined && getTotalCost() >= maxBudgetUsd) {
  yield {
    type: 'result',
    subtype: 'error_max_budget_usd',
    is_error: true,
    errors: [`Reached maximum budget ($${maxBudgetUsd})`],
    ...
  }
  return
}
```

#### Structured Output 重试超限

```ts
if (message.type === 'user' && jsonSchema) {
  const callsThisQuery = countToolCalls(this.mutableMessages, SYNTHETIC_OUTPUT_TOOL_NAME) - initialStructuredOutputCalls
  if (callsThisQuery >= maxRetries) {
    yield { type: 'result', subtype: 'error_max_structured_output_retries', ... }
    return
  }
}
```

### 阶段 7：Loop 结束后组装 Result

`query()` 完成后，QueryEngine 需要判断这次 turn 是否"成功"，并提取最终结果文本：

```ts
const result = messages.findLast(m => m.type === 'assistant' || m.type === 'user')

if (!isResultSuccessful(result, lastStopReason)) {
  yield {
    type: 'result',
    subtype: 'error_during_execution',
    is_error: true,
    errors: [
      `[ede_diagnostic] result_type=${edeResultType} last_content_type=${edeLastContentType} stop_reason=${lastStopReason}`,
      ...allErrors.slice(start).map(_ => _.error),
    ],
    ...
  }
  return
}
```

#### 成功路径

```ts
let textResult = ''
if (result.type === 'assistant') {
  const lastContent = last(result.message.content)
  if (lastContent?.type === 'text' && !SYNTHETIC_MESSAGES.has(lastContent.text)) {
    textResult = lastContent.text
  }
}

yield {
  type: 'result',
  subtype: 'success',
  is_error: isApiError,
  result: textResult,
  stop_reason: lastStopReason,
  total_cost_usd: getTotalCost(),
  usage: this.totalUsage,
  modelUsage: getModelUsage(),
  permission_denials: this.permissionDenials,
  structured_output: structuredOutputFromTool,
  fast_mode_state: getFastModeState(mainLoopModel, initialAppState.fastMode),
  uuid: randomUUID(),
}
```

---

## 三、`ask()` —— QueryEngine 的便捷包装

对于只想发一条 prompt、不需要长期会话管理的 SDK 调用方，`ask()` 是一个便利函数：

```ts
export async function* ask({ commands, prompt, cwd, tools, ... }): AsyncGenerator<SDKMessage> {
  const engine = new QueryEngine({
    cwd,
    tools,
    commands,
    // ... 其他配置
    initialMessages: mutableMessages,
    readFileCache: cloneFileStateCache(getReadFileCache()),
    snipReplay: feature('HISTORY_SNIP')
      ? (yielded, store) => snipModule!.snipCompactIfNeeded(store, { force: true })
      : undefined,
  })

  try {
    yield* engine.submitMessage(prompt, { uuid: promptUuid, isMeta })
  } finally {
    setReadFileCache(engine.getReadFileState())  // 把读文件缓存回传给调用方
  }
}
```

**设计要点**：
- `initialMessages` 让调用方可以维护跨 turn 的消息历史
- `finally` 里回传 `readFileState`，确保下次 `ask()` 能延续文件的"已读"状态

---

## 四、QueryEngine 的 4 个核心设计智慧

### 1. **在 API 调用前写 transcript = 可恢复性第一**
用户的 `UserMessage` 一旦产生，立刻持久化。这是 `--resume`、崩溃恢复、进程重启后对话连续性的基石。

### 2. **Fire-and-forget vs Await 的精细权衡**
- `assistant` 消息：`void recordTranscript(messages)` —— 不阻塞流式输出
- `user/attachment/progress` 消息：`await recordTranscript(messages)` —— 保证顺序正确
- 这是性能与一致性的微妙平衡

### 3. **`mutableMessages` 与 `messages` 双缓冲**
`mutableMessages` 是引擎的内部状态，`messages` 是传给 `query()` 的局部快照。这种分离让 slash command 可以在提交前修改历史，而不污染正在运行的 loop。

### 4. **所有边界条件都在 `submitMessage` 层处理**
`query.ts` 只管"把对话跑完"，但：
- 预算超了？`QueryEngine` 截断
- max turns 到了？`QueryEngine` 截断
- structured output 重试过多？`QueryEngine` 截断

这样 `query.ts` 保持纯粹，`QueryEngine` 负责产品策略。

---

## 五、关键代码索引

| 概念 | 文件 | 关键函数/类 |
|------|------|------------|
| QueryEngine 主类 | `src/QueryEngine.ts` | `QueryEngine`, `submitMessage()` |
| 便捷包装 | `src/QueryEngine.ts` | `ask()` |
| 用户输入处理 | `src/utils/processUserInput/processUserInput.ts` | `processUserInput()` |
| 斜杠命令处理 | `src/utils/processUserInput/processSlashCommand.ts` | `processSlashCommand()` |
| 普通 prompt | `src/utils/processUserInput/processTextPrompt.ts` | `processTextPrompt()` |
| System Prompt 组装 | `src/utils/queryContext.ts` | `fetchSystemPromptParts()` |
| Transcript 持久化 | `src/utils/sessionStorage.ts` | `recordTranscript()`, `flushSessionStorage()` |
| 消息标准化 | `src/utils/queryHelpers.ts` | `normalizeMessage()` |

---

*文档生成时间：2026-04-14*  
*基于源码版本：Claude Code v2.1.88*
