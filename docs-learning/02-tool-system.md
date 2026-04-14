# Claude Code 源码学习：Tool 系统 (`src/tools/*`)

> 工具是 AI Agent 的"手和眼"。Claude Code 内置了 40+ 个工具，从文件操作到子 Agent 递归，每个都经过精密设计。本章聚焦 3 个最重要的工具：BashTool、FileEditTool、AgentTool。

---

## 一、Tool 接口：所有工具的"宪法"

所有工具都通过 `buildTool()` 工厂函数创建，遵循统一的 `Tool` 接口（定义在 `src/Tool.ts`）。

### 核心接口一览

```ts
export type Tool<Input, Output, Progress> = {
  name: string              // 工具名，如 "Bash", "FileEdit", "Agent"
  description(...)          // 给模型看的简短描述
  prompt(...)               // 注入到 system prompt 中的详细说明
  inputSchema               // Zod Schema，模型用它生成参数
  outputSchema?             // 输出校验 Schema
  
  // === 生命周期钩子 ===
  validateInput?(input, context)   // 输入校验（不弹窗，纯逻辑检查）
  checkPermissions(input, context) // 权限检查（返回 allow/ask/deny）
  call(input, context, canUseTool, parentMessage, onProgress) // 实际执行
  
  // === UI 渲染 ===
  renderToolUseMessage(...)        // 渲染工具调用时的 UI
  renderToolResultMessage(...)     // 渲染工具结果
  renderToolUseProgressMessage?(...) // 渲染进度（如转圈+输出预览）
  mapToolResultToToolResultBlockParam(...) // 把执行结果转成 API 的 tool_result
  
  // === 元数据 ===
  getToolUseSummary?(...)          // 紧凑摘要（用于 UI 折叠显示）
  getActivityDescription?(...)     // Spinner 文案，如 "Running npm test"
  toAutoClassifierInput(input)     // 给自动权限分类器看的内容
  maxResultSizeChars: number       // 结果超过此值会持久化到磁盘
}
```

### buildTool 工厂

```ts
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  } as BuiltTool<D>
}
```

所有工具都通过对象展开合并默认行为，保证一致性。

---

## 二、BashTool —— 权限与沙箱的艺术

### 2.1 文件定位

| 文件 | 作用 |
|------|------|
| `src/tools/BashTool/BashTool.tsx` | 主工具定义（1,144 行） |
| `src/tools/BashTool/bashPermissions.ts` | 权限规则引擎 |
| `src/tools/BashTool/bashSecurity.ts` | AST 安全分析 |
| `src/utils/Shell.ts` | 实际 shell 执行封装 |

### 2.2 核心设计：异步生成器 `runShellCommand`

BashTool 的 `call` 方法不直接执行命令，而是创建一个**异步生成器**：

```ts
const commandGenerator = runShellCommand({
  input,
  abortController,
  setAppState,
  setToolJSX,
  preventCwdChanges,  // 子 Agent 不能 cd 出项目目录
  isMainThread,
  toolUseId,
  agentId,
})

// 消费生成器：既能拿到最终结果，也能在过程中 yield 进度
do {
  generatorResult = await commandGenerator.next()
  if (!generatorResult.done && onProgress) {
    onProgress({ toolUseID: `bash-progress-${progressCounter++}`, data: {
      type: 'bash_progress',
      output: progress.output,
      fullOutput: progress.fullOutput,
      elapsedTimeSeconds: progress.elapsedTimeSeconds,
      ...
    }})
  }
} while (!generatorResult.done)

result = generatorResult.value
```

**为什么用生成器？**
1. **流式进度**：长命令执行时，UI 可以实时显示输出行数和字节数
2. **后台化**：命令超时可以优雅地转为后台任务，而不阻塞主循环
3. **可中断**：通过 `AbortController` 随时取消

### 2.3 runShellCommand 内部关键逻辑

