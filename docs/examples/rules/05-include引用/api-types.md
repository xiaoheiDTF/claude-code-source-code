# @include 引用 — API 类型文件引用

> 在规则中引用 TypeScript 类型文件，操作 API 路由时自动获得类型上下文

## 目录结构

```
my-monorepo/
├── .claude/
│   ├── rules/
│   │   ├── api-types.md             ← ★ 本规则（@ 引用类型文件和模板）
│   │   └── templates/
│   │       └── route-handler.ts     ← @ 引用的模板文件
│   └── settings.json
├── packages/
│   └── shared/
│       └── src/
│           └── types/
│               ├── api.ts           ← @ 引用的类型文件（ApiResponse<T>）
│               └── pagination.ts    ← @ 引用的分页类型
└── apps/
    └── api/
        └── src/
            └── routes/
                ├── users.ts         ← 操作此文件时触发规则
                └── orders.ts        ← 操作此文件时触发规则
```

## 规则文件内容

```markdown
# 文件: .claude/rules/api-types.md

---
paths: "apps/api/src/routes/**"
---

# API 路由开发规范

## 响应类型定义
@./packages/shared/src/types/api.ts

## 分页类型
@./packages/shared/src/types/pagination.ts

## 路由模板
@./.claude/templates/route-handler.ts

## 规则
- 路由参数和请求体用 Zod schema 验证
- 响应符合 ApiResponse<T> 类型
- 分页接口使用 PaginationParams / PaginatedResponse
- 按照路由模板结构编写
```

## 被引用文件的内容示例

```typescript
// packages/shared/src/types/api.ts（被 @ 引用后展开）
export interface ApiResponse<T> {
  success: boolean;
  data?: T;
  error?: string;
  meta?: { page: number; total: number };
}

// packages/shared/src/types/pagination.ts（被 @ 引用后展开）
export interface PaginationParams {
  page?: number;
  limit?: number;
  cursor?: string;
}
export interface PaginatedResponse<T> extends ApiResponse<T[]> {
  meta: { page: number; limit: number; total: number; hasNext: boolean };
}
```

## 触发示例

```
场景: 用户让 Claude 添加订单列表 API

操作: Write("apps/api/src/routes/orders.ts")
  → 匹配 "apps/api/src/routes/**" ✓ → api-types.md 加载
  → @ 引用展开 → api.ts + pagination.ts + 模板代码全部注入

Claude 看到的上下文:
  ✅ ApiResponse<T> 类型
  ✅ PaginationParams 类型
  ✅ PaginatedResponse<T> 类型
  ✅ 路由模板代码

生成结果:
  import { z } from 'zod';
  import type { PaginatedResponse, PaginationParams } from '@shared/types';
  import type { Order } from '@shared/types/order';

  const listOrdersSchema = z.object({
    page: z.coerce.number().default(1),
    limit: z.coerce.number().default(20),
    status: z.enum(['pending','paid','shipped']).optional(),
  });

  app.get('/orders', async (req): Promise<PaginatedResponse<Order>> => {
    const params = listOrdersSchema.parse(req.query);
    // ... 符合所有类型约束 ✓
  });
```
