# Claude Code 源码学习：Agent Loop 核心架构 (`src/query.ts`)

> 本系列文档是对 `Claude Code` v2.1.88 完整 TypeScript 源码的逐层拆解，目标是理解业界最顶尖的 AI 编程助手是如何从"简单 Agent 循环"演化为"生产级系统"的。

---

## 一、一句话概括

`src/query.ts` 实现了一个**带状态机的异步生成器循环**：

> 不断地调用 Claude API → 流式接收响应 → 如果有 `tool_use` 就并行执行工具 → 把结果追加回对话历史 → 再调 API → 直到模型直接输出文本为止。

这个模式本身非常简单，但 Claude Code 在其外部包裹了极其厚重的 **production harness**：权限控制、上下文压缩、Token 预算、子 Agent 递归、Hook 系统、错误自愈等。

---

## 二、文件定位

| 属性 | 内容 |
|------|------|
| 文件 | `src/query.ts` |
| 行数 | ~1,729 行（全项目最大单文件之一） |
| 核心函数 | `query()` → `queryLoop()` |
| 设计模式 | `async function*` 生成器 + 不可变状态机 |

---

## 三、代码结构总览

```ts
// 对外暴露的入口：一个异步生成器
export async function* query(params: QueryParams): AsyncGenerator<...> {
  const terminal = yield* queryLoop(params, consumedCommandUuids)
  return terminal
}

// 真正的循环体
async function* queryLoop(
  params: QueryParams,
  consumedCommandUuids: string[],
): AsyncGenerator<..., Terminal> {
  let state: State = {
    messages: params.messages,
    toolUseContext: params.toolUseContext,
    turnCount: 1,
    // ... 其他循环状态
  }

  while (true) {
    // Stage 1: 上下文预处理（压缩、裁剪、token 检查）
    // Stage 2: 流式调用模型 API
    // Stage 3: 解析响应（text / tool_use / error）
    // Stage 4: 分支判断（需要 follow-up 还是直接结束？）
    // Stage 5: 工具执行（并行/串行）
    // Stage 6: 附件注入（memory / skill / 排队命令）
    // Stage 7: 状态更新 & 递归进入下一轮
  }
}
```

---

## 四、一次完整循环迭代的 8 个阶段

```
┌─────────────────────────────────────────────────────────────────────┐
│  Stage 0: 进入循环，解构 state                                       │
│  messages, toolUseContext, autoCompactTracking, turnCount...         │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Stage 1: 上下文预处理（Critical Path）                              │
│  - applyToolResultBudget: 限制单个工具结果长度                       │
│  - snipCompact: 裁剪历史（HISTORY_SNIP）                             │
│  - microcompact: 微压缩（CACHED_MICROCOMPACT）                       │
│  - contextCollapse: 上下文折叠（CONTEXT_COLLAPSE）                   │
│  - autocompact: 自动上下文压缩（Token 超标时触发）                    │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Stage 2: 调用模型 API（流式）                                       │
│  deps.callModel({ messages, systemPrompt, tools, ... })              │
│  → for await (const message of stream) { ... }                       │
└─────────────────────────────────────────────────────────────────────┘
                                │
                    ┌───────────┴───────────┐
                    ▼                       ▼
              收到 text block           收到 tool_use block
                    │                       │
                    │                       ▼
                    │              StreamingToolExecutor.addTool()
                    │              → 后台开始并行执行工具
                    │                       │
                    ▼                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Stage 3: 流式输出处理                                               │
│  - yield 给 UI 层显示（实时打字效果）                                 │
│  - 收集 assistantMessages                                            │
│  - 检测 streaming fallback（模型降级重试）                           │
│  - 检测并 withhold（暂存）可恢复错误：PTL / max_output_tokens        │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Stage 4: API 调用结束后的分支判断                                   │
│  needsFollowUp = (是否有 tool_use ?)                                 │
│                                                                      │
│  IF needsFollowUp == false:                                          │
│    → 处理错误恢复（reactive compact, max_tokens recovery）           │
│    → 执行 stop hooks                                                 │
│    → 检查 token budget                                               │
│    → return { reason: 'completed' }                                  │
│                                                                      │
│  ELSE (有 tool_use):                                                 │
│    → 进入 Stage 5                                                    │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Stage 5: 工具执行                                                   │
│  IF streamingToolExecutor:                                           │
│    → 消费 getRemainingResults()（流式并行执行的结果）                │
│  ELSE:                                                               │
│    → runTools(toolUseBlocks, ...)（批量串行/并行执行）               │
│                                                                      │
│  收集 toolResults（UserMessage/AttachmentMessage）                   │
│  更新 toolUseContext（如文件状态缓存）                               │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Stage 6: 附件注入                                                   │
│  - getAttachmentMessages(): 注入图片、文件变更通知、排队命令等        │
│  - pendingMemoryPrefetch: 注入 CLAUDE.md 记忆                        │
│  - skillPrefetch: 注入发现的 Skill                                   │
│  - refreshTools(): MCP 服务器新连接后刷新可用工具                    │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Stage 7: 状态更新 & 递归                                            │
│  nextState.messages = [                                              │
│    ...messagesForQuery,                                              │
│    ...assistantMessages,  // 模型的 tool_use 请求                    │
│    ...toolResults        // 工具执行结果                             │
│  ]                                                                   │
│  state = nextState                                                   │
│  continue   // 回到 while(true) 下一轮                               │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 五、5 个你必须理解的超关键机制

### 1️⃣ Streaming Tool Execution — 工具"边流边执行"

**问题**：传统实现是等模型完整输出完所有 `tool_use` 块，再开始执行工具。这意味着用户要白白等待模型"说完"所有计划。

**Claude Code 的解法**：

```ts
// 在流式接收过程中，一旦检测到 tool_use 块：
for (const toolBlock of msgToolUseBlocks) {
  streamingToolExecutor.addTool(toolBlock, message)  // 立刻入队执行！
}
```

`StreamingToolExecutor` 会**在模型还在打字的时候就提前并行执行工具**。当模型说完最后一个字时，很多工具已经执行完了。

```ts
// API 流结束后：
const toolUpdates = streamingToolExecutor
  ? streamingToolExecutor.getRemainingResults()   // 拿剩余结果
  : runTools(...)                                 // 老路径：非流式批量执行