```ts
async function* runShellCommand({...}) {
  const timeoutMs = timeout || getDefaultTimeoutMs()
  
  // 1. 调用 exec() 启动实际 shell 进程
  const shellCommand = await exec(command, abortSignal, 'bash', {
    timeout: timeoutMs,
    onProgress(lastLines, allLines, totalLines, totalBytes, isIncomplete) {
      // 通过 resolveProgress 唤醒生成器，yield 最新进度
    },
    preventCwdChanges,     // 子 Agent 禁止改 cwd
    shouldUseSandbox,      // 是否启用沙箱
    shouldAutoBackground   // 是否允许自动后台化
  })

  // 2. 启动后，进入进度轮询循环
  const resultPromise = shellCommand.result
  
  while (!resultDone) {
    // 要么 result 完成，要么 onProgress 有新数据
    const race = await Promise.race([resultPromise, progressSignal])
    if (progressSignal fired) {
      yield { type: 'progress', output: ..., fullOutput: ... }
    }
  }
  
  return result
}
```

### 2.4 权限检查：`bashToolHasPermission`

BashTool 的权限不是简单的是/否，而是基于**命令语义分析**：

```ts
async checkPermissions(input, context): Promise<PermissionResult> {
  return bashToolHasPermission(input, context)
}
```

`bashToolHasPermission` 内部会：
1. **解析命令 AST**：`parseForSecurity(command)` 识别命令结构
2. **提取主命令**：如 `git status`、`npm test`、`rm -rf /`
3. **匹配权限规则**：用户的 `alwaysAllow` / `alwaysAsk` / `alwaysDeny` 规则
4. **特殊处理**：
   - `cd` 命令：子 Agent 默认禁止（`preventCwdChanges`）
   - 破坏命令：`rm`、`dd > disk` 等会标记为 destructive
   - 沙箱命令：启用 sandbox 时，在受限环境中运行

### 2.5 输入校验：`validateInput`

```ts
async validateInput(input) {
  if (feature('MONITOR_TOOL') && !isBackgroundTasksDisabled && !input.run_in_background) {
    const sleepPattern = detectBlockedSleepPattern(input.command)
    if (sleepPattern !== null) {
      return {
        result: false,
        message: `Blocked: ${sleepPattern}. Run blocking commands in the background...`,
        errorCode: 10
      }
    }
  }
  return { result: true }
}
```

**设计意图**：如果检测到 `sleep 100` 这种阻塞命令，且没有 `run_in_background: true`，直接拒绝并提示用户使用后台模式或 Monitor 工具。这是产品层面对用户体验的保护。

### 2.6 大输出处理：持久化到磁盘

Bash 命令可能产生 MB 级甚至 GB 级输出（如日志文件）。直接塞给模型会爆 token：

```ts
const MAX_PERSISTED_SIZE = 64 * 1024 * 1024  // 64MB

if (result.outputFilePath && result.outputTaskId) {
  const fileStat = await fsStat(result.outputFilePath)
  persistedOutputSize = fileStat.size
  await ensureToolResultsDir()
  const dest = getToolResultPath(result.outputTaskId, false)
  if (fileStat.size > MAX_PERSISTED_SIZE) {
    await fsTruncate(result.outputFilePath, MAX_PERSISTED_SIZE)
  }
  // 优先硬链接，失败再复制
  try {
    await link(result.outputFilePath, dest)
  } catch {
    await copyFile(result.outputFilePath, dest)
  }
  persistedOutputPath = dest
}
```

**结果呈现给模型时**：

```ts
if (persistedOutputPath) {
  const preview = generatePreview(processedStdout, PREVIEW_SIZE_BYTES)
  processedStdout = buildLargeToolResultMessage({
    filepath: persistedOutputPath,
    originalSize: persistedOutputSize ?? 0,
    isJson: false,
    preview: preview.preview,
    hasMore: preview.hasMore
  })
}
```

模型看到的是：
> "输出太大已保存到 `/tmp/tool-results/xxx.log`（原始大小 15MB），前 2KB 预览如下：..."

随后模型可以用 `FileReadTool` 读取该文件。

### 2.7 BashTool 的 UI 折叠机制

终端里 `find`、`grep`、`ls` 这类命令的输出往往很长。Claude Code 会把它们**折叠**成一行摘要：

```ts
export function isSearchOrReadBashCommand(command: string) {
  const BASH_SEARCH_COMMANDS = new Set(['find', 'grep', 'rg', 'ag', 'ack'])
  const BASH_READ_COMMANDS = new Set(['cat', 'head', 'tail', 'less', 'jq', 'awk'])
  const BASH_LIST_COMMANDS = new Set(['ls', 'tree', 'du'])
  // ... 解析命令管道，判断整条命令是否纯搜索/读取
}
```

如果是纯读取命令，UI 默认折叠，只显示 "Listed 3 directories" 或 "Read 12 files"。

