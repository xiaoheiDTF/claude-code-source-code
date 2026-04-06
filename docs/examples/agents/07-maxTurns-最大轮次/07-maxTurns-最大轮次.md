# Agent 字段: maxTurns（最大轮次）

> 指南 3.2「权限控制」— 限制 Agent 的对话轮次，防止成本失控

## 目录结构

```
my-project/
├── .claude/
│   └── agents/
│       ├── quick-fixer.md           ← maxTurns: 8（快速修复）
│       ├── feature-builder.md       ← maxTurns: 20（功能开发）
│       └── deep-analyzer.md         ← maxTurns: 50（深度分析）
└── src/
```

## Agent 文件内容

```markdown
# 文件: .claude/agents/quick-fixer.md

---
description: Quick bug fixer for simple, well-defined issues. Fast and focused.
model: sonnet
effort: low
maxTurns: 8                         ← 最多 8 轮，快速完成
tools:
  - Read
  - Edit
  - Grep
  - Glob
  - Bash
---

Fix the specified bug quickly and efficiently.

## Rules
- Locate the bug directly (don't explore unnecessarily)
- Make minimal changes (only fix the bug)
- Run tests once to verify
- Don't refactor surrounding code
```

## 说明

```
maxTurns 限制 Agent 的对话轮次:

  maxTurns: 5    → 极快：lint 检查、格式化
  maxTurns: 8    → 快速：简单 bug 修复
  maxTurns: 15   → 中等：单功能实现
  maxTurns: 20   → 常规：多文件改动
  maxTurns: 30   → 复杂：跨模块重构
  maxTurns: 50   → 深度：完整架构分析
  省略            → 无限制（直到 Agent 认为完成）

达到限制时:
  → Agent 自动停止
  → 返回当前进度摘要
  → 主 Agent 决定是否继续
```

---

## 使用示例 1: maxTurns: 8 — 简单 bug 快速修复

```
用户: "修复登录接口返回 500 的 bug"

→ Agent(quick-fixer, maxTurns: 8)

Turn 1: Grep("login|auth")             ← 定位相关代码
Turn 2: Read("src/routes/auth.ts")     ← 读取文件
Turn 3: 发现缺少 try-catch
Turn 4: Edit("src/routes/auth.ts")     ← 修复
Turn 5: Bash("npm test")               ← 验证
Turn 6: 测试通过，输出结果

  ✅ 6 轮完成（在 8 轮限制内）
  ✅ effort: low → 不做深度思考，快速定位修复

field 效果:
  maxTurns: 8 确保简单 bug 在有限轮次内完成。
  如果 bug 很简单，6 轮就完成，剩余 2 轮不会浪费。
  如果超过 8 轮，Agent 自动停止并返回进度，
  避免在简单任务上花费过多时间和成本。
```

---

## 使用示例 2: maxTurns: 5 — 代码风格批量修复

```
用户: "把所有文件中的 var 替换成 const/let"

→ Agent(var-replacer, maxTurns: 5)

Turn 1: Grep("var ")                    ← 搜索所有 var 声明
Turn 2: 分析结果，确定替换策略
         发现 23 个文件有 var 声明
Turn 3: Bash("npx eslint --fix --rule 'no-var: error' src/")
         ← 用 eslint 自动修复大部分
Turn 4: Grep("var ")                    ← 检查剩余手动修复
Turn 5: 报告结果

  输出:
    ✅ 自动修复 18 个文件（89 处 var → const/let）
    ⚠️ 2 个文件需手动处理（var 在复杂 for 循环中）
    总计: 89/91 完成，剩余 2 处需人工确认

field 效果:
  maxTurns: 5 适合确定性任务：工具执行 + 验证。
  因为不需要"思考"方案，只需执行固定步骤：
  搜索 → 修复 → 验证 → 报告。
  即使只给 5 轮，也能高效完成。
```

---

## 使用示例 3: maxTurns: 20 — 新增一个 API 端点

