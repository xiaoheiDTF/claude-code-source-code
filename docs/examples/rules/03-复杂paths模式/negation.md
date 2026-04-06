---
# 取反模式：以 ! 开头排除特定路径
# gitignore 语法，先匹配正向规则，再排除取反的
paths:
  - "src/**"
  - "!src/**/*.test.*"
  - "!src/**/*.spec.*"
  - "!src/**/__mocks__/**"
---

# 复杂 paths — 排除/取反模式

> `!` 开头的路径表示排除，先匹配正向规则，再从结果中排除取反的路径

## 目录结构

```
my-project/
├── .claude/
│   └── rules/
│       ├── source-code.md           ← ★ 本规则（匹配 src 但排除测试）
│       └── testing.md               ← 条件：**/*.test.*（专门管测试）
├── src/
│   ├── app.ts                       ← 匹配 "src/**" ✓，不匹配 "!*.test.*" ✓ → 注入
│   ├── components/
│   │   ├── Button.tsx               ← 匹配 ✓，不被排除 → 注入
│   │   └── Button.test.tsx          ← 匹配 "src/**" ✓，但被 "!src/**/*.test.*" 排除 ✗ → 不注入
│   ├── services/
│   │   ├── user-service.ts          ← 注入 ✓
│   │   ├── user-service.test.ts     ← 被排除 ✗
│   │   └── user-service.spec.ts     ← 被排除 ✗
│   ├── utils/
│   │   ├── helpers.ts               ← 注入 ✓
│   │   └── __mocks__/
│   │       └── helpers.ts           ← 被排除 ✗（匹配 "!src/**/__mocks__/**"）
│   └── types/
│       └── index.ts                 ← 注入 ✓
└── tests/                           ← 不匹配 "src/**" ✗
    └── e2e/
        └── app.test.ts
```

## 取反逻辑

```
步骤 1: "src/**"           → 匹配 src 下所有文件
步骤 2: "!src/**/*.test.*" → 从结果中排除 .test. 文件
步骤 3: "!src/**/*.spec.*" → 从结果中排除 .spec. 文件
步骤 4: "!src/**/__mocks__/**" → 排除 __mocks__ 目录

最终结果:
  ✅ src/app.ts
  ✅ src/components/Button.tsx
  ✅ src/services/user-service.ts
  ✅ src/utils/helpers.ts
  ❌ src/components/Button.test.tsx     ← 被排除
  ❌ src/services/user-service.test.ts  ← 被排除
  ❌ src/services/user-service.spec.ts  ← 被排除
  ❌ src/utils/__mocks__/helpers.ts     ← 被排除
```

## 触发示例

```
场景 1: 用户让 Claude 修改业务代码
  → Edit("src/services/user-service.ts")
  → 匹配 "src/**" ✓，不被排除 → 注入 source-code.md

场景 2: 用户让 Claude 修改测试文件
  → Edit("src/services/user-service.test.ts")
  → 匹配 "src/**" 但被 "!**/*.test.*" 排除
  → source-code.md 不注入
  → 但 testing.md（paths: **/*.test.*）会注入

场景 3: 用户让 Claude 修改 mock 文件
  → Edit("src/utils/__mocks__/helpers.ts")
  → 被 "!src/**/__mocks__/**" 排除 → 不注入

这样实现了：源码规则和测试规则互不干扰
```
