# Hook 类型: Agent Hook（启动子 Agent）

> 指南 4.3「Hook 类型」— Agent Hook 在事件触发时自动启动一个子 Agent 执行任务

## 目录结构

```
my-project/
├── .claude/
│   ├── settings.json              ← Hook 配置
│   └── agents/
│       ├── code-reviewer.md       ← 代码审查 Agent
│       ├── test-runner.md         ← 测试运行 Agent
│       ├── security-scanner.md    ← 安全扫描 Agent
│       ├── changelog-gen.md       ← 变更日志 Agent
│       └── migration-checker.md   ← 迁移检查 Agent
├── src/
└── package.json
```

## settings.json 配置

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "agent",
            "agentType": "code-reviewer",
            "prompt": "Review the file that was just modified: {{file_path}}"
          }
        ]
      }
    ],
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "agent",
            "agentType": "test-runner",
            "prompt": "Run the full test suite and report results"
          }
        ]
      }
    ]
  }
}
```

## 说明

```
Agent Hook 字段:
  type:      "agent"（固定值）
  agentType: Agent 定义的名称（对应 .claude/agents/xxx.md）
  prompt:    传递给子 Agent 的任务描述

  特点:
    - 自动启动一个子 Agent 执行任务
    - 子 Agent 有自己的工具集和权限
    - 主流程不阻塞（子 Agent 异步执行）
    - 支持模板变量: {{file_path}} 等会被替换为实际值

  与 Command Hook 的区别:
    Command Hook → 执行 shell 命令（脚本）
    Agent Hook   → 启动一个 AI Agent（有理解能力）

  适用场景:
    - 代码审查（需要理解代码语义）
    - 测试运行和结果分析
    - 安全扫描
```

## 使用示例

### 示例 1：每次编辑后自动启动代码审查 Agent

```
settings.json:
  PostToolUse → matcher: "Edit|Write" → type: "agent"
  agentType: "code-reviewer"
  prompt: "Review the file that was just modified: {{file_path}}"

code-reviewer.md:
  ---
  description: Code review specialist
  tools: [Read, Grep, Glob]
  disallowedTools: [Write, Edit, Bash]
  ---

用户: "修改 auth.ts 添加 JWT 验证"

→ Claude 调用 Edit 修改 src/auth/jwt.ts → 成功

→ PostToolUse Agent Hook 触发:
  1. 读取 .claude/agents/code-reviewer.md
  2. 启动子 Agent，prompt: "Review the file: src/auth/jwt.ts"
  3. 子 Agent 执行:
     - 读取 src/auth/jwt.ts
     - 检查安全漏洞（JWT secret 硬编码？）
     - 检查错误处理（try-catch 完整？）
     - 检查类型安全（TypeScript 类型？）
  4. 子 Agent 返回审查结果:
     "审查发现 2 个问题:
      1. [高] JWT secret 不应硬编码，建议使用环境变量
      2. [中] 缺少 token 过期时间的边界检查"

→ Claude 看到审查结果并主动修复

效果:
  → 每次修改自动启动代码审查
  → Agent Hook 比脚本更智能（能理解代码语义）
  → 审查 Agent 是只读的（disallowedTools 确保不会修改代码）
```

### 示例 2：Agent 停止后自动运行完整测试套件

```
settings.json:
  Stop → matcher: "" → type: "agent"
  agentType: "test-runner"
  prompt: "Run the full test suite and report any failures with suggested fixes"

test-runner.md:
  ---
  description: Test runner and analyzer
  tools: [Bash, Read, Grep, Glob]
  model: haiku
  ---

用户: "完成用户模块开发"

→ Claude 完成所有修改，准备停止

→ Stop Agent Hook 触发:
  1. 启动 test-runner 子 Agent
  2. 子 Agent 执行:
     - Bash: npm test -- --coverage
     - 发现 2 个测试失败
     - 读取失败的测试文件
     - 分析失败原因
     - 报告: "测试结果: 48/50 通过，2 个失败
       1. user.test.ts:23 — 验证逻辑期望 email 格式验证
       2. user.test.ts:45 — 缺少 role 字段默认值
       建议修复: 1) 添加 email 正则验证 2) 设置 role 默认值为 'user'"

