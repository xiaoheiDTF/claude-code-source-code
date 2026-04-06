# Agent 字段: omitClaudeMd（省略 CLAUDE.md）

> 指南 3.2「其他」— `omitClaudeMd: true` 不加载 CLAUDE.md，节省 token

## 目录结构

```
my-project/
├── CLAUDE.md                        ← 100 行项目指令（占用 token）
├── .claude/
│   ├── rules/
│   │   ├── coding.md                ← 50 行编码规范
│   │   └── frontend.md              ← 30 行前端规范
│   └── agents/
│       ├── fullstack-dev.md         ← omitClaudeMd: false（默认，加载全部）
│       └── quick-reader.md          ← ★ omitClaudeMd: true（不加载 CLAUDE.md）
└── src/
```

## Agent 文件内容

```markdown
# 文件: .claude/agents/quick-reader.md

---
description: Quick file reader for simple lookups. Minimal context loading.
omitClaudeMd: true                   ← 不加载 CLAUDE.md 和 rules
model: haiku                         ← 快速模型
effort: low                          ← 低推理
maxTurns: 5                          ← 最多 5 轮
tools:
  - Read
  - Grep
  - Glob
---

Quickly read and answer questions about code.
No need to follow project conventions — just read and report.
```

## 说明

```
omitClaudeMd 选项:
  omitClaudeMd: true   → 不加载 CLAUDE.md、rules、子目录指令
  omitClaudeMd: false  → 加载全部（默认）
  省略                 → 加载全部（默认）

节省 token 对比:
  omitClaudeMd: false:
    加载: CLAUDE.md(100行) + rules/coding.md(50行) + rules/frontend.md(30行) + ...
    Token: ~5000 tokens 用于上下文

  omitClaudeMd: true:
    加载: 无
    Token: ~0 tokens，全部留给 Agent 的实际任务

适用场景:
  ✅ 只读查询 Agent（quick-reader, grep-helper）
  ✅ 通用工具 Agent（不需要项目特定知识）
  ✅ 轻量快速任务（节省 token = 节省成本和时间）
  ❌ 需要遵循项目规范的 Agent（如代码审查、功能实现）
```

---

## 使用示例 1: 快速查看函数用途

```
用户: "utils/helpers.ts 里的 formatDate 函数是干什么的？"

→ 主 Agent 委派 quick-reader（omitClaudeMd: true）

Agent 执行:
  1. 跳过 CLAUDE.md、rules 加载（省 ~5000 tokens）
  2. Read("src/utils/helpers.ts")

Agent 输出:
  "formatDate(date: Date): string
   将 Date 对象格式化为 'YYYY-MM-DD' 字符串。
   参数: date - 日期对象
   返回: '2024-01-15' 格式的字符串"

omitClaudeMd 的效果:
  此任务完全不需要项目规范，只读取一个文件并回答。
  省略 CLAUDE.md 后 token 消耗从 ~5500 降至 ~500，节省 90%。
```

## 使用示例 2: 搜索 API 端点定义

```
用户: "项目中哪些文件定义了 POST 请求的路由？"

→ 主 Agent 委派 quick-reader（omitClaudeMd: true）

Agent 执行:
  1. 跳过全部指令文件加载
  2. Grep("app\\.post\\(", "**/*.ts")
  3. 汇总所有匹配文件和行号

Agent 输出:
  "找到 12 个 POST 路由定义:
   - src/routes/auth.ts:15   POST /api/login
   - src/routes/users.ts:42  POST /api/users
   - src/routes/orders.ts:78 POST /api/orders
   ..."

omitClaudeMd 的效果:
  纯搜索任务，不需要知道项目的编码规范、分支策略或提交格式。
  omitClaudeMd: true 避免加载与任务无关的 180 行项目指令。
```

## 使用示例 3: 统计代码行数

```
用户: "统计 src/components 目录下有多少个 .tsx 文件"

→ 主 Agent 委派 quick-reader（omitClaudeMd: true）

Agent 执行:
  1. Glob("src/components/**/*.tsx")
  2. 逐个 Read 并统计行数
  3. 输出汇总表格

Agent 输出:
  "| 文件 | 行数 |
   |------|------|
   | Button.tsx | 45 |
   | Modal.tsx | 120 |
   | Table.tsx | 210 |
   | 总计: 3 个文件, 375 行 |"

omitClaudeMd 的效果:
  纯统计任务，结果完全基于文件内容，与项目规范无关。
  不加载 CLAUDE.md 让 haiku 模型的 200K token 窗口全部用于文件读取。
```

## 使用示例 4: 对比两个文件的差异

```
用户: "config.dev.json 和 config.prod.json 有什么不同？"

→ 主 Agent 委派 quick-reader（omitClaudeMd: true）

Agent 执行:
  1. Read("config.dev.json")
  2. Read("config.prod.json")
  3. 逐字段对比差异

Agent 输出:
  "差异如下:
   - database.host: 'localhost' → 'db.prod.internal'
   - database.port: 5432 → 5433
   - cache.ttl: 300 → 3600
   - log.level: 'debug' → 'warn'
   相同字段: 15 个（如 app.name, version 等）"

omitClaudeMd 的效果:
  配置文件对比不需要知道项目编码规范。
  omitClaudeMd: true 确保 Agent 只关注文件内容本身，不被项目规则干扰。
```

## 使用示例 5: 查找 TODO 和 FIXME 标记

```
用户: "帮我找出代码里所有 TODO 和 FIXME 注释"

→ 主 Agent 委派 quick-reader（omitClaudeMd: true）

Agent 执行:
  1. Grep("TODO|FIXME", "**/*.{ts,tsx,js,jsx}")
  2. 收集所有匹配项，按文件分组

Agent 输出:
  "共找到 8 处待处理标记:
   TODO (5):
   - src/api/auth.ts:23    TODO: add rate limiting
   - src/api/users.ts:67   TODO: validate email format
   - src/utils/cache.ts:12 TODO: implement LRU eviction
   - ...

   FIXME (3):
   - src/components/Form.tsx:89  FIXME: race condition on double submit
   - ..."

omitClaudeMd 的效果:
  扫描标记是纯文本搜索任务，与项目开发规范、Git 策略完全无关。
  不加载 CLAUDE.md 让 Agent 启动更快，搜索结果更纯粹。
```
