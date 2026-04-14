# Claude Code 源码学习：UI 层 (`REPL.tsx` / `App.tsx` / 权限弹窗)

> Claude Code 不是一个普通的命令行程序，而是一个**在终端里运行的 React 应用**。它使用 `ink`（基于 React 的终端 UI 库）来渲染消息流、输入框、权限弹窗和转圈圈动画。这一章我们揭开终端 React 应用的神秘面纱。

---

## 一、整体渲染架构

### 启动链路（从 `main.tsx` 到屏幕）

```ts
// src/replLauncher.tsx
export async function launchRepl(root, appProps, replProps, renderAndRun) {
  const { App } = await import('./components/App.js')
  const { REPL } = await import('./screens/REPL.js')
  await renderAndRun(root,
    <App {...appProps}>
      <REPL {...replProps} />
    </App>
  )
}
```

三层包装：

```
┌─────────────────────────────────────────┐
│  ink Root（终端画布）                    │
├─────────────────────────────────────────┤
│  App.tsx                                │
│  ├─ FpsMetricsProvider                  │
│  ├─ StatsProvider                       │
│  └─ AppStateProvider (全局状态)          │
├─────────────────────────────────────────┤
│  REPL.tsx (5,000+ 行)                   │
│  ├─ VirtualMessageList (消息列表)        │
│  ├─ PromptInput (输入框)                 │
│  ├─ PermissionRequest (权限弹窗)         │
│  ├─ Spinner (转圈圈)                     │
│  └─ 各种 Dialog / Notification           │
└─────────────────────────────────────────┘
```

### App.tsx：极简的 Provider 树

```tsx
export function App({ getFpsMetrics, stats, initialState, children }) {
  return (
    <FpsMetricsProvider getFpsMetrics={getFpsMetrics}>
      <StatsProvider store={stats}>
        <AppStateProvider initialState={initialState} onChangeAppState={onChangeAppState}>
          {children}
        </AppStateProvider>
      </StatsProvider>
    </FpsMetricsProvider>
  )
}
```

`AppStateProvider` 是整个应用的心脏，所有跨组件的状态（消息历史、权限上下文、MCP 连接、任务列表）都从这里分发。

---

## 二、REPL.tsx：5,000 行代码的终端帝国

`REPL.tsx` 是 Claude Code 的**主界面组件**。它负责：
1. 维护本地状态（messages、tools、commands、screen mode）
2. 处理用户输入（`onSubmit`）
3. 调用 `QueryEngine` / `ask()` 进入 Agent Loop
4. 消费 loop 产出的事件并更新 UI
5. 管理权限弹窗队列
6. 渲染消息列表和输入框

### 2.1 核心状态一览

```ts
export function REPL({ ... }): React.ReactNode {
  // 从全局 AppState 读取的关键状态
  const toolPermissionContext = useAppState(s => s.toolPermissionContext)
  const verbose = useAppState(s => s.verbose)
  const mcp = useAppState(s => s.mcp)
  const agentDefinitions = useAppState(s => s.agentDefinitions)
  const tasks = useAppState(s => s.tasks)
  const queuedCommands = useCommandQueue()
  
  // 本地状态
  const [screen, setScreen] = useState<Screen>('prompt')  // prompt | transcript
  const [messages, setMessages] = useState<Message[]>(initialMessages)
  const [localTools, setLocalTools] = useState(initialTools)
  const [abortController, setAbortController] = useState(createAbortController())
  
  // 权限/交互弹窗队列
  const [toolUseConfirmQueue, setToolUseConfirmQueue] = useState<ToolUseConfirm[]>([])
  const [sandboxPermissionRequestQueue, setSandboxPermissionRequestQueue] = useState(...)
  const [promptQueue, setPromptQueue] = useState(...)
  const [elicitation, setElicitation] = useState(...)
}
```

### 2.2 用户输入处理链路：`onSubmit` → `handlePromptSubmit`

当用户在 `PromptInput` 里按回车时：

