---
paths:
  - "apps/api/**"
  - "packages/core/**"
  - "packages/shared/**"
---

# 条件规则 — 后端开发规范

> 有 `paths` → 仅当操作 `apps/api/` 或 `packages/core/` 等文件时注入

## 目录结构

```
my-monorepo/
├── .claude/
│   └── rules/
│       ├── frontend.md              ← 条件：apps/web/**
│       ├── backend.md               ← ★ 本规则（条件：apps/api/** + packages/core/** + packages/shared/**）
│       └── database.md              ← 条件：**/prisma/**
├── apps/
│   ├── api/                         ← paths 匹配
│   │   └── src/
│   │       ├── routes/
│   │       │   └── users.ts         ← 操作此文件 → 触发 backend.md
│   │       ├── middleware/
│   │       │   └── auth.ts          ← 操作此文件 → 触发 backend.md
│   │       └── services/
│   │           └── user-service.ts  ← 操作此文件 → 触发 backend.md
│   └── web/
│       └── src/                     ← 不匹配 backend.md，匹配 frontend.md
├── packages/
│   ├── core/                        ← paths 匹配
│   │   └── src/
│   │       └── business-logic.ts    ← 操作此文件 → 触发 backend.md
│   ├── shared/                      ← paths 匹配
│   │   └── src/
│   │       └── types.ts             ← 操作此文件 → 触发 backend.md
│   └── ui/                          ← 不匹配 backend.md
└── prisma/
    └── schema.prisma                ← 匹配 database.md（不匹配 backend.md）
```

## 规则文件内容

```
技术栈: Node.js + Fastify + Prisma + Zod
API: RESTful，资源名复数，Zod 验证，统一响应格式 { success, data?, error? }
安全: bcrypt(salt≥12)、参数化查询、Rate Limiting、文件上传限制
```

## 触发示例

```
场景 1: 用户让 Claude 写一个新的 API 路由
  → Edit("apps/api/src/routes/orders.ts")
  → 匹配 "apps/api/**" ✓ → backend.md 注入
  → Claude 按 Fastify + Zod 验证 + 统一响应格式编写

场景 2: 用户让 Claude 修改共享类型
  → Edit("packages/shared/src/types/api.ts")
  → 匹配 "packages/shared/**" ✓ → backend.md 注入

场景 3: 用户让 Claude 同时加 API 路由和 Prisma migration
  → Edit("apps/api/src/routes/orders.ts")  → 触发 backend.md
  → Edit("prisma/schema.prisma")            → 触发 database.md
  → 两条规则同时生效，Claude 同时遵循两套规范
```
