# Claude Code 整体架构全景图

> 源码总量: 1884 个 TypeScript 文件 | 37 个顶级模块 | 40+ 工具 | 20+ 服务

---

## 一、系统全景架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            用户入口 (Entry Points)                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
│  │  CLI 命令 │  │ VSCode   │  │  Web App │  │ Desktop  │  │ Agent SDK│     │
│  │  claude   │  │ Extension│  │ claude.ai│  │ App      │  │ API 调用  │     │
│  └─────┬────┘  └─────┬────┘  └─────┬────┘  └─────┬────┘  └─────┬────┘     │
│        └──────────────┴──────────────┴──────────────┴──────────────┘         │
│                                    │                                         │
│                          src/entrypoints/cli.tsx                             │
│                          (快速路径分流 / 主入口)                              │
│                                    │                                         │
│                              src/main.tsx                                    │
│                          (React/Ink 应用初始化)                              │
└────────────────────────────────────┬────────────────────────────────────────┘
                                     │
┌────────────────────────────────────┴────────────────────────────────────────┐
│                          应用层 (Application Layer)                          │
│                                                                              │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  REPL 模式  │  │ Headless 模式│  │  Daemon 模式 │  │ Remote 模式  │     │
│  │ 交互式终端  │  │ 非交互式 CLI │  │ 后台守护进程 │  │ 远程桥接     │     │
│  │replLauncher │  │  print.ts    │  │ daemon       │  │ bridge/rc    │     │
│  └──────┬──────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘     │
│         └────────────────┴─────────────────┴─────────────────┘              │
│                                    │                                         │
│                    ┌───────────────┴───────────────┐                         │
│                    │     核心查询引擎 (Query Engine) │                         │
│                    │       src/query.ts             │                         │
│                    │  ┌─────────────────────────┐   │                         │
│                    │  │ 1. 组装 system prompt    │   │                         │
│                    │  │ 2. 组装 messages          │   │                         │
│                    │  │ 3. 调用 Claude API        │   │                         │
│                    │  │ 4. 解析 tool_use blocks   │   │                         │
│                    │  │ 5. 调度工具执行           │   │                         │
│                    │  │ 6. 收集结果继续对话       │   │                         │
│                    │  └─────────────────────────┘   │                         │
│                    └───────────────┬───────────────┘                         │
└────────────────────────────────────┬────────────────────────────────────────┘
                                     │
         ┌───────────────────────────┼───────────────────────────┐
         │                           │                           │
