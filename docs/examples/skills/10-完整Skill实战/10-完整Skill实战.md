# Skill: 完整实战示例

> 指南 6.2「Skill 定义」— 综合运用所有字段创建生产级 Skill

## 目录结构

```
my-project/
├── .claude/
│   ├── commands/
│   │   ├── commit.md              ← 实战 1: 标准提交
│   │   ├── deploy.md              ← 实战 2: 安全部署
│   │   └── pr-review.md           ← 实战 3: PR 审查
│   └── skills/
│       ├── api-endpoint.md        ← 实战 4: API 生成
│       └── bug-fix.md             ← 实战 5: Bug 修复
├── src/
└── package.json
```

## 使用示例

### 示例 1：标准 commit 命令

```markdown
# 文件: .claude/commands/commit.md

---
description: Generate a conventional commit and push
argument-hint: "[scope] optional description hint"
allowed-tools:
  - Bash
  - Read
---

Generate a conventional commit message based on the current git diff.

## Steps

1. Run `git diff --cached` to see staged changes
2. If no staged changes, run `git diff` to see unstaged changes
3. Analyze the changes and determine:
   - Type: feat, fix, refactor, docs, test, chore, perf
   - Scope: the module/area affected
   - Description: concise summary in imperative mood
4. Stage all changes: `git add -A`
5. Commit with the generated message
6. Show the commit hash and summary

## Rules

- Use Conventional Commits format: type(scope): description
- Breaking changes must include `BREAKING CHANGE:` in footer
- Description in lowercase, imperative mood, no period
- Max 72 characters for the first line

## Additional Context

$ARGUMENTS
```

```
→ /commit
  1. Bash: git diff --cached → 无暂存
  2. Bash: git diff → 3 files changed
  3. Read: src/auth/jwt.ts → 理解变更
  4. 分析: 新增了 token 刷新逻辑 → type: feat, scope: auth
  5. Bash: git add -A
  6. Bash: git commit -m "feat(auth): add JWT token refresh mechanism"
  7. 输出: "已提交: a3f2b1c feat(auth): add JWT token refresh mechanism"

→ /commit auth fix timeout
  → 额外上下文: "auth fix timeout"
  → 生成: "fix(auth): resolve token refresh timeout on slow networks"
```

### 示例 2：安全部署命令

```markdown
# 文件: .claude/commands/deploy.md

---
description: Deploy with pre-flight checks and rollback plan
argument-hint: "staging|production"
allowed-tools:
  - Bash
  - Read
shell: bash
---

Deploy the application to the specified environment.

## Pre-flight Checks

1. Verify working tree is clean: `git status --porcelain`
2. Run tests: `npm test`
3. Run build: `npm run build`
4. Check for security vulnerabilities: `npm audit`

## Deploy Steps

1. Get the current commit hash for rollback: `git rev-parse HEAD`
2. Deploy using the appropriate script based on environment:
   - staging: `npm run deploy:staging`
   - production: `npm run deploy:production`
3. Run smoke tests: `npm run smoke-test:$ENV`

## Rollback Plan

If deployment fails:
1. Show the error output
2. Provide rollback command: `npm run rollback:$ENV -- --to=$COMMIT_HASH`

## Environment

$ARGUMENTS (default: staging)
```

```
→ /deploy production
  1. Bash: git status --porcelain → 干净 ✅
  2. Bash: npm test → 48/48 通过 ✅
  3. Bash: npm run build → 成功 ✅
  4. Bash: npm audit → 无高危漏洞 ✅
  5. Bash: git rev-parse HEAD → "e4f3a2b"（记录用于回滚）
  6. Bash: npm run deploy:production → 部署中...
  7. Bash: npm run smoke-test:production → 全部通过 ✅
  8. 输出: "✅ 生产环境部署成功。回滚点: e4f3a2b"
```

### 示例 3：PR 审查 Skill

