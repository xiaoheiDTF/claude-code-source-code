# Agent 字段: requiredMcpServers（必须可用的 MCP）

> 指南 3.2「其他」— `requiredMcpServers` 声明必须可用的 MCP，不可用则 Agent 不可启动

## 目录结构

```
my-project/
├── .claude/
│   ├── agents/
│   │   ├── db-admin.md              ← ★ 带 requiredMcpServers
│   │   ├── deploy-bot.md            ← ★ 带 requiredMcpServers
│   │   └── doc-writer.md            ← 不需要 requiredMcpServers
│   └── settings.json                ← MCP 配置
└── src/
```

## Agent 文件内容

```markdown
# 文件: .claude/agents/db-admin.md

---
description: Database admin that requires postgres MCP to be available.
tools:
  - Read
  - Bash
  - Grep
  - Glob
mcpServers:
  - "postgres"                       ← 使用 postgres MCP
requiredMcpServers:                  ← 必须可用，否则报错
  - "postgres"
permissionMode: acceptEdits
---

You manage database schemas and migrations.

## Rules
- Always read current schema first
- Migrations must be reversible
- Never drop columns without deprecation period
```

## 说明

```
requiredMcpServers vs mcpServers:

  mcpServers: ["postgres"]
    → 声明使用这个 MCP
    → 如果不可用，Agent 仍会启动，但相关工具不可用

  requiredMcpServers: ["postgres"]
    → 声明这个 MCP 必须可用
    → 如果不可用，Agent 拒绝启动，直接报错
    → 用户看到: "Agent db-admin requires MCP server 'postgres' which is not available"

适用场景:
  ✅ 数据库 Agent（没有数据库连接就毫无意义）
  ✅ 外部 API Agent（没有 API 连接无法工作）
  ✅ 确保用户正确配置了 MCP（友好报错而非运行时失败）
```

---

## 使用示例 1: 数据库管理 — postgres MCP 检查

```
用户: "帮我创建一个用户表"

→ 主 Agent 委派 db-admin（requiredMcpServers: ["postgres"]）

场景 A: postgres MCP 已配置

  检查流程:
    requiredMcpServers: ["postgres"] → 检查 → postgres 可用 ✅
    → Agent 正常启动

  Agent 执行:
    1. 调用 postgres MCP 查询当前表结构
    2. 设计 users 表 schema
    3. 生成并执行 CREATE TABLE 语句

  Agent 输出:
    "已创建 users 表:
     - id SERIAL PRIMARY KEY
     - email VARCHAR(255) UNIQUE NOT NULL
     - name VARCHAR(100) NOT NULL
     - created_at TIMESTAMP DEFAULT NOW()"

场景 B: postgres MCP 未配置

  检查流程:
    requiredMcpServers: ["postgres"] → 检查 → postgres 不可用 ❌
    → Agent 拒绝启动

  错误提示:
    "Agent 'db-admin' requires MCP server 'postgres'.
     Please configure it in .claude/settings.json:
     { mcpServers: { postgres: { command: 'npx',
       args: ['@anthropic-ai/mcp-server-postgres'] } } }"

  → 用户根据提示配置后重试 → 成功

requiredMcpServers 的效果:
  没有数据库连接，这个 Agent 完全无法工作。
  requiredMcpServers 在启动前就检查，避免运行到一半才发现缺少 MCP。
```

## 使用示例 2: 部署机器人 — kubernetes MCP 检查

```
用户: "把最新版本部署到生产环境"

→ 主 Agent 委派 deploy-bot（requiredMcpServers: ["kubernetes"]）

Agent 配置:
  requiredMcpServers:
    - "kubernetes"
  tools:
    - Bash
    - Read

场景: kubernetes MCP 未配置（新成员的电脑）

  检查流程:
    requiredMcpServers: ["kubernetes"] → 检查 → kubernetes 不可用 ❌
    → Agent 拒绝启动

  错误提示:
    "Agent 'deploy-bot' requires MCP server 'kubernetes'.
     This MCP provides kubectl access for deployments.
     Please run: npm install -g @anthropic-ai/mcp-server-kubernetes
     Then add to .claude/settings.json under mcpServers."

  → 新成员按照提示安装并配置 → 重新尝试 → 成功

requiredMcpServers 的效果:
  部署是高风险操作，如果 Agent 在运行中途才发现无法连接 K8s，
  可能导致部署流程中断在中间状态（如已构建未推送）。
  提前检查确保一切就绪再开始。
```