---

## 三、FileEditTool —— 为什么 Claude Code 编辑文件很少出错

### 3.1 文件定位

| 文件 | 作用 |
|------|------|
| `src/tools/FileEditTool/FileEditTool.ts` | 主工具定义（625 行） |
| `src/tools/FileEditTool/utils.ts` | `findActualString`, `getPatchForEdit` |
| `src/tools/FileEditTool/UI.tsx` | UI 渲染 |

### 3.2 编辑模式：字符串替换（StrReplace），而非覆写

FileEditTool 的核心参数是：

```json
{
  "file_path": "src/foo.ts",
  "old_string": "function add(a, b) {",
  "new_string": "function add(a: number, b: number) {"
}
```

**为什么不直接覆写整个文件？**
- 覆写容易丢失并发修改
- 字符串替换让模型必须**先读后改**，减少误操作
- 与用户的 diff 直觉一致

### 3.3 `validateInput` —— 六道安全闸门

FileEditTool 的 `validateInput` 是我见过最严密的输入校验之一：

#### 闸门 1：无意义编辑
```ts
if (old_string === new_string) {
  return { result: false, message: 'No changes to make: old_string and new_string are exactly the same.' }
}
```

#### 闸门 2：权限规则 deny
```ts
const denyRule = matchingRuleForInput(fullFilePath, toolPermissionContext, 'edit', 'deny')
if (denyRule !== null) {
  return { result: false, message: 'File is in a directory that is denied by your permission settings.' }
}
```

#### 闸门 3：UNC 路径安全（Windows 防 NTLM 泄露）
```ts
if (fullFilePath.startsWith('\\') || fullFilePath.startsWith('//')) {
  return { result: true }  // 跳过文件系统检查，让权限层处理
}
```

#### 闸门 4：超大文件保护
```ts
const MAX_EDIT_FILE_SIZE = 1024 * 1024 * 1024  // 1 GiB
if (size > MAX_EDIT_FILE_SIZE) {
  return { result: false, message: `File is too large to edit (${formatFileSize(size)})...` }
}
```

#### 闸门 5：必须先读
```ts
const readTimestamp = toolUseContext.readFileState.get(fullFilePath)
if (!readTimestamp || readTimestamp.isPartialView) {
  return { result: false, message: 'File has not been read yet. Read it first before writing to it.' }
}
```

**这是 FileEditTool 不出错的第一性原理**：模型必须先调用 `FileReadTool` 读取文件，系统会在 `readFileState` 中记录读取时间和完整内容。如果模型想编辑一个它没读过的文件，直接被拒。

#### 闸门 6：文件已被外部修改
```ts
const lastWriteTime = getFileModificationTime(fullFilePath)
if (lastWriteTime > readTimestamp.timestamp) {
  // Windows 下时间戳可能因云同步/杀毒软件变化，再比较内容
  const contentUnchanged = isFullRead && originalFileContents === lastRead.content
  if (!contentUnchanged) {
    return { result: false, message: 'File has been modified since read... Read it again before attempting to write it.' }
  }
}
```

#### 闸门 7：字符串找不到或多处匹配
```ts
const actualOldString = findActualString(file, old_string)
if (!actualOldString) {
  return { result: false, message: `String to replace not found in file.\nString: ${old_string}` }
}

const matches = file.split(actualOldString).length - 1
if (matches > 1 && !replace_all) {
  return { result: false, message: `Found ${matches} matches... To replace only one occurrence, please provide more context...` }
}
```

`findActualString` 还做了**引号智能归一化**：如果模型给的 `old_string` 里有直引号 `"`，但文件里实际是弯引号 `"`（或反之），它会自动匹配文件中的实际字符。

### 3.4 `call` 方法 —— 原子读写

通过所有校验后，进入真正的编辑：

