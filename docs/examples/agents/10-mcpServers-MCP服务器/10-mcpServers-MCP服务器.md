# Agent 字段: mcpServers（专属 MCP 服务器）

> 指南 3.2「MCP 服务器」— Agent 可以连接专属的 MCP 工具

## 目录结构

```
my-project/
├── .claude/
│   ├── agents/
│   │   └── db-analyst.md            ← ★ 带 mcpServers 的 Agent
│   └── settings.json                ← 全局 MCP 配置
├── src/
└── package.json
```

## Agent 文件内容

```markdown
# 文件: .claude/agents/db-analyst.md

---
description: Database analyst that can query databases and analyze schemas.
tools:
  - Read
  - Grep
  - Glob
  - Bash
mcpServers:                         ← Agent 专属 MCP 服务器
  - "shared-postgres"               ← 引用已在 settings.json 中配置的 MCP
  - analytics-db:                   ← 内联定义 MCP（Agent 专属）
      command: "npx"
      args: ["-y", "@anthropic-ai/mcp-server-postgres"]
      env:
        DATABASE_URL: "postgresql://readonly@localhost:5432/analytics"
---

You are a database analyst.

## 能力
- 查询数据库（通过 MCP postgres 工具）
- 分析表结构、索引、查询性能
- 生成优化建议

## Rules
- 只读查询，不修改数据（使用 readonly 账号）
- 大表查询加 LIMIT
- 分析慢查询时使用 EXPLAIN ANALYZE
```

## 说明

```
mcpServers 两种写法:

1. 引用已配置的 MCP（字符串）:
   mcpServers:
     - "shared-postgres"            ← 引用 settings.json 中的配置
   → settings.json 中必须有对应配置

2. 内联定义 MCP（对象）:
   mcpServers:
     - analytics-db:                ← 自定义名称
         command: "npx"
         args: ["-y", "some-mcp-server"]
         env: { ... }
   → Agent 专属，不影响其他 Agent

省略 mcpServers → Agent 使用主 Agent 的 MCP 服务器
```

## 使用示例

### 示例 1：分析数据库表性能

```
用户: "分析一下 users 表的性能问题"

→ Agent(db-analyst)

Agent 执行:
  1. MCP: analytics-db → "SELECT * FROM users LIMIT 5"       ✅ 查数据
  2. MCP: analytics-db → "EXPLAIN ANALYZE SELECT ..."         ✅ 分析查询
  3. Read("prisma/schema.prisma")                             ✅ 读 schema
  4. Grep("User")                                             ✅ 搜索模型定义

mcpServers 效果:
  → Agent 通过 analytics-db MCP 直接连接数据库
  → 可以执行 SQL 查询和 EXPLAIN 分析
  → 不需要用户手动提供数据库连接信息

输出:
  ## Users 表性能分析
  - 缺少 email 字段索引 → 查询走全表扫描
  - created_at 没有索引 → 排序慢
  - 建议: CREATE INDEX CONCURRENTLY idx_users_email ON users(email)
```

### 示例 2：多数据源对比分析

```
用户: "对比主库和从库的数据一致性"

→ Agent(db-analyst) — 同时引用了 shared-postgres 和 analytics-db

Agent 执行:
  1. MCP: shared-postgres → "SELECT COUNT(*) FROM orders"        ✅ 查主库
  2. MCP: analytics-db → "SELECT COUNT(*) FROM orders"           ✅ 查从库
  3. MCP: shared-postgres → "SELECT id, updated_at FROM orders ORDER BY updated_at DESC LIMIT 10"
  4. MCP: analytics-db → "SELECT id, updated_at FROM orders ORDER BY updated_at DESC LIMIT 10"
  5. Bash("diff <(echo '主库数据') <(echo '从库数据')")            ✅ 对比

mcpServers 效果:
  → 一个 Agent 同时连接两个不同的数据库
  → 引用式 (shared-postgres) + 内联式 (analytics-db) 可混合使用
  → 对比结果直接在 Agent 内完成，无需人工搬运数据

输出:
  ## 数据一致性报告
  - 主库 orders: 15,234 行
  - 从库 orders: 15,230 行（差 4 行）
  - 缺失记录 ID: [1042, 2087, 3991, 5210]
  - 建议: 检查从库复制延迟，最后同步时间 2 小时前
```