```markdown
# 文件: .claude/commands/pr-review.md

---
description: Review a GitHub Pull Request
argument-hint: "<PR number>"
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
agent: code-reviewer
---

Review the specified Pull Request comprehensively.

## Steps

1. Fetch PR details: `gh pr view $ARGUMENTS --json title,body,files,additions,deletions`
2. Get the diff: `gh pr diff $ARGUMENTS`
3. For each changed file, analyze:
   - Code quality and readability
   - Potential bugs or edge cases
   - Security implications
   - Performance impact
   - Test coverage
4. Generate a structured review comment

## Review Format

### Summary
[One paragraph summary of the PR]

### Issues Found
- 🔴 [Critical] ...
- 🟡 [Warning] ...
- 🔵 [Suggestion] ...

### Positive Notes
- ✅ ...

### Verdict
Approve / Request Changes / Comment
```

```
→ /pr-review 42
  1. 使用 code-reviewer Agent（只读、高推理强度）
  2. Bash: gh pr view 42 → PR 标题、描述、文件列表
  3. Bash: gh pr diff 42 → 完整 diff
  4. Read: 逐个读取变更文件
  5. Grep: 检查安全模式、TODO 等
  6. 输出结构化审查报告:
     "### Summary
      添加用户搜索 API，支持模糊匹配和分页
     ### Issues Found
      🔴 [Critical] 搜索输入未做 SQL 注入防护
      🟡 [Warning] 缺少搜索结果的缓存策略
      🔵 [Suggestion] 考虑添加搜索建议功能
     ### Verdict: Request Changes"
```

### 示例 4：API 端点生成 Skill

```markdown
# 文件: .claude/skills/api-endpoint.md

---
description: Generate a REST API endpoint with full stack
argument-hint: "<resource-name> [actions: CRUD]"
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Grep
  - Glob
paths:
  - "src/api/**"
  - "src/models/**"
---

Generate a complete REST API endpoint for: $ARGUMENTS

## Generation Checklist

1. Check existing project patterns (read an existing endpoint)
2. Generate model/schema
3. Generate route with CRUD operations
4. Generate validation middleware
5. Generate controller/service layer
6. Generate tests
7. Register route in router
8. Run tests to verify

## Patterns to Follow

- Read `src/api/users.ts` for the pattern to follow
- Use the same validation library
- Follow the same error handling pattern
- Match the project's authentication middleware
```

```
→ /api-endpoint products CRUD
  1. Read: src/api/users.ts → 学习项目 API 模式
  2. Write: src/models/product.ts → 产品模型
  3. Write: src/api/products.ts → CRUD 路由
  4. Write: src/validators/product.ts → 输入验证
  5. Edit: src/routes/index.ts → 注册路由
  6. Write: tests/api/products.test.ts → 测试
  7. Bash: npx jest tests/api/products.test.ts → 验证
  8. 输出: "已生成 products CRUD API: 5 个文件创建，1 个文件修改"
```

### 示例 5：Bug 修复 Skill

```markdown
# 文件: .claude/skills/bug-fix.md

---
description: Fix a bug with root cause analysis
argument-hint: "<bug description or issue number>"
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Grep
  - Glob
---

Fix the following bug with a thorough root cause analysis.

## Bug Description

$ARGUMENTS

## Investigation Steps

1. Understand the bug from the description or issue
2. Find related code using Grep/Glob
3. Read and analyze the relevant files
4. Identify the root cause
5. Develop a fix
6. Write or update tests to prevent regression
7. Run tests to verify the fix
8. Document the fix

## Output Format

### Root Cause
[What caused the bug]

### Fix
[What was changed and why]

### Prevention
[How to prevent this class of bugs in the future]

### Test Coverage
[New/updated tests that verify the fix]
```

```
→ /bug-fix Issue #156: Login fails when password contains special characters

  1. Bash: gh issue view 156 → 获取 Bug 详情
  2. Grep: "password|login|auth" → 找到相关代码
  3. Read: src/auth/login.ts → 发现密码未做 URL 编码
  4. Root cause: 密码中的 & 和 = 字符被解析为查询参数
  5. Edit: src/auth/login.ts → 添加 encodeURIComponent
  6. Write: tests/auth/login-special-chars.test.ts → 回归测试
  7. Bash: npx jest tests/auth/ → 全部通过
  8. 输出:
     "### Root Cause
      密码未做 URL 编码，特殊字符被解析为查询参数分隔符
     ### Fix
      在发送登录请求前对密码做 encodeURIComponent()
     ### Test
      新增 5 个测试用例覆盖特殊字符密码"
```
