# Claude Code 工具调用管线深度分析

> 基于源码 `src/services/tools/toolExecution.ts`、`src/services/tools/toolHooks.ts`、`src/Tool.ts` 的逐行解读

---

## 全局流程图

```
Claude API 返回 tool_use block
        │
        ▼
┌─────────────────────────────────┐
│  1. findToolByName()             │  按名称查找工具
│     找不到 → 返回错误给 Claude    │
└──────────┬──────────────────────┘
           │
           ▼
┌─────────────────────────────────┐
│  2. inputSchema.safeParse()      │  Zod schema 验证
│     失败 → 返回 InputValidationError │
└──────────┬──────────────────────┘
           │
           ▼
┌─────────────────────────────────┐
│  3. tool.validateInput()         │  工具自定义验证
│     (例如 Edit 检查文件是否读过)   │
│     失败 → 返回具体错误信息        │
└──────────┬──────────────────────┘
           │
           ▼
┌─────────────────────────────────┐
│  4. backfillObservableInput()    │  填充派生字段
│     (例如 expandPath 展开路径)    │
└──────────┬──────────────────────┘
           │
           ▼
┌─────────────────────────────────┐
│  5. runPreToolUseHooks()         │  执行前置 Hooks
│     hook 可修改 input / 拒绝执行  │
│     可返回 stop → 终止执行        │
└──────────┬──────────────────────┘
           │
           ▼
┌─────────────────────────────────┐
│  6. resolveHookPermissionDecision│  合并 Hook 权限决策
│     hook 已决定 → 使用 hook 结果  │
│     hook 未决定 → 进入正常流程    │
└──────────┬──────────────────────┘
           │
           ▼
┌─────────────────────────────────┐
│  7. canUseTool()                 │  用户权限检查
│     ├─ allow → 直接执行          │
│     ├─ deny → 返回拒绝信息       │
│     └─ ask → 弹窗让用户确认       │
│         用户允许 → 执行           │
│         用户拒绝 → 返回拒绝信息    │
└──────────┬──────────────────────┘
           │
           ▼
┌─────────────────────────────────┐
│  8. tool.call()                  │  实际执行工具
│     ┌───────────────────────┐    │
│     │ onProgress() 回调      │    │  实时进度报告
│     │ (终端输出/文件读取等)   │    │
│     └───────────────────────┘    │
└──────────┬──────────────────────┘
           │
           ▼
┌─────────────────────────────────┐
│  9. runPostToolUseHooks()        │  执行后置 Hooks
└──────────┬──────────────────────┘
           │
           ▼
┌─────────────────────────────────┐
│ 10. 返回 ToolResult              │
│     ├─ tool_result block          │  正常结果
│     │  (文件内容/diff/命令输出)   │
│     ├─ newMessages               │  附带消息 (Skill)
│     └─ contextModifier           │  修改上下文 (工具白名单等)
└─────────────────────────────────┘
```

---

## Step 1: findToolByName() — 工具查找

### 源码位置
- 定义：`src/Tool.ts:358-360`
- 调用：`src/services/tools/toolExecution.ts:345-356`

### 源码实现
```typescript
// src/Tool.ts
export function findToolByName(tools: Tools, name: string): Tool | undefined {
  return tools.find(t => toolMatchesName(t, name))
}

function toolMatchesName(
  tool: { name: string; aliases?: string[] },
  name: string,
): boolean {
  return tool.name === name || (tool.aliases?.includes(name) ?? false)
}
```

### 调用逻辑
```typescript
// toolExecution.ts runToolUse()
let tool = findToolByName(toolUseContext.options.tools, toolName)

// 如果找不到，检查是否是别名调用的已废弃工具
if (!tool) {
  const fallbackTool = findToolByName(getAllBaseTools(), toolName)
  if (fallbackTool && fallbackTool.aliases?.includes(toolName)) {
    tool = fallbackTool
  }
}
```

### 关键设计
| 设计点 | 说明 |
|--------|------|
| **双重查找** | 先在当前可用工具列表中查找，再在全局基础工具中查找 |
| **别名机制** | 工具重命名后保留旧名称作为 alias，兼容旧会话 |
| **错误处理** | 找不到工具时返回 `No such tool available` 错误给 Claude |
| **MCP 工具** | `mcp__` 前缀的工具名通过 `mcpInfoFromString()` 解析服务器名 |

