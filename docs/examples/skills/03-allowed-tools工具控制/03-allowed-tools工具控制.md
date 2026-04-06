# Skill: allowed-tools 工具控制

> 指南 6.2「Skill 定义」— allowed-tools 指定 Skill 执行时额外允许的工具

## 目录结构

```
my-project/
├── .claude/
│   └── commands/
│       ├── commit.md              ← allowed-tools: [Bash, Read]
│       ├── review.md              ← allowed-tools: [Read, Grep, Glob]
│       ├── deploy.md              ← allowed-tools: [Bash]
│       ├── fix-lint.md            ← allowed-tools: [Edit, Bash, Read]
│       └── analyze.md             ← 无 allowed-tools（继承默认）
└── src/
```

## 说明

```
allowed-tools 字段:
  格式: YAML 列表
  作用: 指定执行 Skill 时额外允许的工具

  可用工具名:
    Read, Write, Edit, Bash, Grep, Glob, Agent, WebSearch, WebFetch, ...

  行为:
    - 不指定 allowed-tools → 使用当前会话的默认工具集
    - 指定 allowed-tools → Skill 执行时只能使用列表中的工具
    - Skill 执行结束后恢复原来的工具集

  与 Agent 的 tools/disallowedTools 的区别:
    Skill 的 allowed-tools → 执行时临时允许的工具
    Agent 的 tools → Agent 可以使用的完整工具集
```

## 使用示例

### 示例 1：commit Skill 只允许 Bash + Read

```
commit.md:
  ---
  description: Generate and execute a git commit
  allowed-tools:
    - Bash
    - Read
  ---

  Based on the current git diff, generate a conventional commit message
  and commit the changes. $ARGUMENTS

→ 用户输入: /commit fix auth timeout

→ Skill 执行时的工具集:
  ✅ Bash → git diff, git add, git commit
  ✅ Read → 读取修改的文件内容
  ❌ Write → 不能创建新文件
  ❌ Edit → 不能修改文件（commit 不应该改代码）
  ❌ Grep → 不需要搜索

→ 执行流程:
  1. Bash: git diff → 查看变更
  2. Bash: git diff --stat → 变更统计
  3. Read: src/auth/login.ts → 理解具体修改
  4. Bash: git add -A
  5. Bash: git commit -m "fix(auth): resolve timeout issue"

效果:
  → allowed-tools 限制 Skill 只做该做的事
  → commit 不应该修改代码（排除 Edit/Write）
  → 工具限制引导 Claude 聚焦任务
```

### 示例 2：review Skill 只读不写

```
review.md:
  ---
  description: Code review without modifying files
  allowed-tools:
    - Read
    - Grep
    - Glob
  ---

  Review the following files for code quality, security, and best practices.
  $ARGUMENTS

→ 用户输入: /review src/auth/

→ Skill 执行时的工具集:
  ✅ Read → 读取源码
  ✅ Grep → 搜索模式
  ✅ Glob → 查找文件
  ❌ Edit → 不能修改（审查不应该改代码）
  ❌ Write → 不能写入
  ❌ Bash → 不能执行命令

→ 执行流程:
  1. Glob: "src/auth/**/*.ts" → 找到 8 个文件
  2. Read: src/auth/login.ts → 审查登录逻辑
  3. Read: src/auth/oauth.ts → 审查 OAuth
  4. Grep: "password|secret|token" → 安全检查
  5. 输出审查报告（不修改任何文件）

效果:
  → 纯只读工具确保审查不会意外修改代码
  → 审查结果以文本形式输出
  → 安全可靠：不可能破坏代码
```

### 示例 3：deploy Skill 只允许 Bash

```
deploy.md:
  ---
  description: Deploy to specified environment
  allowed-tools:
    - Bash
  ---

  Deploy the application to the specified environment.
  Run pre-deploy checks first.

  Environment: $ARGUMENTS (default: staging)

→ 用户输入: /deploy production

→ Skill 执行时的工具集:
  ✅ Bash → 部署命令
  ❌ Read → 不需要读文件
  ❌ Edit/Write → 不需要改代码
  ❌ Grep/Glob → 不需要搜索

→ 执行流程:
  1. Bash: npm run predeploy:check → 前置检查
  2. Bash: npm run build:production → 构建
  3. Bash: npm run deploy:production → 部署
  4. Bash: npm run smoke-test:production → 验证

效果:
  → 部署只需要执行命令
  → 不允许读/写文件防止误操作
  → 工具集最小化原则
```

### 示例 4：fix-lint Skill 允许读写 + 执行

```
fix-lint.md:
  ---
  description: Auto-fix all linting issues
  allowed-tools:
    - Edit
    - Bash
    - Read
  ---

  Find and fix all ESLint errors and warnings in the project.

→ 用户输入: /fix-lint

→ Skill 执行时的工具集:
  ✅ Bash → 运行 eslint
  ✅ Read → 读取有问题的文件
  ✅ Edit → 修复 lint 问题

→ 执行流程:
  1. Bash: npx eslint . --format json → 获取所有问题
  2. Read: src/app.ts → 读取有问题的文件
  3. Edit: src/app.ts → 修复未使用的变量
  4. Read: src/utils.ts
  5. Edit: src/utils.ts → 修复缺少分号
  6. Bash: npx eslint . → 验证全部修复

效果:
  → 需要读+写+执行的完整修复流程
  → Edit 修复代码，Bash 运行 lint 验证
  → 工具集匹配任务需求
```

### 示例 5：不指定 allowed-tools 继承默认工具集

```
analyze.md:
  ---
  description: Analyze code architecture
  ---

  Analyze the project architecture and generate a report.
  $ARGUMENTS

→ 没有 allowed-tools 字段

→ Skill 执行时使用当前会话的默认工具集:
  所有可用工具都可以使用 ✅

→ 执行流程:
  1. Glob: "**/*.ts" → 项目文件列表
  2. Read: package.json → 依赖分析
  3. Grep: "import.*from" → 依赖关系
  4. Bash: npx madge --circular src/ → 循环依赖检测
  5. Edit: docs/architecture.md → 生成架构文档（如果需要）

效果:
  → 不限制工具，Claude 可以自由选择最适合的工具
  → 灵活但需要信任 Claude 不会误操作
  → 适合复杂分析任务
```
