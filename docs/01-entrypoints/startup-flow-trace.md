# Claude Code 启动全链路追踪：从 CLI 入口到 REPL 交互

> 基于源码 `src/entrypoints/cli.tsx` → `src/entrypoints/init.ts` → `src/main.tsx` → `src/replLauncher.tsx` 的逐步分析

---

## 全局启动流程总览

```
用户执行 `claude` 命令
        │
        ▼
┌──────────────────────────────────────┐
│  Phase 0: cli.tsx — 快速路径分发      │  零/最小 import
│  --version / --dump-system-prompt    │
│  remote-control / daemon / bg / mcp  │
└──────────────┬───────────────────────┘
               │ 没有匹配快速路径
               ▼
┌──────────────────────────────────────┐
│  Phase 1: cli.tsx — 预加载 main.tsx  │  动态 import
│  startCapturingEarlyInput()          │  抢先捕获 stdin
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│  Phase 2: main.tsx — 环境与安全初始化  │  ~200 行
│  Windows 安全 / 信号处理 / URL 解析   │
│  SSH / Assistant / DirectConnect     │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│  Phase 3: init.ts — 基础设施初始化    │  ~240 行
│  配置 / 环境变量 / 遥测 / 网络       │
│  OAuth / mTLS / Proxy / LSP         │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│  Phase 4: main.tsx — Commander 解析  │  ~4000 行
│  CLI 参数解析 / Flag 验证            │
│  子命令注册 (mcp/auth/plugin/...)    │
└──────────────┬───────────────────────┘
               │ 主命令 action handler
               ▼
┌──────────────────────────────────────┐
│  Phase 5: main.tsx — Session 准备    │
│  认证 / GrowthBook / 工具注册        │
│  MCP 服务器 / 权限模式 / 信任对话框  │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│  Phase 6: launchRepl() — 启动 REPL   │
│  React/Ink 渲染 / 主循环开始         │
└──────────────────────────────────────┘
```

---

## Phase 0: cli.tsx — 快速路径分发

### 源码位置
`src/entrypoints/cli.tsx:1-303`

### 第一步：顶级副作用（在任何函数之前执行）

```typescript
// 1. 修复 corepack auto-pinning 问题
process.env.COREPACK_ENABLE_AUTO_PIN = '0'

// 2. 远程容器环境：设置子进程最大堆内存
if (process.env.CLAUDE_CODE_REMOTE === 'true') {
  process.env.NODE_OPTIONS += ' --max-old-space-size=8192'
}

// 3. 消融实验基线：设置简化模式环境变量
if (feature('ABLATION_BASELINE') && process.env.CLAUDE_CODE_ABLATION_BASELINE) {
  // 批量设置: SIMPLE, DISABLE_THINKING, DISABLE_COMPACT, DISABLE_AUTO_MEMORY, ...
}
```

**为什么在顶级执行**：BashTool/AgentTool/PowerShellTool 在 import 时将 `DISABLE_BACKGROUND_TASKS` 捕获为模块级常量，`init()` 运行时已经太晚了。

### 第二步：快速路径分发

