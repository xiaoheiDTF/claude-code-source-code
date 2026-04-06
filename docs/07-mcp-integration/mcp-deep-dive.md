# Claude Code MCP (Model Context Protocol) 深度分析

> 基于 `src/services/mcp/` 全部源码的逐层解读

---

## 一、MCP 协议是什么

**Model Context Protocol (MCP)** 是 Anthropic 定义的开放协议，让 AI 助手通过统一接口连接外部工具和数据源。

### 核心理念

```
┌───────────────┐     JSON-RPC      ┌───────────────┐
│               │ ◄──────────────► │               │
│  Claude Code  │   stdio/SSE/HTTP  │  MCP Server   │
│  (MCP Client) │   /WebSocket    │  (第三方进程) │
│               │ ►──────────────► │               │
└───────────────┘                  └───────────────┘

Claude Code 是 MCP Client，向 MCP Server 发送 JSON-RPC 请求。
MCP Server 可以是任何语言的任何进程。
```

### JSON-RPC 方法

| 方法 | 方向 | 说明 |
|------|------|------|
| `tools/list` | Client → Server | 获取服务器提供的工具列表 |
| `tools/call` | Client → Server | 调用某个工具执行 |
| `prompts/list` | Client → Server | 获取服务器提供的提示词模板列表 |
| `prompts/get` | Client → Server | 获取某个提示词的内容 |
| `resources/list` | Client → Server | 获取服务器提供的资源列表 |
| `resources/read` | Client → Server | 读取某个资源的内容 |
| `notifications/*` | Server → Client | 服务器主动推送通知（进度、日志等） |

---

## 二、6 种传输层

### 源码位置
`src/services/mcp/types.ts`

| 类型 | 配置字段 | 适用场景 | 连接方式 |
|------|---------|----------|----------|
| **stdio** | `command` + `args` + `env` | 本地子进程 | stdin 管道（最常用） |
| **sse** | `url` + `headers` | 远程 HTTP | Server-Sent Events 长连接 |
| **sse-ide** | `url` + `ideName` | IDE 集成 | IDE 内部 SSE |
| **http** | `url` + `headers` | 远程 HTTP | JSON-RPC over HTTP（Streamable） |
| **ws** | `url` + `headers` | 远程 WebSocket | 全双工 WebSocket |
| **sdk** | `name` | 进程内 | 内存中直接调用 |

### 配置示例

```json
// stdio：启动一个本地进程
{
  "my-server": {
    "command": "npx",
    "args": ["-y", "@anthropic/my-server"],
    "env": { "API_KEY": "xxx" }
  }
}

// sse：连接远程 SSE 端点
{
  "remote-server": {
    "type": "sse",
    "url": "https://mcp.example.com/sse"
  }
}

// http：JSON-RPC over HTTP
{
  "api-server": {
    "type": "http",
    "url": "https://mcp.example.com/api"
  }
}
```

---

## 三、配置加载：7 个源与合并优先级

### 源码位置
`src/services/mcp/config.ts` — `getAllMcpConfigs()` / `getClaudeCodeMcpConfigs()`

### 加载流程

```
getAllMcpConfigs()
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│  1. Enterprise（企业管理配置）                                    │
│     路径: {managed_path}/managed-mcp.json                        │
│     如果存在 → 独占模式，跳过所有其他源                              │
│     适用: hasAllowManagedMcpServersOnly() = true                 │
└──────────────┬──────────────────────────────────────────────────┘
               │ 企业配置不存在
               ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. User（用户全局配置）                                         │
│     路径: ~/.claude.json (mcpServers 字段)                      │
│     命令: claude mcp add <name> --scope user                    │
└──────────────┬──────────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. Project（项目配置，需审批）                                   │
│     路径: <project>/.mcp.json                                   │
│     安全: 项目配置需用户明确 approve 才生效                       │
│     原因: 防止恶意仓库注入 MCP 服务器                             │
└──────────────┬──────────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. Local（项目本地配置）                                        │
│     命令: claude mcp add <name> --scope project --local          │
│     不提交到 git，个人使用                                       │
└──────────────┬──────────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────────┐
│  5. Plugin（插件 MCP）                                          │
│     命名: plugin:<pluginName>:<serverName>                      │
│     去重: 与手动配置签名相同的插件服务器被抑制                       │
└──────────────┬──────────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────────┐
│  6. Claude.ai Connectors                                       │
│     从 claude.ai 服务拉取，最低优先级                             │
│     去重: 与手动配置签名相同的 connector 被抑制                     │
└──────────────┬──────────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────────┐
│  7. Dynamic（运行时动态配置）                                     │
│     CLI 参数 --mcp-config 传入                                   │
│     最低优先级                                                   │
└─────────────────────────────────────────────────────────────────┘
               │
               ▼
          合并 & 策略过滤
               │
               ▼
          最终 MCP 服务器列表
```