```ts
const onSubmit = useCallback(async (input: string, helpers: PromptInputHelpers) => {
  // 1. 预处理：expand pasted text refs、trim
  const trimmedInput = expandPastedTextRefs(input, pastedContents).trim()
  
  // 2. 判断是 slash command 还是普通 prompt
  const isSlashCommand = !speculationAccept && input.trim().startsWith('/')
  
  // 3. 如果是远程模式（Remote/Bridge），直接转发
  if (activeRemote.isRemoteMode && !isSlashCommand) {
    activeRemote.send(input)
    return
  }
  
  // 4. 调用 handlePromptSubmit（核心！）
  await handlePromptSubmit({
    input,
    helpers,
    messages,
    setMessages,
    setLoading,
    abortController,
    setAbortController,
    canUseTool,
    tools: localTools,
    commands,
    onQuery: async ({ messages, abortController }) => {
      // 5. 创建 QueryEngine 并进入 Agent Loop
      for await (const msg of ask({
        prompt: input,
        mutableMessages: messages,
        tools: localTools,
        commands,
        abortController,
        // ...
      })) {
        // 6. 消费 SDKMessage，更新本地 messages
        setMessages(prev => [...prev, normalizeMessage(msg)])
      }
    },
    // ... 还有很多回调
  })
}, [messages, localTools, commands, ...])
```

`handlePromptSubmit` 在 `src/utils/handlePromptSubmit.ts` 中，它做了：
- 如果是 slash command，先执行 `processSlashCommand`
- 如果不是 slash command 但当前正在 loading，把输入**入队**（queued），等当前 turn 结束后再自动提交
- 否则直接调用 `onQuery` 进入 API 调用

### 2.3 消息消费：从 `ask()` 到 `setMessages()`

`ask()` 生成的是 `SDKMessage` 流。REPL 把它们转换成内部 `Message` 并追加到消息列表：

```ts
for await (const msg of ask({ ... })) {
  switch (msg.type) {
    case 'assistant':
      setMessages(prev => [...prev, msg])
      break
    case 'user':
      setMessages(prev => [...prev, msg])
      break
    case 'progress':
      // 工具进度消息（如 bash 实时输出）
      setMessages(prev => updateProgressMessage(prev, msg))
      break
    case 'result':
      // Turn 结束，重置 loading 状态
      setLoading(false)
      break
    // ...
  }
}
```

**关键设计**：`setMessages` 在流式过程中会被调用几十次。Claude Code 的消息列表使用 `VirtualMessageList`（虚拟滚动）来避免大量重渲染拖垮终端。

---

## 三、消息列表：VirtualMessageList

`REPL.tsx` 的消息区不是简单渲染一个数组，而是用 `VirtualMessageList` 做**虚拟滚动**。

### 为什么终端里需要虚拟滚动？

长会话可能有数百条消息。如果全部渲染：
- ink 的渲染引擎需要计算每个组件的输出
- 终端滚动历史会无限增长
- 帧率暴跌

`VirtualMessageList` 只渲染**当前视口可见**的消息行。

```ts
export function VirtualMessageList({
  messages,
  scrollRef,
  columns,
  itemKey,
  renderItem,
  onItemClick,
}) {
  const { visibleItems, totalHeight, scrollToIndex } = useVirtualScroll({
    items: messages,
    itemHeight: estimatedHeight,
    overscan: 3,
  })
  
  return (
    <Box flexDirection="column" height={totalHeight}>
      {visibleItems.map((msg, idx) => (
        <Box key={itemKey(msg)}>
          {renderItem(msg, idx)}
        </Box>
      ))}
    </Box>
  )
}
```

### 附加功能：搜索跳转

`VirtualMessageList` 还暴露了一个 `JumpHandle`，支持 `/` 搜索、`n/N` 跳转：

```ts
export type JumpHandle = {
  jumpToIndex: (i: number) => void
  setSearchQuery: (q: string) => void
  nextMatch: () => void
  prevMatch: () => void
  warmSearchIndex: () => Promise<number>
  disarmSearch: () => void
}
```

当你在 transcript 模式按 `/` 时，REPL 会调用 `jumpRef.current.setSearchQuery('foo')`，然后 `VirtualMessageList` 会：
1. 为每条消息提取可搜索文本（`renderableSearchText`）
2. 高亮匹配项
3. 自动滚动到第一个匹配

---

## 四、PromptInput：终端里的"富文本编辑器"

`PromptInput` 不是简单的 `readline`，它是一个功能丰富的输入组件：

### 核心能力