```

**学习点**：性能优化不要只盯着算法复杂度，**并行度**和**流水线重叠**往往带来更大的用户体验提升。

---

### 2️⃣ 上下文压缩三剑客（+ 两路急救）

Token 超限是长会话的头号杀手。Claude Code 有多层防御：

| 机制 | 代码位置 | 作用 | 触发时机 |
|------|---------|------|---------|
| **Snip** | `snipCompact.ts` | 删除历史中间部分，只保留开头+结尾 | 编译时开关 `HISTORY_SNIP` |
| **Microcompact** | `compact.ts` | 轻量级压缩，保留最近对话 | 每轮查询前 |
| **Autocompact** | `autoCompact.ts` | 主动调用侧模型总结长历史 | Token 达到阈值 |
| **Reactive Compact** | `reactiveCompact.ts` | API 返回 413 后的**被动急救** | 收到 PTL 错误 |
| **Context Collapse** | `contextCollapse/` | 按主题折叠对话为新视角 | 编译时开关 `CONTEXT_COLLAPSE` |

**主动 vs 被动**：
- **主动（Proactive）**：每轮开始前检查 token 数，超标就 `autocompact`。
- **被动（Reactive）**：如果还是 413 了，流式循环会 **withhold（暂扣）** 错误消息，然后尝试 `reactive compact`（紧急总结），成功后再自动 retry 同一轮。

```ts
// 流中检测到 PTL 时先 withhold 不显示
if (reactiveCompact?.isWithheldPromptTooLong(message)) {
  withheld = true
}

// 流结束后尝试恢复
if (isWithheld413 && reactiveCompact) {
  const compacted = await reactiveCompact.tryReactiveCompact({...})
  if (compacted) {
    state = next  // 带着压缩后的消息 continue
    continue
  }
}
```

**学习点**：错误恢复要分层。不要一有错误就抛给用户，**能自愈的先自愈**。

---

### 3️⃣ Model Fallback — 模型降级重试

如果当前模型（如 Claude 4 Opus）高负载不可用，API 会触发 fallback：

```ts
catch (innerError) {
  if (innerError instanceof FallbackTriggeredError && fallbackModel) {
    currentModel = fallbackModel
    attemptWithFallback = true

    // 清空本轮已收集的消息
    assistantMessages.length = 0
    toolResults.length = 0
    toolUseBlocks.length = 0

    // 给 UI 一个系统提示
    yield createSystemMessage(
      `Switched to ${renderModelName(fallbackModel)} due to high demand...`
    )
    continue  // 重试
  }
}
```

这是一个**无缝的用户体验设计**：用户完全无感知，只是看到一条"已切换模型"的提示，对话继续。

---

### 4️⃣ Max Output Tokens Recovery — 输出长度超限时自我恢复

如果模型输出到一半被截断（`max_output_tokens`）：

```ts
if (isWithheldMaxOutputTokens(lastMessage)) {
  // 第一次：尝试把输出上限从 8k 提升到 64k
  if (maxOutputTokensOverride === undefined) {
    state = { ..., maxOutputTokensOverride: ESCALATED_MAX_TOKENS }
    continue
  }

  // 后续：最多重试 3 次，每次注入一条"继续写"的 meta message
  if (maxOutputTokensRecoveryCount < MAX_OUTPUT_TOKENS_RECOVERY_LIMIT) {
    const recoveryMessage = createUserMessage({
      content: `Output token limit hit. Resume directly — no apology, ` +
               `no recap of what you were doing. Pick up mid-thought...`,
      isMeta: true,
    })
    state = { ..., messages: [..., recoveryMessage] }
    continue
  }
}
```

这就是你有时看到 Claude 说"我继续"的底层原因——它在自动做**断点续传**。

---

### 5️⃣ Stop Hooks — 每轮结束前的"刹车片"

当模型没有调用工具、直接回复文本时，loop 不会立刻退出，它会先跑 **stop hooks**：

```ts
const stopHookResult = yield* handleStopHooks(
  messagesForQuery,
  assistantMessages,
  systemPrompt,
  userContext,
  systemContext,
  toolUseContext,
  querySource,
  stopHookActive,
)