### 合并优先级

```
plugin < user < project < local < dynamic
```

后者覆盖前者。Enterprise 是独占模式，存在时跳过所有其他源。

### 去重机制（Dedup）

```typescript
// config.ts — getMcpServerSignature()
// 签名 = type:content
// stdio 服务器 → "stdio:["command","arg1","arg2"]"
// url 服务器 → "url:https://mcp.example.com"
// sdk 服务器 → null（不去重）

// 手动配置优先于插件配置
// 先加载的插件优先于后加载的插件（同名同签名）
```

### 企业策略过滤

```typescript
// config.ts — isMcpServerAllowedByPolicy()
// 1. denylist（黑名单）优先级最高，绝对阻止
// 2. allowlist（白名单）为空 → 阻止所有
// 3. allowlist 存在 → 必须匹配 name/command/url 才允许
// 4. 无 allowlist → 默认允许（除非在 denylist 中）
```

---

## 四、服务器连接生命周期

### 源码位置
`src/services/mcp/client.ts` — `connectToServer()`

### 连接状态

```typescript
type MCPServerConnection =
  | ConnectedMCPServer    // 已连接
  | FailedMCPServer       // 连接失败
  | NeedsAuthMCPServer    // 需要认证
  | PendingMCPServer      // 正在连接
  | DisabledMCPServer     // 被用户禁用
```

### 连接流程

```
connectToServer(config)
        │
        ▼
┌──────────────────────────────────────┐
│  1. 选择 Transport                  │
│                                      │
│  stdio → StdioClientTransport       │
│    启动子进程，通过 stdin/stdout 通信  │
│                                      │
│  sse → SSEClientTransport           │
│    建立 SSE 长连接                    │
│    支持 OAuth 认证                    │
│                                      │
│  http → HTTPClientTransport         │
│    JSON-RPC over HTTP               │
│    支持 Session 管理                 │
│                                      │
│  ws → WebSocketClientTransport      │
│    全双工 WebSocket                  │
│                                      │
│  sdk → InProcessTransport           │
│    内存中直接调用                     │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│  2. 建立 MCP Client 连接             │
│                                      │
│  const client = new Client({         │
│    name: "claude-code",              │
│    version: "2.1.88"                 │
│  })                                  │
│  await client.connect(transport)     │
│                                      │
│  → 交换 capabilities                │
│  → 获取 serverInfo (名称/版本)       │
│  → 确定支持的功能 (tools/prompts/    │
│    resources)                        │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│  3. 返回 ConnectedMCPServer         │
│  {                                   │
│    client,                           │
│    name,                             │
│    type: 'connected',               │
│    capabilities,                     │
│    config,                           │
│    cleanup                           │
│  }                                   │
└──────────────────────────────────────┘
```

### 错误处理

| 错误类型 | 触发条件 | 处理方式 |
|----------|----------|----------|
| `McpAuthError` | 401 Unauthorized | 标记为 `needs-auth`，缓存 15 分钟 |
| `McpToolCallError` | 工具返回 `isError: true` | 包装错误信息，保留 `_meta` |
| `McpSessionExpiredError` | HTTP 404 + JSON-RPC -32001 | 清除连接缓存，自动重连 |
| 超时 | `MCP_TOOL_TIMEOUT` 环境变量（默认 27.8 小时） | 抛出超时错误 |
| 连接断开 | SSE/WebSocket 断开 | 指数退避重连 |

---

## 五、MCP Tools 注册与发现

### 源码位置
`src/services/mcp/client.ts` — `fetchToolsForClient()`

### 注册流程

