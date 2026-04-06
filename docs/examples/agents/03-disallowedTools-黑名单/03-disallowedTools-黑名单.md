# Agent 字段: disallowedTools（工具黑名单）

> 指南 3.2「工具控制」— disallowedTools 禁止特定工具，优先级高于 tools

## 目录结构

```
my-project/
├── .claude/
│   └── agents/
│       ├── readonly-analyst.md      ← 用 tools 白名单（02 示例）
│       └── code-reviewer.md         ← ★ 用 disallowedTools 黑名单
├── src/
└── package.json
```

## Agent 文件内容

```markdown
# 文件: .claude/agents/code-reviewer.md

---
description: Code review specialist. Reviews code for bugs, security issues, and style violations.
disallowedTools:
  - Write
  - Edit
  - NotebookEdit
---

You are a code reviewer. You ONLY review code, never modify it.

## Checklist
1. Security: XSS, SQL injection, hardcoded secrets
2. Performance: N+1 queries, memory leaks
3. Error handling: unhandled promises, empty catches
4. TypeScript: any usage, missing types

## Output
For each issue:
- Severity: Critical / Warning / Info
- File: path:line
- Issue: description
- Fix: suggested code block

End with summary: total by severity.
```

## 说明

```
disallowedTools 黑名单效果:
  不在黑名单 → 可用
  在黑名单   → 禁用

本例声明了 [Write, Edit, NotebookEdit]:
  ✅ Read        ← 不在黑名单，可用
  ✅ Grep        ← 不在黑名单，可用
  ✅ Glob        ← 不在黑名单，可用
  ✅ Bash        ← 不在黑名单，可用
  ❌ Write       ← 在黑名单，禁用
  ❌ Edit        ← 在黑名单，禁用
  ❌ NotebookEdit ← 在黑名单，禁用

tools vs disallowedTools 优先级:
  如果同时设置:
    tools: [Read, Write, Edit]
    disallowedTools: [Edit]
  → Edit 被移除 → 最终只有 [Read, Write]
  → disallowedTools 优先级更高
```

## 使用示例 1: 审查认证模块安全性

```
用户说: "审查一下 src/auth/ 下的代码"
→ 调用 Agent(agentType: "code-reviewer", prompt: "审查 src/auth/")

Agent 执行:
  1. Glob("src/auth/**/*.ts")     ✅ 找到文件
  2. Read("src/auth/jwt.ts")      ✅ 读取代码
  3. Read("src/auth/oauth.ts")    ✅ 读取代码

  ❌ 不会尝试 Write/Edit（黑名单 + prompt 都说只读）

输出:
  ## Code Review: src/auth/

  ### Critical (1)
  - **File**: src/auth/jwt.ts:42
  - **Issue**: Password compared with === instead of timingSafeEqual
  - **Fix**: `import { timingSafeEqual } from 'crypto'`

  ### Warning (2)
  - **File**: src/auth/oauth.ts:18 — Missing state validation
  - **File**: src/auth/jwt.ts:7 — Token expiry too long (24h)

  **Total**: 1 Critical, 2 Warning, 0 Info
```

## 使用示例 2: 审查 PR 中的 API 路由

```
用户说: "帮我 review PR #38 里新增的 API 路由"
→ 调用 Agent(agentType: "code-reviewer", prompt: "审查 PR #38 的 API 路由变更")

Agent 执行:
  1. Bash("gh pr diff 38")                            ✅ 获取 PR diff
  2. Grep("router\.(get|post|put|delete|patch)")       ✅ 搜索路由定义
  3. Read("src/routes/orders.ts")                      ✅ 读取相关路由文件

  ❌ 不会自动修复代码（Write/Edit 被黑名单禁止）
  ❌ 不会修改 Notebook（NotebookEdit 被黑名单禁止）

输出:
  ## PR #38 Code Review: API Routes

  ### Critical (2)
  - **File**: src/routes/orders.ts:25
  - **Issue**: DELETE /orders/:id 缺少权限校验，任何用户可删除他人订单
  - **Fix**: 添加 `requireOwner` middleware

  - **File**: src/routes/orders.ts:48
  - **Issue**: 用户输入直接拼入 SQL 查询（SQL 注入风险）
  - **Fix**: 使用参数化查询 `pool.query('SELECT * FROM orders WHERE id = $1', [id])`

  ### Warning (1)
  - **File**: src/routes/orders.ts:12 — 缺少 rate limiting

  **Total**: 2 Critical, 1 Warning, 0 Info
```

