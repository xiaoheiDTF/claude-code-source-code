# @include 引用 — 数据库 Schema 引用

> 在规则中引用 Prisma Schema，操作数据库文件时自动获得当前数据模型上下文

## 目录结构

```
my-project/
├── .claude/
│   ├── rules/
│   │   ├── database-context.md      ← ★ 本规则（@ 引用 schema.prisma）
│   │   └── templates/
│   │       └── migration.sql        ← @ 引用的 migration 模板
│   └── settings.json
├── prisma/
│   ├── schema.prisma                ← @ 引用的目标文件（当前数据库模型）
│   ├── seed.ts
│   └── migrations/
│       ├── 20240101_init/
│       │   └── migration.sql
│       └── 20240102_add_users/
│           └── migration.sql
├── src/
│   └── db/
│       ├── connection.ts            ← 操作此文件触发规则
│       └── repositories/
│           └── user-repo.ts         ← 操作此文件触发规则
└── package.json
```

## 规则文件内容

```markdown
# 文件: .claude/rules/database-context.md

---
paths:
  - "**/prisma/**"
  - "**/db/**"
  - "**/migrations/**"
---

# 数据库上下文

## 当前 Schema（自动注入最新内容）
@./prisma/schema.prisma

## Migration 模板
@./.claude/templates/migration.sql

## 规范
- Schema 变更必须生成 migration
- 禁止直接修改数据库
- 新增表必须有 id, createdAt, updatedAt
- 字段用 camelCase（Prisma 自动映射 snake_case）
```

## 被引用的 schema.prisma 展开

```prisma
// 当规则被加载时，schema.prisma 的完整内容被注入:

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(uuid())
  email     String   @unique
  name      String?
  role      Role     @default(USER)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  posts     Post[]
}

model Post {
  id        String   @id @default(uuid())
  title     String
  content   String?
  published Boolean  @default(false)
  authorId  String
  author    User     @relation(fields: [authorId], references: [id], onDelete: Cascade)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

enum Role {
  USER
  ADMIN
}
```

## 触发示例

```
场景 1: 用户让 Claude 加一个 Comment 表

操作: Edit("prisma/schema.prisma")
  → 匹配 "**/prisma/**" ✓ → database-context.md 加载
  → @./prisma/schema.prisma 展开 → Claude 看到 User, Post 模型
  → Claude 知道:
    ✅ Comment 需要关联 Post（已有模型）
    ✅ 必须有 id, createdAt, updatedAt
    ✅ 字段用 camelCase
    ✅ onDelete 行为

  生成:
    model Comment {
      id        String   @id @default(uuid())
      content   String
      postId    String
      post      Post     @relation(fields: [postId], references: [id], onDelete: Cascade)
      authorId  String
      author    User     @relation(fields: [authorId], references: [id])
      createdAt DateTime @default(now())
      updatedAt DateTime @updatedAt
    }

    // 还需要更新 Post 和 User 模型添加 comments 关联字段

场景 2: 用户让 Claude 写一个 migration

操作: Write("prisma/migrations/20240103_add_comments/migration.sql")
  → 匹配 "**/migrations/**" ✓ → 规则加载
  → schema.prisma 展开 → Claude 看到当前完整 Schema
  → 模板展开 → Claude 按 migration 模板格式编写
```
