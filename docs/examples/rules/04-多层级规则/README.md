# 多层级规则示例

> 展示规则在不同层级的放置方式和优先级关系

## 目录结构

```
层级 0（托管，最高）  /etc/claude-code/rules/*.md       企业托管规则
层级 1（最低优先级）  ~/.claude/rules/*.md              全局个人规则
层级 2               <project>/.claude/rules/*.md       项目规则
层级 3（最高非托管）  <subdir>/.claude/rules/*.md       子目录规则
```

```
my-monorepo/
│
│─── 全局规则（~/.claude/rules/）           ← 所有项目生效
│    └── coding-global.md                   ← 优先级 1
│
├── .claude/rules/                          ← 项目级
│   ├── coding-project.md                   ← 优先级 2（覆盖全局同名）
│   └── frontend-project.md                ← 优先级 2（条件规则）
│
├── apps/web/.claude/rules/                 ← 子目录级
│   └── subdir-frontend.md                 ← 优先级 3（最高非托管）
│
└── apps/api/.claude/rules/                 ← 另一个子目录级
    └── ...
```

## 优先级机制

```
操作 apps/web/src/components/Button.tsx 时的规则加载顺序:

  ┌─────────────────────────────────────────────────────┐
  │ 优先级 1 — ~/.claude/rules/coding-global.md         │
  │   "TypeScript strict, 禁止 any, kebab-case 文件名"   │
  │                                                     │
  │ 优先级 2 — .claude/rules/coding-project.md          │
  │   "Next.js 14, 允许 metadata any"                   │
  │   → 覆盖同名全局 coding-global.md                    │
  │                                                     │
  │ 优先级 2 — .claude/rules/frontend-project.md        │
  │   paths: "apps/web/**" → 匹配 ✓                    │
  │   "React 18, Tailwind, 函数组件"                     │
  │   → 覆盖: kebab-case → 前端用 PascalCase            │
  │                                                     │
  │ 优先级 3 — apps/web/.claude/rules/subdir-frontend.md│
  │   paths: "src/components/**" → 匹配 ✓              │
  │   "组件必须有 Storybook, 允许 CSS Modules"           │
  │   → 覆盖: Tailwind-only → 组件允许 CSS Modules      │
  └─────────────────────────────────────────────────────┘
```

## 各层级示例文件

| 文件 | 层级 | 有 paths | 说明 |
|------|------|---------|------|
| [global-rules/coding-global.md](./global-rules/coding-global.md) | 全局 | 无 | 所有项目通用规范 |
| [project-rules/coding-project.md](./project-rules/coding-project.md) | 项目 | 无 | 覆盖全局同名 |
| [project-rules/frontend-project.md](./project-rules/frontend-project.md) | 项目 | 有 | 条件规则：apps/web/** |
| [subdir-rules/subdir-frontend.md](./subdir-rules/subdir-frontend.md) | 子目录 | 有 | 最高优先级：src/components/** |
