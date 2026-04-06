---
paths:
  - "**/prisma/**"
  - "**/db/**"
  - "**/migrations/**"
  - "**/schema.prisma"
---

# 条件规则 — 数据库规范

> 有 `paths` → 仅当操作数据库相关文件时注入
> `**` 在中间表示任意层级子目录

## 目录结构

```
my-project/
├── .claude/
│   └── rules/
│       ├── database.md              ← ★ 本规则
│       ├── backend.md
│       └── ...
├── prisma/                          ← 匹配 "**/prisma/**" ✓
│   ├── schema.prisma                ← 同时匹配 "**/schema.prisma" ✓
│   ├── seed.ts                      ← 匹配 "**/prisma/**" ✓
│   └── migrations/                  ← 匹配 "**/migrations/**" ✓
│       ├── 20240101_init/
│       │   └── migration.sql        ← 匹配 "**/migrations/**" ✓
│       └── 20240102_add_users/
│           └── migration.sql
├── src/
│   └── db/                          ← 匹配 "**/db/**" ✓
│       ├── connection.ts
│       ├── repositories/
│       │   └── user-repo.ts
│       └── seeds/
│           └── dev-seed.ts
├── packages/
│   └── core/
│       └── db/                      ← 匹配 "**/db/**" ✓（任意深度）
│           └── client.ts
└── apps/
    └── api/
        └── prisma/                  ← 匹配 "**/prisma/**" ✓（任意深度）
            └── schema.prisma
```

## 规则文件内容

```
Prisma: 模型按分组，必须有 id/createdAt/updatedAt，关系显式定义 onDelete/onUpdate
Migration: 必须有 down，数据和 schema 分开，禁止删除有数据列（先 deprecated）
索引: 外键必建，频繁查询建复合索引，唯一约束用 @@unique
```

## 触发示例

```
场景 1: 用户让 Claude 加一个新表
  → Edit("prisma/schema.prisma")
  → 匹配 "**/schema.prisma" ✓ → database.md 注入
  → Claude 按 id+createdAt+updatedAt+onDelete 规范添加

场景 2: 用户让 Claude 写 migration
  → Write("prisma/migrations/20240103_add_orders/migration.sql")
  → 匹配 "**/migrations/**" ✓ → database.md 注入
  → Claude 同时写 up 和 down migration

场景 3: 用户让 Claude 修改数据库连接配置
  → Edit("src/db/connection.ts")
  → 匹配 "**/db/**" ✓ → database.md 注入
```
