# Claude Code 完整架构图

> 本文档基于 Claude Code v2.1.88 源码，用多层级架构图展示系统全貌。建议配合 `01-05` 专题文档逐层阅读。

---

## 一、系统分层总览

```mermaid
flowchart TB
    subgraph L4["L4: 用户交互层 (CLI / UI)"]
        A1["main.tsx<br/>CLI 入口、参数解析、初始化"] --> A2["App.tsx<br/>全局状态 Provider"]
        A2 --> A3["REPL.tsx<br/>终端 React 应用主界面"]
        A3 --> A4["PromptInput<br/>输入框、历史、补全"]
        A3 --> A5["VirtualMessageList<br/>虚拟滚动消息列表"]
        A3 --> A6["PermissionRequest<br/>权限弹窗系统"]
    end

    subgraph L3["L3: 编排层 (QueryEngine)"]
        B1["QueryEngine.submitMessage()<br/>会话生命周期管理"] --> B2["processUserInput()<br/>Slash 命令 / Prompt 分支"]
        B1 --> B3["ask()<br/>QueryEngine 便捷包装"]
        B1 --> B4["recordTranscript()<br/>持久化与恢复"]
    end

    subgraph L2["L2: 核心引擎层 (Agent Loop)"]
        C1["query() / queryLoop()<br/>异步生成器主循环"] --> C2["callModel()<br/>流式 API 调用"]
        C1 --> C3["StreamingToolExecutor<br/>边流边执行工具"]
        C1 --> C4["Autocompact / Reactive Compact<br/>上下文压缩与急救"]
        C1 --> C5["Stop Hooks<br/>计划模式刹车片"]
    end

    subgraph L1["L1: 基础设施层 (Tools / Services)"]
        D1["40+ Tools"] --> D1a["BashTool<br/>Shell 执行"]
        D1 --> D1b["FileEditTool<br/>字符串替换编辑"]
        D1 --> D1c["AgentTool<br/>子 Agent 递归"]
        D1 --> D1d["FileRead / Glob / Grep<br/>文件操作"]
        D2["权限系统"] --> D2a["hasPermissionsToUseTool<br/>7 步决策流水线"]
        D2 --> D2b["Bash Classifier<br/>AI 自动分类器"]
        D2 --> D2c["Safety Check<br/>不可绕过安全闸"]
        D3["服务层"] --> D3a["API Client<br/>流式请求、重试"]
        D3 --> D3b["Compact Service<br/>Token 压缩"]
        D3 --> D3c["MCP Client<br/>外部工具协议"]
        D3 --> D3d["Analytics / GrowthBook<br/>遥测与实验"]
    end

    A3 -->|调用| B1
    B1 -->|进入| C1
    C1 -->|调度| D1
    C1 -->|检查| D2
    D1a -.->|沙箱| D2
    D1c -.->|递归调用| C1
```

---

## 二、数据流：一次用户输入的完整旅程

```mermaid
sequenceDiagram
    actor User
    participant REPL as REPL.tsx
    participant PUI as processUserInput
    participant QE as QueryEngine
    participant QL as query.ts Loop
    participant API as Claude API
    participant TE as ToolExecutor
    participant Tool as BashTool/FileEditTool
    participant PM as PermissionSystem

    User->>REPL: 输入 "帮我把 utils.js 拆成两个文件"
    REPL->>PUI: 判断是普通 prompt
    PUI->>PUI: 提取附件、执行 UserPromptSubmit Hooks
    PUI->>QE: shouldQuery = true
    QE->>QL: 组装 messages + systemPrompt + tools
    
    loop Agent Loop
        QL->>API: 流式调用 (stream_request_start)
        API-->>QL: 返回 assistant 消息
        
        alt 包含 tool_use (如 Bash, FileEdit, Agent)
            QL->>TE: 提前开始并行执行
            TE->>Tool: 调用 call()
            Tool->>PM: checkPermissions()
            PM-->>Tool: allow / ask / deny
            
            alt ask
                PM->>REPL: 推入 PermissionRequest 队列
                REPL->>User: 显示权限弹窗
                User->>REPL: 点击 "Allow"
                REPL->>PM: resolve allow
                PM-->>Tool: 继续执行
            end
            
            Tool-->>TE: 返回 tool_result
            TE-->>QL: 结果追加到 messages
            QL->>API: 下一轮调用
        else 纯文本回复
            API-->>QL: 直接输出建议
            QL->>QE: return completed
        end
    end
    
    QE->>REPL: yield result (success)
    REPL->>User: 显示最终回复
```

