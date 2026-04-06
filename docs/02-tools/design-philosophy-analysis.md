# Claude Code 工具管线设计哲学与优缺点分析

> 基于 `src/services/tools/toolExecution.ts`、`src/services/tools/toolHooks.ts`、`src/Tool.ts` 源码分析

---

## 一、总体设计理念

Claude Code 的工具调用管线采用 **「纵深防御 + 管道-过滤器」** 架构，核心思想是：

```
每一步都是一道独立的防线，任何一层拦截成功都不会进入下一步。
同时每一步都有独立的可观测性（遥测/日志），形成完整的审计链。
```

这不是一个简单的"验证 → 执行"流程，而是一个精心设计的 **10 阶段状态机**，每个阶段都有明确的输入/输出契约和失败处理策略。

---

## 二、设计的六大好处

### 好处 1：纵深防御——每一层都是独立的安全防线

```
Schema 验证（类型安全）
    ↓ 失败 → InputValidationError
自定义验证（业务逻辑）
    ↓ 失败 → 工具自定义错误
前置 Hooks（用户脚本）
    ↓ 拒绝 → hook deny
规则权限（settings.json）
    ↓ 拒绝 → deny rule
用户交互（权限对话框）
    ↓ 拒绝 → user reject
工具执行
    ↓ 异常 → error handling
后置 Hooks（审计/修改）
```

**为什么好**：即使某一层被绕过或出 bug，下一层仍然能拦截。例如：
- Hook 返回 `allow` 但 `checkRuleBasedPermissions()` 仍然检查 deny/ask 规则
- Schema 验证通过了，`validateInput()` 还能检查业务逻辑（文件是否读过）
- `_simulatedSedEdit` 字段即使绕过了 schema strictObject 检查，在 Step 4 还会被手动剥离

源码中的注释明确说明了这个设计意图：
```typescript
// Defense-in-depth: strip _simulatedSedEdit from model-provided Bash input.
// The schema's strictObject should already reject it, but we strip here
// as a safeguard against future regressions.
```

### 好处 2：Hook 不高于规则——权限层级清晰

`resolveHookPermissionDecision()` 实现了一个关键不变量：

```
Hook allow → checkRuleBasedPermissions() → deny 规则覆盖 → ask 规则要求弹窗
```

这意味着用户在 `settings.json` 中写的 deny 规则**永远不可被 hook 绕过**。即使第三方 hook 脚本返回 allow，用户的显式 deny 规则仍然生效。

**为什么好**：这让用户拥有最终控制权。Hook 系统是扩展能力，不是覆盖能力。权限层级是：
```
用户 deny 规则 > Hook allow > Hook ask > 正常权限流程
```

### 好处 3：输入不可变性——callInput vs processedInput 的分离

```typescript
let callInput = processedInput                    // 原始输入
const backfilledClone = { ...processedInput }      // 浅拷贝
tool.backfillObservableInput!(backfilledClone)     // 修改拷贝
processedInput = backfilledClone                   // hooks/权限看到拷贝
```

到 `tool.call()` 时：
```typescript
// 如果 hook 没替换 input，恢复原始值给 call()
if (backfilledClone && processedInput !== callInput) {
  callInput = { ...processedInput, file_path: callInput.file_path }
}
```

**为什么好**：
- 工具结果嵌入的路径保持模型原始值（如 `~/foo` 而非 `/home/user/foo`）
- Transcript 和 VCR fixture hash 保持稳定
- Hooks 和权限系统看到展开后的绝对路径（用于规则匹配）

### 好处 4：渐进式进度反馈——AsyncGenerator + onProgress 双通道

```typescript
// 进度通道 1：onProgress 回调（实时）
onProgress({ toolUseID, data: progress.data })

// 进度通道 2：Stream 消息流（最终结果）
stream.enqueue({ message: createProgressMessage(...) })
```