if (stopHookResult.preventContinuation) {
  return { reason: 'stop_hook_prevented' }
}

if (stopHookResult.blockingErrors.length > 0) {
  // 把 hook 的阻塞错误注入 messages，继续下一轮
  state = { ..., messages: [..., ...blockingErrors] }
  continue
}
```

**Stop hooks 的典型用途**：
- **计划模式（Plan Mode）**：如果用户还没批准计划，hook 会阻塞并返回错误，迫使模型停下来等待用户确认
- **后置验证**：检查某些业务规则是否满足
- **安全拦截**：在关键操作前做二次确认

**学习点**：不要把"流程控制"全部交给模型靠 prompt 自律，**显式的 hook/闸门机制**更可靠。

---

## 六、状态机设计：`State` 与 `transition`

这是 `query.ts` 里最优雅的工程设计之一。整个循环用**不可变状态更新**代替大量零散的 `let` 赋值：

```ts
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined  // ← 记录"为什么进入下一轮"
}
```

每次 `continue` 都构造一个全新的 `next: State`：

```ts
const next: State = {
  messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
  toolUseContext: toolUseContextWithQueryTracking,
  autoCompactTracking: tracking,
  turnCount: nextTurnCount,
  maxOutputTokensRecoveryCount: 0,
  hasAttemptedReactiveCompact: false,
  pendingToolUseSummary: nextPendingToolUseSummary,
  maxOutputTokensOverride: undefined,
  stopHookActive,
  transition: { reason: 'next_turn' },
}
state = next
continue
```

### 好处

1. **可测试性**：单元测试可以断言 `transition.reason` 来判断走了哪条分支
2. **可调试性**：每次状态变更都有明确的 reason
3. **函数式风格**：避免在循环体里到处修改变量

### transition reason 枚举

| reason | 含义 |
|--------|------|
| `next_turn` | 正常进入下一轮（有 tool results） |
| `collapse_drain_retry` | Context collapse 触发后重试 |
| `reactive_compact_retry` | Reactive compact 触发后重试 |
| `max_output_tokens_escalate` | 把输出上限从 8k 提升到 64k |
| `max_output_tokens_recovery` | 输出截断后注入恢复消息重试 |
| `stop_hook_blocking` | stop hook 返回阻塞错误，继续追问 |
| `token_budget_continuation` | Token 预算未耗尽，自动继续 |

---

## 七、从 `query.ts` 学到的 3 个工程智慧

### 1. "Generator + 状态机" 是复杂 Agent Loop 的最佳拍档
- `async function*` 让 loop 可以一边执行一边 `yield` 事件给 UI
- 状态机让 7 种 continue 路径清晰可控，不会变成意大利面条代码

### 2. 错误恢复要分层
- 流内暂扣（withhold）→ 流后分级恢复（collapse drain → reactive compact → escalation）
- **不要让错误直接炸给用户，能自愈的先自愈**

### 3. 性能藏在并行度里
- Streaming Tool Executor：模型还在输出时就开始执行工具
- Memory prefetch / skill prefetch / task summary：全部后台异步，不阻塞主 loop
- 用户体验的"快"往往不是代码运行快，而是**等待时间被重叠利用了**

---

## 八、关键代码索引（快速查找）

| 概念 | 文件 | 关键函数/类 |
|------|------|------------|
| Agent Loop 入口 | `src/query.ts` | `query()`, `queryLoop()` |
| 流式工具执行 | `src/services/tools/StreamingToolExecutor.ts` | `StreamingToolExecutor` |
| 工具调度 | `src/services/tools/toolOrchestration.ts` | `runTools()` |
| 自动上下文压缩 | `src/services/compact/autoCompact.ts` | `autocompact()` |
| 被动压缩急救 | `src/services/compact/reactiveCompact.ts` | `tryReactiveCompact()` |
| 上下文折叠 | `src/services/contextCollapse/index.ts` | `applyCollapsesIfNeeded()` |
| Stop Hooks | `src/query/stopHooks.ts` | `handleStopHooks()` |
| Token 预算 | `src/query/tokenBudget.ts` | `checkTokenBudget()` |
| API 调用封装 | `src/services/api/claude.ts` | `queryModelWithStreaming()` |

---

## 九、延伸阅读预告

- **Part 2**: Tool 系统 — `BashTool` / `AgentTool` / `FileEditTool`
- **Part 3**: QueryEngine — 一次用户输入如何变成 API 请求
- **Part 4**: UI 层 — React + Ink 在终端里渲染世界
- **Part 5**: 权限与安全 — `useCanUseTool` 和权限规则引擎

---

*文档生成时间：2026-04-14*  
*基于源码版本：Claude Code v2.1.88*
