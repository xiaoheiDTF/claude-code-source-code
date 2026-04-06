# settings.json: mcpServers MCP 服务器

> 指南 7.2「主要配置项」— mcpServers 配置外部 MCP 服务

## settings.json 配置

```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "postgresql://localhost/mydb"],
      "env": {}
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_xxx"
      }
    }
  }
}
```

## 说明

```
mcpServers 字段:
  格式: 对象，key 是服务器名称，value 是配置
  作用: 配置外部 MCP (Model Context Protocol) 服务器

  每个服务器的配置:
    command: 启动命令（必填）
    args: 命令参数数组（可选）
    env: 环境变量对象（可选）

  服务器启动方式:
    npx -y @mcp/server-name → 自动从 npm 安装并运行
    node /path/to/server.js → 运行本地 MCP 服务器
    python /path/to/server.py → 运行 Python MCP 服务器

  常用 MCP 服务器:
    @modelcontextprotocol/server-postgres  → PostgreSQL 数据库
    @modelcontextprotocol/server-github    → GitHub API
    @modelcontextprotocol/server-filesystem → 文件系统
    @modelcontextprotocol/server-brave-search → 搜索
```

## 使用示例

### 示例 1：PostgreSQL 数据库 MCP

```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-postgres",
        "postgresql://user:pass@localhost:5432/mydb"
      ]
    }
  }
}
```

```
→ Claude 可以直接操作数据库:
  MCP: postgres.query("SELECT * FROM users LIMIT 5")
  MCP: postgres.query("EXPLAIN ANALYZE SELECT ...")
  MCP: postgres.describeTable("users")

→ Claude 执行:
  "查看用户表结构..." → MCP 调用 describeTable
  "找出慢查询..." → MCP 调用 query + EXPLAIN
  "统计用户增长..." → MCP 调用 query + GROUP BY

效果:
  → Claude 直接查询数据库，不需要 Bash + psql
  → 结构化数据返回，比文本解析更可靠
  → 适合数据分析和数据库管理
```

### 示例 2：GitHub API MCP

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "$GITHUB_TOKEN"
      }
    }
  }
}
```

```
→ Claude 可以直接操作 GitHub:
  MCP: github.create_issue({title, body})
  MCP: github.list_prs({state: "open"})
  MCP: github.get_pr_diff({number: 42})
  MCP: github.search_code({query: "TODO"})

→ Claude 执行:
  "查看待处理的 PR" → list_prs
  "审查 PR #42" → get_pr_diff + review
  "搜索所有 TODO" → search_code

效果:
  → Claude 通过 API 而非 Bash gh 命令操作 GitHub
  → 结构化数据，更高效
  → Token 通过环境变量传入（安全）
```

### 示例 3：多个 MCP 服务器组合

```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "postgresql://localhost/mydb"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_PERSONAL_ACCESS_TOKEN": "$GITHUB_TOKEN" }
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/tmp/shared"]
    },
    "brave-search": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-brave-search"],
      "env": { "BRAVE_API_KEY": "$BRAVE_API_KEY" }
    }
  }
}
```

```
→ Claude 同时使用 4 个 MCP 服务:
  postgres      → 数据库查询
  github        → PR/Issue 管理
  filesystem    → /tmp/shared 文件访问
  brave-search  → 网络搜索

→ 跨服务协作:
  "分析 GitHub Issue 中报告的数据库问题"
  → github: 读取 Issue #156
  → postgres: 查询相关表数据
  → github: 创建 PR 修复
  → postgres: 验证修复

效果:
  → 多服务组合扩展 Claude 能力
  → 每个服务专注一个领域
  → 跨服务协作完成复杂任务
```

### 示例 4：自定义本地 MCP 服务器

```json
{
  "mcpServers": {
    "my-company-api": {
      "command": "node",
      "args": ["/path/to/my-mcp-server/build/index.js"],
      "env": {
        "API_BASE_URL": "https://internal.company.com/api",
        "API_KEY": "$COMPANY_API_KEY"
      }
    },
    "legacy-system": {
      "command": "python",
      "args": ["/path/to/legacy-mcp/server.py"],
      "env": {
        "LEGACY_HOST": "legacy.internal",
        "LEGACY_PORT": "8080"
      }
    }
  }
}
```

```
→ Claude 使用自定义 MCP 服务:
  my-company-api:
    → 访问公司内部 API（客户系统、订单系统）
    → 通过 Node.js MCP 服务器桥接

  legacy-system:
    → 访问遗留系统（Python 桥接）
    → 通过 Python MCP 服务器连接

效果:
  → Claude 可以访问公司内部系统
  → 自定义 MCP 服务器桥接任意 API
  → 环境变量传递认证信息
```

### 示例 5：禁用 MCP 服务器

```json
{
  "mcpServers": {}
}
```

```
→ 不配置任何 MCP 服务器
→ Claude 只使用内置工具（Read, Edit, Bash, Grep 等）

→ 临时禁用某个 MCP:
  删除或注释该服务器配置即可

→ 项目级覆盖全局 MCP:
  # ~/.claude/settings.json（全局）
  {
    "mcpServers": {
      "postgres": { ... },
      "github": { ... }
    }
  }

  # .claude/settings.json（项目 — 只用 postgres）
  {
    "mcpServers": {
      "github": null
    }
  }

效果:
  → 空配置禁用所有 MCP
  → 按需启用，不用时不消耗资源
  → MCP 服务器在 Claude Code 启动时初始化
```
