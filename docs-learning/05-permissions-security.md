# Claude Code 源码学习：权限与安全机制 (`useCanUseTool` / `permissions/`)

> 权限系统是 Claude Code 区别于普通 AI 编程助手的核心产品能力。它不仅仅是"弹个窗问用户同不同意"，而是一个**多层决策引擎**，融合了规则匹配、模式切换、AI 自动分类器和沙箱隔离。

---

## 一、权限系统的总入口：`useCanUseTool`

每次模型想要调用工具时，`query.ts` 都会调用 `canUseTool()`。这个函数由 `useCanUseTool` hook 提供：

```ts
// src/hooks/useCanUseTool.tsx
function useCanUseTool(setToolUseConfirmQueue, setToolPermissionContext) {
  return async (tool, input, toolUseContext, assistantMessage, toolUseID, forceDecision) => {
    return new Promise(resolve => {
      const ctx = createPermissionContext(tool, input, toolUseContext, assistantMessage, toolUseID, ...)
      
      // 1. 如果已被 abort，直接返回取消
      if (ctx.resolveIfAborted(resolve)) return
      
      // 2. 调用核心权限判断
      const decisionPromise = forceDecision !== undefined
        ? Promise.resolve(forceDecision)
        : hasPermissionsToUseTool(tool, input, toolUseContext, assistantMessage, toolUseID)
      
      decisionPromise.then(async result => {
        switch (result.behavior) {
          case 'allow':
            // 直接放行
            resolve(ctx.buildAllow(result.updatedInput ?? input, { decisionReason: result.decisionReason }))
            break
          case 'deny':
            // 直接拒绝
            resolve(result)
            break
          case 'ask':
            // 进入交互式权限流程（弹窗）
            handleInteractivePermission({ ctx, description, result, ... }, resolve)
            break
        }
      })
    })
  }
}
```

**核心设计**：所有权限判断都返回一个 `Promise<PermissionDecision>`，这个 Promise 可能：
- **立即 resolve**（`allow` / `deny`）
- **长期 pending**（`ask`，直到用户点击弹窗按钮才 resolve）

这意味着 `query.ts` 的工具执行流程会在权限弹窗处**同步阻塞**，直到用户做出选择。

---

## 二、核心决策引擎：`hasPermissionsToUseToolInner`

`hasPermissionsToUseTool` 的底层实现是 `hasPermissionsToUseToolInner`（`src/utils/permissions/permissions.ts`）。这是一个**严格按优先级排序**的 7 步决策流水线：

```
┌─────────────────────────────────────────────────────────────────────┐
│  Step 1: 工具级 deny 规则                                           │
│  用户是否配置了 "永远不要使用 Bash"？                                │
│  → 是：直接 deny                                                    │
├─────────────────────────────────────────────────────────────────────┤
│  Step 2: 工具级 ask 规则                                            │
│  用户是否配置了 "每次使用 Bash 都要问我"？                           │
│  → 是：ask（但 sandbox 命令可能例外）                                │
├─────────────────────────────────────────────────────────────────────┤
│  Step 3: 工具的 checkPermissions()                                  │
│  工具自己怎么说？（Bash 分析 AST，FileEdit 检查路径规则）            │
│  → deny：直接 deny                                                  │
│  → ask：继续 Step 4-7                                               │
│  → passthrough：转为 ask                                            │
├─────────────────────────────────────────────────────────────────────┤
│  Step 4: bypassPermissions 模式                                     │
│  当前是 bypass 模式吗？                                             │
│  → 是：allow（但 safetyCheck 和 requiresUserInteraction 免疫）      │
├─────────────────────────────────────────────────────────────────────┤
│  Step 5: 工具级 allow 规则                                          │
│  用户是否配置了 "总是允许 Bash(git status)"？                        │
│  → 是：allow                                                        │
├─────────────────────────────────────────────────────────────────────┤
│  Step 6: passthrough → ask                                          │
│  前面都没有明确结果 → 转为 ask                                       │
├─────────────────────────────────────────────────────────────────────┤
│  Step 7: 外层模式转换（auto / dontAsk）                             │
│  hasPermissionsToUseTool 外层再做一次模式覆盖                        │
└─────────────────────────────────────────────────────────────────────┘
```

