# settings.json: 完整实战配置

> 指南 7.2「主要配置项」— 综合运用所有配置项的生产级 settings.json

## 使用示例

### 示例 1：前端团队项目配置

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Edit",
      "Write",
      "Glob",
      "Grep",
      "Bash(git *)",
      "Bash(npm test*)",
      "Bash(npm run lint*)",
      "Bash(npm run build*)",
      "Bash(npx jest*)",
      "Bash(npx eslint*)",
      "Bash(npx prettier*)",
      "Bash(npx tsc*)"
    ],
    "deny": [
      "Bash(rm -rf*)",
      "Bash(npm publish*)",
      "Bash(git push --force*)",
      "Write(production/*)"
    ],
    "defaultMode": "acceptEdits",
    "additionalDirectories": ["/shared/design-tokens"]
  },
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/auto-lint.sh",
            "async": true,
            "statusMessage": "Auto-linting..."
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/check-branch.sh"
          }
        ]
      }
    ]
  },
  "mcpServers": {
    "figma": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-figma"],
      "env": { "FIGMA_ACCESS_TOKEN": "$FIGMA_TOKEN" }
    }
  },
  "agentModel": "sonnet",
  "theme": "dark",
  "verbose": false
}
```

```
→ 团队前端开发配置:
  permissions:
    → 自动允许: 读写、git、npm test/lint/build
    → 拒绝: rm -rf、npm publish、force push
    → 默认模式: acceptEdits（自动编辑）
    → 额外目录: 共享设计 token

  hooks:
    → 编辑后自动 lint
    → 编辑前检查分支（不允许直接改 main）

  mcpServers:
    → Figma MCP 读取设计稿

  agentModel: sonnet → 平衡能力和成本
```

### 示例 2：后端/数据库项目配置

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Edit",
      "Write",
      "Glob",
      "Grep",
      "Bash(git *)",
      "Bash(npm test*)",
      "Bash(npx prisma *)",
      "Bash(docker compose *)",
      "Bash(docker logs*)",
      "Bash(curl localhost*)"
    ],
    "deny": [
      "Bash(rm -rf*)",
      "Bash(docker rm*)",
      "Bash(docker rmi*)",
      "Bash(npm publish*)",
      "Edit(.env.production)",
      "Write(production/*)"
    ],
    "defaultMode": "default",
    "additionalDirectories": ["../api-types"]
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/check-migration.sh",
            "once": true,
            "statusMessage": "Checking pending migrations..."
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/auto-test.sh",
            "async": true,
            "asyncRewake": true
          }
        ]
      }
    ]
  },
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "postgresql://localhost:5432/mydb_dev"]
    }
  },
  "agentModel": "opus",
  "verbose": false
}
```

```
→ 后端/数据库项目配置:
  permissions:
    → 允许: Prisma、Docker Compose（只读）、curl localhost
    → 拒绝: docker rm、生产环境文件
    → 默认模式: default（手动确认，后端更谨慎）

  hooks:
    → 首次编辑前检查数据库迁移状态
    → 编辑后自动运行相关测试

  mcpServers:
    → PostgreSQL MCP 直接查数据库

  agentModel: opus → 后端需要更强推理
```

### 示例 3：安全审计项目配置

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep"
    ],
    "deny": [
      "Edit",
      "Write",
      "Bash(*)"
    ],
    "defaultMode": "default"
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/audit-log.sh"
          }
        ]
      }
    ]
  },
  "agentModel": "opus",
  "effort": "high",
  "verbose": true
}
```

```
→ 安全审计配置（严格只读）:
  permissions:
    → 只允许: Read、Glob、Grep
    → 拒绝: Edit、Write、所有 Bash
    → 审计 Agent 不能修改任何文件

  hooks:
    → 所有工具调用记录审计日志

  agentModel: opus → 安全分析需要最强推理
  effort: high → 深度分析
  verbose: true → 记录详细日志

效果:
  → 纯只读审计，不可能修改代码
  → 所有操作有审计追踪
  → 使用最强模型做深度安全分析
```

### 示例 4：CI/CD 自动化配置

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Write",
      "Edit",
      "Bash(*)",
      "Glob",
      "Grep"
    ],
    "deny": [
      "Bash(rm -rf /)",
      "Bash(npm publish*)"
    ],
    "defaultMode": "auto"
  },
  "hooks": {
    "SessionEnd": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/summary.sh",
            "async": true
          }
        ]
      }
    ]
  },
  "agentModel": "haiku",
  "verbose": false
}
```

```
→ CI/CD 自动化配置:
  permissions:
    → 全自动（auto 模式）
    → 允许所有工具
    → 只禁止 rm -rf / 和 npm publish

  hooks:
    → 会话结束生成摘要

  agentModel: haiku → CI 用便宜模型
  verbose: false → 简洁输出

效果:
  → 全自动，无需人工干预
  → 最低成本（haiku 模型）
  → 适合批量任务（代码生成、测试、文档）
```

### 示例 5：新开发者推荐配置

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep",
      "Bash(git log*)",
      "Bash(git diff*)",
      "Bash(git status*)",
      "Bash(npm test*)",
      "Bash(npm run build*)"
    ],
    "deny": [
      "Bash(rm -rf*)",
      "Bash(git push --force*)",
      "Bash(npm publish*)",
      "Write(.env*)",
      "Edit(.env*)"
    ],
    "defaultMode": "default"
  },
  "hooks": {
    "UserPromptSubmit": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/inject-context.sh"
          }
        ]
      }
    ]
  },
  "agentModel": "sonnet",
  "theme": "dark",
  "verbose": true
}
```

```
→ 新开发者友好配置:
  permissions:
    → 只读 git 命令自动通过
    → 测试和构建自动通过
    → 其他操作需手动确认（学习过程）
    → 禁止危险操作和 .env 文件修改

  hooks:
    → 每条消息自动注入项目上下文（帮助新人理解）

  agentModel: sonnet → 平衡选择
  verbose: true → 帮助理解 Claude 的操作

效果:
  → 新人安全的入门配置
  → 自动允许安全的只读操作
  → 写入操作需确认（学习阶段）
  → verbose 帮助理解 Claude 在做什么
```
