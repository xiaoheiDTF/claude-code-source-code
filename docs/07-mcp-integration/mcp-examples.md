# Claude Code MCP 实战示例

> 覆盖 MCP 服务器配置、Tool 调用、Prompt 命令、资源访问的完整示例

---

## 示例 1：最简 stdio MCP 服务器

### 项目结构

```
my-project/
├── .mcp.json          ← MCP 配置（需提交 git）
├── src/
└── package.json
```

### .mcp.json

```json
{
  "mcpServers": {
    "time-server": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-time"]
    }
  }
}
```

启动 Claude Code 后，Claude 自动获得一个 `mcp__time_server__*` 工具，可以直接查询当前时间。

**不写代码、不装依赖**，一个 JSON 文件就能扩展 Claude 的能力。

---

## 示例 2：带环境变量的 stdio 服务器

### ~/.claude.json（用户级配置）

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

Claude 启动时：
1. 读取 `~/.claude.json` 中的 `mcpServers.github`
2. 展开 `${GITHUB_TOKEN}` 为实际环境变量值
3. 启动 `npx -y @anthropic/mcp-server-github` 子进程
4. 通过 stdin/stdout 发送 JSON-RPC
5. 调用 `tools/list` 获取：`create_issue`、`list_prs`、`search_code` 等工具
6. 注册为 `mcp__github__create_issue`、`mcp__github__list_prs` 等

Claude 可直接调用：
```
用户: "帮我创建一个 issue，标题是 'Fix login bug'"
Claude: 调用 mcp__github__create_issue({ title: "Fix login bug", body: "..." })
```

---

## 示例 3：SSE 远程 MCP 服务器

### .mcp.json

```json
{
  "mcpServers": {
    "slack": {
      "type": "sse",
      "url": "https://mcp.slack.com/sse",
      "headers": {
        "Authorization": "Bearer ${SLACK_BOT_TOKEN}"
      }
    }
  }
}
```

连接流程：
```
Claude Code ──SSE 连接──► https://mcp.slack.com/sse
        │
        │  JSON-RPC: tools/list
        │  ← [{ name: "send_message", ... }, { name: "search_messages", ... }]
        │
        │  注册为:
        │  mcp__slack__send_message
        │  mcp__slack__search_messages
        │
        │  JSON-RPC: prompts/list
        │  ← [{ name: "daily_summary", ... }]
        │
        │  注册为:
        │  /mcp__slack__daily_summary
```

---

## 示例 4：HTTP Streamable MCP 服务器

### ~/.claude.json

```json
{
  "mcpServers": {
    "jira": {
      "type": "http",
      "url": "https://mcp.example.com/jira/api",
      "headers": {
        "X-Auth-Token": "${JIRA_API_TOKEN}"
      },
      "oauth": {
        "authServerMetadataUrl": "https://auth.example.com/.well-known/oauth-authorization-server",
        "clientId": "my-client-id"
      }
    }
  }
}
```

支持 OAuth 2.0 认证流程：
```
1. 首次连接 → 401 Unauthorized
2. Claude 打开浏览器 → OAuth 登录页面
3. 用户授权 → 回调获取 token
4. 后续请求自动携带 Authorization header
```

---

## 示例 5：自定义 MCP 服务器（Node.js）

### 编写 MCP 服务器

```typescript
// my-mcp-server.ts
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({
  name: "my-tools",
  version: "1.0.0",
});

// ── 注册一个 Tool（Claude 自主调用）──
server.tool(
  "get_weather",                            // 工具名
  "Get current weather for a city",         // 描述（Claude 看到这个决定是否调用）
  { city: z.string().describe("City name") }, // 输入 Schema
  async ({ city }) => {                      // 执行函数
    const temp = Math.floor(Math.random() * 35 + 5);
    return {
      content: [{
        type: "text",
        text: `${city}: ${temp}°C, ${temp > 25 ? "sunny" : "cloudy"}`,
      }],
    };
  }
);

server.tool(
  "search_docs",
  "Search internal documentation",
  {
    query: z.string().describe("Search query"),
    limit: z.number().optional().default(5),
  },
  async ({ query, limit }) => {
    // 实际实现：搜索内部文档系统
    const results = await searchInternalDocs(query, limit);
    return {
      content: results.map(r => ({
        type: "text",
        text: `## ${r.title}\n${r.content}`,
      })),
    };
  }
);