```ts
async call(input, { readFileState, updateFileHistoryState, ... }) {
  // 1. 确保目录存在（在原子区外）
  await fs.mkdir(dirname(absoluteFilePath))
  
  // 2. 备份（fileHistoryTrackEdit）
  await fileHistoryTrackEdit(updateFileHistoryState, absoluteFilePath, parentMessage.uuid)
  
  // 3. 原子区开始：重新读文件 +  staleness 检查
  const { content: originalFileContents, fileExists, encoding, lineEndings } = readFileForEdit(absoluteFilePath)
  
  // 再次检查文件是否被外部修改
  if (fileExists) {
    const lastWriteTime = getFileModificationTime(absoluteFilePath)
    const lastRead = readFileState.get(absoluteFilePath)
    if (!lastRead || lastWriteTime > lastRead.timestamp) {
      const contentUnchanged = ...
      if (!contentUnchanged) throw new Error(FILE_UNEXPECTEDLY_MODIFIED_ERROR)
    }
  }
  
  // 4. 再次 findActualString + preserveQuoteStyle
  const actualOldString = findActualString(originalFileContents, old_string) || old_string
  const actualNewString = preserveQuoteStyle(old_string, actualOldString, new_string)
  
  // 5. 生成 patch 和最终内容
  const { patch, updatedFile } = getPatchForEdit({
    filePath: absoluteFilePath,
    fileContents: originalFileContents,
    oldString: actualOldString,
    newString: actualNewString,
    replaceAll: replace_all,
  })
  
  // 6. 写入磁盘
  writeTextContent(absoluteFilePath, updatedFile, encoding, lineEndings)
  
  // 7. 通知 LSP 服务器（didChange + didSave）
  const lspManager = getLspServerManager()
  if (lspManager) {
    lspManager.changeFile(absoluteFilePath, updatedFile).catch(...)
    lspManager.saveFile(absoluteFilePath).catch(...)
  }
  
  // 8. 通知 VSCode MCP 做 diff 展示
  notifyVscodeFileUpdated(absoluteFilePath, originalFileContents, updatedFile)
  
  // 9. 更新 readFileState，让后续编辑以新内容为基准
  readFileState.set(absoluteFilePath, {
    content: updatedFile,
    timestamp: getFileModificationTime(absoluteFilePath),
    offset: undefined,
    limit: undefined,
  })
}
```

### 3.5 关键工程细节

| 细节 | 意义 |
|------|------|
| `await fs.mkdir()` 在原子区外 | 避免 yield 期间并发修改插入 |
| 两次 staleness 检查 | `validateInput` 一次，`call` 前再一次（时间窗更小） |
| `preserveQuoteStyle` | 模型用直引号，文件用弯引号，也能正确替换 |
| 通知 LSP | 改完文件后 TypeScript 语言服务器立即重新检查类型 |
| `readFileState` 更新 | 连续编辑同一文件时，后续编辑基于最新内容 |

---

## 四、AgentTool —— 子 Agent 的递归世界

### 4.1 文件定位

| 文件 | 作用 |
|------|------|
| `src/tools/AgentTool/AgentTool.tsx` | 主工具定义（1,398 行） |
| `src/tools/AgentTool/runAgent.ts` | 子 Agent 运行器 |
| `src/tools/AgentTool/loadAgentsDir.ts` | Agent 定义加载 |
| `src/tools/AgentTool/forkSubagent.ts` | Fork 子 Agent 实验 |

### 4.2 AgentTool 的输入 Schema

```ts
{
  description: string,           // 3-5 字任务描述
  prompt: string,                // 给子 Agent 的完整 prompt
  subagent_type?: string,        // 专用 Agent 类型（如 explore, plan）
  model?: 'sonnet' | 'opus' | 'haiku',
  run_in_background?: boolean,   // 是否在后台运行
  name?: string,                 // 命名（多 Agent 协作时用）
  team_name?: string,            // 团队名（Agent Swarms）
  isolation?: 'worktree',        // 隔离模式
  cwd?: string                   // 覆盖工作目录
}
```

### 4.3 Agent 选择逻辑

```ts
// Fork subagent 实验路由
const effectiveType = subagent_type ?? (isForkSubagentEnabled() ? undefined : GENERAL_PURPOSE_AGENT.agentType)
const isForkPath = effectiveType === undefined

if (isForkPath) {
  // Fork 路径：继承父 Agent 的 system prompt 和工具集
  // 用于 cache-identical 优化（父子共享 prompt 缓存前缀）
  selectedAgent = FORK_AGENT
} else {
  // 普通路径：按 subagent_type 查找对应 AgentDefinition
  const agents = filterDeniedAgents(allAgents, toolPermissionContext, AGENT_TOOL_NAME)
  selectedAgent = agents.find(agent => agent.agentType === effectiveType)
}
```

### 4.4 隔离模式：`worktree`