### 示例 3：生成数据库迁移建议

```
用户: "我们的订单表需要支持多币种，帮我看看怎么改"

→ Agent(db-analyst)

Agent 执行:
  1. MCP: shared-postgres → "SELECT column_name, data_type FROM information_schema.columns WHERE table_name = 'orders'"
  2. MCP: shared-postgres → "SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'orders'"
  3. Grep("Order")                                             ✅ 搜索代码中的模型定义
  4. Read("src/types/order.ts")                                ✅ 读取 TypeScript 类型定义

mcpServers 效果:
  → 通过 MCP 直接读取数据库实际表结构，不依赖文档
  → 获取真实的索引和外键约束信息
  → 结合代码中的类型定义，生成准确的迁移 SQL

输出:
  ## 多币种改造方案
  当前: amount DECIMAL(10,2), 仅支持单一币种

  迁移 SQL:
    ALTER TABLE orders ADD COLUMN currency VARCHAR(3) DEFAULT 'CNY';
    ALTER TABLE orders ADD COLUMN amount_usd DECIMAL(10,2);
    CREATE INDEX idx_orders_currency ON orders(currency);
  影响: 需同步修改 3 个代码文件
```

### 示例 4：监控慢查询日志

```
用户: "最近线上有慢查询，帮我排查"

→ Agent(db-analyst)

Agent 执行:
  1. MCP: analytics-db → "SELECT query, calls, mean_exec_time, max_exec_time FROM pg_stat_statements ORDER BY mean_exec_time DESC LIMIT 20"
  2. MCP: analytics-db → "SELECT pid, query, state, duration FROM pg_stat_activity WHERE state = 'active'"
  3. 对最慢的 3 条查询执行 EXPLAIN ANALYZE

mcpServers 效果:
  → 通过 MCP 访问 pg_stat_statements 系统视图
  → 实时获取当前正在运行的查询状态
  → 自动化性能排查流程，无需 DBA 手动介入

输出:
  ## 慢查询排查报告
  Top 3 慢查询:
  1. SELECT * FROM orders WHERE status = 'pending' (avg 3.2s) → 缺索引
  2. SELECT u.*, COUNT(o.id) FROM users u JOIN orders o ... (avg 2.1s) → JOIN 缺优化
  3. SELECT * FROM products WHERE description LIKE '%手机%' (avg 1.8s) → 全文搜索建议用 pg_trgm

  当前活跃长事务: PID 12345, 运行 45 秒, 建议 KILL
```

### 示例 5：跨环境 Schema 演巡检

```
用户: "帮我检查开发和生产环境的数据库 schema 是否一致"

→ Agent(db-analyst)

Agent 执行:
  1. MCP: shared-postgres → 开发环境 schema 查询
     "SELECT table_name, column_name, data_type, is_nullable FROM information_schema.columns ORDER BY table_name, ordinal_position"
  2. MCP: analytics-db → 生产环境同样查询（只读连接）
  3. Bash → 对比两个 schema 的差异

mcpServers 效果:
  → 两个 MCP 连接分别指向不同环境的数据库
  → 通过 readonly 账号安全查询生产数据
  → 自动化巡检，发现 schema 漂移问题

输出:
  ## Schema 巡检结果
  一致表: 42 个
  不一致表: 3 个
    - users: 开发多了 phone_verified 字段（生产未上线）
    - payments: 生产多了 stripe_session_id 字段（热修复添加）
    - notifications: status 字段类型不同（VARCHAR vs ENUM）
  建议: 同步 schema 并更新迁移脚本
```