→ 用户看到完整的测试报告和修复建议

效果:
  → 每次工作结束自动运行测试
  → 使用 haiku 模型节省成本
  → 不仅运行测试，还分析失败原因并建议修复
```

### 示例 3：子 Agent 完成后触发安全扫描

```
settings.json:
  SubagentStop → matcher: "" → type: "agent"
  agentType: "security-scanner"
  prompt: "Scan all files modified in this session for security vulnerabilities"

security-scanner.md:
  ---
  description: Security vulnerability scanner
  tools: [Read, Grep, Glob, Bash]
  disallowedTools: [Write, Edit]
  effort: high
  ---

→ 一个功能开发子 Agent 刚完成工作并停止

→ SubagentStop Agent Hook 触发:
  1. 启动 security-scanner 子 Agent
  2. 子 Agent 执行:
     - Bash: git diff --name-only HEAD~1 → 找到修改的文件
     - 逐个读取修改过的文件
     - 检查 OWASP Top 10 漏洞:
       ✅ SQL 注入: 使用参数化查询
       ⚠️ XSS: 发现一处未转义的用户输入
       ✅ CSRF: Token 验证已实现
       ✅ 认证: JWT 实现正确
  3. 报告: "安全扫描发现 1 个中危漏洞:
     src/components/Comment.tsx:15 — 用户输入未转义，
     使用 dangerouslySetInnerHTML 可能导致 XSS 攻击"

效果:
  → 子 Agent 完成后自动触发安全审查
  → 多层安全防护：代码审查 + 安全扫描
  → 使用 effort: high 确保深度扫描
```

### 示例 4：Git 提交后自动生成变更日志

```
settings.json:
  PostToolUse → matcher: "Bash" → type: "agent"
  agentType: "changelog-gen"
  prompt: "Generate changelog entry for the latest commit"

changelog-gen.md:
  ---
  description: Changelog generator
  tools: [Bash, Read, Write, Grep]
  ---

用户: "提交代码"

→ Claude 调用 Bash: git commit -m "feat: add user search API" → 成功

→ PostToolUse Agent Hook 触发:
  1. 检测到 Bash 命令包含 "git commit"
  2. 启动 changelog-gen 子 Agent
  3. 子 Agent 执行:
     - Bash: git log -1 --pretty=format:"%H %s"
     - Bash: git diff HEAD~1 --stat
     - 读取现有 CHANGELOG.md
     - 分析提交类型: feat → 新功能
     - Write 更新 CHANGELOG.md:
       ## [Unreleased]
       ### Added
       - 用户搜索 API (feat: add user search API)
         变更: src/api/search.ts (+45), src/models/user.ts (+12)

效果:
  → 每次提交自动更新变更日志
  → Agent 理解 conventional commit 格式
  → 不需要手动维护 CHANGELOG
```

### 示例 5：修改数据库文件后启动迁移检查 Agent

```
settings.json:
  PostToolUse → matcher: "Write|Edit" → type: "agent"
  agentType: "migration-checker"
  prompt: "Check if database schema changes require a new migration"

migration-checker.md:
  ---
  description: Database migration checker
  tools: [Read, Grep, Glob, Bash]
  model: haiku
  ---

用户: "修改 Prisma schema 添加 User.email 字段"

→ Claude 调用 Edit 修改 prisma/schema.prisma → 成功

→ PostToolUse Agent Hook 触发:
  1. 检测到文件路径匹配 prisma/**
  2. 启动 migration-checker 子 Agent
  3. 子 Agent 执行:
     - 读取 prisma/schema.prisma
     - 对比上一次迁移: prisma/migrations/last/
     - 检测到变更: User model 新增 email 字段
     - Bash: npx prisma migrate diff --from-migrations --to-schema-datamodel
     - 报告: "检测到 schema 变更:
       User model:
         + email String @unique  (新增字段)
       需要: npx prisma migrate dev --name add-user-email
       注意: @unique 约束需要确保现有数据无重复"
  4. 自动执行: npx prisma migrate dev --name add-user-email

效果:
  → 修改 schema 后自动检查是否需要迁移
  → Agent 使用 haiku 模型节省成本
  → 自动检测变更并建议迁移命令
  → 还能提示潜在的迁移风险（如 @unique 约束）
```