### 代码对应

```ts
async function hasPermissionsToUseToolInner(tool, input, context): Promise<PermissionDecision> {
  // Step 1a: 整个工具被 deny
  const denyRule = getDenyRuleForTool(appState.toolPermissionContext, tool)
  if (denyRule) return { behavior: 'deny', ... }
  
  // Step 1b: 整个工具被 ask
  const askRule = getAskRuleForTool(appState.toolPermissionContext, tool)
  if (askRule) return { behavior: 'ask', ... }
  
  // Step 1c: 工具自己的 checkPermissions
  const parsedInput = tool.inputSchema.parse(input)
  let toolPermissionResult = await tool.checkPermissions(parsedInput, context)
  
  // Step 1d: 工具 deny
  if (toolPermissionResult.behavior === 'deny') return toolPermissionResult
  
  // Step 1e/f/g: 某些 ask 结果免疫 bypass 模式
  if (tool.requiresUserInteraction?.() && toolPermissionResult.behavior === 'ask') return toolPermissionResult
  if (toolPermissionResult.decisionReason?.type === 'rule' && ...) return toolPermissionResult
  if (toolPermissionResult.decisionReason?.type === 'safetyCheck') return toolPermissionResult
  
  // Step 2a: bypassPermissions 模式
  if (shouldBypassPermissions) {
    return { behavior: 'allow', decisionReason: { type: 'mode', mode: ... } }
  }
  
  // Step 2b: 整个工具被 allow
  const alwaysAllowedRule = toolAlwaysAllowedRule(...)
  if (alwaysAllowedRule) return { behavior: 'allow', ... }
  
  // Step 3: passthrough → ask
  if (toolPermissionResult.behavior === 'passthrough') {
    return { ...toolPermissionResult, behavior: 'ask' }
  }
  
  return toolPermissionResult
}
```

---

## 三、权限模式（PermissionMode）

Claude Code 有 6 种权限模式，按"自由度"从低到高排列：

| 模式 | 行为 | 适用场景 |
|------|------|---------|
| **`dontAsk`** | 所有 `ask` 转为 `deny` | 完全无人值守，宁可不做也不打扰 |
| **`plan`** | 需要计划批准后才执行 | 高价值/高风险变更前的确认 |
| **`default`** | 标准流程：规则 → ask → 弹窗 | 日常交互 |
| **`auto`** | `ask` 走 AI 自动分类器判断 | 信任度较高的日常开发 |
| **`acceptEdits`** | 自动允许编辑类操作 | 子 Agent / 快速迭代模式 |
| **`bypassPermissions`** | 几乎全部放行 | 内部测试、完全信任环境 |

模式切换命令：`/permissions` 或 `/auto`。

---

## 四、权限规则系统：Allow / Ask / Deny

用户可以配置细粒度规则，语法类似：

```
Bash(git status)              → allow
Bash(rm *)                    → ask
Bash(curl * | sh)             → deny
FileEditTool(src/*.ts)        → allow
Agent(explore)                → allow
```

### 规则来源（按优先级）

```ts
const PERMISSION_RULE_SOURCES = [
  'policySettings',   // 组织策略（最高优先级）
  'flagSettings',     // Feature flag 注入
  'cliArg',           // 命令行参数
  'command',          // 运行时命令（如 /permissions）
  'projectSettings',  // 项目级 .claude/settings.json
  'userSettings',     // 用户级 ~/.claude/settings.json
  'localSettings',    // 本地覆盖
  'session',          // 当前会话临时设置
] as const
```

**来源越靠前，优先级越高**。这意味着组织管理员可以通过 `policySettings` 锁定某些规则，普通用户无法覆盖。

### Bash 的特殊规则匹配

Bash 命令的权限规则不是简单字符串匹配，而是基于**命令语义分析**：

```ts
// BashTool 的 preparePermissionMatcher
async preparePermissionMatcher({ command }) {
  return pattern => matchWildcardPattern(pattern, command)
}

// 例如规则 "Bash(git status)" 会匹配：
// - "git status"
// - "git status --short"
// 但不会匹配：
// - "git push"
```

---

## 五、Safety Check：不可绕过的安全闸门