## 使用示例 3: 文档生成 — github MCP 检查

```
用户: "根据最近的 PR 生成变更日志"

→ 主 Agent 委派 changelog-generator（requiredMcpServers: ["github"]）

Agent 配置:
  requiredMcpServers:
    - "github"
  mcpServers:
    - "github"
  tools:
    - Read
    - Write

场景 A: github MCP 已配置且有 token

  检查流程:
    requiredMcpServers: ["github"] → 检查 → github 可用 ✅

  Agent 执行:
    1. 调用 github MCP: 获取最近 30 个已合并的 PR
    2. 按 label 分类: feature / bugfix / breaking
    3. 生成 CHANGELOG.md

  Agent 输出:
    "已生成 CHANGELOG.md:
     ## v2.3.0 (2024-01-15)
     ### Features
     - Add user avatar upload (#142) by @alice
     ### Bug Fixes
     - Fix login redirect loop (#139) by @bob
     ### Breaking Changes
     - Remove deprecated /api/v1 endpoints (#135)"

场景 B: github MCP 配置了但 token 过期

  检查流程:
    requiredMcpServers: ["github"] → 检查 → MCP 存在但连接失败 ❌
    → Agent 拒绝启动

  错误提示:
    "Agent 'changelog-generator' requires MCP server 'github'.
     The server exists but connection failed (401 Unauthorized).
     Please update your GitHub token in settings."

requiredMcpServers 的效果:
  不仅检查 MCP 是否声明，还检查是否真正可用。
  避免 Agent 开始读取 PR 后因认证失败产生不完整的变更日志。
```

## 使用示例 4: 监控告警 — 多个必需 MCP

```
用户: "检查生产环境的监控指标和最近告警"

→ 主 Agent 委派 monitor-agent（requiredMcpServers: ["prometheus", "grafana"]）

Agent 配置:
  requiredMcpServers:
    - "prometheus"                    ← 指标数据源
    - "grafana"                       ← 仪表盘和告警
  mcpServers:
    - "prometheus"
    - "grafana"
  tools:
    - Read
    - Bash

场景: prometheus 已配置但 grafana 未配置

  检查流程:
    requiredMcpServers: ["prometheus", "grafana"]
    → prometheus: 可用 ✅
    → grafana: 不可用 ❌
    → Agent 拒绝启动

  错误提示:
    "Agent 'monitor-agent' requires MCP server 'grafana' which is not available.
     Required servers: prometheus ✅, grafana ❌
     Please configure grafana in .claude/settings.json."

  → 用户补充 grafana 配置 → 重新启动 → 成功

requiredMcpServers 的效果:
  当 Agent 依赖多个 MCP 时，缺一不可。
  提前全部检查，列出每个 MCP 的状态，帮助用户快速定位缺失项。
  如果用 mcpServers 代替 requiredMcpServers，Agent 会启动，
  但查询 grafana 告警时才发现工具不存在，导致报告不完整。
```

## 使用示例 5: 云存储操作 — s3 MCP 检查

```
用户: "把构建产物上传到 S3"

→ 主 Agent 委派 upload-agent（requiredMcpServers: ["s3"]）

Agent 配置:
  requiredMcpServers:
    - "s3"
  mcpServers:
    - "s3"
  tools:
    - Bash
    - Glob

场景: 新项目，还没有配置任何 MCP

  检查流程:
    requiredMcpServers: ["s3"] → 检查 → s3 不可用 ❌
    → Agent 拒绝启动

  错误提示:
    "Agent 'upload-agent' requires MCP server 's3'.
     Example configuration for .claude/settings.json:
     {
       mcpServers: {
         s3: {
           command: 'npx',
           args: ['-y', '@anthropic-ai/mcp-server-s3'],
           env: {
             AWS_REGION: 'us-east-1',
             AWS_ACCESS_KEY_ID: '<your-key>',
             AWS_SECRET_ACCESS_KEY: '<your-secret>'
           }
         }
       }
     }
     Never commit credentials to version control."

  → 用户按照模板配置 → 首次配置后启动 → 成功

requiredMcpServers 的效果:
  云存储操作涉及凭证和权限，运行中失败可能产生半上传的文件。
  提前检查并给出完整的配置示例（包括安全提示），
  让新用户也能一步到位地完成配置，而不是反复试错。
```