```
用户: "添加一个 DELETE /api/users/:id 端点"

→ Agent(feature-builder, maxTurns: 20)

Turn  1: Read("src/routes/users.ts")        ← 了解现有路由结构
Turn  2: Read("src/models/User.ts")         ← 了解数据模型
Turn  3: Read("src/middleware/auth.ts")      ← 了解鉴权中间件
Turn  4: Edit("src/routes/users.ts")         ← 添加 delete 路由
Turn  5: Edit("src/services/userService.ts") ← 添加 delete 方法
Turn  6: Edit("tests/routes/users.test.ts")  ← 添加测试用例
Turn  7: Bash("npm test")                    ← 运行测试
Turn  8: 测试失败 — 缺少权限检查
Turn  9: Edit("src/routes/users.ts")         ← 添加 admin 权限校验
Turn 10: Bash("npm test")                    ← 再次测试
Turn 11: 测试通过
Turn 12: 报告完成

  ✅ 12 轮完成（在 20 轮限制内）
  包含了读取上下文、编写代码、测试、修复的完整循环

field 效果:
  maxTurns: 20 为功能开发提供足够空间：
  - 前几轮读取上下文（了解现有模式）
  - 中间几轮编写代码（路由 + 服务 + 测试）
  - 后几轮测试和修复（可能需要 2-3 轮迭代）
  预留的轮次足够处理"测试失败→修复→重测"的循环。
```

---

## 使用示例 4: maxTurns: 50 — 跨模块架构重构

```
用户: "将单体 src/services/ 拆分为微服务架构"

→ Agent(deep-analyzer, maxTurns: 50)

Turn  1-5:   分析现有代码结构，梳理模块依赖图
Turn  6-10:  识别服务边界，确定拆分策略
Turn 11-15:  设计服务间通信方案（REST/gRPC/消息队列）
Turn 16-20:  创建 user-service 目录和基础结构
Turn 21-25:  迁移用户相关代码到 user-service
Turn 26-30:  创建 order-service，迁移订单代码
Turn 31-35:  实现服务间通信层
Turn 36-40:  配置 Docker Compose 多服务编排
Turn 41-45:  运行集成测试，修复发现的问题
Turn 46-50:  生成迁移文档和下一步计划

  输出（第 50 轮自动停止）:
    ## 重构进度报告

    ✅ 已完成:
    - user-service: 用户 CRUD + 认证（独立运行）
    - order-service: 订单处理（独立运行）
    - 通信层: REST API + 健康检查
    - Docker Compose 配置

    ⏳ 未完成:
    - payment-service（待拆分）
    - 服务发现（未配置）
    - API Gateway（未配置）

    📋 下一步:
    1. 继续拆分 payment-service
    2. 引入服务发现（Consul/Eureka）
    3. 搭建 API Gateway

field 效果:
  maxTurns: 50 允许大型任务的深度执行。
  即使 50 轮用完未完成，也会输出清晰的进度和下一步计划，
  不会突然中断。主 Agent 可据此决定是否继续。
  50 轮约 50 次 API 调用，成本可控（相比无限制）。
```

---

## 使用示例 5: maxTurns 不足 — 任务被截断

```
用户: "重写整个测试套件，从 Jest 迁移到 Vitest"

→ Agent(test-migrator, maxTurns: 15)

Turn 1-3:   分析测试文件结构，共 45 个测试文件
Turn 4-5:   配置 Vitest，修改 package.json
Turn 6-8:   迁移前 10 个文件（jest.fn → vi.fn 等）
Turn 9-11:  迁移第 11-20 个文件
Turn 12-13: 迁移第 21-25 个文件
Turn 14:    Bash("npx vitest run") ← 部分测试通过
Turn 15:    输出截断报告 ← 达到限制，自动停止

  ⚠️ maxTurns 不足，任务被截断:

  输出:
    ## 测试迁移进度

    迁移进度: 25/45 文件（55%）
    ✅ 修改完成: 25 个文件
    ⏳ 待迁移: 20 个文件

    已知问题:
    - jest.mock → vi.mock 的 hoisting 差异（3 处）
    - jsdom 环境配置差异（2 处）

    要继续迁移，请再次调用并指定剩余文件:
    "继续迁移测试，从第 26 个文件开始"

field 效果:
  maxTurns 不足时，Agent 不会崩溃或丢失进度。
  它会在最后一轮输出清晰的进度报告，包含：
  - 完成了什么
  - 还剩什么
  - 如何继续
  这让用户可以分批完成任务，每次都有明确的起点。
  建议: 不确定任务复杂度时，先设较小的 maxTurns 试探，
  根据截断报告决定后续策略。
```