// ── 注册一个 Prompt（用户 /command 调用）──
server.prompt(
  "code_review",
  "Generate a code review prompt for the given code",
  { code: z.string().describe("Code to review") },
  async ({ code }) => {
    return {
      messages: [{
        role: "user",
        content: {
          type: "text",
          text: `Please review the following code for security issues, bugs, and style:\n\n\`\`\`\n${code}\n\`\`\``,
        },
      }],
    };
  }
);

// ── 注册一个 Resource（数据源）──
server.resource(
  "config",
  "config://app-settings",
  async () => ({
    contents: [{
      uri: "config://app-settings",
      text: JSON.stringify(await loadAppSettings(), null, 2),
    }],
  })
);

// 启动
const transport = new StdioServerTransport();
await server.connect(transport);
```

### 配置使用

```json
// .mcp.json
{
  "mcpServers": {
    "my-tools": {
      "command": "npx",
      "args": ["tsx", "my-mcp-server.ts"],
      "env": {
        "DOCS_API_KEY": "${MY_DOCS_API_KEY}"
      }
    }
  }
}
```

### Claude 获得的能力

```
Tools (Claude 自主调用):
  mcp__my_tools__get_weather   → 输入: { city: "Beijing" }
  mcp__my_tools__search_docs   → 输入: { query: "auth flow", limit: 5 }

Prompts (用户 /command):
  /mcp__my_tools__code_review  → 输入: code="function add(a,b){return a+b}"

Resources (数据源):
  config://app-settings        → Claude 可读取应用配置
```

---

## 示例 6：MCP Tool vs Prompt 的完整对比

### MCP 服务器定义

```typescript
// 同一个 MCP 服务器中
const server = new McpServer({ name: "git-helper", version: "1.0.0" });

// Tool: Claude 在需要时自主调用
server.tool(
  "list_branches",
  "List all git branches",
  { remote: z.boolean().optional() },
  async ({ remote }) => {
    const cmd = remote ? "git branch -r" : "git branch";
    const { stdout } = await exec(cmd);
    return { content: [{ type: "text", text: stdout }] };
  }
);

// Prompt: 用户主动调用 /mcp__git_helper__release_checklist
server.prompt(
  "release_checklist",
  "Generate a release checklist for the current branch",
  { version: z.string().describe("Version number for the release") },
  async ({ version }) => {
    const branch = (await exec("git branch --show-current")).stdout.trim();
    const commits = (await exec("git log --oneline -10")).stdout.trim();
    return {
      messages: [{
        role: "user",
        content: {
          type: "text",
          text: `# Release Checklist v${version}\n\n` +
            `Current branch: ${branch}\n\n` +
            `## Recent Commits\n${commits}\n\n` +
            `## Checklist\n` +
            `- [ ] All tests pass\n` +
            `- [ ] Version bumped to ${version}\n` +
            `- [ ] Changelog updated\n` +
            `- [ ] No TODOs remaining\n` +
            `- [ ] Documentation updated\n`,
        },
      }],
    };
  }
);
```

### 使用方式对比

```
═══════════════════════════════════════════════════════════
  Tool: mcp__git_helper__list_branches
═══════════════════════════════════════════════════════════

  用户: "当前有哪些分支？"
        │
        ▼
  Claude 自主决定: 我需要调用 mcp__git_helper__list_branches
        │
        ▼
  Tool Pipeline 10 步 → callMCPTool() → JSON-RPC tools/call
        │
        ▼
  结果: "* main\n  develop\n  feature/auth"
        │
        ▼
  Claude 回复: "当前有以下分支：main（当前）、develop、feature/auth"

═══════════════════════════════════════════════════════════
  Prompt: /mcp__git_helper__release_checklist
═══════════════════════════════════════════════════════════

  用户: /mcp__git_helper__release_checklist 2.1.0
        │
        ▼
  SkillTool 匹配到 MCP Prompt 命令
        │
        ▼
  JSON-RPC prompts/get → 获取提示词内容
        │
        ▼
  提示词注入当前对话:
    "# Release Checklist v2.1.0\n
     Current branch: main\n
     ## Recent Commits\n..."

  （和 Skill 的内联执行模式一样，提示词注入上下文）
