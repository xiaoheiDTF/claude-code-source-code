---
# 花括号展开 {a,b}：一条规则覆盖多个平行目录
# 源码 splitPathInFrontmatter() 会自动展开
# "{apps/web,apps/mobile}/src/**" → ["apps/web/src/**", "apps/mobile/src/**"]
paths: "{apps/web,apps/mobile}/src/**"
---

# 复杂 paths — 花括号展开

> `{a,b}` 语法同时匹配多个路径，等价于写多条 paths 条目

## 目录结构

```
my-monorepo/
├── .claude/
│   └── rules/
│       └── cross-platform.md        ← ★ 本规则（花括号展开）
├── apps/
│   ├── web/                         ← 匹配 "apps/web/src/**" ✓
│   │   └── src/
│   │       ├── components/
│   │       │   └── Button.tsx       ← 触发
│   │       └── App.tsx              ← 触发
│   ├── mobile/                      ← 匹配 "apps/mobile/src/**" ✓
│   │   └── src/
│   │       ├── components/
│   │       │   └── Button.tsx       ← 触发
│   │       └── App.tsx              ← 触发
│   └── api/                         ← 不匹配 ✗
│       └── src/
│           └── routes.ts            ← 不触发
└── packages/
    └── shared/
        └── src/                     ← 不匹配 ✗
```

## 花括号展开过程

```
原始:  "{apps/web,apps/mobile}/src/**"
         ↓ 展开
等价于: ["apps/web/src/**", "apps/mobile/src/**"]
         ↓ 效果
一条规则同时管 web 和 mobile 两个应用
```

## 其他花括号写法

```yaml
# 匹配 .ts 和 .tsx
paths: "src/*.{ts,tsx}"

# 匹配三个 package
paths: "{packages/ui,packages/core,packages/shared}/**"

# 嵌套花括号
paths: "apps/{web,mobile}/src/{components,pages}/**"
```

## 触发示例

```
场景 1: 用户让 Claude 修改 Web 端 Button
  → Edit("apps/web/src/components/Button.tsx")
  → 展开后匹配 "apps/web/src/**" ✓ → 注入

场景 2: 用户让 Claude 修改 Mobile 端 Button
  → Edit("apps/mobile/src/components/Button.tsx")
  → 展开后匹配 "apps/mobile/src/**" ✓ → 注入

场景 3: 用户让 Claude 修改 API
  → Edit("apps/api/src/routes.ts")
  → 不匹配 ✗ → 不注入
```