### 错误返回格式
```xml
<tool_use_error>Error: No such tool available: {toolName}</tool_use_error>
```

---

## Step 2: inputSchema.safeParse() — Zod Schema 验证

### 源码位置
`src/services/tools/toolExecution.ts:615-680`

### 源码实现
```typescript
const parsedInput = tool.inputSchema.safeParse(input)
if (!parsedInput.success) {
  let errorContent = formatZodValidationError(tool.name, parsedInput.error)

  // 如果是延迟加载工具（schema 未发送到 API），追加提示
  const schemaHint = buildSchemaNotSentHint(tool, ...)
  if (schemaHint) errorContent += schemaHint

  return [{
    message: createUserMessage({
      content: [{
        type: 'tool_result',
        content: `<tool_use_error>InputValidationError: ${errorContent}</tool_use_error>`,
        is_error: true,
        tool_use_id: toolUseID,
      }],
    }),
  }]
}
```

### 关键设计
| 设计点 | 说明 |
|--------|------|
| **Zod v4** | 使用 `zod/v4` 的 `safeParse`，不抛异常而是返回 `{success, error}` |
| **延迟工具提示** | 如果工具 schema 未发送到 API（ToolSearch 机制），提示模型先调用 `ToolSearch` |
| **类型安全** | 每个工具定义自己的 `inputSchema`，如 FileEditTool 使用 `inputSchema()` 函数 |
| **错误格式化** | `formatZodValidationError()` 将 Zod 错误转为人类可读的消息 |

### ToolSearch 提示机制
当工具是 deferred（延迟加载）且模型没有先调用 ToolSearch 时：
```
This tool's schema was not sent to the API — it was not in the discovered-tool set.
Without the schema, typed parameters get emitted as strings and the parser rejects them.
Load the tool first: call ToolSearch with query "select:{toolName}", then retry.
```

---

## Step 3: tool.validateInput() — 工具自定义验证

### 源码位置
- 定义：`src/Tool.ts:489-492`
- 调用：`src/services/tools/toolExecution.ts:683-733`

### 源码实现
```typescript
// Tool.ts 接口定义
validateInput?(
  input: z.infer<Input>,
  context: ToolUseContext,
): Promise<ValidationResult>

// ValidationResult 类型
type ValidationResult =
  | { result: true }
  | { result: false; message: string; errorCode: number }
```

### FileEditTool 的 validateInput 示例
```typescript
// FileEditTool.ts
async validateInput(input: FileEditInput, toolUseContext: ToolUseContext) {
  const { file_path, old_string, new_string, replace_all = false } = input
  const fullFilePath = expandPath(file_path)

  // 1. 检查是否在团队记忆文件中引入 secrets
  const secretError = checkTeamMemSecrets(fullFilePath, new_string)
  if (secretError) return { result: false, message: secretError, errorCode: 0 }

  // 2. old_string === new_string → 无意义操作
  if (old_string === new_string) {
    return { result: false, message: 'No changes to make', errorCode: 1 }
  }

  // 3. 检查权限设置中的 deny 规则
  const denyRule = matchingRuleForInput(fullFilePath, ...)
  if (denyRule !== null) {
    return { result: false, message: 'File denied by permission settings', errorCode: 2 }
  }

  // 4. 安全检查：UNC 路径防止 NTLM 凭据泄露（Windows）
  if (fullFilePath.startsWith('\\\\') || fullFilePath.startsWith('//')) {
    // 跳过文件系统操作，交给权限检查处理
  }

  // 5. 检查文件是否被读过（必须先 Read 再 Edit）
  // 6. 检查文件是否被外部修改
  // ...
}
```

### 各工具的典型验证逻辑

| 工具 | 验证内容 |
|------|----------|
| **FileEditTool** | 文件是否读过、是否被外部修改、路径 deny 规则、secrets 泄露、UNC 安全 |
| **FileWriteTool** | 路径权限、文件大小限制 |
| **FileReadTool** | 文件是否存在、大小限制 |
| **BashTool** | 危险命令检测、沙盒检查 |

---

## Step 4: backfillObservableInput() — 派生字段填充

### 源码位置
- 定义：`src/Tool.ts:481`
- 调用：`src/services/tools/toolExecution.ts:783-793`