`streamedCheckPermissionsAndCallTool()` 用 `Stream` 类桥接了这两个通道：
```typescript
const stream = new Stream<MessageUpdateLazy>()
checkPermissionsAndCallTool(..., progress => {
  stream.enqueue({ message: createProgressMessage(...) })  // 进度
})
.then(results => {
  for (const result of results) stream.enqueue(result)      // 最终结果
})
.finally(() => stream.done())
```

**为什么好**：用户在终端看到实时进度（命令输出、文件读取），同时最终结果通过消息流返回给 API。两个关注点完全分离。

### 好处 5：buildTool() 工厂——安全默认值

```typescript
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: (_input?) => false,     // 默认不安全
  isReadOnly: (_input?) => false,            // 默认可写
  isDestructive: (_input?) => false,
  checkPermissions: (input) => ({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: (_input?) => '',    // 默认跳过分类器
  userFacingName: (_input?) => '',
}
```

**为什么好**：新工具只需实现核心方法（`call`、`inputSchema`、`prompt`），其他方法自动获得安全默认值。这降低了工具开发的心智负担，同时保证忘记实现的方法走最安全的路径。

### 好处 6：完整的可观测性——每个决策点都有遥测

| 事件 | 数据 |
|------|------|
| `tengu_tool_use_error` | 工具名、错误类型、MCP 信息、query chain ID |
| `tengu_tool_use_success` | 持续时间、结果大小、文件扩展名 |
| `tengu_tool_use_progress` | 进度事件 |
| `tool_decision` (OTel) | 决策来源（config/hook/user_permanent/user_temporary/user_reject） |
| `tool_result` (OTel) | 工具参数、输入、输出大小、决策上下文 |

**为什么好**：问题排查时能精确知道工具在哪个阶段失败、权限决策由谁做出、耗时多久。OTel 集成支持分布式追踪。

---

## 三、设计优点详细分析

### 3.1 清晰的关注点分离

| 层次 | 关注点 | 文件 |
|------|--------|------|
| `Tool.ts` | 工具接口定义、查找逻辑 | 纯类型 + 工具函数 |
| `toolHooks.ts` | Hook 生命周期管理 | Pre/Post Hook 编排 |
| `toolExecution.ts` | 完整管线编排 | 10 步状态机 |
| `useCanUseTool.tsx` | 用户权限交互 | React 组件 |
| 各工具文件 | 工具特定逻辑 | 自包含 |

每个文件职责明确，修改某一层不影响其他层。

### 3.2 优雅的错误恢复

```typescript
// 权限被拒绝后，自动模式分类器可以通过 PermissionDenied hook 建议重试
if (permissionDecision.decisionReason?.type === 'classifier') {
  for await (const result of executePermissionDeniedHooks(...)) {
    if (result.retry) hookSaysRetry = true
  }
  if (hookSaysRetry) {
    resultingMessages.push({
      message: createUserMessage({
        content: 'The PermissionDenied hook indicated this command is now approved. You may retry it.'
      })
    })
  }
}
```

分类器拒绝后，不是死路一条——hook 可以在运行时改变决策。

### 3.3 投机优化——BashTool 分类器并行启动

```typescript
// 权限检查和 hook 执行的同时，提前启动分类器
if (tool.name === BASH_TOOL_NAME && 'command' in parsedInput.data) {
  startSpeculativeClassifierCheck(
    parsedInput.data.command,
    appState.toolPermissionContext,
    ...
  )
}
```

分类器检查是网络请求（调用 Claude 做 side-query），与 hooks 并行执行可以节省几百毫秒。

### 3.4 MCP 与内置工具的差异化处理

```typescript
// 内置工具：先添加 tool_result，再跑 hooks
if (!isMcpTool(tool)) {
  await addToolResult(toolOutput, mappedToolResultBlock)
}

// MCP 工具：先跑 hooks（可能修改输出），再添加 tool_result
if (isMcpTool(tool)) {
  await addToolResult(toolOutput)
}
```

MCP 工具的输出可以被 PostToolUse hooks 修改，内置工具不行。这给第三方 MCP 集成留了扩展空间。

---

## 四、设计缺点详细分析

