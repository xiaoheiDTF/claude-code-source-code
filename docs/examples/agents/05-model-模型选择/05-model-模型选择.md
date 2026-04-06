# Agent 字段: model（模型选择）

> 指南 3.2「模型控制」— 选择 Agent 使用的模型

## 目录结构

```
my-project/
├── .claude/
│   └── agents/
│       ├── fast-linter.md           ← model: haiku
│       ├── code-reviewer.md         ← model: sonnet
│       ├── architect.md             ← model: opus
│       └── general-helper.md        ← model: inherit（继承主模型）
├── src/
└── package.json
```

## Agent 文件内容

```markdown
# 文件: .claude/agents/architect.md

---
description: Senior architect for deep code analysis and complex refactoring plans.
model: opus
---

You are a senior software architect.

Perform deep analysis:
1. Architecture design review
2. Performance bottleneck analysis
3. Security vulnerability deep-dive
4. Refactoring plan with trade-offs

For each finding, provide 2-3 solution options with pros/cons.
```

## 说明

```
model 选项:
  - haiku    → 最快最便宜，适合简单重复任务（lint, 格式化）
  - sonnet   → 平衡性价比，适合大多数任务（代码审查, bug 修复）
  - opus     → 最强推理，适合复杂任务（架构分析, 安全审计）
  - inherit  → 继承主 Agent 的模型（默认值，省略 model 字段时）

省略 model → 默认 inherit（跟主 Agent 用同一个模型）

成本参考（相对值）:
  haiku   → 1x  （最低成本，最高速度）
  sonnet  → 5x  （性价比之选）
  opus    → 25x （最强推理，最贵）
```

---

## 使用示例 1: model: opus — 复杂架构安全分析

```
用户: "深入分析认证模块的安全性"

→ Claude 匹配 architect 的 description "deep analysis"
→ 调用 Agent(agentType: "architect", prompt: "分析认证模块安全性")

architect (opus):
  - 深度读取 auth 模块所有文件
  - 分析 JWT 实现、OAuth 流程、密码处理
  - 每个问题给 2-3 个方案 + 利弊分析

  输出:
    ## JWT Secret 配置
    **问题**: secret 从 env 读取但无启动验证
    **方案A**: 启动时检查 secret 存在且长度 >= 32（简单有效）
    **方案B**: 使用 Vault 动态获取（安全但引入依赖）
    **方案C**: 使用 K8s Secret + 环境变量注入（推荐生产环境）
    **推荐**: 方案C → 方案A 作为 fallback

field 效果:
  opus 模型具有最强推理能力，能发现 JWT secret 未验证、
  refresh token 无轮换策略等深层安全问题，并为每个问题
  提供多个方案及其 trade-off 分析，而非简单的"修复建议"。
```

---

## 使用示例 2: model: haiku — 批量代码格式化

```
用户: "格式化 src/ 下所有 TypeScript 文件"

→ Claude 匹配 fast-linter 的 description
→ 调用 Agent(agentType: "fast-linter", prompt: "格式化所有 TS 文件")

fast-linter (haiku):
  Turn 1: Glob("src/**/*.ts")          ← 找到 47 个文件
  Turn 2: Bash("npx prettier --write src/**/*.ts")
  Turn 3: 报告格式化结果

  输出:
    ✅ 已格式化 47 个文件，其中 12 个有改动
    改动统计:
    - 缩进修正: 8 处
    - 行尾分号: 3 处
    - 尾随空格: 5 处
    耗时: 3.2s

field 效果:
  haiku 模型速度最快、成本最低。格式化是机械性重复任务，
  不需要深度推理，用 haiku 可以在几秒内处理大量文件，
  成本仅为 opus 的 1/25。
```

---

## 使用示例 3: model: sonnet — 日常代码审查

```
用户: "审查 PR #142 的代码变更"

→ Claude 匹配 code-reviewer 的 description
→ 调用 Agent(agentType: "code-reviewer", prompt: "审查 PR #142")

code-reviewer (sonnet):
  Turn 1: Bash("gh pr diff 142")           ← 获取变更
  Turn 2: 分析 diff，读取相关上下文
  Turn 3: 逐文件审查

  输出:
    ## PR #142 Review

    ### src/services/payment.ts
    ⚠️ L42: 金额计算使用浮点数，建议用整数（分）或 decimal.js
    ⚠️ L67: 缺少对 currency 参数的枚举校验
    ✅ L78: 错误处理得当，有重试机制

    ### src/utils/format.ts
    ✅ 新增的 formatCurrency 函数逻辑清晰
    💡 建议: 可考虑 i18n，当前硬编码了人民币符号

    总结: 2 个需要修改，1 个建议，整体质量良好

field 效果:
  sonnet 平衡了推理能力和成本。能识别浮点精度问题、
  参数校验缺失等代码审查中常见的问题，同时不会过度
  分析（不像 opus 那样为每个小问题写长篇分析）。
```

---

## 使用示例 4: model: inherit — 跟随主模型执行

```
场景: 用户当前使用 opus 模型与 Claude 对话

用户: "帮我重构 utils 目录，提取公共函数"

→ Claude 匹配 general-helper 的 description
→ 调用 Agent(agentType: "general-helper", prompt: "重构 utils 目录")

general-helper (inherit → 当前是 opus):
  - 继承用户当前使用的 opus 模型
  - 按 opus 的推理深度执行重构

如果用户切换到 sonnet 后再次调用:
general-helper (inherit → 当前是 sonnet):
  - 继承 sonnet 模型
  - 按 sonnet 的推理深度执行

field 效果:
  inherit 让 Agent 跟随用户的模型选择，不做额外指定。
  好处: 用户切换模型后，所有 inherit 的 Agent 自动适配，
  无需手动修改每个 Agent 文件。
  适用于: 通用型 Agent，不特定依赖某个模型能力。
```

---

## 使用示例 5: model 组合对比 — 同一任务不同模型

```
用户: "检查 src/api/users.ts 的潜在问题"

→ 分别用三个模型执行同一任务，对比输出:

--- haiku 输出（快但浅）---
  ⚠️ L12: 未使用的变量 'temp'
  ⚠️ L34: console.log 应移除
  耗时: 1.5s

--- sonnet 输出（均衡）---
  ⚠️ L12: 未使用的变量 'temp'
  ⚠️ L34: console.log 应移除（生产环境会泄露信息）
  ⚠️ L45: 缺少对 email 格式的服务端校验
  💡 L56: getUserById 可加缓存提升性能
  耗时: 4.2s

--- opus 输出（深但慢）---
  ⚠️ L12: 未使用的变量 'temp'（可能是重构遗留）
  ⚠️ L34: console.log 应移除 — 生产环境可能泄露 user PII
  ⚠️ L45: 缺少 email 格式校验 — 可被用于注入攻击
  ⚠️ L56: getUserById 无缓存 — 在高并发下可能导致 DB 过载
  🔴 L67: deleteUser 的 WHERE 条件使用字符串拼接，
         存在 SQL 注入风险，攻击者可构造 id="1 OR 1=1"
         删除全表数据
  💡 建议重构: 将用户 CRUD 抽象为 Repository 模式，
           统一参数化查询和错误处理
  耗时: 12.8s

field 效果:
  haiku  → 适合快速扫描，只抓表面问题，速度快成本低
  sonnet → 日常首选，兼顾速度和深度，性价比最高
  opus   → 关键代码审查时使用，能发现深层安全和性能问题
```