---

## 三、Agent Loop 内部状态机

```mermaid
stateDiagram-v2
    [*] --> Stage0: 解构 state
    Stage0 --> Stage1: 上下文预处理
    Stage1 --> Stage2: 流式调用 API
    
    Stage2 --> Stage3a: 收到 text block
    Stage2 --> Stage3b: 收到 tool_use block
    Stage3b --> Stage4: StreamingToolExecutor.addTool()
    
    Stage4 --> Stage5a: needsFollowUp = false
    Stage4 --> Stage5b: needsFollowUp = true
    
    Stage5a --> StopHooks: 执行 stop hooks
    StopHooks --> Stage6a: preventContinuation
    StopHooks --> Stage6b: blockingErrors
    StopHooks --> Stage6c: 正常完成
    Stage6c --> [*]: return completed
    Stage6b --> Stage0: continue (带错误消息)
    
    Stage5b --> Stage7: 并行/串行执行工具
    Stage7 --> Stage8: 注入附件 (memory/skill/commands)
    Stage8 --> Stage0: continue (新一轮)
```

---

## 四、权限系统决策树

```mermaid
flowchart TD
    Start([模型请求使用工具]) --> S1{Step 1a:<br/>工具级 deny 规则?}
    S1 -->|是| Deny1[deny]
    S1 -->|否| S2{Step 1b:<br/>工具级 ask 规则?}
    
    S2 -->|是| Ask1[ask]
    S2 -->|否| S3{Step 1c:<br/>tool.checkPermissions()}
    
    S3 -->|deny| Deny2[deny]
    S3 -->|ask| S4{Step 1e/f/g:<br/>免疫 bypass?}
    S3 -->|passthrough| S8
    
    S4 -->|是| Ask2[ask]
    S4 -->|否| S5{Step 2a:<br/>bypassPermissions 模式?}
    
    S5 -->|是| Allow1[allow]
    S5 -->|否| S6{Step 2b:<br/>工具级 allow 规则?}
    
    S6 -->|是| Allow2[allow]
    S6 -->|否| S8[passthrough → ask]
    
    S8 --> S9{外层模式转换?}
    S9 -->|dontAsk| Deny3[deny]
    S9 -->|auto| S10{AI 分类器}
    S9 -->|其他| Ask3[ask 弹窗]
    
    S10 -->|高置信度 allow| Allow3[auto allow]
    S10 -->|低置信度 / 拒识| Ask4[ask 弹窗]
    S10 -->|连续拒绝超限| Ask5[降级到弹窗]
```

---

## 五、子 Agent 递归架构

```mermaid
flowchart LR
    subgraph Parent["父 Agent (Main Thread)"]
        P1["queryLoop"] --> P2["收到 tool_use: AgentTool"]
        P2 --> P3{"同步 / 异步?"}
        P3 -->|同步| P4["阻塞等待子 Agent 完成"]
        P3 -->|异步| P5["注册后台任务<br/>立即返回 async_launched"]
    end

    subgraph Child["子 Agent (Subagent)"]
        C1["runAgent()"] --> C2["新建 ToolUseContext<br/>独立工具池"]
        C2 --> C3["进入独立的 queryLoop()"]
        C3 --> C4["可进一步递归 AgentTool"]
    end

    subgraph Isolation["隔离机制"]
        I1["worktree"] --> I2["git worktree<br/>独立文件系统副本"]
        I3["remote"] --> I4["CCR 远程环境<br/>完全隔离"]
    end

    P4 -->|调用| C1
    P5 -.->|后台运行| C1
    C1 -.->|可选| I1
    C1 -.->|可选| I3
    C4 -->|递归| Child
```

---

## 六、上下文压缩防御体系

