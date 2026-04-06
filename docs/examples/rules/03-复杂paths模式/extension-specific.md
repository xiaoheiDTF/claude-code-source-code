---
paths:
  - "src/**/*.d.ts"
  - "src/**/*.interface.ts"
---

# 复杂 paths — 精确扩展名匹配

> `*.{ts,tsx}` 只匹配特定扩展名，不会匹配 `.js` `.jsx` `.d.ts` 等

## 目录结构

```
my-project/
├── .claude/
│   └── rules/
│       └── type-declarations.md     ← ★ 本规则
├── src/
│   ├── types/
│   │   ├── env.d.ts                 ← 匹配 "**/*.d.ts" ✓
│   │   ├── api.d.ts                 ← 匹配 ✓
│   │   ├── global.d.ts              ← 匹配 ✓
│   │   └── utils.interface.ts       ← 匹配 "**/*.interface.ts" ✓
│   ├── services/
│   │   ├── user-service.ts          ← 不匹配（是 .ts 不是 .d.ts）✗
│   │   └── user-service.test.ts     ← 不匹配 ✗
│   ├── components/
│   │   └── Button.tsx               ← 不匹配 ✗
│   └── app.ts                       ← 不匹配 ✗
└── package.json
```

## 匹配细节

```
模式: "src/**/*.d.ts"
  src/          → 必须在 src 目录下
  **/           → 任意层子目录
  *.d.ts        → 文件名任意，但扩展名必须是 .d.ts

匹配列表:
  ✅ src/types/env.d.ts
  ✅ src/types/api.d.ts
  ✅ src/services/declarations.d.ts   （深层也匹配）
  ❌ src/types/env.ts                 （不是 .d.ts）
  ❌ src/types/env.d.ts.map           （多了 .map）
  ❌ types/env.d.ts                   （不在 src 下）
  ❌ test/types/mock.d.ts             （不在 src 下）
```

## 触发示例

```
场景 1: 用户让 Claude 添加环境变量类型
  → Edit("src/types/env.d.ts")
  → 匹配 "src/**/*.d.ts" ✓ → 注入

场景 2: 用户让 Claude 创建模块声明
  → Write("src/types/svg.d.ts")
  → 匹配 ✓ → 注入

场景 3: 用户让 Claude 修改业务代码
  → Edit("src/services/user-service.ts")
  → 不匹配 ✗ → 不注入
```