### 源码实现
```typescript
// Tool.ts 接口定义
backfillObservableInput?(input: Record<string, unknown>): void

// toolExecution.ts 调用
let callInput = processedInput
const backfilledClone =
  tool.backfillObservableInput &&
  typeof processedInput === 'object' &&
  processedInput !== null
    ? ({ ...processedInput } as typeof processedInput)  // 浅拷贝
    : null
if (backfilledClone) {
  tool.backfillObservableInput!(backfilledClone as Record<string, unknown>)
  processedInput = backfilledClone
}
```

### 关键设计
| 设计点 | 说明 |
|--------|------|
| **浅拷贝** | 对输入做浅拷贝后再修改，不污染原始数据 |
| **幂等性** | 文档要求必须幂等（`Must be idempotent`） |
| **不影响 call()** | `callInput` 保留原始值，`processedInput` 给 hooks/权限系统看 |
| **路径展开** | FileEditTool/FileWriteTool 将 `~` 或相对路径展开为绝对路径 |

### FileEditTool 的 backfill 示例
```typescript
// FileEditTool.ts
backfillObservableInput(input) {
  if (typeof input.file_path === 'string') {
    input.file_path = expandPath(input.file_path)  // ~/foo → /home/user/foo
  }
}
```

### 为什么要分开 callInput 和 processedInput？
> hooks/canUseTool 需要看到展开后的绝对路径来匹配 allowlist 规则，但 `tool.call()` 的结果中嵌入路径（如 "File created at: ~/foo"）需要保留模型原始值，否则会改变 transcript 和 VCR fixture hash。

---

## Step 5: runPreToolUseHooks() — 前置 Hook 执行

### 源码位置
`src/services/tools/toolHooks.ts:435-650`

### 源码实现
```typescript
export async function* runPreToolUseHooks(
  toolUseContext: ToolUseContext,
  tool: Tool,
  processedInput: Record<string, unknown>,
  toolUseID: string,
  ...
): AsyncGenerator<
  | { type: 'message'; message: ... }
  | { type: 'hookPermissionResult'; hookPermissionResult: PermissionResult }
  | { type: 'hookUpdatedInput'; updatedInput: Record<string, unknown> }
  | { type: 'preventContinuation'; shouldPreventContinuation: boolean }
  | { type: 'stopReason'; stopReason: string }
  | { type: 'additionalContext'; message: ... }
  | { type: 'stop' }  // 终止执行
> { ... }
```

### Hook 返回的事件类型

| 事件类型 | 含义 | 后续行为 |
|----------|------|----------|
| `message` | Hook 产生的进度/日志消息 | 推入消息队列 |
| `hookPermissionResult` | Hook 做出了权限决策（allow/deny/ask） | 传递给 resolveHookPermissionDecision |
| `hookUpdatedInput` | Hook 修改了输入但未做权限决策 | 更新 processedInput |
| `preventContinuation` | Hook 要求阻止后续执行 | 设置标志位，工具执行后停止 |
| `stopReason` | 停止原因描述 | 配合 preventContinuation 使用 |
| `additionalContext` | Hook 提供的额外上下文 | 推入消息队列 |
| `stop` | 立即终止执行 | 直接 return |

### 慢 Hook 日志
```typescript
const SLOW_PHASE_LOG_THRESHOLD_MS = 2000
if (preToolHookDurationMs >= SLOW_PHASE_LOG_THRESHOLD_MS) {
  logForDebugging(`Slow PreToolUse hooks: ${preToolHookDurationMs}ms for ${tool.name}`)
}
```

---

## Step 6: resolveHookPermissionDecision() — 合并 Hook 权限决策

### 源码位置
`src/services/tools/toolHooks.ts:332-433`

### 核心逻辑

```typescript
export async function resolveHookPermissionDecision(
  hookPermissionResult: PermissionResult | undefined,
  tool: Tool,
  input: Record<string, unknown>,
  toolUseContext: ToolUseContext,
  canUseTool: CanUseToolFn,
  ...
): Promise<{ decision: PermissionDecision; input: Record<string, unknown> }>
```

### 决策矩阵