```mermaid
flowchart TB
    Input[每轮 Loop 的 messages] --> Snip["Snip<br/>删除历史中间部分"]
    Snip --> Micro["Microcompact<br/>轻量压缩"]
    Micro --> Collapse["Context Collapse<br/>按主题折叠"]
    Collapse --> Auto["Autocompact<br/>主动总结 (Haiku)"]
    
    API[调用 API] -->|返回 413 PTL| Reactive["Reactive Compact<br/>被动急救"]
    Reactive --> Retry["自动 retry 同一轮"]
    Retry --> API
    
    Auto -->|仍超标| Block["Blocking Limit<br/>硬阻断"]
```

---

## 七、组件依赖关系图

```mermaid
flowchart BT
    subgraph UI["UI 组件"]
        REPL[REPL.tsx]
        PIN[PromptInput]
        VML[VirtualMessageList]
        PERM[PermissionRequest]
        SP[Spinner]
    end

    subgraph Engine["引擎"]
        QE[QueryEngine.ts]
        QU[query.ts]
        PUI[processUserInput.ts]
    end

    subgraph Tools["工具"]
        BT[BashTool.tsx]
        FT[FileEditTool.ts]
        AT[AgentTool.tsx]
        GT[Glob/Grep/Read]
    end

    subgraph Perm["权限"]
        CUT[useCanUseTool.tsx]
        HP[hasPermissionsToUseTool]
        BC[bashClassifier.ts]
    end

    subgraph Services["服务"]
        AC[api/claude.ts]
        CS[compact/autoCompact.ts]
        MCP[mcp/client.ts]
        SS[sessionStorage.ts]
    end

    REPL --> QE
    REPL --> PIN
    REPL --> VML
    REPL --> PERM
    REPL --> SP
    
    QE --> PUI
    QE --> QU
    QE --> SS
    
    QU --> AC
    QU --> CS
    QU --> CUT
    QU --> BT
    QU --> FT
    QU --> AT
    QU --> GT
    
    CUT --> HP
    HP --> BC
    HP --> BT
    HP --> FT
    
    AT --> QU
    BT --> AC
    
    MCP --> GT
```

---

## 八、文件速查导航

按功能模块快速定位源码：

### 核心循环
| 文件 | 内容 |
|------|------|
| `src/query.ts` | Agent Loop 心脏 |
| `src/QueryEngine.ts` | 会话编排与 SDK 接口 |
| `src/replLauncher.tsx` | REPL 启动器 |

### 工具系统
| 文件 | 内容 |
|------|------|
| `src/Tool.ts` | 工具接口与 `buildTool` |
| `src/tools.ts` | 工具注册中心 |
| `src/tools/BashTool/BashTool.tsx` | Bash 执行 |
| `src/tools/FileEditTool/FileEditTool.ts` | 文件编辑 |
| `src/tools/AgentTool/AgentTool.tsx` | 子 Agent |
| `src/services/tools/StreamingToolExecutor.ts` | 流式工具执行器 |

### 权限与安全
| 文件 | 内容 |
|------|------|
| `src/hooks/useCanUseTool.tsx` | 权限总入口 |
| `src/utils/permissions/permissions.ts` | 7 步决策引擎 |
| `src/utils/permissions/bashClassifier.ts` | Bash 分类器 |
| `src/utils/permissions/denialTracking.ts` | 连续拒绝追踪 |
| `src/components/permissions/PermissionRequest.tsx` | 弹窗分发器 |

### 上下文压缩
| 文件 | 内容 |
|------|------|
| `src/services/compact/autoCompact.ts` | 主动压缩 |
| `src/services/compact/reactiveCompact.ts` | 被动急救 |
| `src/services/compact/compact.ts` | 压缩实现 |

### UI 与状态
| 文件 | 内容 |
|------|------|
| `src/screens/REPL.tsx` | 主界面 |
| `src/components/App.tsx` | 全局 Provider |
| `src/components/PromptInput/PromptInput.tsx` | 输入框 |
| `src/components/VirtualMessageList.tsx` | 虚拟滚动 |
| `src/state/AppState.tsx` | 全局状态定义 |

### 输入处理
| 文件 | 内容 |
|------|------|
| `src/utils/processUserInput/processUserInput.ts` | 输入分支处理 |
| `src/utils/handlePromptSubmit.ts` | 提交处理器 |
| `src/utils/queryContext.ts` | System Prompt 组装 |

---

*文档生成时间：2026-04-14*  
*基于源码版本：Claude Code v2.1.88*