## 使用示例 3: 审查数据库迁移脚本

```
用户说: "检查一下 migrations/ 目录里的迁移脚本有没有问题"
→ 调用 Agent(agentType: "code-reviewer", prompt: "审查数据库迁移脚本")

Agent 执行:
  1. Glob("migrations/**/*.sql")                         ✅ 找到迁移文件
  2. Read("migrations/20240315_add_orders.sql")           ✅ 读取迁移
  3. Read("migrations/20240320_add_index.sql")            ✅ 读取迁移
  4. Grep("DROP|ALTER|DELETE|TRUNCATE")                   ✅ 搜索危险操作

输出:
  ## Migration Review

  ### Critical (1)
  - **File**: migrations/20240315_add_orders.sql:8
  - **Issue**: ALTER TABLE 缺少 IF NOT EXISTS，重复执行会失败
  - **Fix**: `ALTER TABLE IF EXISTS orders ADD COLUMN IF NOT EXISTS status VARCHAR(20)`

  ### Warning (2)
  - **File**: migrations/20240320_add_index.sql:3 — 大表加索引没有 CONCURRENTLY，会锁表
  - **File**: migrations/20240315_add_orders.sql:1 — 缺少事务包裹（BEGIN ... COMMIT）

  ### Info (1)
  - **File**: migrations/20240320_add_index.sql:5 — 建议添加回滚脚本（down migration）

  **Total**: 1 Critical, 2 Warning, 1 Info
```

## 使用示例 4: 审查第三方依赖安全性

```
用户说: "检查一下 package.json 里的依赖有没有安全隐患"
→ 调用 Agent(agentType: "code-reviewer", prompt: "审查依赖安全性")

Agent 执行:
  1. Read("package.json")                     ✅ 读取依赖列表
  2. Bash("npm audit --json")                 ✅ 运行安全审计
  3. Grep("import.*from.*lodash")             ✅ 检查具体使用方式

  ❌ 不能 Edit package.json 来自动升级依赖
  ❌ 不能 Write 生成 audit report 文件
  → 只能输出文本报告，由用户决定是否升级

输出:
  ## Dependency Security Review

  ### Critical (1)
  - **Package**: lodash@4.17.19
  - **Issue**: 原型污染漏洞（CVE-2024-XXXX）
  - **Fix**: 升级到 lodash@4.17.21 或使用 lodash-es

  ### Warning (1)
  - **Package**: axios@0.21.0
  - **Issue**: SSRF 漏洞（CVE-2023-XXXX）
  - **Fix**: 升级到 axios@1.6.0+

  ### Info (1)
  - 发现 3 个未使用的依赖: moment, jquery, core-js
  - 建议执行 npm uninstall 移除，减少攻击面

  **Total**: 1 Critical, 1 Warning, 1 Info
```

## 使用示例 5: disallowedTools vs tools — 选择场景对比

```
场景: 需要一个"只读审查"Agent，有两种写法

写法 A — 用 tools 白名单:
  ---
  tools:
    - Read
    - Grep
    - Glob
    - Bash
  ---
  → 只能用这 4 个工具
  → 新增工具不会自动获得
  → 适合: 严格限制，确保 Agent 绝对不能修改任何东西

写法 B — 用 disallowedTools 黑名单（本例）:
  ---
  disallowedTools:
    - Write
    - Edit
    - NotebookEdit
  ---
  → 禁止这 3 个修改类工具
  → 其他所有工具（包括未来新增的）都可用
  → 适合: 只需要排除"写操作"，允许其他一切

选择原则:
  ✅ 需要精确控制 → 用 tools 白名单（白名单更严格）
  ✅ 只需排除少数工具 → 用 disallowedTools 黑名单（黑名单更灵活）
  ✅ 两者可以组合使用 → 04-工具池组装规则 详解
```