```typescript
async function main() {
  const args = process.argv.slice(2)

  // ── Fast-path 1: --version（零 import）──
  if (args.length === 1 && (args[0] === '--version' || args[0] === '-v')) {
    console.log(`${MACRO.VERSION} (Claude Code)`)  // MACRO 在构建时内联
    return
  }

  // ── 加载性能分析器 ──
  const { profileCheckpoint } = await import('../utils/startupProfiler.js')
  profileCheckpoint('cli_entry')

  // ── Fast-path 2: --dump-system-prompt（导出系统提示词）──
  if (feature('DUMP_SYSTEM_PROMPT') && args[0] === '--dump-system-prompt') { ... }

  // ── Fast-path 3: Chrome MCP / Native Host ──
  if (process.argv[2] === '--claude-in-chrome-mcp') { ... }
  if (process.argv[2] === '--chrome-native-host') { ... }

  // ── Fast-path 4: --daemon-worker（daemon 子工作进程）──
  if (feature('DAEMON') && args[0] === '--daemon-worker') { ... }

  // ── Fast-path 5: remote-control / bridge（远程控制）──
  if (feature('BRIDGE_MODE') && args[0] === 'remote-control') { ... }

  // ── Fast-path 6: daemon（长驻守护进程）──
  if (feature('DAEMON') && args[0] === 'daemon') { ... }

  // ── Fast-path 7: bg sessions（后台会话管理）──
  if (feature('BG_SESSIONS') && args[0] === 'ps' || ...) { ... }

  // ── Fast-path 8: templates（模板任务）──
  if (feature('TEMPLATES') && args[0] === 'new' || ...) { ... }

  // ── Fast-path 9: environment-runner / self-hosted-runner ──
  if (feature('BYOC_ENVIRONMENT_RUNNER') && args[0] === 'environment-runner') { ... }

  // ── Fast-path 10: --worktree --tmux（tmux 工作树）──
  if (hasTmuxFlag && args.includes('--worktree')) { ... }

  // ── Fast-path 11: --update / --upgrade 重定向 ──
  if (args[0] === '--update' || args[0] === '--upgrade') {
    process.argv = [..., 'update']  // 重定向到 update 子命令
  }

  // ── Fast-path 12: --bare（简化模式）──
  if (args.includes('--bare')) {
    process.env.CLAUDE_CODE_SIMPLE = '1'
  }
  ...
}
```

### 第三步：加载完整 CLI

```typescript
  // 抢先捕获 stdin 输入（用户可能在 CLI 加载时就已经开始打字）
  const { startCapturingEarlyInput } = await import('../utils/earlyInput.js')
  startCapturingEarlyInput()

  // 动态导入 main.tsx（这是最大的 import，~135ms）
  profileCheckpoint('cli_before_main_import')
  const { main: cliMain } = await import('../main.js')
  profileCheckpoint('cli_after_main_import')

  // 启动主函数
  await cliMain()
  profileCheckpoint('cli_after_main_complete')
```

**设计要点**：
- 所有 import 都是**动态的**，最小化模块加载时间
- `--version` 零 import，直接返回
- `startCapturingEarlyInput()` 在加载 main.tsx 的同时捕获用户输入

---

## Phase 1: main.tsx — 环境与安全初始化

### 源码位置
`src/main.tsx:1-200`（import + 副作用）、`src/main.tsx:585-800`（main 函数前半）

### 第四步：模块级副作用（import 时执行）

```typescript
// main.tsx 顶部，import 时立即执行

// 1. 性能检查点：标记 main.tsx 入口
profileCheckpoint('main_tsx_entry')

// 2. 启动 MDM 读取（macOS 企业管理配置）
//    发起 plutil/reg query 子进程，与后续 import 并行运行
startMdmRawRead()

// 3. 启动 Keychain 预读（macOS OAuth + API key）
//    否则在 applySafeConfigEnvironmentVariables() 中会同步串行读取（~65ms）
startKeychainPrefetch()
```

### 第五步：main() 函数初始化

```typescript
export async function main() {
  // 1. Windows 安全：防止从当前目录执行命令（PATH 劫持防护）
  process.env.NoDefaultCurrentDirectoryInExePath = '1'

  // 2. 初始化 React 警告处理器
  initializeWarningHandler()

  // 3. 进程退出处理
  process.on('exit', () => resetCursor())  // 恢复终端光标
  process.on('SIGINT', () => {
    if (process.argv.includes('-p')) return  // print 模式有自己的处理
    process.exit(0)
  })
```

### 第六步：URL / SSH / Assistant 参数预处理

```typescript
  // ── cc:// URL 处理（DirectConnect 模式）──
  if (feature('DIRECT_CONNECT')) {
    // 解析 cc://serverUrl?authToken=xxx
    // Headless (-p) → 重写为 `open` 子命令
    // Interactive → 保存 URL，后续传给 REPL
  }

  // ── Deep Link URI 处理（macOS 协议处理器）──
  if (feature('LODESTONE') && process.argv.includes('--handle-uri')) {
    // 解析 URI 并打开终端，然后退出
  }

  // ── Assistant 模式处理 ──
  if (feature('KAIROS') && rawArgs[0] === 'assistant') {
    // `claude assistant [sessionId]` → 保存 sessionId，strip 参数
  }

  // ── SSH 远程模式处理 ──
  if (feature('SSH_REMOTE') && rawCliArgs[0] === 'ssh') {
    // `claude ssh <host> [dir]` → 提取 host/dir/flags
    // 支持 --permission-mode, --dangerously-skip-permissions
    // 转发 --continue, --resume, --model 到远程 CLI
  }
```

