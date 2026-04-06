---
paths:
  - "**/*.test.*"
  - "**/*.spec.*"
  - "**/test/**"
  - "**/tests/**"
  - "**/__tests__/**"
---

# 条件规则 — 测试规范

> 有 `paths` → 仅当操作测试文件时注入
> `**` 表示匹配任意层级子目录

## 目录结构

```
my-project/
├── .claude/
│   └── rules/
│       ├── testing.md               ← ★ 本规则（条件：**/*.test.* 等）
│       ├── frontend.md              ← 条件：apps/web/**
│       └── backend.md               ← 条件：apps/api/**
├── src/
│   ├── user-service.ts              ← 不匹配（非测试文件）
│   ├── user-service.test.ts         ← 匹配 "**/*.test.*" ✓
│   └── utils/
│       ├── helpers.ts               ← 不匹配
│       └── helpers.spec.ts          ← 匹配 "**/*.spec.*" ✓
├── test/                            ← 匹配 "**/test/**" ✓
│   ├── integration/
│   │   └── api.test.ts              ← 同时匹配 "*.test.*" 和 "test/**"
│   └── e2e/
│       └── checkout.test.ts
├── tests/                           ← 匹配 "**/tests/**" ✓
│   └── fixtures/
│       └── mock-data.ts
├── __tests__/                       ← 匹配 "**/__tests__/**" ✓
│   └── setup.ts
└── apps/
    ├── api/
    │   └── src/
    │       └── routes/
    │           └── users.test.ts    ← 匹配 "**/*.test.*" ✓（深层也能匹配）
    └── web/
        └── src/
            └── Button.test.tsx       ← 匹配 "**/*.test.*" ✓
```

## 规则文件内容

```
框架: Vitest（单元）、Playwright（E2E）、Supertest（API）
命名: 同目录 user-service.test.ts，describe 用模块名，it 描述行为
覆盖: 新代码 ≥ 80%，关键逻辑 ≥ 95%
规则: Mock 外部依赖不 mock 内部模块，测试独立，async/await 禁 done callback
```

## 触发示例

```
场景 1: 用户让 Claude 为 user-service 写测试
  → Write("src/user-service.test.ts")
  → 匹配 "**/*.test.*" ✓ → testing.md 注入
  → Claude 按 Vitest 规范 + describe/it 命名 + 覆盖率要求编写

场景 2: 用户让 Claude 修改集成测试
  → Edit("test/integration/api.test.ts")
  → 匹配 "**/test/**" ✓ → testing.md 注入

场景 3: 用户让 Claude 在 __tests__ 下创建测试
  → Write("__tests__/setup.ts")
  → 匹配 "**/__tests__/**" ✓ → testing.md 注入

场景 4: 用户让 Claude 修改源码（非测试文件）
  → Edit("src/user-service.ts")
  → 不匹配任何 paths ✗ → testing.md 不注入
```