某些路径和命令被标记为 **safetyCheck**，即使处于 `bypassPermissions` 模式也会强制弹窗：

```ts
if (toolPermissionResult?.behavior === 'ask' &&
    toolPermissionResult.decisionReason?.type === 'safetyCheck') {
  return toolPermissionResult  // 免疫 bypass
}
```

### 典型的 safetyCheck 场景

- 编辑 `.git/` 目录下的文件
- 修改 shell 配置文件（`.bashrc`, `.zshrc`）
- 修改 `.claude/` 设置文件
- 修改系统级路径（`/etc/`, `C:\Windows\`）
- 读取/写入 SSH 密钥

这是**防止 AI 意外破坏关键系统文件**的最后一道防线。

---

## 六、Auto Mode 与 AI 分类器（ANT-ONLY）

当用户开启 `auto` 模式时，所有 `ask` 不再弹窗，而是交给**AI 分类器**自动判断。

### 6.1 分类器决策流程

```ts
if (appState.toolPermissionContext.mode === 'auto') {
  // 1. 某些 safetyCheck 不可分类，保持 ask（在 headless 中转为 deny）
  if (result.decisionReason?.type === 'safetyCheck' && !result.decisionReason.classifierApprovable) {
    return appState.toolPermissionContext.shouldAvoidPermissionPrompts
      ? { behavior: 'deny', ... }  // headless / 后台 Agent
      : result  // 保持 ask
  }
  
  // 2. 需要用户交互的工具跳过分类器
  if (tool.requiresUserInteraction?.() && result.behavior === 'ask') {
    return result
  }
  
  // 3. 调用分类器
  const classifierResult = await classifyYoloAction(...)
  // 或 bash-specific classifier
  const bashClassifierResult = await classifyBashCommand(...)
  
  // 4. 根据分类结果决定 allow / deny / ask
  if (classifierResult.matches && classifierResult.confidence === 'high') {
    return { behavior: 'allow', decisionReason: { type: 'classifier', ... } }
  }
  // 低置信度或拒识 → 保持 ask
}
```

### 6.2 Bash 专用分类器

Bash 命令有一个专门的**推测性分类器**（Speculative Classifier）：

```ts
// 在弹窗显示的同时，后台启动分类器
if (feature('BASH_CLASSIFIER') && result.pendingClassifierCheck) {
  const speculativePromise = peekSpeculativeClassifierCheck(input.command)
  
  //  race：分类器结果 vs 2秒超时
  const raceResult = await Promise.race([
    speculativePromise.then(r => ({ type: 'result', result: r })),
    new Promise(res => setTimeout(res, 2000, { type: 'timeout' }))
  ])
  
  if (raceResult.type === 'result' && 
      raceResult.result.matches && 
      raceResult.result.confidence === 'high') {
    // 高置信度匹配 → 自动允许，用户甚至还没看到弹窗
    resolve(ctx.buildAllow(input, { decisionReason: { type: 'classifier', classifier: 'bash_allow' } }))
    return
  }
}
```

**这个设计的妙处**：
- 弹窗和分类器**并行**
- 如果分类器在 2 秒内高置信度通过，用户完全无感知
- 如果分类器没通过或超时，弹窗继续等待用户确认
- 一旦用户开始与弹窗交互（按方向键/Tab），分类器立即被取消

---

## 七、Denial Tracking：连续拒绝后的智能降级

在 `auto` 模式下，如果分类器**连续拒绝**太多次，系统会认为当前任务不适合自动模式，自动降级：

```ts
const denialState = context.localDenialTracking ?? appState.denialTracking ?? createDenialTrackingState()

if (shouldFallbackToPrompting(denialState)) {
  // 连续拒绝太多 → 强制回到弹窗模式（或告知用户）
}
```

```ts
// src/utils/permissions/denialTracking.ts
export const DENIAL_LIMITS = {
  fallbackThreshold: 3,   // 连续 3 次拒绝后降级
}

export function recordDenial(state: DenialTrackingState): DenialTrackingState {
  return {
    ...state,
    consecutiveDenials: state.consecutiveDenials + 1,
  }
}

