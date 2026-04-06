# 企业级场景 — Monorepo 完整规则体系

> 大型 Monorepo 的规则体系：按编号分层、按目录分区、条件+无条件混合

## 目录结构

```
my-monorepo/
├── CLAUDE.md                                  ← 项目总指令
│   （内容：架构概述、Turborepo、pnpm workspace）
│
├── .claude/
│   ├── settings.json                          ← 项目配置（hooks + 权限）
│   │
│   ├── rules/
│   │   │
│   │   │─────────────────────────────────
│   │   │  00-09: 无条件规则（始终加载）
│   │   │─────────────────────────────────
│   │   ├── 00-general.md                      ← 全局编码规范
│   │   ├── 01-commit.md                       ← 提交规范
│   │   ├── 02-dependencies.md                 ← 依赖管理规范
│   │   │
│   │   │─────────────────────────────────
│   │   │  10-19: 前端条件规则
│   │   │─────────────────────────────────
│   │   ├── 10-frontend.md                     ← paths: "apps/web/**, packages/ui/**"
│   │   ├── 11-frontend-test.md                ← paths: "apps/web/**/*.test.*"
│   │   ├── 12-components.md                   ← paths: "apps/web/src/components/**"
│   │   ├── 13-hooks.md                        ← paths: "apps/web/src/hooks/**"
│   │   ├── 14-pages.md                        ← paths: "apps/web/src/app/**"
│   │   │
│   │   │─────────────────────────────────
│   │   │  20-29: 后端条件规则
│   │   │─────────────────────────────────
│   │   ├── 20-backend.md                      ← paths: "apps/api/**, packages/core/**"
│   │   ├── 21-backend-test.md                 ← paths: "apps/api/**/*.test.*"
│   │   ├── 22-routes.md                       ← paths: "apps/api/src/routes/**"
│   │   ├── 23-middleware.md                   ← paths: "apps/api/src/middleware/**"
│   │   │
│   │   │─────────────────────────────────
│   │   │  30-39: 数据库条件规则
│   │   │─────────────────────────────────
│   │   ├── 30-database.md                     ← paths: "**/prisma/**, **/db/**"
│   │   ├── 31-migrations.md                   ← paths: "**/migrations/**"
│   │   │
│   │   │─────────────────────────────────
│   │   │  40-49: 基础设施条件规则
│   │   │─────────────────────────────────
│   │   ├── 40-infra.md                        ← paths: "Dockerfile*, k8s/**, .github/**"
│   │   ├── 41-ci.md                           ← paths: ".github/workflows/**"
│   │   │
│   │   │─────────────────────────────────
│   │   │  50-59: 安全条件规则
│   │   │─────────────────────────────────
│   │   ├── 50-security.md                     ← paths: "**/auth/**, **/middleware/**"
│   │   │
│   │   │─────────────────────────────────
│   │   │  60-69: 文档条件规则
│   │   │─────────────────────────────────
│   │   └── 60-docs.md                         ← paths: "docs/**, **/*.md"
│   │
│   ├── agents/
│   │   ├── code-reviewer.md
│   │   ├── test-writer.md
│   │   └── migration-runner.md
│   └── commands/
│       ├── deploy.md
│       └── db-migrate.md
│
├── apps/
│   ├── web/                                   ← 前端应用
│   │   ├── CLAUDE.md
│   │   ├── .claude/rules/                     ← 子目录级规则（最高优先级）
│   │   │   ├── components.md
│   │   │   ├── hooks.md
│   │   │   └── pages.md
│   │   └── src/
│   │       ├── components/
│   │       ├── hooks/
│   │       └── app/
│   └── api/                                   ← 后端应用
│       ├── CLAUDE.md
│       ├── .claude/rules/                     ← 子目录级规则
│       │   ├── routes.md
│       │   ├── middleware.md
│       │   └── services.md
│       └── src/
│           ├── routes/
│           ├── middleware/
│           └── services/
│
├── packages/
│   ├── shared/                                ← 共享代码
│   │   └── .claude/rules/types.md
│   ├── ui/                                    ← UI 组件库
│   │   └── .claude/rules/components.md
│   └── core/                                  ← 核心业务逻辑
│       └── .claude/rules/business-logic.md
│
└── services/
    ├── auth/                                  ← 独立认证服务
    │   └── .claude/rules/security.md
    └── payment/                               ← 独立支付服务
        └── .claude/rules/pci-compliance.md
```

## 触发示例 — 多规则联动

```
场景: 用户让 Claude 在 Web 端添加一个订单列表页面（涉及前后端+数据库）

Step 1: 创建 API 路由
  → Write("apps/api/src/routes/orders.ts")
  → 触发:
    ✅ 00-general.md          ← 无条件，始终加载
    ✅ 01-commit.md           ← 无条件
    ✅ 20-backend.md          ← 匹配 "apps/api/**"
    ✅ 22-routes.md           ← 匹配 "apps/api/src/routes/**"（子目录）
  → Claude 按 Fastify + Zod + 统一响应规范编写

Step 2: 修改数据库 Schema
  → Edit("prisma/schema.prisma")
  → 触发:
    ✅ 00-general.md          ← 始终
    ✅ 30-database.md         ← 匹配 "**/prisma/**"
  → Claude 按 Prisma 规范添加 Order 模型

Step 3: 创建前端页面
  → Write("apps/web/src/app/orders/page.tsx")
  → 触发:
    ✅ 00-general.md          ← 始终
    ✅ 10-frontend.md         ← 匹配 "apps/web/**"
    ✅ 14-pages.md            ← 匹配 "apps/web/src/app/**"（子目录）
  → Claude 按 Next.js App Router + Tailwind 编写

Step 4: 编写测试
  → Write("apps/api/src/routes/orders.test.ts")
  → 触发:
    ✅ 00-general.md          ← 始终
    ✅ 20-backend.md          ← 匹配 "apps/api/**"
    ✅ 21-backend-test.md     ← 匹配 "apps/api/**/*.test.*"
  → Claude 按 Vitest 规范编写测试

整个过程：每步只加载相关规则，不浪费上下文窗口
```