┌────────┴────────┐     ┌───────────┴──────────┐    ┌──────────┴──────────┐
│  提示词系统      │     │   工具执行层          │    │   API 通信层         │
│ (Prompt Layer)  │     │   (Tool Execution)   │    │   (API Layer)        │
│                 │     │                      │    │                      │
│ src/constants/  │     │ src/services/tools/  │    │ src/services/api/    │
│  prompts.ts     │     │  toolOrchestration   │    │  claude.ts           │
│  systemPrompt   │     │  toolExecution       │    │  client.ts           │
│  Sections.ts    │     │  toolHooks           │    │  bootstrap.ts        │
│                 │     │  StreamingToolExec   │    │  errors.ts           │
│ src/tools/*/    │     │                      │    │  usage.ts            │
│  prompt.ts      │     │ src/tools/ (40+工具) │    │                      │
│                 │     │  AgentTool           │    │ 权限、速率限制、      │
│ 静态+动态分区   │     │  BashTool            │    │ 重试、缓存           │
│ 全局缓存策略   │     │  FileEditTool         │    │                      │
│                 │     │  FileReadTool         │    └──────────────────────┘
│                 │     │  FileWriteTool        │
│                 │     │  GlobTool/GrepTool    │
│                 │     │  SkillTool            │
│                 │     │  TodoWriteTool        │
│                 │     │  WebSearchTool        │
│                 │     │  ...                  │
│                 │     │                      │
│                 │     │ 验证→权限→Hook→执行   │
│                 │     │ 并发/串行调度         │
└─────────────────┘     └──────────────────────┘
         │                           │
┌────────┴──────────────────────────┴────────────────────────────────────────┐
│                          服务层 (Service Layer)                              │
│                                                                              │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐              │
│  │ MCP 协议   │ │ Context    │ │ Analytics  │ │ LSP 语言   │              │
│  │ 第三方工具 │ │ 压缩/管理  │ │ 遥测统计   │ │ 服务协议   │              │
│  │ mcp/       │ │ compact/   │ │ analytics/ │ │ lsp/       │              │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘              │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐              │
│  │ Session    │ │ Memory     │ │ OAuth      │ │ Policy     │              │
│  │ 记忆管理   │ │ 记忆同步   │ │ 认证授权   │ │ 限制策略   │              │
│  │ SessionMem │ │ extractMem │ │ oauth/     │ │ policyLim  │              │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘              │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐              │
│  │ Auto Dream │ │ Magic Docs │ │ Settings   │ │ Team       │              │
│  │ 后台思考   │ │ 文档管理   │ │ 同步       │ │ 记忆同步   │              │
│  │ autoDream/ │ │ MagicDocs/ │ │ settingsSy │ │ teamMemory │              │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘              │
└──────────────────────────────────────────────────────────────────────────────┘
         │
┌────────┴────────────────────────────────────────────────────────────────────┐
│                        基础设施层 (Infrastructure)                           │
│                                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │ 状态管理     │  │ 权限系统     │  │ 配置管理     │  │ UI 框架      │   │
│  │ state/       │  │ permissions/ │  │ settings/    │  │ ink/ React   │   │
│  │ AppState     │  │ 规则引擎     │  │ 分层配置     │  │ 终端渲染     │   │
│  │ Store 模式   │  │ 分类器       │  │ 验证/缓存    │  │ 组件库       │   │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │ Skills 机制  │  │ Plugin 系统  │  │ Vim 模式     │  │ 远程会话     │   │
│  │ skills/      │  │ plugins/     │  │ vim/         │  │ remote/      │   │
│  │ 内置/用户    │  │ 市场/内置    │  │ 键位/操作    │  │ WebSocket    │   │
│  │ MCP 技能     │  │ 生命周期     │  │ 状态机       │  │ 权限桥接     │   │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │ Hooks 钩子   │  │ 沙箱执行     │  │ 任务管理     │  │ 键位绑定     │   │
│  │ 前置/后置    │  │ sandbox/     │  │ tasks/       │  │ keybindings/ │   │
│  │ 用户自定义   │  │ 隔离执行     │  │ 本地/远程    │  │ 默认映射     │   │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘   │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 二、目录结构与模块职责

### 顶级模块 (src/ 下 37 个目录)

| 目录 | 文件数 | 职责 |
|------|--------|------|
| **tools/** | 40+ 子目录 | 40+ 个工具实现，每个工具一个目录 |
| **services/** | 20+ 子目录 | 后台服务层 (API, MCP, 压缩, 遥测等) |
| **utils/** | 298 文件 | 工具函数库 (文件、路径、权限、设置等) |
| **components/** | - | React/Ink 终端 UI 组件 |
| **state/** | - | 全局状态管理 (AppState Store) |
| **context/** | - | React Context (通知、邮箱、弹窗等) |
| **hooks/** | - | React Hooks (useCanUseTool 等) |
| **constants/** | - | 常量定义 (提示词、系统配置) |
| **skills/** | - | 技能系统 (内置/加载/MCP) |
| **plugins/** | - | 插件系统 |
| **types/** | - | 全局类型定义 |
| **ink/** | - | 自定义 React 终端渲染器 |
| **vim/** | - | Vim 模式模拟 |
| **entrypoints/** | - | 应用入口 (CLI) |
| **cli/** | - | CLI 参数解析和快速路径 |
| **query/** | - | 查询引擎子模块 |
| **coordinator/** | - | 多 Agent 协调器模式 |
| **remote/** | - | 远程会话支持 (WebSocket) |
| **server/** | - | 本地服务器 (SDK/远程) |
| **screens/** | - | 主屏幕 (REPL, Doctor, Resume) |
| **native-ts/** | - | 原生 TypeScript 模块 |
| **keybindings/** | - | 键位绑定系统 |
| **migrations/** | - | 数据迁移 |
| **voice/** | - | 语音模式 |
| **schemas/** | - | JSON Schema 定义 |
| **moreright/** | - | 内部构建工具 |
| **upstreamproxy/** | - | 上游代理配置 |
| **outputStyles/** | - | 输出风格 |
| **bridge/** | - | 进程桥接 |
| **buddy/** | - | Buddy 陪伴系统 |
| **memdir/** | - | 记忆目录管理 |
| **bootstrap/** | - | 启动状态和初始化 |
| **commands/** | - | 斜杠命令实现 |
| **tasks/** | - | 任务类型 (Shell/Agent/Remote) |

---

## 三、核心数据流

### 3.1 主循环 (一次完整交互)

```
用户输入 (文本/斜杠命令)
    │
    ▼
┌─ src/entrypoints/cli.tsx ─────────────────────────────────┐
│  快速路径判断: --version? /help? daemon? bridge?           │
│  否 → 加载 main.tsx                                        │
└────────────────────┬───────────────────────────────────────┘
                     │
                     ▼
┌─ src/main.tsx ────────────────────────────────────────────┐
│  1. setup.ts → Node版本检查、终端初始化、权限验证           │
│  2. AppState 初始化 (状态管理)                              │
│  3. MCP 客户端连接                                         │
│  4. 插件加载                                               │
│  5. 启动 REPL / Headless / Dialog                          │
└────────────────────┬───────────────────────────────────────┘
                     │
                     ▼
┌─ src/replLauncher.tsx / src/cli/print.ts ─────────────────┐
│  交互模式: Ink React 渲染终端 UI                           │
│  Headless: 结构化 IO 输出                                  │
└────────────────────┬───────────────────────────────────────┘
                     │
                     ▼
┌─ 用户输入处理 ────────────────────────────────────────────┐
│  1. 斜杠命令? → commands.ts 处理                           │
│  2. 普通文本 → 构建 UserMessage                            │
│  3. 附件处理 (图片、文件、@mention)                         │
└────────────────────┬───────────────────────────────────────┘
                     │
                     ▼
┌─ src/query.ts → queryLoop() ──────────────────────────────┐
│                                                             │
│  ┌───────────────────────────────────────────────────────┐ │
│  │ while (对话未结束) {                                    │ │
│  │                                                         │ │
│  │   1. getSystemPrompt() → 组装系统提示词                 │ │
│  │      - 静态部分 (缓存)                                  │ │
│  │      - 动态部分 (每轮更新)                               │ │
│  │                                                         │ │
│  │   2. normalizeMessages() → 规范化消息历史               │ │
│  │      - 移除已删除消息                                   │ │
│  │      - 压缩长内容                                       │ │
│  │      - 附加系统提醒                                     │ │
│  │                                                         │ │
│  │   3. queryModel() → 调用 Claude API                     │ │
│  │      src/services/api/claude.ts                         │ │
│  │      - 流式接收响应                                     │ │
│  │      - 解析 text / tool_use blocks                      │ │
│  │                                                         │ │
│  │   4. 检测 stop_reason:                                  │ │
│  │      - "end_turn" → 展示回复，等待用户输入              │ │
│  │      - "tool_use"  → 进入工具执行流程 ──┐               │ │
│  │      - "max_tokens" → 继续生成           │              │ │
│  │                                          │              │ │
│  │   5. ← 工具结果返回 ← ──────────────────┘              │ │
│  │      tool_result 追加到 messages                         │ │
│  │      回到步骤 1 继续循环                                 │ │
│  │ }                                                       │ │
│  └───────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 工具执行数据流

```
Claude API 返回 tool_use blocks
         │
         ▼
┌─ toolOrchestration.ts ──────────────────────────────┐
│  partitionToolCalls():                                │
│  遍历所有 tool_use blocks                             │
│  按 isConcurrencySafe() 分组:                        │
│    Batch 1: [Read, Glob, Grep] → 并发执行             │
│    Batch 2: [Edit, Bash]        → 串行执行             │
│    Batch 3: [Read]              → 并发执行             │
└────────────────────┬─────────────────────────────────┘
                     │
          ┌──────────┴──────────┐
          │                     │
    runToolsConcurrently   runToolsSerially
    (Promise.allSettled)   (for...of 顺序)
          │                     │
          └──────────┬──────────┘
                     │
                     ▼
┌─ toolExecution.ts → runToolUse() ───────────────────┐
│                                                       │
│  1. findToolByName(tools, "Edit")                     │
│     → 在已注册工具列表中查找                           │
│                                                       │
│  2. inputSchema.safeParse(input)                      │
│     → Zod 校验 file_path/old_string/new_string 类型   │
│     失败 → 返回 InputValidationError                  │
│                                                       │
│  3. tool.validateInput(parsed, context)                │
│     → 工具自定义验证 (文件存在? 已读? 唯一?)           │
│     失败 → 返回具体错误码和消息                        │
│                                                       │
│  4. backfillObservableInput()                          │
│     → expandPath, 补充派生字段                         │
│                                                       │
│  5. runPreToolUseHooks(tool, input)                    │
│     → 用户配置的 hook (PreToolUse)                     │
│     → hook 可以: 修改 input / 拒绝 / 终止 / 通过      │
│                                                       │
│  6. resolveHookPermissionDecision()                    │
│     → 合并 hook 的权限决策                             │
│                                                       │
│  7. canUseTool() / checkPermissions()                  │
│     → 用户权限规则检查                                 │
│     → allow: 直接执行                                  │
│     → ask: 弹窗让用户确认                              │
│     → deny: 返回拒绝                                   │
│                                                       │
│  8. tool.call(input, context, canUseTool, msg, progress)│
│     → 执行工具核心逻辑                                 │
│     → onProgress 回调实时报告进度                      │
│                                                       │
│  9. runPostToolUseHooks(tool, input, result)           │
│     → 用户配置的 hook (PostToolUse)                    │
│                                                       │
│  10. 返回 MessageUpdate:                               │
│     - tool_result block (结果)                         │
│     - newMessages (附带消息, 如 Skill)                 │
│     - contextModifier (修改上下文)                     │
└───────────────────────────────────────────────────────┘
```

---

## 四、40+ 工具分类

### 文件操作 (4 个)
| 工具 | 功能 |
|------|------|
| Read | 读取文件 (文本/PDF/图片/Notebook) |
| Edit | 精确字符串替换编辑 |
| Write | 写入/创建文件 |
| NotebookEdit | Jupyter Notebook 编辑 |

### 搜索 (3 个)
| 工具 | 功能 |
|------|------|
| Glob | 文件名模式匹配搜索 |
| Grep | 文件内容正则搜索 (基于 ripgrep) |
| ToolSearch | 动态工具发现和加载 |

### 执行 (2 个)
| 工具 | 功能 |
|------|------|
| Bash | Shell 命令执行 (含沙箱) |
| PowerShell | Windows PowerShell 执行 |

### 代理 (1 个)
| 工具 | 功能 |
|------|------|
| Agent | 启动子代理 (Explore/Plan/general/fork) |

### 任务管理 (4 个)
| 工具 | 功能 |
|------|------|
| TaskCreate | 创建任务 |
| TaskUpdate | 更新任务状态 |
| TaskList | 列出任务 |
| TaskGet | 获取单个任务 |

### 团队协作 (3 个)
| 工具 | 功能 |
|------|------|
| TeamCreate | 创建团队 |
| TeamDelete | 删除团队 |
| SendMessage | 团队内消息传递 |

### 规划 (3 个)
| 工具 | 功能 |
|------|------|
| EnterPlanMode | 进入规划模式 |
| ExitPlanMode | 退出规划模式 |
| TodoWrite | 待办事项管理 |

### 用户交互 (1 个)
| 工具 | 功能 |
|------|------|
| AskUserQuestion | 向用户提问获取输入 |

### 技能/外部 (3 个)
| 工具 | 功能 |
|------|------|
| Skill | 执行斜杠命令/技能 |
| MCPTool | 调用 MCP 服务器工具 |
| WebSearch | 网络搜索 |

### 远程/会话 (5 个)
| 工具 | 功能 |
|------|------|
| TaskOutput | 获取后台任务输出 |
| TaskStop | 停止后台任务 |
| ScheduleCron | 定时任务 |
| RemoteTrigger | 远程触发 |
| WebFetch | 获取网页内容 |

### MCP 资源 (3 个)
| 工具 | 功能 |
|------|------|
| ListMcpResources | 列出 MCP 资源 |
| ReadMcpResource | 读取 MCP 资源 |
| McpAuth | MCP 认证 |

### Git/工作区 (2 个)
| 工具 | 功能 |
|------|------|
| EnterWorktree | 进入 Git Worktree |
| ExitWorktree | 退出 Git Worktree |

### 其他 (6 个)
| 工具 | 功能 |
|------|------|
| Config | 配置管理 |
| LSPTool | 语言服务器操作 |
| Sleep | 延迟等待 |
| Brief | 简报生成 |
| SyntheticOutput | 合成输出 |
| BriefTool | 摘要工具 |

---

## 五、20+ 服务分类

### 核心 (4 个)
| 服务 | 目录 | 功能 |
|------|------|------|
| API 通信 | api/ | Claude API 调用、流式、缓存、重试 |
| MCP 协议 | mcp/ | 第三方工具集成 (SSE/Stdio/WebSocket) |
| Context 压缩 | compact/ | 自动压缩上下文、微压缩、消息分组 |
| 工具执行 | tools/ | 工具编排、执行、Hook |

### 智能 (4 个)
| 服务 | 目录 | 功能 |
|------|------|------|
| 会话记忆 | SessionMemory/ | 自动维护对话笔记 |
| 记忆提取 | extractMemories/ | 从对话提取持久记忆 |
| 后台思考 | autoDream/ | 后台记忆整合 |
| 文档管理 | MagicDocs/ | 自动更新项目文档 |

### 基础设施 (7 个)
| 服务 | 目录 | 功能 |
|------|------|------|
| 遥测分析 | analytics/ | Datadog、GrowthBook、事件追踪 |
| 语言服务 | lsp/ | Language Server Protocol 集成 |
| OAuth 认证 | oauth/ | 认证流程、令牌管理 |
| 策略限制 | policyLimits/ | 组织级功能限制 |
| 设置同步 | settingsSync/ | 跨环境设置同步 |
| 团队记忆 | teamMemorySync/ | 团队记忆同步 |
| 插件系统 | plugins/ | 插件安装管理 |

### 辅助 (5 个)
| 服务 | 目录 | 功能 |
|------|------|------|
| 提示建议 | PromptSuggestion/ | 输入建议 |
| 代理摘要 | AgentSummary/ | 子代理结果摘要 |
| 工具摘要 | toolUseSummary/ | 工具使用摘要 |
| 提示技巧 | tips/ | 使用提示 |
| 托管设置 | remoteManagedSettings/ | 远程配置管理 |

---

## 六、状态管理架构

```
┌──────────────────────────────────────────────────┐
│                  AppState (单一状态源)             │
│              src/state/AppStateStore.ts            │
│                                                    │
│  70+ 属性:                                         │
│  ├── 工具权限上下文 (toolPermissionContext)         │
│  ├── MCP 连接状态 (mcp)                            │
│  ├── 消息历史 (messages)                           │
│  ├── 会话 ID (sessionId)                           │
│  ├── 当前模型 (model)                              │
│  ├── 任务列表 (tasks)                              │
│  ├── 插件状态 (plugins)                            │
│  ├── 通知队列 (notifications)                      │
│  ├── 文件读取状态 (readFileState)                   │
│  ├── 努力级别 (effortValue)                        │
│  ├── UI 状态 (expanded views, selection)           │
│  └── ...                                           │
│                                                    │
│  更新模式: immutable setState(prev => {...prev})   │
│  副作用: onChangeAppState.ts 集中处理               │
│  选择器: selectors.ts 计算派生状态                  │
└──────────────────────────────────────────────────┘
```

---

## 七、权限系统架构

```
┌──────────────────────────────────────────────────────────┐
│                       权限决策流程                         │
│                                                           │
│  用户输入 → 权限模式选择:                                  │
│  ├── default     → 每次询问用户                            │
│  ├── plan        → 只允许计划模式工具                      │
│  ├── acceptEdits → 自动接受编辑                            │
│  ├── bypassPerms → 绕过所有权限 (危险)                     │
│  ├── dontAsk     → 使用已有规则，不弹窗                    │
│  └── auto        → 自动模式 + 分类器                      │
│                                                           │
│  规则引擎:                                                │
│  ├── alwaysAllowRules (始终允许)                           │
│  ├── alwaysDenyRules  (始终拒绝)                           │
│  ├── 按工具名匹配                                         │
│  ├── 按内容匹配 (文件路径/命令/Skill 名)                   │
│  └── 通配符模式 (/.claude/**, **/*.ts)                     │
│                                                           │
│  Hook 系统:                                               │
│  ├── PreToolUse  → 前置检查，可修改 input 或拒绝           │
│  ├── PostToolUse → 后置处理，可修改结果                    │
│  └── Notification → 事件通知                               │
│                                                           │
│  分类器 (Bash 自动模式):                                   │
│  └── startSpeculativeClassifierCheck()                    │
│      → 并行运行安全分类                                    │
│      → 决定 allow/deny/ask                                 │
└──────────────────────────────────────────────────────────┘
```

---

## 八、技能 (Skills) 架构

```
┌──────────────────────────────────────────────────────────┐
│                      Skills 生命周期                       │
│                                                           │
│  来源:                                                    │
│  ├── Bundled Skills (编译内置)                            │
│  │   initBundledSkills() → registerBundledSkill()         │
│  │   15+ 内置技能 (commit, debug, simplify, skillify...)  │
│  │                                                        │
│  ├── User Skills (用户定义 .md 文件)                       │
│  │   .claude/skills/*.md / .claude/commands/*.md           │
│  │   loadSkillsDir() → 解析 frontmatter + Markdown        │
│  │                                                        │
│  ├── MCP Skills (MCP 服务器提供)                           │
│  │   mcpSkillBuilders.ts → 桥接加载                        │
│  │                                                        │
│  ├── Plugin Skills (插件系统)                              │
│  │   市场/内置插件注册                                     │
│  │                                                        │
│  └── Remote Skills (远程加载, 实验性)                      │
│      AKI/GCS → loadRemoteSkill()                          │
│                                                           │
│  执行模式:                                                │
│  ├── inline (默认) → 展开为 UserMessage 注入当前对话       │
│  └── fork        → 启动独立子 Agent (runAgent)             │
│                                                           │
│  触发方式:                                                │
│  ├── 用户输入 /skill-name                                 │
│  ├── Claude 主动调用 SkillTool                            │
│  └── 自动激活 (条件匹配: 路径/文件类型)                    │
└──────────────────────────────────────────────────────────┘
```

---

## 九、构建与运行

```
源码 (TypeScript)
    │
    ▼
scripts/build.mjs (esbuild)
    │
    ├── feature('X') → false  (条件编译死代码消除)
    ├── MACRO.VERSION → "1.0.33"  (宏替换)
    ├── Bun runtime APIs → polyfill
    │
    ▼
dist/cli.js (单文件打包)
    │
    ▼
Bun 运行时执行
```

### 运行模式

| 模式 | 入口 | 用途 |
|------|------|------|
| **Interactive** | `claude` → REPL | 交互式终端，Ink React UI |
| **Headless** | `claude -p "msg"` | 非交互式，结构化输出 |
| **Daemon** | `claude daemon` | 后台守护进程，长连接 |
| **Bridge** | `claude bridge/rc` | 远程控制桥接 |
| **VSCode** | 扩展调用 | IDE 内嵌 |
| **SDK** | Agent SDK API | 程序化调用 |
