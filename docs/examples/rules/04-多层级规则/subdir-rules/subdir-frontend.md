---
# paths 相对于 apps/web/（.claude 的父目录）
paths:
  - "src/components/**"
---

# 多层级规则 — 子目录层（最高优先级）

> 放在 `apps/web/.claude/rules/`，仅在该子目录范围内生效
> paths 相对于该 `.claude` 目录的**父目录**（即 `apps/web/`）

## 目录结构

```
my-monorepo/
├── .claude/rules/
│   ├── coding.md                    ← 全局无条件（优先级 1）
│   └── frontend.md                  ← 项目条件: apps/web/**（优先级 2）
│
├── apps/
│   ├── web/                         ← 这是子目录 .claude 的父目录
│   │   ├── .claude/
│   │   │   └── rules/
│   │   │       ├── components.md    ← ★ 本规则（子目录级，优先级 3 最高）
│   │   │       ├── hooks.md         ← 子目录级: src/hooks/**
│   │   │       └── pages.md         ← 子目录级: src/app/**
│   │   └── src/
│   │       ├── components/
│   │       │   ├── Button.tsx       ← 匹配 "src/components/**" ✓
│   │       │   ├── Modal.tsx        ← 匹配 ✓
│   │       │   └── Form/
│   │       │       └── index.tsx    ← 匹配 ✓（深层也匹配）
│   │       ├── hooks/
│   │       │   └── useAuth.ts       ← 不匹配 components.md，匹配 hooks.md
│   │       └── app/
│   │           └── page.tsx         ← 不匹配 components.md，匹配 pages.md
│   │
│   └── api/                         ← apps/api/ 下没有 .claude/rules/
│       └── src/
│
└── packages/
    └── ui/
        └── src/                     ← 不匹配 apps/web/.claude/rules 的 paths
```

## paths 相对路径解释

```
文件位置: apps/web/.claude/rules/components.md
paths: "src/components/**"

解释:
  paths 是相对于 apps/web/（.claude 的父目录）
  所以 "src/components/**" 实际匹配:
    apps/web/src/components/**/*.tsx
    apps/web/src/components/**/*.ts

  而不是项目根目录的 src/components/**
```

## 三级覆盖示例（操作 Button.tsx 时）

```
操作文件: apps/web/src/components/Button.tsx

规则叠加（从低到高）:

优先级 1 — 全局/项目无条件规则:
  "TypeScript strict, 禁止 any, kebab-case 文件名"

优先级 2 — 项目条件规则（frontend.md, paths: "apps/web/**"）:
  "React 18, Tailwind CSS only, 函数组件"
  → 覆盖: 全局的 "kebab-case 文件名" → 前端组件用 PascalCase

优先级 3 — 子目录条件规则（components.md, paths: "src/components/**"）:
  "组件必须有 Storybook, 复杂动画允许 CSS Modules"
  → 覆盖: 项目级 "Tailwind CSS only" → 组件目录允许 CSS Modules

最终效果:
  ✅ TypeScript strict
  ✅ 禁止 any
  ✅ PascalCase 文件名（覆盖全局 kebab-case）
  ✅ 函数组件
  ✅ 必须有 Storybook story
  ✅ 复杂动画可用 CSS Modules（覆盖项目级 Tailwind-only）
```

## 触发示例

```
场景 1: 操作 apps/web/src/components/Button.tsx
  → 全局 coding.md          ← 始终加载
  → 项目 frontend.md         ← 匹配 "apps/web/**" ✓
  → 子目录 components.md     ← 匹配 "src/components/**" ✓
  → 三级规则全部叠加

场景 2: 操作 apps/web/src/hooks/useAuth.ts
  → 全局 coding.md          ← 始终加载
  → 项目 frontend.md         ← 匹配 "apps/web/**" ✓
  → 子目录 components.md     ← 不匹配 ✗（不是 components 目录）
  → 子目录 hooks.md          ← 匹配 "src/hooks/**" ✓
  → 三级规则叠加（hooks.md 代替 components.md）

场景 3: 操作 packages/ui/src/Modal.tsx
  → 全局 coding.md          ← 始终加载
  → 项目 frontend.md         ← 不匹配 ✗
  → 子目录 components.md     ← 不匹配 ✗
  → 只有全局规则生效
```
