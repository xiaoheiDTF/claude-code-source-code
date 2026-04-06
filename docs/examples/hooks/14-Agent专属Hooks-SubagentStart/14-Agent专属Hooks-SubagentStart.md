# Agent 专属 Hooks: SubagentStart（Agent 启动时）

> 指南 5.3「Agent 可用的 Hook 事件」— SubagentStart 在 Agent 启动时触发

## 目录结构

```
my-project/
├── .claude/
│   └── agents/
│       ├── migrator.md             ← 数据库迁移 Agent
│       ├── onboarding.md           ← 新人引导 Agent
│       ├── perf-analyzer.md        ← 性能分析 Agent
│       └── security-reviewer.md    ← 安全审查 Agent
├── prisma/
│   └── schema.prisma
└── package.json
```

## Agent 文件内容

```markdown
# 文件: .claude/agents/migrator.md

---
description: Database migration specialist
tools:
  - Read
  - Write
  - Edit
  - Bash
hooks:
  SubagentStart:
    - matcher: ""
      hooks:
        - type: command
          command: |
            SCHEMA_VERSION=$(grep '// version:' prisma/schema.prisma | head -1)
            PENDING=$(npx prisma migrate status 2>&1 | grep "pending" || echo "none")
            echo "{\"additionalContext\": \"Schema: $SCHEMA_VERSION | 待执行迁移: $PENDING\"}"
---

You handle database migrations...
```

## 说明

```
SubagentStart 事件:
  触发时机: Agent 启动时（session 创建后，执行任务前）
  matcher: 通常为空 ""（匹配所有）

  典型用途:
    - 加载项目上下文（数据库版本、环境变量）
    - 检查前置条件（依赖是否安装、服务是否运行）
    - 注入启动信息给 Agent

  与 initialPrompt 的区别:
    SubagentStart → Hook 事件，执行 shell 命令，可以返回 additionalContext
    initialPrompt → 预加载的斜杠命令文本
    两者可以配合使用
```

## 使用示例

### 示例 1：数据库迁移 Agent 启动时加载 schema 版本

```
Agent: .claude/agents/migrator.md
  SubagentStart → 读取 prisma/schema.prisma + 检查 pending migrations

→ 用户: "用 migrator 添加用户邮箱字段"

→ Agent 启动 → SubagentStart Hook 触发:
  1. 执行: grep '// version:' prisma/schema.prisma
     → "// version: 3.2.1"
  2. 执行: npx prisma migrate status
     → "2 pending migrations: add-user-table, add-order-table"
  3. 返回: {"additionalContext": "Schema: // version: 3.2.1 | 待执行迁移: 2 pending"}

→ Agent 收到上下文:
  "当前 schema 版本 3.2.1，有 2 个待执行迁移。
   先执行待处理的迁移，再添加新字段..."

效果:
  → Agent 启动时自动了解当前数据库状态
  → 不需要先花一轮查询 schema
  → 避免在旧 schema 上创建迁移
```

### 示例 2：安全审查 Agent 启动时检查依赖漏洞

```
Agent: .claude/agents/security-reviewer.md
  SubagentStart → 运行 npm audit + 检查已知漏洞

→ 用户: "用 security-reviewer 检查认证模块"

→ Agent 启动 → SubagentStart Hook 触发:
  1. 执行: npm audit --json 2>/dev/null
  2. 提取: vulnerabilities: { high: 3, medium: 7, low: 12 }
  3. 检查: .env 文件是否存在、是否有敏感信息泄露
  4. 返回: {"additionalContext": "依赖安全状态: 3高危/7中危/12低危。
     高危包: lodash@4.17.20(prototype pollution), ..."}
  5. 执行: cat .gitignore | grep -c ".env" → 确认 .env 已忽略

→ Agent 收到安全概览:
  "项目有 3 个高危依赖漏洞，我会优先检查这些包的使用情况..."

效果:
  → 安全审查前自动获取整体安全状况
  → Agent 可以聚焦在最高优先级的问题
  → SubagentStart 做预处理，Agent 直接进入深度分析
```

### 示例 3：性能分析 Agent 启动时获取项目指标

```
Agent: .claude/agents/perf-analyzer.md
  SubagentStart → 获取 bundle size + 依赖数量 + TypeScript 配置

→ 用户: "用 perf-analyzer 分析首页加载性能"

→ Agent 启动 → SubagentStart Hook 触发:
  1. 执行: du -sh node_modules/ → "245M"
  2. 执行: cat package.json | jq '.dependencies | length' → "42"
  3. 执行: ls -la .next/ 2>/dev/null || echo "未构建"
  4. 执行: cat next.config.js | grep -E "compress|image|experimental" || echo "默认配置"
  5. 返回: {"additionalContext": "项目规模: 42个依赖, node_modules 245MB,
     构建状态: 未构建, Next.js 配置: 默认"}

→ Agent 基于项目规模制定分析策略:
  "项目有 42 个依赖且未优化过，我从 bundle 分析开始..."

效果:
  → Agent 启动时掌握项目全貌
  → 根据项目规模调整分析策略
  → 不需要先花 2-3 轮收集基础信息
```

### 示例 4：新人引导 Agent 启动时展示项目地图

```
Agent: .claude/agents/onboarding.md
  SubagentStart → 生成项目结构概览 + 技术栈 + 最近活动

→ 用户: "用 onboarding 帮新同事了解项目"

→ Agent 启动 → SubagentStart Hook 触发:
  1. 执行: find src -type d | head -15 → 目录结构
  2. 执行: cat package.json | jq '{deps: .dependencies, scripts: .scripts}' → 技术栈
  3. 执行: git log --oneline -5 → 最近提交
  4. 执行: git shortlog -sn --all | head -5 → 贡献者
  5. 返回: {"additionalContext": "项目概览:
     目录: src/api, src/components, src/hooks, src/utils
     技术栈: React 18 + Next.js 14 + Prisma + tRPC
     最近活动: feat: add search, fix: auth timeout
     主要贡献者: zhangsan(120), lisi(89)..."}

→ Agent 直接开始引导:
  "欢迎加入项目！这是一个 React + Next.js 全栈应用，
   主要目录: api(后端)、components(前端)、hooks(自定义hooks)..."

效果:
  → 新人 Agent 启动时自动展示项目地图
  → 包含技术栈、目录结构、团队信息
  → 新成员不需要自己探索项目结构
```

### 示例 5：启动时检查前置条件，不满足则注入警告

```
Agent: .claude/agents/e2e-tester.md
  SubagentStart → 检查 Playwright 是否安装 + 测试环境是否就绪

→ 用户: "用 e2e-tester 运行端到端测试"

→ Agent 启动 → SubagentStart Hook 触发:
  1. 检查 Playwright:
     npx playwright --version → 未安装 ❌
  2. 检查测试环境:
     curl -s http://localhost:3000 → 连接失败 ❌
  3. 检查测试数据库:
     npx prisma db push --check → 数据库未运行 ❌
  4. 返回: {"additionalContext": "⚠️ 前置条件检查失败:
     1. Playwright 未安装 → 运行: npx playwright install
     2. 开发服务器未启动 → 运行: npm run dev
     3. 测试数据库未运行 → 运行: docker-compose up -d db"}

→ Agent 先处理前置条件:
  "检测到 3 个前置条件未满足，我先逐一安装和启动...
   → npx playwright install
   → docker-compose up -d db
   → npm run dev &"
  等待就绪后开始运行测试

效果:
  → Agent 启动时自动检查运行环境
  → 不满足条件时注入警告和修复建议
  → 避免盲目执行导致大量错误
  → 前置条件检查 + 自动修复
```