```
fetchToolsForClient(connectedClient)
        │
        ▼
┌─────────────────────────────────────────────────────────┐
│  1. 检查 capabilities.tools                             │
│     服务器声明支持工具功能                                │
│     不支持 → 返回空数组                                  │
└──────────────┬──────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────┐
│  2. JSON-RPC: tools/list                                │
│     const result = await client.request(                │
│       { method: 'tools/list' },                         │
│       ListToolsResultSchema                             │
│     )                                                   │
│                                                         │
│     返回: { tools: [{ name, description, inputSchema }] }│
└──────────────┬──────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────┐
│  3. 转换为 Claude Code Tool 对象                         │
│     每个 MCP tool 变成一个 Tool 对象：                    │
│                                                         │
│     {                                                   │
│       name: "mcp__serverName__toolName",  // 命名规则    │
│       isMcp: true,                                      │
│       mcpInfo: { serverName, toolName },                │
│       inputJSONSchema: tool.inputSchema,                │
│       description() { return tool.description },        │
│       prompt() { return truncated description },        │
│       call(args) { ... },                               │
│       checkPermissions() { ... },                       │
│       isReadOnly() { return annotations.readOnlyHint }, │
│       isDestructive() { return annotations.destructiveHint }, │
│     }                                                   │
└─────────────────────────────────────────────────────────┘
```

### 命名规则

```typescript
// mcpStringUtils.ts
export function buildMcpToolName(serverName: string, toolName: string): string {
  return `${getMcpPrefix(serverName)}${normalizeNameForMCP(toolName)}`
}

// 结果: mcp__servername__toolname
// 例如: mcp__slack__post_message
//       mcp__github__create_issue
//       mcp__filesystem__read_file

// SDK 特殊模式：可跳过 mcp__ 前缀
// CLAUDE_AGENT_SDK_MCP_NO_PREFIX=true
```

### Tool Annotations 映射

MCP 工具可以通过 `annotations` 声明元信息：

| MCP Annotation | Claude Code 映射 | 说明 |
|----------------|------------------|------|
| `readOnlyHint` | `isReadOnly()` | 只读工具，不修改文件 |
| `destructiveHint` | `isDestructive()` | 破坏性操作 |
| `openWorldHint` | `isOpenWorld()` | 开放世界工具 |
| `anthropic/searchHint` | `searchHint` | 搜索提示 |
| `anthropic/alwaysLoad` | `alwaysLoad` | 始终加载（不延迟） |

---

## 六、MCP Prompts 注册与发现（Slash Commands）

### 源码位置
`src/services/mcp/client.ts` — `fetchCommandsForClient()`

### MCP Tools vs Prompts 的区别

这是理解 MCP 集成最关键的区别：

```
┌─────────────────────────────────────────────────────────────────┐
│                    MCP Tools vs Prompts                          │
├──────────────────────┬──────────────────────────────────────────┤
│     MCP Tool         │          MCP Prompt                      │
├──────────────────────┼──────────────────────────────────────────┤
│ Claude 调用的工具     │ 用户调用的斜杠命令                       │
│                      │                                          │
│ 在工具列表中显示      │ 在 / 命令自动补全中显示                  │
│                      │                                          │
│ 模型自主决定何时使用  │ 用户主动输入 /command 触发               │
│                      │                                          │
│ JSON-RPC: tools/call │ JSON-RPC: prompts/get                   │
│                      │                                          │
│ 输入: JSON 参数      │ 输入: 参数字符串                         │
│                      │                                          │
│ 输出: 工具结果       │ 输出: 提示词内容（注入对话）              │
│                      │                                          │
│ 权限: 标准工具权限   │ 权限: 技能权限规则                       │
│                      │                                          │
│ 命名: mcp__s__t      │ 命名: mcp__s__prompt_name               │
└──────────────────────┴──────────────────────────────────────────┘
```

### Prompts 发现流程

