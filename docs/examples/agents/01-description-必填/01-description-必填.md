# Agent 字段: description（必填）

> 指南 3.2「必填字段」— description 是唯一必填字段，告诉主 Agent 何时使用

## 目录结构

```
my-project/
├── .claude/
│   └── agents/
│       └── summarizer.md            ← ★ 最小 Agent（只有 description）
├── src/
└── package.json
```

## Agent 文件内容

```markdown
# 文件: .claude/agents/summarizer.md

---
description: Summarize code changes or PR descriptions concisely.
---

Summarize the provided content in 3-5 bullet points.
Focus on: what changed, why it changed, and potential impact.
```

## 说明

```
description 是唯一必填字段:
  ✅ description: "Summarize code changes concisely."
  ❌ 缺少 description → Agent 注册失败

写法要求:
  - 一句话说清楚 Agent 做什么
  - 包含使用场景关键词（方便主 Agent 匹配）
  - 例: "Code review specialist for security and quality"
       ↑ 功能              ↑ 触发关键词
```

## 使用示例 1: 总结代码变更

```
用户说: "总结一下最近改了什么"
→ 主 Agent 匹配 description 中的 "Summarize"
→ 调用 Agent(agentType: "summarizer", prompt: "总结最近的代码变更")

summarizer 执行:
  - 读取 git log
  - 输出 3-5 条要点

输出:
  - 新增 POST /api/users 注册接口
  - 修复登录超时 bug（5s → 30s）
  - 重构 auth middleware 为可复用模块
```

## 使用示例 2: 总结 PR 描述

```
用户说: "帮我看看 PR #42 的内容摘要"
→ 主 Agent 匹配 description 中的 "PR descriptions"
→ 调用 Agent(agentType: "summarizer", prompt: "总结 PR #42 的变更内容")

summarizer 执行:
  - 通过 Bash 执行 gh pr view 42
  - 解析 PR title、body、files changed

输出:
  - PR #42: 重构支付模块为独立微服务
  - 主要变更: 支付逻辑从 monolith 抽离为 payment-service
  - 影响范围: orders 路由需要更新 API 调用地址
  - 风险点: 需要同步部署，否则下单会失败
```

## 使用示例 3: 总结文件差异

```
用户说: "总结一下 src/auth/ 目录这周的改动"
→ 主 Agent 匹配 "Summarize" + "code changes"
→ 调用 Agent(agentType: "summarizer", prompt: "总结 src/auth/ 最近一周的变更")

summarizer 执行:
  - Bash("git log --since='1 week ago' -- src/auth/")
  - 分析每次 commit 的 diff

输出:
  - jwt.ts: Token 过期时间从 1h 改为 15min（安全加固）
  - oauth.ts: 新增 Google OAuth provider 支持
  - middleware.ts: 修复刷新 token 的竞态条件
  - 删除了旧的 session-based auth 代码（约 200 行）
```

## 使用示例 4: 总结错误日志

```
用户说: "把刚才跑测试的报错总结一下"
→ 主 Agent 匹配 "Summarize"
→ 调用 Agent(agentType: "summarizer", prompt: "总结测试失败的错误信息")

summarizer 执行:
  - Bash("npm test 2>&1 | tail -100") 或读取最近的测试日志
  - 提取失败用例和错误堆栈

输出:
  - 总计 12 个测试失败，全部集中在 user.service.test.ts
  - 原因: getByEmail() 返回类型从 User 改为 User | null
  - 修复建议: 在所有调用处加 null 检查
  - 影响评估: 不是随机失败，改一处类型定义即可批量修复
```

## 使用示例 5: description 写法对比 — 好与坏

```
用户说: "我想创建一个自动审查代码的 Agent"

❌ 糟糕的 description（无法被匹配）:
  description: "Helps with stuff."
  → 主 Agent 不知道什么时候该用它 → 永远不会被调用

❌ 模糊的 description（匹配不精准）:
  description: "Code assistant."
  → 太宽泛，用户问任何代码问题都可能误触发

✅ 好的 description（精准匹配）:
  description: "Code review specialist for security, performance, and style."
  → 关键词: code review, security, performance, style
  → 用户说"审查代码"或"看看有没有安全问题"都能精准匹配

✅ 最佳的 description（场景 + 关键词）:
  description: "Summarize code changes or PR descriptions concisely."
  ↑ 功能动词      ↑ 两个使用场景           ↑ 输出特点
  → 覆盖 "总结变更" 和 "总结 PR" 两个场景
  → "concisely" 告诉主 Agent 输出格式偏好
```
