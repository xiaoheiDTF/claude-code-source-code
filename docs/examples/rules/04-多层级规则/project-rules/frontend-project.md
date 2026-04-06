---
paths:
  - "apps/web/**"
---

# 多层级规则 — 项目条件规则

> 放在项目 `.claude/rules/`，但只在操作匹配路径时注入
> 优先级：中（覆盖全局条件规则，被子目录条件规则覆盖）

## 目录结构

```
my-monorepo/
├── .claude/
│   └── rules/
│       ├── coding.md                ← 无条件（始终加载）
│       ├── frontend.md              ← ★ 本规则（条件：apps/web/**）
│       └── backend.md               ← 条件：apps/api/**
├── apps/
│   ├── web/
│   │   ├── CLAUDE.md                ← 子目录指令
│   │   ├── .claude/rules/
│   │   │   └── components.md        ← 子目录条件规则（更高优先级）
│   │   └── src/
│   │       ├── components/
│   │       │   └── Button.tsx       ← 匹配 frontend.md + components.md
│   │       ├── hooks/
│   │       │   └── useAuth.ts       ← 只匹配 frontend.md
│   │       └── utils/
│   │           └── format.ts        ← 只匹配 frontend.md
│   └── api/
│       └── src/                     ← 匹配 backend.md，不匹配 frontend.md
└── packages/
    └── shared/                      ← 不匹配任何条件规则
```

## 三级规则叠加示例（操作 Button.tsx）

```
操作文件: apps/web/src/components/Button.tsx

加载的规则（按优先级从低到高）:
┌──────────────────────────────────────────────────────────────┐
│ 1. 全局规则（~/.claude/rules/coding.md）                     │
│    → "TypeScript strict, 禁止 any"                           │
│                                                              │
│ 2. 项目无条件规则（.claude/rules/coding.md）                  │
│    → "Next.js 14, pnpm, 允许 metadata any"                   │
│    → 覆盖全局同名 coding.md                                  │
│                                                              │
│ 3. 项目条件规则（.claude/rules/frontend.md）   ★ 本规则       │
│    → paths 匹配 "apps/web/**" ✓                             │
│    → "React 18, Tailwind, shadcn/ui"                         │
│                                                              │
│ 4. 子目录条件规则（apps/web/.claude/rules/components.md）     │
│    → paths 匹配 "src/components/**" ✓                       │
│    → "组件文件夹结构, Storybook, CSS Modules"                 │
│    → 覆盖项目级 frontend.md 中的 Tailwind-only 规则          │
└──────────────────────────────────────────────────────────────┘
```

## 触发示例

```
场景 1: 操作 apps/web/src/hooks/useAuth.ts
  → frontend.md 匹配 ✓ → 注入
  → components.md 不匹配 ✗ → 不注入
  → Claude 遵循 React 18 + hooks 规范

场景 2: 操作 apps/web/src/components/Button.tsx
  → frontend.md 匹配 ✓ → 注入
  → components.md 匹配 ✓ → 注入（更高优先级）
  → 两条规则同时生效，components.md 覆盖冲突部分

场景 3: 操作 apps/api/src/routes.ts
  → frontend.md 不匹配 ✗ → 不注入
  → backend.md 匹配 ✓ → 注入
```
