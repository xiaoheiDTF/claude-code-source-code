# 企业级场景 — 微服务架构规则

> 每个微服务有独立规则，通过共享 proto 包和跨服务规则保持一致性

## 目录结构

```
my-platform/
├── CLAUDE.md                                  ← 平台总指令
│   （微服务架构概述、服务列表、通信方式）
│
├── .claude/
│   ├── rules/
│   │   ├── 00-general.md                      ← 无条件：平台级编码规范
│   │   ├── 01-grpc.md                         ← 条件: **/*.proto, **/grpc/**
│   │   │   @./packages/proto/README.md        ← @ 引用 proto 规范
│   │   ├── 02-monitoring.md                   ← 条件: **/monitoring/**, **/metrics/**
│   │   └── 03-shared-libs.md                  ← 条件: packages/shared/**
│   │
│   └── settings.json
│
├── packages/
│   ├── proto/                                 ← 共享 Protobuf 定义
│   │   ├── README.md                          ← proto 编写规范
│   │   ├── user/
│   │   │   └── user.proto                     ← @ 被规则引用
│   │   ├── order/
│   │   │   └── order.proto
│   │   └── common/
│   │       └── types.proto
│   │
│   ├── shared/                                ← 共享库
│   │   └── src/
│   │       ├── types/
│   │       ├── errors/
│   │       └── utils/
│   │
│   └── grpc-client/                           ← gRPC 客户端封装
│       └── src/
│
├── services/
│   ├── user-service/                          ← 用户服务（Go）
│   │   ├── CLAUDE.md                          ← 服务说明
│   │   ├── .claude/rules/
│   │   │   └── service-rules.md               ← ★ 用户服务专属规则
│   │   │       paths: "services/user-service/**"
│   │   │       @./packages/proto/user/user.proto
│   │   ├── main.go
│   │   ├── internal/
│   │   │   ├── handler/
│   │   │   ├── repository/
│   │   │   └── service/
│   │   └── go.mod
│   │
│   ├── order-service/                         ← 订单服务（Node.js）
│   │   ├── CLAUDE.md
│   │   ├── .claude/rules/
│   │   │   └── service-rules.md               ← ★ 订单服务专属规则
│   │   │       paths: "services/order-service/**"
│   │   │       @./packages/proto/order/order.proto
│   │   ├── src/
│   │   │   ├── routes/
│   │   │   ├── services/
│   │   │   └── consumers/                     ← 消息队列消费者
│   │   └── package.json
│   │
│   ├── payment-service/                       ← 支付服务（Node.js）
│   │   ├── CLAUDE.md
│   │   ├── .claude/rules/
│   │   │   ├── service-rules.md               ← 支付服务规则
│   │   │   └── pci-compliance.md              ← ★ PCI 合规规则（最高安全级别）
│   │   │       paths: "services/payment-service/**"
│   │   └── src/
│   │
│   └── notification-service/                  ← 通知服务（Python）
│       ├── CLAUDE.md
│       ├── .claude/rules/
│       │   └── service-rules.md               ← 通知服务规则
│       │       paths: "services/notification-service/**"
│       ├── app/
│       │   ├── handlers/
│       │   └── providers/
│       └── requirements.txt
│
├── infrastructure/                            ← 基础设施
│   ├── .claude/rules/
│   │   └── infra.md                           ← 条件: k8s/**, terraform/**
│   ├── k8s/
│   └── terraform/
│
└── scripts/
    └── ...
```

## 触发示例 — 跨服务开发

```
场景 1: 用户让 Claude 修改用户服务的 API

操作: Edit("services/user-service/internal/handler/user.go")
  → 触发:
    ✅ 00-general.md                 ← 平台级规范（始终加载）
    ✅ 01-grpc.md                    ← 不匹配（不是 proto/grpc 文件）
    ✅ services/user-service/.claude/rules/service-rules.md  ← 匹配
       → @./packages/proto/user/user.proto 展开
       → Claude 看到用户 proto 定义
  → Claude 按 Go + Gin + gRPC 规范编写

场景 2: 用户让 Claude 添加新的 proto 定义

操作: Write("packages/proto/order/order.proto")
  → 触发:
    ✅ 00-general.md                 ← 始终
    ✅ 01-grpc.md                    ← 匹配 "**/*.proto" ✓
       → @./packages/proto/README.md 展开 → proto 编写规范
  → Claude 按 proto3 语法 + 命名规范编写

场景 3: 用户让 Claude 修改支付服务

操作: Edit("services/payment-service/src/routes/checkout.ts")
  → 触发:
    ✅ 00-general.md                 ← 始终
    ✅ service-rules.md              ← 匹配
    ✅ pci-compliance.md             ← 匹配（PCI 最高安全级别）
  → Claude 按 PCI-DSS 规范：禁止存储卡号、CVV，使用 tokenization

场景 4: 用户让 Claude 同时修改两个服务的 proto

操作:
  Edit("packages/proto/user/user.proto")      → 触发 01-grpc.md
  Edit("services/user-service/internal/handler/user.go") → 触发 service-rules.md
  → Claude 同时看到 proto 规范和服务实现规范
```