---

## Phase 2: init.ts — 基础设施初始化

### 源码位置
`src/entrypoints/init.ts:57-238`

### 第七步：init() 函数（memoized，只执行一次）

```typescript
export const init = memoize(async (): Promise<void> => {
  // ── 7a. 启用配置系统 ──
  enableConfigs()

  // ── 7b. 应用安全环境变量（信任对话框之前）──
  applySafeConfigEnvironmentVariables()  // 只应用"安全"变量
  applyExtraCACertsFromConfig()          // NODE_EXTRA_CA_CERTS

  // ── 7c. 注册优雅退出 ──
  setupGracefulShutdown()

  // ── 7d. 初始化第一方事件日志（异步，不阻塞）──
  void Promise.all([
    import('../services/analytics/firstPartyEventLogger.js'),
    import('../services/analytics/growthbook.js'),
  ]).then(([fp, gb]) => {
    fp.initialize1PEventLogging()
    gb.onGrowthBookRefresh(() => fp.reinitialize1PEventLoggingIfConfigChanged())
  })

  // ── 7e. 填充 OAuth 账户信息（异步）──
  void populateOAuthAccountInfoIfNeeded()

  // ── 7f. 检测 JetBrains IDE（异步，缓存结果）──
  void initJetBrainsDetection()

  // ── 7g. 检测 Git 仓库（异步，缓存结果）──
  void detectCurrentRepository()

  // ── 7h. 初始化远程管理设置加载 Promise ──
  if (isEligibleForRemoteManagedSettings()) {
    initializeRemoteManagedSettingsLoadingPromise()
  }

  // ── 7i. 策略限制加载 ──
  if (isPolicyLimitsEligible()) {
    initializePolicyLimitsLoadingPromise()
  }

  // ── 7j. 记录首次启动时间 ──
  recordFirstStartTime()

  // ── 7k. 配置 mTLS ──
  configureGlobalMTLS()

  // ── 7l. 配置 HTTP 代理 ──
  configureGlobalAgents()

  // ── 7m. 预连接 Anthropic API（TCP+TLS 握手，~100-200ms）──
  preconnectAnthropicApi()  // 与后续工作并行

  // ── 7n. CCR 上游代理初始化 ──
  if (process.env.CLAUDE_CODE_REMOTE === 'true') {
    await initUpstreamProxy()
  }

  // ── 7o. Windows Git Bash 设置 ──
  setShellIfWindows()

  // ── 7p. 注册清理回调 ──
  registerCleanup(shutdownLspServerManager)
  registerCleanup(cleanupSessionTeams)

  // ── 7q. Scratchpad 目录初始化 ──
  if (isScratchpadEnabled()) {
    await ensureScratchpadDir()
  }
})
```

**关键设计**：
- 大量异步操作用 `void` 标记为 fire-and-forget，不阻塞主流程
- `preconnectAnthropicApi()` 提前完成 TCP+TLS 握手，节省首次 API 调用的 ~200ms
- MDM 读取和 Keychain 预读在 import 时就已启动

---

## Phase 3: main.tsx — Commander CLI 解析

### 源码位置
`src/main.tsx:884-4510`

### 第八步：run() — 构建 Commander 程序

```typescript
async function run(): Promise<CommanderCommand> {
  const program = new CommanderCommand()

  program
    .name('claude')
    .description('Claude Code - AI-powered coding assistant')
    .argument('[prompt]', 'Prompt to start a conversation')
    // 大量 option 定义...
    .option('-p, --print [prompt]', 'Print mode (headless)')
    .option('--model <model>', 'Model for the session')
    .option('--permission-mode <mode>', 'Permission mode')
    .option('-c, --continue', 'Continue most recent conversation')
    .option('-r, --resume [value]', 'Resume a conversation')
    .option('--allowedTools <tools...>', 'Allowed tools')
    .option('--disallowedTools <tools...>', 'Disallowed tools')
    .option('--mcp-config <configs...>', 'MCP server configs')
    // ... 50+ 个选项

    .action(async (prompt, options) => {
      // ── Phase 4 的主入口 ──
    })
```