```
fetchCommandsForClient(connectedClient)
        │
        ▼
┌─────────────────────────────────────────────────────────┐
│  1. 检查 capabilities.prompts                           │
│     服务器声明支持 prompts 功能                           │
└──────────────┬──────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────┐
│  2. JSON-RPC: prompts/list                              │
│     const result = await client.request(                │
│       { method: 'prompts/list' },                       │
│       ListPromptsResultSchema                           │
│     )                                                   │
│                                                         │
│     返回: { prompts: [{ name, description, arguments }] }│
└──────────────┬──────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────┐
│  3. 转换为 Command (PromptCommand) 对象                  │
│                                                         │
│     {                                                   │
│       type: 'prompt',                                   │
│       name: 'mcp__server__prompt_name',                 │
│       isMcp: true,                                      │
│       description: prompt.description,                  │
│       argNames: [...],                                  │
│       userFacingName() { return "server:name (MCP)" },  │
│                                                         │
│       async getPromptForCommand(args) {                 │
│         // JSON-RPC: prompts/get                        │
│         const result = await client.getPrompt({          │
│           name: prompt.name,                            │
│           arguments: { arg1: val1, ... }                │
│         })                                              │
│         // 转换返回的消息内容                             │
│         return transformResultContent(result.messages)  │
│       }                                                 │
│     }                                                   │
└─────────────────────────────────────────────────────────┘
```

### 用户使用方式

```
# 列出可用命令（含 MCP prompts）
/help

# 调用 MCP prompt（如 /mcp__slack__search_messages）
/mcp__slack__search_messages query: "meeting tomorrow"

# 或者通过 SkillTool 自动匹配
Claude 内部会查找所有 Command（含 MCP prompts）并执行
```

---

## 七、MCP Tool Call 完整执行流程

### 源码位置
`src/services/mcp/client.ts` — `callMCPTool()`

```
Claude API 返回 tool_use block: mcp__slack__post_message
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│  标准 10 步 Tool Pipeline（和内置工具完全相同）                    │
│                                                                  │
│  1. findToolByName("mcp__slack__post_message")                   │
│     → 找到 MCP tool 对象                                         │
│                                                                  │
│  2. inputSchema.safeParse(input)                                 │
│     → 使用 MCP 服务器提供的 JSON Schema 验证输入                  │
│                                                                  │
│  3. validateInput() → 无额外验证                                 │
│                                                                  │
│  4. backfillObservableInput() → 无                                │
│                                                                  │
│  5. runPreToolUseHooks()                                         │
│     → 前置 Hooks（用户/技能定义的）                               │
│                                                                  │
│  6. resolveHookPermissionDecision()                              │
│     → 合并 Hook 权限决策                                         │
│                                                                  │
│  7. canUseTool()                                                 │
│     → MCP tool 默认 behavior: 'passthrough'                      │
│     → 需要用户确认（除非有 allow 规则）                            │
│                                                                  │
│  8. tool.call(args, context) ← MCP 特殊路径                      │
│     ┌─────────────────────────────────────────────────────┐      │
│     │  MCP Tool.call() 实现 (client.ts:1833)                  │  │
│     │                                                         │  │
│     │  a. ensureConnectedClient(client)                       │  │
│     │     → 确保连接还活着，断开则重连                            │  │
│     │                                                         │  │
│     │  b. callMCPToolWithUrlElicitationRetry()               │  │
│     │     → 调用 callMCPTool()                                │  │
│     │                                                         │  │
│     │     c. callMCPTool() 核心实现 (client.ts:3029)         │  │
│     │        ┌──────────────────────────────────────────┐    │  │
│     │        │ Promise.race([                            │    │  │
│     │        │   client.callTool({                      │    │  │
│     │        │     name: "post_message",                │    │  │
│     │        │     arguments: { channel, text },        │    │  │
│     │        │     _meta: { toolUseId }                 │    │  │
│     │        │   },                                    │    │  │
│     │        │   timeoutPromise  // 超时保护             │    │  │
│     │        │ ])                                        │    │  │
│     │        │                                           │    │  │
│     │        │ 返回: { content, _meta, structuredContent }   │  │
│     │        └──────────────────────────────────────────┘    │  │
│     │                                                         │  │
│     │  d. Session 过期重试（最多 1 次）                        │  │
│     │     McpSessionExpiredError → 清缓存 + 重连 + 重试       │  │
│     └─────────────────────────────────────────────────────┘    │
│                                                                  │
│  9. runPostToolUseHooks()                                        │
│     → 后置 Hooks                                                 │
│     → 特殊: MCP 工具的输出可以被 PostHook 修改                   │
│     → 内置工具的输出不可被修改                                    │
│                                                                  │
│  10. 返回 ToolResult                                             │
│     → processMCPResult() 处理大输出、图片、Blob                  │
└─────────────────────────────────────────────────────────────────┘
```