| 功能 | 实现 |
|------|------|
| **历史记录** | 上下箭头浏览之前的输入（`useArrowKeyHistory`） |
| **Tab 补全** | 斜杠命令、文件路径、Shell 历史（`useTypeahead`） |
| **图片粘贴** | 从剪贴板读取图片，resize 后作为 ContentBlockParam 发送 |
| **@ 提及** | `@agent-xxx` 语法触发子 Agent 提及（`useIdeAtMentioned`） |
| **引用展开** | `[Pasted text #1]` 自动展开为完整内容 |
| **Vim 模式** | 可选的 Vim 键绑定 |
| **彩虹输入** | `ultrathink` 关键词触发彩虹渐变效果 |

### PromptInput 的状态管理

```ts
export function PromptInput({ ... }) {
  const [inputValue, setInputValue, inputBuffer] = useInputBuffer(initialValue)
  const [mode, setMode] = useState<PromptInputMode>('prompt')
  const [vimMode, setVimMode] = useState<VimMode>('normal')
  
  // 各种 suggestion hooks
  const { suggestion, acceptSuggestion } = useTypeahead(inputValue, ...)
  const { historyItem, navigateHistory } = useArrowKeyHistory(inputValue, ...)
  const { searchQuery, searchMatches, navigateSearch } = useHistorySearch(...)
  
  // 图片/附件
  const [pastedContents, setPastedContents] = useState<Record<number, PastedContent>>({})
  
  // 当用户按 Enter 时
  const handleSubmit = () => {
    const contentBlocks = buildContentBlocks(inputValue, pastedContents)
    onSubmit(contentBlocks, helpers)
    setInputValue('')
    clearPastedContents()
  }
}
```

---

## 五、权限弹窗系统：PermissionRequest

权限弹窗是 Claude Code 产品体验的核心之一。当模型调用一个需要用户确认的工具时，UI 会弹出一个专门的权限请求组件。

### 5.1 弹窗队列机制

REPL 里维护了一个 `toolUseConfirmQueue`：

```ts
const [toolUseConfirmQueue, setToolUseConfirmQueue] = useState<ToolUseConfirm[]>([])
```

`useCanUseTool` hook（下一章详讲）在权限决策为 `ask` 时，会把请求推入这个队列：

```ts
const canUseTool = useCanUseTool(setToolUseConfirmQueue, setToolPermissionContext)
```

REPL 渲染时只显示队列里的第一个：

```tsx
const toolPermissionOverlay = focusedInputDialog === 'tool-permission'
  ? <PermissionRequest
      key={toolUseConfirmQueue[0]?.toolUseID}
      onDone={() => setToolUseConfirmQueue(([_, ...tail]) => tail)}
      onReject={handleQueuedCommandOnCancel}
      toolUseConfirm={toolUseConfirmQueue[0]!}
      toolUseContext={...}
      verbose={verbose}
      workerBadge={...}
      setStickyFooter={...}
    />
  : null
```

### 5.2 PermissionRequest：按工具类型分发

`PermissionRequest.tsx` 是一个**分发器**，根据工具类型返回不同的权限组件：

```ts
function permissionComponentForTool(tool: Tool): React.ComponentType<PermissionRequestProps> {
  switch (tool) {
    case FileEditTool:
      return FileEditPermissionRequest
    case FileWriteTool:
      return FileWritePermissionRequest
    case BashTool:
      return BashPermissionRequest
    case PowerShellTool:
      return PowerShellPermissionRequest
    case WebFetchTool:
      return WebFetchPermissionRequest
    case NotebookEditTool:
      return NotebookEditPermissionRequest
    case SkillTool:
      return SkillPermissionRequest
    case AskUserQuestionTool:
      return AskUserQuestionPermissionRequest
    case GlobTool:
    case GrepTool:
    case FileReadTool:
      return FilesystemPermissionRequest
    default:
      return FallbackPermissionRequest
  }
}
```

每个工具都有**定制化的弹窗 UI**：
- **BashPermissionRequest**：显示完整命令、是否destructive、是否在沙箱中运行
- **FileEditPermissionRequest**：显示 diff 预览（old_string → new_string）
- **FileWritePermissionRequest**：显示即将写入的文件内容预览
- **AskUserQuestionPermissionRequest**：显示问题和选项

### 5.3 Bash 权限弹窗示例