当 `isolation: 'worktree'` 时，Claude Code 会为子 Agent 创建一个**独立的 git worktree**：

```ts
const earlyAgentId = createAgentId()
const slug = `agent-${earlyAgentId.slice(0, 8)}`
const worktreeInfo = await createAgentWorktree(slug)
```

这保证了：
- 子 Agent 的修改不影响父 Agent 的工作目录
- 子 Agent 可以自由做破坏性操作（如大量重构）
- 完成后，如果 worktree 有变更则保留，无变更则自动清理

```ts
const cleanupWorktreeIfNeeded = async () => {
  if (headCommit) {
    const changed = await hasWorktreeChanges(worktreePath, headCommit)
    if (!changed) {
      await removeAgentWorktree(worktreePath, worktreeBranch, gitRoot)
      return {}
    }
  }
  return { worktreePath, worktreeBranch }  // 有变更，保留
}
```

### 4.5 异步 vs 同步执行

AgentTool 支持两种执行模式：

#### 异步（Background Agent）

触发条件：
```ts
const shouldRunAsync = (
  run_in_background === true ||
  selectedAgent.background === true ||
  isCoordinator ||
  forceAsync ||           // fork 子 Agent 强制异步
  assistantForceAsync ||  // KAIROS 模式强制异步
  proactiveModule?.isProactiveActive()
) && !isBackgroundTasksDisabled
```

异步 Agent 的执行流程：
```ts
const agentBackgroundTask = registerAsyncAgent({ agentId, description, prompt, selectedAgent, setAppState, toolUseId })

// 在独立上下文中运行，不阻塞父 loop
void runWithAgentContext(asyncAgentContext, () => runAsyncAgentLifecycle({
  taskId: agentBackgroundTask.agentId,
  abortController: agentBackgroundTask.abortController,
  makeStream: params => runAgent({ ...runAgentParams, ...params }),
  metadata,
  description,
  toolUseContext,
  rootSetAppState,
  getWorktreeResult: cleanupWorktreeIfNeeded
}))

// 立即返回，父 Agent 继续
return {
  data: {
    isAsync: true,
    status: 'async_launched',
    agentId,
    outputFile: getTaskOutputPath(agentId),
    canReadOutputFile
  }
}
```

异步 Agent 完成后会通过 `enqueueAgentNotification` 发送通知，用户可用 `/tasks` 查看进度。

#### 同步（Foreground Agent）

同步 Agent 阻塞父 loop，但可以被**中途后台化**：

```ts
// 注册为 foreground task
const { taskId, backgroundSignal } = registerAgentForeground({ agentId, description, prompt, ... })

while (true) {
  // 每次迭代都与 backgroundSignal 赛跑
  const raceResult = await Promise.race([
    agentIterator.next().then(r => ({ type: 'message', result: r })),
    backgroundSignal.then(() => ({ type: 'background' }))
  ])
  
  if (raceResult.type === 'background') {
    // 用户点了 "Background" 或自动超时后台化
    wasBackgrounded = true
    // 关闭 foreground iterator，切换到 background 模式继续运行
    await Promise.race([agentIterator.return(undefined).catch(() => {}), sleep(1000)])
    
    void runWithAgentContext(..., async () => {
      for await (const msg of runAgent({ ...runAgentParams, isAsync: true })) {
        // 继续收集消息...
      }
      completeAsyncAgent(agentResult, rootSetAppState)
      enqueueAgentNotification({ taskId, status: 'completed', ... })
    })
    
    // 立即返回 async_launched
    return { data: { isAsync: true, status: 'async_launched', agentId, ... } }
  }
  
  // 正常处理消息
  const message = raceResult.result.value
  agentMessages.push(message)
  // 转发进度到父 Agent UI...
}
```

**这个设计的妙处**：
- 默认同步运行，用户体验最直观（像普通工具调用一样等待结果）
- 如果任务耗时过长，用户可以手动点击 "Run in background" 按钮
- 也可以配置 `autoBackgroundMs`（如 120 秒）自动后台化

### 4.6 子 Agent 的工具池独立组装

子 Agent 不会继承父 Agent 的受限工具集，而是**独立重新组装**：

```ts
const workerPermissionContext = {
  ...appState.toolPermissionContext,
  mode: selectedAgent.permissionMode ?? 'acceptEdits'
}
const workerTools = assembleToolPool(workerPermissionContext, appState.mcp.tools)
```

