---
paths:
  - "apps/web/**"
  - "packages/ui/**"
---

# 条件规则 — 前端开发规范

> 有 `paths` → 仅当 Claude 操作匹配路径的文件时才注入

## 目录结构

```
my-monorepo/
├── CLAUDE.md
├── .claude/
│   ├── rules/
│   │   ├── frontend.md              ← ★ 本规则（条件：apps/web/** + packages/ui/**）
│   │   ├── backend.md               ← 条件：apps/api/**
│   │   └── testing.md               ← 条件：**/*.test.*
│   └── settings.json
├── apps/
│   ├── web/                         ← paths 匹配这个目录
│   │   ├── src/
│   │   │   ├── components/
│   │   │   │   └── Button.tsx       ← 操作此文件 → 触发 frontend.md
│   │   │   └── hooks/
│   │   │       └── useAuth.ts       ← 操作此文件 → 触发 frontend.md
│   │   └── package.json
│   └── api/                         ← 不匹配，不触发
│       └── src/
├── packages/
│   ├── ui/                          ← paths 匹配这个目录
│   │   └── src/
│   │       └── Modal.tsx            ← 操作此文件 → 触发 frontend.md
│   └── core/
│       └── src/                     ← 不匹配，不触发
└── package.json
```

## 规则文件内容

```
技术栈: React 18 + TypeScript + Tailwind CSS + shadcn/ui + Zustand
组件: 函数组件 + Hooks，禁止 class 组件
Props: 使用 interface 定义，以 Props 后缀命名
文件名: PascalCase（如 UserProfile.tsx）
禁止: 直接操作 DOM、useEffect 条件渲染、内联样式
```

## 触发示例

```
场景 1: 用户让 Claude 修改 Button.tsx
  → Read("apps/web/src/components/Button.tsx")
  → 路径匹配 "apps/web/**" ✓
  → frontend.md 被注入到 Claude 上下文中
  → Claude 按照 React + Tailwind 规范修改

场景 2: 用户让 Claude 修改 Modal.tsx
  → Read("packages/ui/src/Modal.tsx")
  → 路径匹配 "packages/ui/**" ✓
  → frontend.md 被注入

场景 3: 用户让 Claude 修改 user-service.ts
  → Read("apps/api/src/user-service.ts")
  → 路径不匹配任何 paths ✗
  → frontend.md 不被注入（backend.md 可能被注入）

场景 4: 用户让 Claude 同时修改前后端文件
  → Edit("apps/web/src/App.tsx")     → 触发 frontend.md
  → Edit("apps/api/src/routes.ts")   → 触发 backend.md
  → 两条规则同时注入上下文
```