```
hookPermissionResult.behavior
    │
    ├─ "allow" ──────────────────────────────────────────────┐
    │   ├─ 工具需要用户交互且 hook 未提供 updatedInput?       │
    │   │   └─ 是 → 调用 canUseTool()                        │
    │   ├─ requireCanUseTool 为 true?                        │
    │   │   └─ 是 → 调用 canUseTool()                        │
    │   ├─ checkRuleBasedPermissions() 检查 deny/ask 规则     │
    │   │   ├─ null → 直接允许（hook allow 生效）             │
    │   │   ├─ deny → 规则覆盖 hook，拒绝                    │
    │   │   └─ ask → 规则要求弹窗，调用 canUseTool()          │
    │   └─ 否则 → hook allow 直接生效                        │
    │                                                          │
    ├─ "deny" ──────────────────────────────────────────────│
    │   └─ 直接拒绝，返回 hook 的拒绝消息                     │
    │                                                          │
    └─ "ask" / undefined ──────────────────────────────────│
        └─ 调用 canUseTool()，传入 forceDecision（如 ask）   │
```

### 关键不变量
> **Hook 的 `allow` 不会绕过 `settings.json` 中的 `deny/ask` 规则**——`checkRuleBasedPermissions()` 始终会执行。这是安全防线。

---

## Step 7: canUseTool() — 用户权限检查

### 源码位置
`src/hooks/useCanUseTool.tsx:28-100`

### 权限流程

```typescript
// hasPermissionsToUseTool() 返回三种行为
const result = await hasPermissionsToUseTool(tool, input, toolUseContext, ...)

switch (result.behavior) {
  case "allow":
    // 自动允许，记录日志
    resolve(ctx.buildAllow(result.updatedInput ?? input, { decisionReason }))
    break

  case "deny":
    // 自动拒绝（配置规则或分类器决定）
    resolve(result)
    break

  case "ask":
    // 需要用户确认 → 根据模式走不同 handler
    if (coordinator模式) {
      handleCoordinatorPermission(...)
    } else if (swarm worker) {
      handleSwarmWorkerPermission(...)
    } else {
      handleInteractivePermission(...)  // 弹出权限对话框
    }
    break
}
```

### 三种权限处理器

| 处理器 | 场景 | 行为 |
|--------|------|------|
| `handleInteractivePermission` | 交互式 REPL | 弹出终端 UI 确认对话框 |
| `handleCoordinatorPermission` | 协调器模式 | 等待分类器结果后自动决定 |
| `handleSwarmWorkerPermission` | Swarm 工作节点 | 自动拒绝（后台 agent 无 UI） |

### BashTool 的投机分类器
```typescript
// 对于 BashTool，提前启动分类器检查（与 hooks 并行执行）
if (tool.name === BASH_TOOL_NAME && 'command' in parsedInput.data) {
  startSpeculativeClassifierCheck(
    parsedInput.data.command,
    appState.toolPermissionContext,
    toolUseContext.abortController.signal,
    toolUseContext.options.isNonInteractiveSession,
  )
}
```

---

## Step 8: tool.call() — 工具实际执行

### 源码位置
`src/services/tools/toolExecution.ts:1207-1222`

### 源码实现
```typescript
const result = await tool.call(
  callInput,                        // 输入参数（可能经 hook 修改）
  {
    ...toolUseContext,               // 工具使用上下文
    toolUseId: toolUseID,
    userModified: permissionDecision.userModified ?? false,
  },
  canUseTool,                       // 权限检查函数（工具内部可能调用子工具）
  assistantMessage,                 // 父消息
  progress => {
    onToolProgress({                // 进度回调
      toolUseID: progress.toolUseID,
      data: progress.data,
    })
  },
)
```

### callInput 的智能选择
```typescript
// 如果 hook/权限返回了新 input，使用新 input
if (processedInput !== backfilledClone) {
  callInput = processedInput
}
// 如果 file_path 被展开但 hook 未修改，恢复模型原始路径
else if (backfilledClone && processedInput.file_path === backfilledClone.file_path) {
  callInput = { ...processedInput, file_path: callInput.file_path }
}
```

### Tool 接口的 call 签名
```typescript
// src/Tool.ts
call(
  args: z.infer<Input>,
  context: ToolUseContext,
  canUseTool: CanUseToolFn,           // 子工具权限检查
  parentMessage: AssistantMessage,
  onProgress?: ToolCallProgress<P>,   // 进度回调
): Promise<ToolResult<Output>>
```