### 子命令注册

```typescript
  // MCP 子命令
  mcp.command('serve').action(...)    // 启动 MCP 服务器
  mcp.command('add-json').action(...) // 添加 MCP 服务器
  mcp.command('list').action(...)     // 列出 MCP 服务器

  // Auth 子命令
  auth.command('login').action(...)   // 登录
  auth.command('status').action(...)  // 认证状态
  auth.command('logout').action(...)  // 登出

  // Plugin 子命令
  pluginCmd.command('install').action(...)
  pluginCmd.command('uninstall').action(...)

  // 其他
  program.command('update').action(...)     // 更新
  program.command('doctor').action(...)     // 健康检查
  program.command('agents').action(...)     // 代理列表
```

---

## Phase 4: main.tsx — 主 Action Handler

### 源码位置
`src/main.tsx:1006-3800`（action handler 内部）

### 第九步：Session 配置准备

```typescript
.action(async (prompt, options) => {
  // ── 9a. --bare 简化模式 ──
  if (options.bare) process.env.CLAUDE_CODE_SIMPLE = '1'

  // ── 9b. Assistant 模式初始化 ──
  if (feature('KAIROS') && assistantModule?.isAssistantMode()) {
    // 初始化团队上下文、加载助手设置
  }

  // ── 9c. 调用 init()（如果还没执行）──
  await init()

  // ── 9d. 认证检查 ──
  const hasApiKey = getClaudeAIOAuthTokens()?.accessToken || getApiKey()
  if (!hasApiKey) { /* 显示登录流程 */ }

  // ── 9e. GrowthBook Feature Flag 初始化 ──
  await initializeGrowthBook({ ... })

  // ── 9f. 订阅类型 & Beta 检查 ──
  const subscriptionType = getSubscriptionType()

  // ── 9g. 模型选择 ──
  const model = options.model || getMainLoopModel()

  // ── 9h. 工具注册 ──
  const tools = getTools({ permissionMode, ... })

  // ── 9i. MCP 服务器初始化 ──
  const mcpClients = await initializeMCPServers(...)

  // ── 9j. LSP 服务器管理器 ──
  initializeLspServerManager()

  // ── 9k. 插件加载 ──
  const plugins = await loadAllPluginsCacheOnly(...)

  // ── 9l. 权限模式配置 ──
  const permissionMode = options.permissionMode || getDefaultPermissionMode()

  // ── 9m. 信任对话框（如果是首次在该目录运行）──
  if (!checkHasTrustDialogAccepted(cwd)) {
    await showTrustDialog(cwd)
  }

  // ── 9n. 遥测初始化（信任后）──
  initializeTelemetryAfterTrust()
```

### 第十步：创建 React/Ink 应用

```typescript
  // ── 10a. 创建 AppState Store ──
  const store = createStore(getDefaultAppState())

  // ── 10b. 创建 Ink Root ──
  const root = createRoot()

  // ── 10c. 构建初始消息列表 ──
  const initialMessages = []

  // ── 10d. 恢复早期输入 ──
  const earlyInput = seedEarlyInput()
  if (earlyInput) {
    initialMessages.push(createUserMessage(earlyInput))
  }
```

---

## Phase 5: launchRepl() — 启动交互循环

### 源码位置
`src/replLauncher.tsx:12`

### 第十一步：REPL 启动

```typescript
export async function launchRepl(
  root: Root,
  appProps: AppWrapperProps,
  replProps: REPLProps,
  renderAndRun: (root: Root, element: React.ReactNode) => Promise<void>
): Promise<void>
```

### 多种启动路径

main.tsx 中有 **7 处不同的 launchRepl() 调用**：