export function recordSuccess(state: DenialTrackingState): DenialTrackingState {
  return {
    ...state,
    consecutiveDenials: 0,
  }
}
```

**为什么需要这个？**
> 如果分类器一直拒绝，说明当前任务的复杂度和风险超出了自动模式的舒适区。强行继续只会让用户不断收到"denied by auto mode"通知，体验很差。自动降级是产品层面的自我保护。

---

## 八、交互式权限处理：`handleInteractivePermission`

当最终走到 `ask` 且需要用户确认时，由 `handleInteractivePermission` 负责：

```ts
function handleInteractivePermission(params, resolve) {
  // 1. 推入权限弹窗队列
  ctx.pushToQueue({
    assistantMessage: ctx.assistantMessage,
    tool: ctx.tool,
    description,
    input: displayInput,
    toolUseContext: ctx.toolUseContext,
    toolUseID: ctx.toolUseID,
    permissionResult: result,
    permissionPromptStartTimeMs,
    // 回调函数
    onAllow(updates) { resolve(ctx.buildAllow(...)) },
    onReject() { resolve({ behavior: 'deny', ... }) },
    onAbort() { resolve(ctx.cancelAndAbort(...)) },
    onUserInteraction() { userInteracted = true; ... },
    recheckPermission() { ... },
  })
  
  // 2. 同时启动后台 classifier / hook 检查（如果支持）
  // 3. 如果用户通过 Bridge/Channel 远程确认，也走这里的回调
}
```

**弹窗队列的 resolve-once 机制**：
- 弹窗可能同时有多个"允许"来源（用户点击、分类器自动通过、Bridge 远程确认）
- `createResolveOnce` 保证只 resolve 一次，避免竞态条件

---

## 九、权限系统的 5 个工程智慧

### 1. **决策流水线必须严格有序**
`deny` → `ask` → `checkPermissions` → `bypass` → `allow` → `ask(passthrough)`。任何顺序调整都会带来安全漏洞或用户体验倒退。

### 2. **safetyCheck 是免疫层**
某些操作（改 `.git/`、shell 配置）无论模式如何都必须人工确认。这是从无数次 AI 幻觉导致的意外中总结出的铁律。

### 3. **分类器和弹窗要并行赛跑**
2 秒的高置信度自动允许 vs 用户手动确认。最好的权限体验是"用户没感觉到权限检查"。

### 4. **连续拒绝要降级**
AI 分类器不是万能的。当它在某个任务上连续失败时，系统要优雅地退回到人工确认，而不是硬撑。

### 5. **权限规则要分层来源**
组织策略 > 命令行 > 项目设置 > 用户设置 > 会话临时设置。这种分层让企业级部署成为可能。

---

## 十、关键代码索引

| 概念 | 文件 | 关键函数/类 |
|------|------|------------|
| 权限总入口 | `src/hooks/useCanUseTool.tsx` | `useCanUseTool`, `CanUseToolFn` |
| 核心决策引擎 | `src/utils/permissions/permissions.ts` | `hasPermissionsToUseTool`, `hasPermissionsToUseToolInner` |
| 交互式处理 | `src/hooks/toolPermission/handlers/interactiveHandler.ts` | `handleInteractivePermission` |
| 权限结果类型 | `src/types/permissions.ts` | `PermissionDecision`, `PermissionResult` |
| 拒绝追踪 | `src/utils/permissions/denialTracking.ts` | `recordDenial`, `shouldFallbackToPrompting` |
| Bash 分类器 | `src/utils/permissions/bashClassifier.ts` | `classifyBashCommand` (ANT-ONLY stub) |
| YOLO 分类器 | `src/utils/permissions/yoloClassifier.ts` | `classifyYoloAction` |
| 权限弹窗上下文 | `src/hooks/toolPermission/PermissionContext.ts` | `createPermissionContext`, `createResolveOnce` |
| 权限规则解析 | `src/utils/permissions/permissionRuleParser.ts` | `permissionRuleValueFromString` |
| 权限更新持久化 | `src/utils/permissions/PermissionUpdate.ts` | `applyPermissionUpdate`, `persistPermissionUpdates` |

---

*文档生成时间：2026-04-14*  
*基于源码版本：Claude Code v2.1.88*
