---
paths:
  - "docker-compose.*"
  - "Dockerfile*"
  - "k8s/**"
  - ".github/workflows/**"
  - "terraform/**"
---

# 条件规则 — 基础设施规范

> 有 `paths` → 仅当操作 Docker/K8s/CI/Terraform 文件时注入
> `docker-compose.*` 匹配 `.yml` `.yaml` 等后缀
> `Dockerfile*` 匹配 `Dockerfile`、`Dockerfile.prod`、`Dockerfile.dev`

## 目录结构

```
my-project/
├── .claude/
│   └── rules/
│       └── infra.md                 ← ★ 本规则
├── Dockerfile                       ← 匹配 "Dockerfile*" ✓
├── Dockerfile.prod                  ← 匹配 "Dockerfile*" ✓
├── Dockerfile.dev                   ← 匹配 "Dockerfile*" ✓
├── docker-compose.yml               ← 匹配 "docker-compose.*" ✓
├── docker-compose.prod.yaml         ← 匹配 "docker-compose.*" ✓
├── k8s/                             ← 匹配 "k8s/**" ✓
│   ├── deployment.yaml              ← 操作此文件 → 触发
│   ├── service.yaml
│   ├── hpa.yaml
│   └── ingress.yaml
├── .github/
│   └── workflows/                   ← 匹配 ".github/workflows/**" ✓
│       ├── ci.yml                   ← 操作此文件 → 触发
│       ├── deploy.yml
│       └── release.yml
├── terraform/                       ← 匹配 "terraform/**" ✓
│   ├── main.tf                      ← 操作此文件 → 触发
│   ├── variables.tf
│   └── modules/
│       └── ec2/
│           └── main.tf
└── src/                             ← 不匹配任何 infra paths ✗
    └── app.ts
```

## 规则文件内容

```
Docker: 多阶段构建，node:20-alpine 基础镜像，非 root 运行，.dockerignore
CI/CD: lint→type-check→test→build，生产需 manual approval，tag 用 git SHA
K8s: 必设 resources requests/limits，liveness+readiness probe，ConfigMap/Secret 分离
```

## 触发示例

```
场景 1: 用户让 Claude 优化 Dockerfile
  → Edit("Dockerfile.prod")
  → 匹配 "Dockerfile*" ✓ → infra.md 注入
  → Claude 加多阶段构建 + USER node + .dockerignore 建议

场景 2: 用户让 Claude 添加 CI pipeline
  → Write(".github/workflows/ci.yml")
  → 匹配 ".github/workflows/**" ✓ → infra.md 注入
  → Claude 按 lint→check→test→build 顺序编排

场景 3: 用户让 Claude 配 K8s 部署
  → Write("k8s/deployment.yaml")
  → 匹配 "k8s/**" ✓ → infra.md 注入
  → Claude 加 resources limits + probes + HPA

场景 4: 用户让 Claude 改业务代码
  → Edit("src/app.ts")
  → 不匹配 ✗ → infra.md 不注入
```