| 路径 | 场景 | 触发条件 |
|------|------|----------|
| `--continue` | 继续上次对话 | `options.continue` |
| `--resume` | 恢复指定会话 | `options.resume` |
| `DirectConnect` | 连接远程服务器 | `cc://` URL |
| `SSH` | SSH 远程会话 | `claude ssh host` |
| `Assistant` | 助手模式 | `claude assistant` |
| `--print` | 无头模式 | `options.print` |
| **默认** | 新对话 | 无特殊 flag |

每条路径都传递不同的 `initialMessages` 和 `sessionConfig`，但最终都汇入同一个 REPL。

### 第十二步：React/Ink 渲染

```
launchRepl()
    │
    ├─ 创建 <AppWrapper> 组件
    │   ├─ AppState Provider
    │   ├─ Notification Provider
    │   └─ <App> 组件
    │       ├─ <REPL> 主循环
    │       │   ├─ 消息列表渲染
    │       │   ├─ 工具执行状态
    │       │   └─ 输入提示符
    │       ├─ <PermissionDialog> 权限对话框
    │       └─ <ToolJSX> 工具自定义 UI
    │
    └─ renderAndRun(root, element)
        └─ Ink 渲染到终端
```

### 第十三步：主循环开始

```
REPL 主循环
    │
    ▼
┌──────────────────────────────────┐
│ 等待用户输入                      │
│ ├─ 命令行输入 (prompt)            │
│ ├─ /命令 (slash command)         │
│ ├─ stdin (pipe 模式)             │
│ └─ 早期输入 (early input)         │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│ processUserInput()               │
│ ├─ 解析输入类型                   │
│ ├─ /命令 → processSlashCommand() │
│ └─ 普通消息 → 构造 UserMessage   │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│ query() — 发送到 Claude API      │
│ ├─ 组装系统提示词                 │
│ ├─ 组装消息历史                   │
│ ├─ 注册工具列表                   │
│ ├─ 调用 Anthropic API            │
│ ├─ 接收流式响应                   │
│ └─ 处理 tool_use blocks          │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│ runToolUse() — 工具调用管线       │
│ (见 02-tools/tool-call-pipeline) │
└──────────────┬───────────────────┘
               │
               ▼
          返回 REPL 主循环
```

---

## 启动性能优化手段总结

| 优化手段 | 位置 | 节省时间 |
|----------|------|----------|
| **零 import 快速路径** | cli.tsx --version | 全部加载时间 |
| **动态 import** | cli.tsx 所有子命令 | 非主路径的模块加载 |
| **MDM 并行读取** | main.tsx import 时 | ~50ms |
| **Keychain 并行预读** | main.tsx import 时 | ~65ms |
| **API 预连接** | init.ts | ~200ms TLS 握手 |
| **早期输入捕获** | cli.tsx | 用户打字不丢失 |
| **Fire-and-forget 异步** | init.ts 多处 | 不阻塞主流程 |
| **Lazy import** | 遥测/OTel/gRPC | ~1.1MB 延迟加载 |
| **Feature Gate DCE** | feature() 标记 | 构建时消除死代码 |
| **profileCheckpoint** | 全链路 | 性能瓶颈定位 |

---

## 启动耗时分布（估算）

```
┌─────────────────────────────────────────┐
│ cli.tsx 快速路径检查          ~1ms       │
├─────────────────────────────────────────┤
│ 动态 import main.tsx          ~135ms     │  ← 最大的瓶颈
│  (含 ~200 个模块的评估)                   │
├─────────────────────────────────────────┤
│ init() 基础设施初始化          ~50ms      │  (并行操作多)
├─────────────────────────────────────────┤
│ Commander 参数解析             ~5ms       │
├─────────────────────────────────────────┤
│ 认证 + GrowthBook              ~100ms     │  (网络请求)
├─────────────────────────────────────────┤
│ 工具注册 + MCP 服务器           ~200ms     │  (MCP 连接)
├─────────────────────────────────────────┤
│ Ink 渲染 + REPL 启动           ~20ms      │
├─────────────────────────────────────────┤
│ 总计                          ~510ms     │
└─────────────────────────────────────────┘
```
