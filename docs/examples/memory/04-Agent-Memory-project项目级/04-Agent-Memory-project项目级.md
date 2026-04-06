# Memory: Agent Memory — project scope（项目级共享）

> 指南 8.2「Agent Memory」— project scope 随项目提交 VCS，团队共享

## 目录结构

```
my-project/
├── .claude/
│   ├── agents/
│   │   └── migrator.md
│   └── agent-memory/
│       └── migrator/
│           └── MEMORY.md          ← project scope，提交 Git
└── package.json
```

## Agent 文件内容

```markdown
# .claude/agents/migrator.md
---
description: Database migration specialist
memory: project                    ← project scope
tools: [Read, Write, Edit, Bash]
---
Handle Prisma migrations.
```

## 说明

```
Agent Memory — project scope:
  路径: .claude/agent-memory/<agentType>/MEMORY.md
  范围: 项目级（当前项目）
  提交 VCS: 是 → 团队成员通过 Git 共享
  优先级: 高于 user scope（项目知识覆盖通用知识）
```

## 使用示例

### 示例 1：migrator Agent 记住 schema 版本

```
.claude/agent-memory/migrator/MEMORY.md:

## 项目数据库状态
- 数据库: PostgreSQL 15
- Schema 版本: 3.2.1
- 最近迁移: add-user-email (2026-04-03)
- 待处理: 添加 orders 表索引

→ git add .claude/agent-memory/migrator/MEMORY.md
→ 团队所有成员共享 schema 状态
```

### 示例 2：api-gen Agent 记住项目 API 模式

```
.claude/agent-memory/api-gen/MEMORY.md:

## 项目 API 约定
- 框架: Express + TypeScript
- 验证: Zod
- 认证: JWT Bearer token
- 响应格式: { success, data, error }
- 分页: cursor-based

→ 团队成员调用 api-gen → 遵循相同模式
```

### 示例 3：reviewer Agent 记住项目代码规范

```
.claude/agent-memory/reviewer/MEMORY.md:

## 项目规范
- 组件: .tsx 函数式组件
- 测试: Vitest 放 __tests__/
- 样式: CSS Modules
- 状态: Zustand（不是 Redux）

## 项目常见问题
- 新成员忘加 error boundary
- API 调用缺少错误处理
```

### 示例 4：project scope 与 user scope 共存

```
user scope (~/.claude/agent-memory/reviewer/MEMORY.md):
  "通用审查经验: 80% 项目缺少 input validation"

project scope (.claude/agent-memory/reviewer/MEMORY.md):
  "本项目用 Vitest，不用 Jest"

→ Agent 加载两者，project scope 覆盖冲突部分
```

### 示例 5：通过 Git 共享 Agent 记忆

```
# .gitignore 配置
.claude/agent-memory-local/       ← local 不提交
# .claude/agent-memory/           ← project scope 要提交

# 提交:
git add .claude/agent-memory/
git commit -m "chore: update agent memory"

# 团队成员:
git pull → 自动获得 Agent 记忆
```