```tsx
// BashPermissionRequest.tsx 简化示意
export function BashPermissionRequest({ toolUseConfirm, onDone, onReject, ... }) {
  const { input, permissionResult } = toolUseConfirm
  
  return (
    <PermissionDialog
      title={`Allow ${input.command}?`}
      description={permissionResult.message}
      onAllow={() => onDone()}
      onDeny={() => onReject()}
      options={[
        { label: 'Allow', value: 'allow' },
        { label: 'Allow for this conversation', value: 'allow_session' },
        { label: 'Always allow in this directory', value: 'allow_dir' },
        { label: 'Deny', value: 'deny' },
      ]}
    />
  )
}
```

### 5.4 多重弹窗优先级

REPL 里有很多需要用户交互的场景，它们有明确的显示优先级：

```ts
function getFocusedInputDialog() {
  if (showingCostDialog) return 'cost'
  if (toolUseConfirmQueue[0]) return 'tool-permission'
  if (sandboxPermissionRequestQueue[0]) return 'sandbox-permission'
  if (promptQueue[0]) return 'prompt'
  if (workerSandboxPermissions.queue[0]) return 'worker-permission'
  if (elicitation.queue[0]) return 'elicitation'
  if (ultraplanPendingChoice) return 'ultraplan'
  return null
}
```

这保证了：
- 如果同时有成本阈值弹窗和权限弹窗，**成本弹窗先显示**
- 用户处理完一个，下一个自动顶上

---

## 六、输入框上方的 Spinner

当模型在思考或工具在执行时，REPL 会在 `PromptInput` 上方显示一个 spinner：

```tsx
<SpinnerWithVerb
  mode={streamMode}
  isLoading={isLoading}
  verb={activityDescription}
  tip={spinnerTip}
/>
```

`streamMode` 有几种状态：
- `requesting`：等待 API 响应
- `responding`：模型正在流式输出文本
- `tool_use`：模型调用了工具，正在等待工具执行
- `tool_use_progress`：工具执行中，有进度更新

`activityDescription` 来自当前执行中的工具的 `getActivityDescription()`，例如：
- "Running npm test"
- "Reading src/query.ts"
- "Searching for pattern"

---

## 七、UI 层的 4 个关键设计智慧

### 1. **在终端里用 React = 状态管理的一致性**
传统的 CLI 工具用命令式代码拼凑 UI，很容易陷入"回调地狱"。Claude Code 用 React + ink 把终端当成浏览器来开发，获得了完整的状态管理和组件复用能力。

### 2. **虚拟滚动是长会话的生命线**
`VirtualMessageList` 让 500+ 条消息的会话也能保持 60fps。这是从 Web 前端直接借鉴过来的技术，但在终端 UI 里极其少见。

### 3. **权限弹窗按工具类型定制**
不是用同一个通用弹窗套所有工具，而是为 Bash、FileEdit、FileWrite 等分别设计弹窗。用户看到的 diff 预览、命令详情、沙箱警告都是最贴合场景的信息。

### 4. **所有弹窗走队列而不是互相覆盖**
`toolUseConfirmQueue`、`sandboxPermissionRequestQueue`、`promptQueue` 等队列机制保证了用户不会被突如其来的弹窗打断当前操作，也不会丢失需要确认的权限请求。

---

## 八、关键代码索引

| 概念 | 文件 | 关键函数/类 |
|------|------|------------|
| 顶层 Provider | `src/components/App.tsx` | `App` |
| 主界面 | `src/screens/REPL.tsx` | `REPL`, `onSubmit` |
| 输入提交处理 | `src/utils/handlePromptSubmit.ts` | `handlePromptSubmit` |
| 输入框 | `src/components/PromptInput/PromptInput.tsx` | `PromptInput` |
| 虚拟消息列表 | `src/components/VirtualMessageList.tsx` | `VirtualMessageList`, `JumpHandle` |
| 权限弹窗分发 | `src/components/permissions/PermissionRequest.tsx` | `PermissionRequest`, `permissionComponentForTool` |
| Bash 权限弹窗 | `src/components/permissions/BashPermissionRequest/BashPermissionRequest.tsx` | `BashPermissionRequest` |
| 文件编辑权限弹窗 | `src/components/permissions/FileEditPermissionRequest/FileEditPermissionRequest.tsx` | `FileEditPermissionRequest` |
| Spinner | `src/components/Spinner.tsx` | `SpinnerWithVerb` |
| 通知系统 | `src/context/notifications.js` | `useNotifications` |

---

*文档生成时间：2026-04-14*  
*基于源码版本：Claude Code v2.1.88*