这意味着：
- 父 Agent 处于 `plan` 模式（所有操作需确认），子 Agent 可以被设置为 `acceptEdits` 模式自动执行
- 子 Agent 可以使用自己的 MCP 工具
- Fork 子 Agent 为了 prompt cache 共享，会例外地使用父 Agent 的 exact tools

### 4.7 MCP 依赖等待

Agent 定义可以声明 `requiredMcpServers`：

```ts
if (requiredMcpServers?.length) {
  // 如果有 pending 的 MCP 服务器，最多等 30 秒
  const MAX_WAIT_MS = 30_000
  while (Date.now() < deadline) {
    await sleep(POLL_INTERVAL_MS)
    currentAppState = toolUseContext.getAppState()
    const stillPending = currentAppState.mcp.clients.some(c => c.type === 'pending' && ...)
    if (!stillPending) break
  }
  
  // 检查是否所有 required 服务器都有可用工具
  if (!hasRequiredMcpServers(selectedAgent, serversWithTools)) {
    throw new Error(`Agent '${selectedAgent.agentType}' requires MCP servers matching: ${missing.join(', ')}`)
  }
}
```

---

## 五、三个工具的横向对比

| 维度 | BashTool | FileEditTool | AgentTool |
|------|---------|--------------|-----------|
| **核心操作** | 执行 shell 命令 | 字符串替换编辑文件 | 启动子 Agent |
| **输入校验** | 检测阻塞 sleep | 6+ 层安全闸门 | 检查 Agent 类型、MCP 依赖 |
| **权限检查** | AST 语义分析 | 文件路径规则匹配 | Agent 类型 deny 规则 |
| **流式/进度** | ✅ 实时输出进度 | ❌ 一次性完成 | ✅ 子 Agent 消息流式转发 |
| **后台化** | ✅ 可后台运行 | ❌ 瞬时完成 | ✅ 同步/异步双模式 |
| **大结果处理** | 持久化到磁盘 | 无需（结果很小） | 输出文件跟踪 |
| **隔离性** | 沙箱可选 | 原子读写 | Worktree / 独立上下文 |

---

## 六、从 Tool 系统学到的工程智慧

### 1. **"先读后写" 是防止 AI 乱改代码的第一性原理**
FileEditTool 强制要求 `readFileState` 命中，且两次检查 staleness。没有这层约束，模型很容易基于幻觉编辑文件。

### 2. **权限系统要分层：validateInput → checkPermissions → call**
- `validateInput`：纯逻辑校验，不依赖用户交互
- `checkPermissions`：基于规则的动态决策
- `call`：只有前面都通过才执行

### 3. **长任务必须支持"可中断 + 可后台化"**
BashTool 和 AgentTool 都有 `foreground → background` 的降级路径。用户不会愿意干等 5 分钟。

### 4. **工具结果超过阈值必须落盘**
不要把 MB 级的输出直接塞给模型。Claude Code 用 `maxResultSizeChars` + `persistedOutputPath` 优雅地解决了这个问题。

---

## 七、关键代码索引

| 概念 | 文件 | 关键函数/类 |
|------|------|------------|
| Tool 接口 | `src/Tool.ts` | `Tool`, `buildTool`, `ToolDef` |
| 工具注册 | `src/tools.ts` | `getTools`, `assembleToolPool` |
| Bash 执行 | `src/utils/Shell.ts` | `exec` |
| Bash 权限 | `src/tools/BashTool/bashPermissions.ts` | `bashToolHasPermission` |
| Bash 流式生成器 | `src/tools/BashTool/BashTool.tsx` | `runShellCommand` |
| 文件编辑校验 | `src/tools/FileEditTool/FileEditTool.ts` | `validateInput`, `call` |
| 字符串匹配 | `src/tools/FileEditTool/utils.ts` | `findActualString`, `getPatchForEdit` |
| Agent 运行器 | `src/tools/AgentTool/runAgent.ts` | `runAgent` |
| Fork 子 Agent | `src/tools/AgentTool/forkSubagent.ts` | `buildForkedMessages` |
| 后台任务管理 | `src/tasks/LocalAgentTask/LocalAgentTask.ts` | `registerAsyncAgent`, `registerAgentForeground` |

---

*文档生成时间：2026-04-14*  
*基于源码版本：Claude Code v2.1.88*
