# 企业级场景 — 合规/安全规则

> 面向 GDPR、PCI-DSS、SOC2 等合规要求的规则体系
> 多层级联动：托管策略（最高）→ 安全规则 → 业务规则

## 目录结构

```
/etc/claude-code/                               ← ★ 托管策略目录（IT 安全团队管理）
├── rules/
│   ├── security-policy.md                      ← 强制安全策略（最高优先级，不可覆盖）
│   └── enterprise-coding.md                    ← 强制编码标准
└── settings.json                               ← 强制配置（禁止危险命令）

~/.claude/                                      ← 用户全局
├── rules/
│   └── no-any.md                               ← 个人偏好

my-project/                                     ← 项目级
├── CLAUDE.md
├── .claude/
│   ├── rules/
│   │   │
│   │   │─────────────────────────────────
│   │   │  安全规则（paths 匹配敏感区域）
│   │   │─────────────────────────────────
│   │   ├── security/
│   │   │   ├── auth.md                         ← paths: "**/auth/**, **/middleware/auth*"
│   │   │   ├── crypto.md                       ← paths: "**/crypto/**, **/encryption/**"
│   │   │   ├── gdpr.md                         ← paths: "**/users/**, **/profile/**, **/personal-data/**"
│   │   │   ├── pci.md                          ← paths: "**/payment/**, **/billing/**, **/checkout/**"
│   │   │   ├── audit.md                        ← paths: "**/audit/**, **/logging/**"
│   │   │   └── secrets.md                      ← paths: "**/*.env*, **/*.key, **/*.pem, **/secrets/**"
│   │   │
│   │   │─────────────────────────────────
│   │   │  业务规则
│   │   │─────────────────────────────────
│   │   ├── coding.md                           ← 无条件：项目编码规范
│   │   ├── frontend.md                         ← 条件: apps/web/**
│   │   └── backend.md                          ← 条件: apps/api/**
│   │
│   └── settings.json                           ← hooks: 敏感文件审计
│
├── src/
│   ├── auth/                                   ← 匹配 auth.md + security-policy.md
│   │   ├── jwt.ts                              ← 还匹配 crypto.md
│   │   ├── oauth.ts
│   │   └── middleware/
│   │       └── require-auth.ts
│   ├── users/                                  ← 匹配 gdpr.md
│   │   ├── user-service.ts
│   │   ├── profile-service.ts
│   │   └── data-export.ts                      ← GDPR 数据导出
│   ├── payment/                                ← 匹配 pci.md
│   │   ├── stripe-service.ts
│   │   ├── checkout.ts
│   │   └── billing.ts
│   ├── audit/                                  ← 匹配 audit.md
│   │   ├── audit-logger.ts
│   │   └── compliance-report.ts
│   └── secrets/                                ← 匹配 secrets.md
│       └── vault-client.ts
│
├── .env.example                                ← 匹配 secrets.md
└── infra/
    └── ssl/                                    ← 匹配 secrets.md (*.pem)
        └── cert.pem
```

## 优先级叠加（操作支付相关文件时）

```
操作文件: src/payment/checkout.ts

优先级从低到高:
┌────────────────────────────────────────────────────────────────────┐
│ 层级 0 — 托管策略（/etc/claude-code/rules/security-policy.md）    │
│   "禁止 eval, md5/sha1, http, 硬编码凭证"                         │
│   "依赖必须通过 Snyk 审查"                                        │
│   "不可被任何下级规则覆盖"                                          │
│                                                                    │
│ 层级 1 — 用户全局（~/.claude/rules/no-any.md）                     │
│   "禁止 any"                                                       │
│                                                                    │
│ 层级 2 — 项目无条件（.claude/rules/coding.md）                     │
│   "TypeScript strict, Zod 验证"                                    │
│                                                                    │
│ 层级 3 — 项目条件: 后端（.claude/rules/backend.md）                │
│   "Fastify + Prisma, 统一响应格式"                                  │
│                                                                    │
│ 层级 4 — 项目条件: PCI 合规（.claude/rules/security/pci.md）       │
│   "禁止存储卡号/CVV, tokenization, TLS 1.3"                       │
│   "金额用整数(分), 审计日志保留 ≥ 1年"                              │
│                                                                    │
│ 层级 5 — 项目条件: 审计（.claude/rules/security/audit.md）          │
│   "所有写操作记录审计日志"                                          │
└────────────────────────────────────────────────────────────────────┘
```

## 触发示例

```
场景 1: 用户让 Claude 实现结账功能

操作: Write("src/payment/checkout.ts")
  → 匹配:
    ✅ security-policy.md（托管，强制）
    ✅ no-any.md（全局）
    ✅ coding.md（项目无条件）
    ✅ backend.md（匹配 apps/api/** 如果在 api 下）
    ✅ pci.md（匹配 "**/payment/**"）
    ✅ audit.md（匹配 "**/audit/**" — 如果代码引用 audit）
  → Claude 必须遵守:
    ✅ 不存储卡号/CVV（PCI）
    ✅ 使用 tokenization（PCI）
    ✅ 金额用整数/分（PCI）
    ✅ 审计日志（Audit）
    ✅ TypeScript strict（Coding）
    ✅ 不用 any（全局）

场景 2: 用户让 Claude 写 GDPR 数据导出

操作: Write("src/users/data-export.ts")
  → 匹配:
    ✅ security-policy.md（托管）
    ✅ gdpr.md（匹配 "**/users/**"）
  → Claude 必须遵守:
    ✅ 支持访问权（用户查看所有数据）
    ✅ 支持删除权（被遗忘权）
    ✅ 支持携带权（JSON/CSV 导出）
    ✅ 导出前验证用户身份
    ✅ 记录数据访问审计日志

场景 3: 用户让 Claude 修改 JWT 认证

操作: Edit("src/auth/jwt.ts")
  → 匹配:
    ✅ security-policy.md（托管）
    ✅ auth.md（匹配 "**/auth/**"）
    ✅ crypto.md（匹配 "**/crypto/**" — 间接）
  → Claude 必须遵守:
    ✅ access_token 过期 15min
    ✅ 密钥从 Vault 读取
    ✅ 使用 RS256 或 ES256（非对称）
    ❌ 不使用 HS256（托管策略禁止弱算法）
```