### ToolResult 结构
```typescript
type ToolResult<T> = {
  data: T                               // 工具输出数据
  newMessages?: Message[]               // 附带消息（Skill 工具展开技能提示词）
  contextModifier?: (ctx) => ctx        // 修改后续上下文（白名单等）
  mcpMeta?: {                           // MCP 协议元数据
    _meta?: Record<string, unknown>
    structuredContent?: Record<string, unknown>
  }
}
```

### 进度报告机制
```
tool.call() 内部
    │
    ├─ onProgress({ toolUseID, data: BashProgress })
    │   └─ terminal output, stderr, exit code
    │
    ├─ onProgress({ toolUseID, data: FileReadProgress })
    │   └─ file content chunks
    │
    └─ onProgress({ toolUseID, data: HookProgress })
        └─ hook execution status
```

---

## Step 9: runPostToolUseHooks() — 后置 Hook 执行

### 源码位置
`src/services/tools/toolHooks.ts:39-191`

### 两种后置 Hook

| Hook | 触发条件 | 作用 |
|------|----------|------|
| `runPostToolUseHooks` | 工具执行成功后 | 审计、日志、修改 MCP 输出 |
| `runPostToolUseFailureHooks` | 工具执行失败后 | 错误处理、重试建议 |

### Hook 结果处理

```typescript
for await (const result of executePostToolHooks(tool.name, ...)) {
  // 1. Hook 被取消（abort）
  if (result.message?.attachment?.type === 'hook_cancelled') {
    yield { message: createAttachmentMessage({ type: 'hook_cancelled' }) }
    continue
  }

  // 2. Hook 返回阻止错误
  if (result.blockingError) {
    yield { message: createAttachmentMessage({ type: 'hook_blocking_error' }) }
  }

  // 3. Hook 阻止继续执行
  if (result.preventContinuation) {
    yield { message: createAttachmentMessage({ type: 'hook_stopped_continuation' }) }
    return  // 终止
  }

  // 4. Hook 提供额外上下文
  if (result.additionalContexts?.length > 0) {
    yield { message: createAttachmentMessage({ type: 'hook_additional_context' }) }
  }

  // 5. Hook 修改 MCP 工具输出（仅 MCP 工具）
  if (result.updatedMCPToolOutput && isMcpTool(tool)) {
    toolOutput = result.updatedMCPToolOutput
  }
}
```

### MCP vs 内置工具的不同处理
```typescript
// 内置工具：先添加 tool_result，再跑 hooks
if (!isMcpTool(tool)) {
  await addToolResult(toolOutput, mappedToolResultBlock)
}

// MCP 工具：先跑 hooks（可能修改输出），再添加 tool_result
for await (const hookResult of runPostToolUseHooks(...)) {
  if ('updatedMCPToolOutput' in hookResult) {
    toolOutput = hookResult.updatedMCPToolOutput  // hooks 修改了输出
  }
}
if (isMcpTool(tool)) {
  await addToolResult(toolOutput)  // 用修改后的输出
}
```

---

## Step 10: 返回 ToolResult — 最终结果组装

### 源码位置
`src/services/tools/toolExecution.ts:1397-1588`

### 结果组装逻辑

```typescript
// 1. 映射工具输出为 API 格式
const mappedToolResultBlock = tool.mapToolResultToToolResultBlockParam(
  result.data,
  toolUseID,
)

// 2. 处理结果大小（超大结果持久化到磁盘）
const toolResultBlock = await processToolResultBlock(
  tool, toolUseResult, toolUseID,
)

// 3. 构建内容块
const contentBlocks = [
  toolResultBlock,                    // 工具结果
  // 用户审批时附带的反馈文本
  permissionDecision.acceptFeedback,
  // 用户粘贴的图片等
  ...permissionDecision.contentBlocks,
]

// 4. 添加 newMessages（技能展开的提示词等）
if (result.newMessages?.length > 0) {
  for (const message of result.newMessages) {
    resultingMessages.push({ message })
  }
}

// 5. 应用 contextModifier（修改后续工具白名单等）
resultingMessages.push({
  message: createUserMessage({ content: contentBlocks }),
  contextModifier: toolContextModifier
    ? { toolUseID, modifyContext: toolContextModifier }
    : undefined,
})
```

### 错误处理路径

