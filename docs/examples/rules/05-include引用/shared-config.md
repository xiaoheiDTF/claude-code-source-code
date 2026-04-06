# @include 引用 — 共享配置片段

> 在 CLAUDE.md 或 rules 文件中用 `@` 引用外部文件，避免重复
> 最大嵌套 5 层，支持 60+ 文本格式

## 目录结构

```
my-project/
├── CLAUDE.md                        ← 项目总指令（使用 @ 引用）
├── docs/
│   ├── tech-stack.md                ← 被 CLAUDE.md @ 引用
│   ├── api-guide.md                 ← 被 rules @ 引用
│   └── design-tokens.md             ← 被 rules @ 引用
├── .claude/
│   ├── rules/
│   │   ├── api-rules.md             ← ★ 本规则（内含 @ 引用）
│   │   └── templates/
│   │       └── route-handler.ts     ← 被 api-rules.md @ 引用
│   └── settings.json
├── packages/
│   └── shared/
│       └── src/
│           └── types/
│               └── api.ts           ← 被 api-rules.md @ 引用
└── src/
    └── ...
```

## 使用 @ 的规则文件

```markdown
# 文件: .claude/rules/api-rules.md

---
paths: "apps/api/src/routes/**"
---

# API 路由规范

## 共享类型（引用 TypeScript 文件作为上下文）
@./packages/shared/src/types/api.ts

## 路由模板
@./.claude/templates/route-handler.ts

## API 文档
@./docs/api-guide.md

## 规则
- 参数必须用 Zod 验证
- 响应符合 ApiResponse<T> 类型（见上方引用）
- 按照路由模板（见上方引用）的结构编写
```

## CLAUDE.md 中的 @ 引用

```markdown
# 文件: CLAUDE.md

# 项目概述
电商平台 Monorepo。

## 技术栈
@./docs/tech-stack.md

## API 规范
@./.claude/rules/api-rules.md
```

## 引用展开过程

```
@ 引用展开（当规则被加载时）:

原始内容:
  "共享类型（引用 TypeScript 文件作为上下文）
   @./packages/shared/src/types/api.ts"

展开后:
  "共享类型（引用 TypeScript 文件作为上下文）
   // 以下是 api.ts 的实际内容
   export interface ApiResponse<T> {
     success: boolean;
     data?: T;
     error?: string;
   }
   ..."

→ Claude 能同时看到规则文本和被引用的类型定义
→ 确保生成的代码符合类型约束
```

## 触发示例

```
场景: 用户让 Claude 写一个新的 API 路由

1. Edit("apps/api/src/routes/orders.ts")
   → 路径匹配 "apps/api/src/routes/**" ✓ → api-rules.md 被加载

2. api-rules.md 中的 @ 引用被展开:
   → @./packages/shared/src/types/api.ts  → 注入完整类型定义
   → @./.claude/templates/route-handler.ts → 注入路由模板代码
   → @./docs/api-guide.md                → 注入 API 文档

3. Claude 上下文中包含:
   ✅ 规则文本（Zod 验证 + 统一响应）
   ✅ api.ts 类型定义（ApiResponse<T>）
   ✅ route-handler.ts 模板代码
   ✅ API 文档

4. 生成的代码完全符合项目规范
```