### 缺点 1：函数过长——`checkPermissionsAndCallTool()` 约 700 行

`src/services/tools/toolExecution.ts` 中的 `checkPermissionsAndCallTool()` 函数从第 599 行到第 1745 行，**超过 1100 行**。它包含了：
- Schema 验证
- 自定义验证
- Input 处理
- Pre hooks
- 权限决策
- 工具执行
- Post hooks
- 错误处理
- 遥测日志

**问题**：
- 难以单元测试某个独立步骤
- 新开发者理解成本极高
- 修改一处容易引入回归

**可能的改进**：将每个阶段拆分为独立函数，由一个薄的编排器串联。

### 缺点 2：重复的遥测日志代码

几乎每个步骤都有结构几乎相同的遥测日志：

```typescript
logEvent('tengu_tool_use_xxx', {
  messageID: messageId as AnalyticsMetadata_...,
  toolName: sanitizeToolNameForAnalytics(tool.name),
  isMcp: tool.isMcp ?? false,
  queryChainId: toolUseContext.queryTracking?.chainId as AnalyticsMetadata_...,
  queryDepth: toolUseContext.queryTracking?.depth,
  ...(mcpServerType && { mcpServerType: mcpServerType as AnalyticsMetadata_... }),
  ...(mcpServerBaseUrl && { mcpServerBaseUrl: ... }),
  ...(requestId && { requestId: ... }),
  ...mcpToolDetailsForAnalytics(tool.name, mcpServerType, mcpServerBaseUrl),
})
```

这段代码在 `toolExecution.ts` 中重复了 **7-8 次**。

**问题**：
- 添加新的遥测维度需要改所有地方
- 代码膨胀严重

**可能的改进**：抽取 `createTelemetryContext()` 工厂函数，一次构建通用字段。

### 缺点 3：输入处理的状态管理复杂

```typescript
let processedInput = parsedInput.data       // Step 3 后
let callInput = processedInput               // 保存原始值
const backfilledClone = { ...processedInput } // Step 4 拷贝
processedInput = backfilledClone             // hooks 看到的
// ... hooks 可能再修改 processedInput
// ... resolveHookPermissionDecision 可能再修改
// ... 最终决定 callInput
if (backfilledClone && processedInput !== callInput ...) {
  callInput = { ...processedInput, file_path: callInput.file_path }
}
```

**问题**：`processedInput` 和 `callInput` 的关系依赖时序理解，有 4 个不同的赋值路径：
1. backfill 修改 → callInput 保持原始
2. hook 修改 → callInput 跟随
3. 权限返回 updatedInput → callInput 跟随
4. backfill 修改但 hook 也修改了 → 特殊路径恢复 file_path

**可能的改进**：用不可变数据结构 + Builder 模式，每步返回新对象而非就地修改。

### 缺点 4：MCP vs 内置工具的条件分支散落各处

```typescript
// 至少 5 处 MCP 专用逻辑
if (isMcpTool(tool)) { ... }
if (!isMcpTool(tool)) { ... }
if (error instanceof McpAuthError) { ... }  // MCP 认证错误
if (error instanceof McpToolCallError) { ... }  // MCP 调用错误
// MCP 工具的 PostHook 输出可以被修改
```

源码中甚至有 TODO 注释承认这个问题：
```typescript
// TOOD(hackyon): refactor so we don't have different experiences for MCP tools
```

**问题**：
- MCP 工具的特殊逻辑散落在主流程中，增加阅读负担
- 添加新的工具类型（如 LSP 工具）需要在多个地方添加条件

**可能的改进**：引入工具类型策略模式，不同类型有不同的 post-execution 行为。

### 缺点 5：Hook 系统的 Generator 协议复杂

`runPreToolUseHooks()` 返回 7 种类型的 AsyncGenerator yield：

```typescript
type: 'message' | 'hookPermissionResult' | 'hookUpdatedInput'
    | 'preventContinuation' | 'stopReason' | 'additionalContext' | 'stop'
```