```typescript
// tool.call() 抛出异常时
catch (error) {
  // 1. MCP 认证错误 → 更新服务器状态为 needs-auth
  if (error instanceof McpAuthError) {
    toolUseContext.setAppState(prev => ({
      ...prev,
      mcp: { ...prev.mcp, clients: updatedClients }
    }))
  }

  // 2. 运行 PostToolUseFailure hooks
  for await (const hookResult of runPostToolUseFailureHooks(...)) {
    hookMessages.push(hookResult)
  }

  // 3. 返回错误结果
  return [
    { message: createUserMessage({ content: [{
      type: 'tool_result',
      content: formatError(error),
      is_error: true,
      tool_use_id: toolUseID,
    }]})},
    ...hookMessages,
  ]
}
```

---

## Tool 接口完整定义

> `src/Tool.ts:362-695`

### 核心方法（每个工具必须实现）

| 方法 | 必需 | 说明 |
|------|------|------|
| `name` | 是 | 工具名称 |
| `inputSchema` | 是 | Zod schema 定义输入格式 |
| `call()` | 是 | 工具执行逻辑 |
| `description()` | 是 | 返回工具描述 |
| `prompt()` | 是 | 返回给 Claude 的系统提示词 |
| `mapToolResultToToolResultBlockParam()` | 是 | 将输出映射为 API 格式 |
| `renderToolUseMessage()` | 是 | 渲染工具调用消息 UI |
| `isConcurrencySafe()` | 否（默认 false） | 是否支持并行执行 |
| `isReadOnly()` | 否（默认 false） | 是否只读 |
| `isDestructive()` | 否（默认 false） | 是否破坏性操作 |

### 权限相关方法

| 方法 | 说明 |
|------|------|
| `checkPermissions()` | 工具级权限检查（默认 allow） |
| `validateInput()` | 自定义输入验证 |
| `backfillObservableInput()` | 填充派生字段 |
| `preparePermissionMatcher()` | 准备权限匹配器（用于 hook 的 if 条件） |
| `getPath()` | 获取工具操作的文件路径 |
| `requiresUserInteraction()` | 是否需要用户交互 |

### buildTool() 工厂函数

```typescript
// src/Tool.ts:757-792
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: (_input?) => false,
  isReadOnly: (_input?) => false,
  isDestructive: (_input?) => false,
  checkPermissions: (input) => Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: (_input?) => '',
  userFacingName: (_input?) => '',
}

export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return { ...TOOL_DEFAULTS, userFacingName: () => def.name, ...def }
}
```

**设计原则**：安全默认值（fail-closed）——未实现的方法默认最安全的选项。

---

## 遥测与可观测性

每个步骤都有对应的遥测事件：

| 步骤 | 遥测事件 |
|------|----------|
| 工具未找到 | `tengu_tool_use_error` (error: "No such tool") |
| Schema 验证失败 | `tengu_tool_use_error` (error: "InputValidationError") |
| 自定义验证失败 | `tengu_tool_use_error` (error: custom message) |
| 进度报告 | `tengu_tool_use_progress` |
| 权限拒绝 | `tengu_tool_use_can_use_tool_rejected` |
| 权限允许 | `tengu_tool_use_can_use_tool_allowed` |
| 执行成功 | `tengu_tool_use_success` (含 duration, resultSize) |
| 执行失败 | `tengu_tool_use_error` (含 classifyToolError) |
| Hook 取消 | `tengu_pre_tool_hooks_cancelled` / `tengu_post_tool_hooks_cancelled` |
| Hook 错误 | `tengu_pre_tool_hook_error` / `tengu_post_tool_hook_error` |
| OTel 工具决策 | `tool_decision` (source: config/hook/user_permanent/user_temporary/user_reject) |
| OTel 工具结果 | `tool_result` (含 duration_ms, tool_parameters, tool_input) |

---

## 总结：设计哲学

1. **纵深防御** — Schema 验证 → 自定义验证 → Hook 检查 → 权限检查 → 执行 → 后置审计，每层都是独立防线
2. **Hook 不高于规则** — Hook 的 `allow` 无法绕过 `settings.json` 的 `deny/ask` 规则
3. **安全默认值** — `buildTool()` 的默认值全部是最安全的选项（`isConcurrencySafe: false`、`isReadOnly: false`）
4. **渐进式进度** — `AsyncGenerator` + `onProgress` 回调实现实时进度报告
5. **可观测性** — 每个决策点都有遥测事件，支持 OTel 追踪
6. **输入不可变性** — `callInput` 保留原始值，`processedInput` 供 hooks/权限系统使用
