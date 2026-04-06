# 多层级规则 — 项目层（中优先级）

> 放在 `<project>/.claude/rules/`，覆盖全局，被子目录覆盖

## 目录结构

```
~/projects/my-monorepo/              ← 项目根目录
├── CLAUDE.md                        ← 项目总指令
├── .claude/
│   ├── settings.json
│   └── rules/
│       ├── coding.md                ← ★ 本规则（项目级，覆盖全局同名）
│       ├── frontend.md              ← 项目条件规则
│       └── backend.md
├── apps/
│   ├── web/
│   │   ├── CLAUDE.md                ← 子目录指令
│   │   └── .claude/rules/
│   │       └── components.md        ← 子目录级（覆盖项目级）
│   └── api/
│       └── .claude/rules/
│           └── routes.md            ← 子目录级
└── packages/
    └── shared/
```

## 优先级加载顺序（操作 `apps/web/src/Button.tsx` 时）

```
1. ~/.claude/rules/coding.md                    全局（最低）
   → "禁止 any，函数有返回类型，ESLint + Prettier"

2. my-monorepo/.claude/rules/coding.md          项目级（覆盖全局同名）
   → "Next.js 14 App Router, pnpm, Turborepo"
   → "允许 Next.js metadata 导出中使用 any"
   → 全局 coding.md 被同名项目级覆盖，不再生效

3. my-monorepo/.claude/rules/frontend.md        项目条件规则
   → "React 18 + Tailwind + shadcn/ui"

4. apps/web/.claude/rules/components.md          子目录条件规则（最高优先级）
   → "组件必须有 Storybook story"
   → "复杂动画允许 CSS Modules"

最终生效 = 2 + 3 + 4（全局被项目级同名覆盖，不再生效）
```

## 触发示例

```
场景 1: 操作 apps/web/src/components/Button.tsx
  → 全局 coding.md                    ← 加载
  → 项目 coding.md                    ← 加载（覆盖全局同名）
  → 项目 frontend.md                  ← 匹配 "apps/web/**" → 加载
  → 子目录 components.md              ← 匹配 "src/components/**" → 加载
  → 4 条规则全部注入（同名覆盖生效）

场景 2: 操作 packages/shared/src/types.ts
  → 全局 coding.md                    ← 加载
  → 项目 coding.md                    ← 加载（覆盖全局）
  → frontend.md                       ← 不匹配 → 不加载
  → 子目录 components.md              ← 不匹配 → 不加载
  → 只有 2 条规则注入
```