消费端用 `switch` 处理 7 种情况：
```typescript
for await (const result of runPreToolUseHooks(...)) {
  switch (result.type) {
    case 'message': ...
    case 'hookPermissionResult': ...
    case 'hookUpdatedInput': ...
    case 'preventContinuation': ...
    case 'stopReason': ...
    case 'additionalContext': ...
    case 'stop': ...
  }
}
```

**问题**：
- 新增 yield 类型需要同时改生产者和消费者
- 编译器不保证 switch 穷尽性（因为 type 是字符串字面量联合类型）
- 语义不够直观——`preventContinuation` 和 `stopReason` 需要配合使用

**可能的改进**：将 Hook 结果分为 `HookDecision`（权限决策）和 `HookMessage`（消息/上下文）两类，减少联合类型成员。

### 缺点 6：硬编码的工具名检查

```typescript
if (tool.name === BASH_TOOL_NAME && ...) { ... }
if (tool.name === FILE_READ_TOOL_NAME && ...) { ... }
if (tool.name === FILE_EDIT_TOOL_NAME || tool.name === FILE_WRITE_TOOL_NAME) { ... }
if (tool.name === NOTEBOOK_EDIT_TOOL_NAME && ...) { ... }
if (tool.name === POWERSHELL_TOOL_NAME && ...) { ... }
```

这些硬编码散落在遥测日志、权限检查、内容记录等多个位置。

**问题**：
- 新增工具需要找到所有需要特殊处理的地方
- 容易遗漏某些位置

**可能的改进**：工具接口增加声明式的分类属性（如 `isFileTool`、`isShellTool`、`needsFileExtensionTracking`）。

---

## 五、架构设计总评

### 总体架构图

```
                    ┌─────────────────────────┐
                    │   Claude API Response    │
                    │   (tool_use block)       │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │   runToolUse()           │  ← 入口函数
                    │   toolExecution.ts       │
                    └────────────┬────────────┘
                                 │
          ┌──────────────────────┼──────────────────────┐
          │                      │                      │
   ┌──────▼──────┐    ┌─────────▼─────────┐   ┌───────▼───────┐
   │ 查找 & 验证  │    │ Hooks & 权限      │   │  执行 & 输出   │
   │ (Step 1-4)  │    │ (Step 5-7)        │   │ (Step 8-10)   │
   └─────────────┘    └───────────────────┘   └───────────────┘
   纯函数/同步         AsyncGenerator          Promise + Stream
```

### 三阶段特征

| 阶段 | 步骤 | 特征 | 失败策略 |
|------|------|------|----------|
| **查找 & 验证** | 1-4 | 同步/快、确定性 | 立即返回错误 |
| **Hooks & 权限** | 5-7 | 异步、可交互 | deny → 返回拒绝，ask → 弹窗 |
| **执行 & 输出** | 8-10 | 长时间运行 | try/catch + PostFailure hooks |

### 设计评分

| 维度 | 评分 | 说明 |
|------|------|------|
| **安全性** | 9/10 | 纵深防御完善，Hook 不高于规则 |
| **可扩展性** | 8/10 | buildTool() 工厂 + 声明式接口，新工具开发简单 |
| **可观测性** | 9/10 | 每个决策点都有遥测 + OTel 追踪 |
| **可维护性** | 5/10 | 超长函数、重复代码、复杂状态管理 |
| **可读性** | 6/10 | 注释详尽但函数太长，需要大量上下文 |
| **性能** | 8/10 | 投机分类器、AsyncGenerator 流式处理 |
| **测试性** | 6/10 | 超长函数难以隔离测试，但各工具有独立测试 |

### 核心取舍

这个设计做的一个明确取舍是：**用代码复杂度换取安全性**。

10 层管线、输入的双重管理、Hook 和规则的多重检查——这一切都是为了确保 Claude 调用的任何工具都不会绕过用户的控制。在 AI Agent 直接操作用户文件系统的场景下，这个取舍是合理的。但如果工具调用的安全性要求较低，这个管线就显得过于复杂。
