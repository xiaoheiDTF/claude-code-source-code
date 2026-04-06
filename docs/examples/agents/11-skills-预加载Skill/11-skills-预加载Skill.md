# Agent 字段: skills（预加载 Skill）

> 指南 3.2「预加载 Skill」— Agent 启动时自动加载指定 Skill

## 目录结构

```
my-project/
├── .claude/
│   ├── agents/
│   │   └── feature-dev.md           ← ★ 带 skills 的 Agent
│   ├── commands/
│   │   └── commit.md                ← /commit Skill
│   ├── skills/
│   │   └── deploy/
│   │       └── SKILL.md             ← /deploy Skill
│   └── rules/
├── src/
└── package.json
```

## Agent 文件内容

```markdown
# 文件: .claude/agents/feature-dev.md

---
description: Feature implementer that writes code, tests, and commits.
tools:
  - Read
  - Write
  - Edit
  - Bash
  - Grep
  - Glob
skills:                              ← 启动时预加载的 Skill
  - "commit"                         ← 加载 /commit Skill
  - "deploy"                         ← 加载 /deploy Skill
model: sonnet
permissionMode: acceptEdits
---

You implement features end-to-end.

## Workflow
1. Read specs and understand requirements
2. Implement the feature
3. Write tests
4. Use /commit to commit with proper message
5. Optionally use /deploy to deploy
```

## 说明

```
skills 字段效果:
  skills: ["commit", "deploy"]
  → Agent 启动时自动加载这些 Skill
  → Agent 可以使用 Skill 工具调用它们
  → 等价于用户在对话中输入 /commit 或 /deploy

与主 Agent 的 Skill 共享:
  主 Agent 可用的 Skill → Agent 也能用（如果不在 disallowedTools 中）
  skills 字段 → 额外确保这些 Skill 在 Agent 启动时就加载

省略 skills → Agent 可以用 Skill 工具，但不会自动预加载
```

## 使用示例

### 示例 1：实现功能后自动提交并部署

```
用户: "实现用户个人资料编辑功能"

→ Agent(feature-dev) — 预加载了 commit 和 deploy

Agent 执行:
  1. Read("docs/specs/profile-edit.md")         ← 了解需求
  2. Write("src/routes/profile.ts")             ← 实现路由
  3. Write("src/routes/profile.test.ts")        ← 写测试
  4. Bash("npm test -- --grep profile")         ← 运行测试
  5. Skill("commit")                            ← ✅ 用预加载的 /commit 提交
     → 自动生成: "feat(profile): add profile editing API"
  6. Skill("deploy", "staging")                 ← ✅ 用预加载的 /deploy 部署到 staging

skills 效果:
  → commit Skill 在 Agent 启动时已加载，无需用户手动输入 /commit
  → deploy Skill 同样预加载，Agent 可以端到端完成「开发→提交→部署」
  → 用户只需一句话，Agent 自动完成整个工作流
```

### 示例 2：Bug 修复流程中自动使用 commit Skill

```
用户: "修复 #1234 bug：用户登录后偶尔看到空白页面"

→ Agent(feature-dev) — 预加载了 commit 和 deploy

Agent 执行:
  1. Bash("gh issue view 1234")                 ← 读取 issue 详情
  2. Grep("useAuth")                            ← 定位相关代码
  3. Read("src/hooks/useAuth.ts")               ← 阅读认证 hook
  4. Edit("src/hooks/useAuth.ts")               ← 修复竞态条件
  5. Write("src/hooks/useAuth.test.ts")         ← 补充回归测试
  6. Bash("npm test")                           ← 验证修复
  7. Skill("commit")                            ← ✅ 自动提交
     → "fix(auth): resolve race condition causing blank page after login"
     → 自动关联 #1234

skills 效果:
  → 无需用户提醒「记得提交」，commit Skill 已预加载
  → 提交消息自动包含 fix 前缀和 issue 编号
  → Agent 知道 commit Skill 可用，所以会在适当时机调用
```

### 示例 3：多 Skill 协作完成重构任务

```
用户: "把用户模块从 REST 迁移到 GraphQL，完成后提交并部署到测试环境"

→ Agent(feature-dev) — 预加载了 commit 和 deploy

Agent 执行:
  1. Grep("router.get.*users")                  ← 找到所有 REST 端点
  2. Read("src/routes/users.ts")                ← 读取现有实现
  3. Write("src/graphql/resolvers/user.ts")     ← 写 GraphQL resolver
  4. Write("src/graphql/schema/user.graphql")   ← 写 GraphQL schema
  5. Edit("src/index.ts")                       ← 注册 GraphQL 路由
  6. Write("src/graphql/resolvers/user.test.ts")← 测试
  7. Bash("npm test")                           ← 跑测试
  8. Skill("commit")                            ← ✅ 提交迁移代码
  9. Skill("deploy", "testing")                 ← ✅ 部署到测试环境

skills 效果:
  → 两个 Skill 串联使用：先 commit 再 deploy
  → Agent 知道 deploy Skill 可用，所以主动询问部署目标
  → commit → deploy 形成完整的交付流水线
  → 用户不需要分两步操作
```

### 示例 4：只预加载 commit Skill 做快速修补

```
用户: "API 返回的错误码不一致，帮我统一修复"

→ Agent(feature-dev) — skills: ["commit"]

Agent 执行:
  1. Grep("res.status(4")                       ← 找所有错误响应
  2. Grep("res.status(5")                       ← 找 500 系列
  3. Read("src/utils/errors.ts")                ← 读取错误处理工具
  4. Edit("src/routes/orders.ts")               ← 统一错误码
  5. Edit("src/routes/payments.ts")             ← 统一错误码
  6. Edit("src/routes/users.ts")                ← 统一错误码
  7. Bash("npm test")                           ← 验证
  8. Skill("commit")                            ← ✅ 自动提交
     → "refactor: standardize API error response codes across all routes"

skills 效果:
  → 只预加载 commit，没有 deploy（这类修改不需要部署）
  → Agent 知道只有 commit 可用，不会尝试调用不存在的 Skill
  → 修改涉及多文件，Skill 自动将所有变更一起提交
```

### 示例 5：Skill 预加载失败时的降级行为

```
用户: "添加日志中间件，然后提交部署"

→ Agent(feature-dev) — skills: ["commit", "deploy"]

Agent 执行:
  1. Write("src/middleware/logger.ts")           ← 实现日志中间件
  2. Edit("src/index.ts")                       ← 注册中间件
  3. Write("src/middleware/logger.test.ts")      ← 写测试
  4. Bash("npm test")                           ← 测试通过

  5. Skill("commit")                            ← ✅ commit Skill 正常
     → "feat: add structured logging middleware"

  6. Skill("deploy", "production")              ← ❌ deploy Skill 配置有误
     → Agent 报告: "deploy Skill 加载失败，请检查 deploy/SKILL.md 配置"
     → 代码已提交到 git，只是部署步骤需要手动处理

skills 效果:
  → 预加载在 Agent 启动时进行，如果某个 Skill 配置有误会立即报告
  → 其他正常加载的 Skill 不受影响
  → Agent 能优雅降级，而不是整个流程失败
  → 用户收到明确提示，知道哪一步需要手动处理
```