### MCP vs 内置工具的执行差异

| 维度 | 内置工具 | MCP 工具 |
|------|---------|---------|
| Schema | Zod schema (TypeScript) | JSON Schema (服务器提供) |
| 执行位置 | 本进程 | 外部进程（JSON-RPC） |
| 超时 | 无 | `MCP_TOOL_TIMEOUT`（默认 ~27.8h） |
| 进度报告 | onProgress 回调 | MCP `notifications/*` + onProgress |
| 结果持久化 | processToolResultBlock | processMCPResult（支持图片/Blob） |
| PostHook 可修改输出 | 不可以 | 可以（仅 MCP 工具） |
| 认证 | 无 | OAuth / API Key / Token |
| 重连 | 不适用 | Session 过期自动重连 |

---

## 八、结果处理管线

### 源码位置
`src/services/mcp/client.ts` — `transformMCPResult()` + `processMCPResult()`

```
MCP Server 返回结果
        │
        ▼
┌─────────────────────────────────────────────────────────┐
│  transformMCPResult()                                   │
│  处理每种内容类型：                                       │
│                                                         │
│  text → 直接保留                                        │
│  image → 压缩 + 调整大小 + 持久化到临时文件               │
│  audio → 持久化到临时文件 + 文本描述                     │
│  resource → 嵌入资源链接                                │
└──────────────┬──────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────┐
│  processMCPResult()                                     │
│  处理输出大小和类型：                                     │
│                                                         │
│  超大文本 → 持久化到临时文件，返回文件路径               │
│  图片数据 → 调整大小后嵌入                               │
│  二进制 Blob → 持久化到临时文件                          │
│  正常文本 → 直接返回                                    │
└─────────────────────────────────────────────────────────┘
```

---

## 九、安全机制总结

```
┌─────────────────────────────────────────────────────────────┐
│                     MCP 安全层                              │
│                                                             │
│  第 1 层: 企业策略                                          │
│  ├─ managed-mcp.json 独占模式                               │
│  ├─ denylist（黑名单）绝对阻止                               │
│  └─ allowlist（白名单）按 name/command/url 过滤              │
│                                                             │
│  第 2 层: 项目配置审批                                       │
│  └─ .mcp.json 需要用户明确 approve 才生效                   │
│                                                             │
│  第 3 层: 连接认证                                           │
│  ├─ OAuth 2.0 流程                                          │
│  ├─ API Key（env 字段注入）                                  │
│  └─ 自定义 headers                                          │
│                                                             │
│  第 4 层: 工具权限                                           │
│  ├─ 标准 Tool Pipeline 10 步权限流程                        │
│  ├─ MCP tool 默认 behavior: 'passthrough'（需确认）         │
│  └─ 用户可配置 allow/deny 规则按工具名匹配                  │
│                                                             │
│  第 5 层: Hook 拦截                                         │
│  ├─ PreToolUse hooks 可拒绝 MCP 工具调用                    │
│  └─ PostToolUse hooks 可审计/修改 MCP 工具输出              │
└─────────────────────────────────────────────────────────────┘
```

---

## 十、配置方式汇总

### CLI 命令

```bash
# 添加 stdio 服务器（用户级）
claude mcp add my-server -s user -- npx -y @scope/server

# 添加 SSE 远程服务器（项目级）
claude mcp add remote-api -s project --transport sse --url https://mcp.example.com/sse

# 列出所有服务器
claude mcp list

# 删除服务器
claude mcp remove my-server

# 启用/禁用服务器
claude mcp enable my-server
claude mcp disable my-server
```

### 配置文件

```json
// .mcp.json（项目级，需提交到 git）
{
  "mcpServers": {
    "my-api": {
      "type": "http",
      "url": "https://api.example.com/mcp"
    }
  }
}

// ~/.claude.json（用户级）
{
  "mcpServers": {
    "local-tool": {
      "command": "npx",
      "args": ["-y", "@myorg/mcp-server"],
      "env": {
        "API_KEY": "${MY_API_KEY}"
      }
    }
  }
}
```

### CLI 参数

```bash
# 运行时动态传入 MCP 配置
claude --mcp-config '{"servers":{"my-server":{"command":"npx","args":["server"]}}}'
```
