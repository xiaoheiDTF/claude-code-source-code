---
paths:
  - "**/grpc/**"
  - "**/proto/**"
  - "**/*.proto"
  - "**/generated/**"
---

# 复杂 paths — 深层子目录匹配

> `**/` 在开头表示从项目根目录开始匹配任意层级子目录
> 无论文件嵌套多深都能匹配到

## 目录结构

```
my-project/
├── .claude/
│   └── rules/
│       └── grpc-proto.md            ← ★ 本规则
├── proto/                           ← 匹配 "**/proto/**" ✓
│   ├── user.proto                   ← 同时匹配 "**/*.proto" ✓
│   └── order.proto                  ← 同时匹配 "**/*.proto" ✓
├── grpc/                            ← 匹配 "**/grpc/**" ✓
│   ├── server.ts
│   └── client.ts
├── generated/                       ← 匹配 "**/generated/**" ✓
│   ├── user_pb.ts
│   └── order_pb.ts
├── services/
│   ├── user-service/
│   │   ├── proto/                   ← 匹配 "**/proto/**" ✓（任意深度）
│   │   │   └── user.proto           ← 匹配 ✓
│   │   ├── grpc/                    ← 匹配 "**/grpc/**" ✓（任意深度）
│   │   │   └── handler.ts
│   │   └── generated/               ← 匹配 "**/generated/**" ✓
│   │       └── user_pb.ts
│   └── order-service/
│       ├── proto/
│       │   └── order.proto          ← 匹配 ✓
│       └── grpc/
│           └── handler.ts           ← 匹配 ✓
└── src/                             ← 不匹配 ✗
    └── app.ts
```

## `**/` 匹配深度说明

```
模式: "**/proto/**"

匹配:
  ✅ proto/user.proto                 （1 层）
  ✅ services/user/proto/user.proto   （3 层）
  ✅ a/b/c/d/proto/file.proto         （5 层）

不匹配:
  ❌ src/proto.ts                     （不是目录）
  ❅ protocols/config.yaml            （不是 proto 目录）
```

## 触发示例

```
场景 1: 用户让 Claude 修改顶层 proto 文件
  → Edit("proto/user.proto")
  → 匹配 "**/proto/**" + "**/*.proto" ✓ → 注入

场景 2: 用户让 Claude 修改服务内的 proto
  → Edit("services/user-service/proto/user.proto")
  → 匹配 "**/proto/**" ✓（深度不影响匹配）→ 注入

场景 3: 用户让 Claude 重新生成 gRPC 代码
  → Write("generated/user_pb.ts")
  → 匹配 "**/generated/**" ✓ → 注入
  → Claude 看到 "不要手动修改生成代码" 的规则
```