```

---

## 示例 7：企业级 MCP 配置（策略控制）

### managed-mcp.json（企业管理员配置）

```json
{
  "mcpServers": {
    "internal-api": {
      "type": "http",
      "url": "https://mcp.internal.company.com/api",
      "headers": {
        "Authorization": "Bearer ${INTERNAL_API_TOKEN}"
      }
    },
    "code-search": {
      "command": "/opt/company/mcp-code-search",
      "args": ["--index", "/repos"],
      "env": {
        "SEARCH_API_KEY": "${SEARCH_API_KEY}"
      }
    }
  }
}
```

### settings.json（企业策略）

```json
{
  "allowedMcpServers": [
    { "serverName": "internal-api" },
    { "serverName": "code-search" }
  ],
  "deniedMcpServers": [
    { "serverUrl": "https://mcp.slack.com/*" },
    { "serverCommand": ["npx", "-y", "@thirdparty/*"] }
  ]
}
```

效果：
- `managed-mcp.json` 存在 → **独占模式**，用户/项目配置全部忽略
- 只允许 `internal-api` 和 `code-search`
- 阻止所有 slack MCP 和第三方 npx 服务器
- 用户无法自行添加 MCP 服务器

---

## 示例 8：插件提供 MCP 服务器

### 插件 manifest

```json
{
  "name": "slack-integration",
  "version": "1.0.0",
  "mcpServers": {
    "slack": {
      "type": "sse",
      "url": "https://mcp.slack.com/sse"
    }
  }
}
```

### 去重规则

```
用户手动配置: ~/.claude.json 中有 "slack" → SSE 连接到 mcp.slack.com
插件配置:     slack-integration 插件提供 "slack" → 同样连 mcp.slack.com

去重结果: 手动配置优先，插件的被抑制
日志: "Suppressing plugin MCP server 'slack': duplicates manually-configured 'slack'"

原因: 避免同一个 MCP 服务器连两次，浪费 ~600 chars/turn 的上下文
```

---

## 示例 9：多 MCP 服务器组合

### .mcp.json

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-filesystem", "/home/user/project"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-github"],
      "env": { "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}" }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-postgres"],
      "env": { "DATABASE_URL": "${DATABASE_URL}" }
    },
    "slack": {
      "type": "sse",
      "url": "https://mcp.slack.com/sse",
      "headers": { "Authorization": "Bearer ${SLACK_TOKEN}" }
    }
  }
}
```

### Claude 获得的所有能力

```
═══ mcp__filesystem ═══
  mcp__filesystem__read_file
  mcp__filesystem__write_file
  mcp__filesystem__list_directory
  mcp__filesystem__create_directory

═══ mcp__github ═══
  mcp__github__create_issue
  mcp__github__list_pull_requests
  mcp__github__search_code
  mcp__github__get_file_contents

═══ mcp__postgres ═══
  mcp__postgres__query
  mcp__postgres__list_tables
  mcp__postgres__describe_table

═══ mcp__slack ═══
  mcp__slack__send_message
  mcp__slack__search_messages
  /mcp__slack__daily_summary (Prompt)
```

用户可以这样说：
```
"查一下数据库中 users 表有多少条记录，
 然后创建一个 PR 把迁移脚本加进去，
 完成后在 Slack #dev 频道通知团队"
```

Claude 会依次调用：
1. `mcp__postgres__query({ sql: "SELECT COUNT(*) FROM users" })`
2. `mcp__github__create_pull_request({ title: "Add users migration", ... })`
3. `mcp__slack__send_message({ channel: "#dev", text: "PR #42 已创建..." })`

---

## 示例 10：MCP 工具的权限配置

### settings.json

```json
{
  "permissions": {
    "allow": [
      "mcp__filesystem__read_file",
      "mcp__filesystem__list_directory",
      "mcp__github__search_code",
      "mcp__github__get_file_contents",
      "mcp__postgres__list_tables",
      "mcp__postgres__describe_table",
      "mcp__slack__search_messages"
    ],
    "deny": [
      "mcp__filesystem__write_file",
      "mcp__filesystem__create_directory",
      "mcp__postgres__query",
      "mcp__slack__send_message"
    ]
  }
}
```

效果：
- 只读操作自动允许（不弹权限对话框）
- 写操作被 deny 规则阻止（Claude 无法调用）
- 未配置的工具走默认流程（弹出权限确认）

### 前缀匹配

```json
{
  "permissions": {
    "allow": [
      "mcp__github__",       // 允许所有 GitHub 工具
      "mcp__postgres__list*" // 允许所有 list 开头的 postgres 工具
    ],
    "deny": [
      "mcp__slack__send*"    // 阻止所有 send 开头的 Slack 工具
    ]
  }
}
```
