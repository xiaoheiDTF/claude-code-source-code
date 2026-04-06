# Claude Code Skills 实战示例

> 从简单到复杂，覆盖所有 Skill 定义模式

---

## 示例 1：最简纯文本 Skill（安全属性自动允许）

创建 `.claude/skills/code-review/SKILL.md`：

```markdown
---
name: code-review
description: Review code changes for quality and security issues
when_to_use: Use when the user asks to review code, check PR quality, or audit changes. Examples: 'review my code', 'check this PR', 'audit changes'.
---

# Code Review

Review the code changes for:

## Checklist
1. **Security** — XSS, SQL injection, secrets in code, OWASP top 10
2. **Correctness** — Logic errors, edge cases, null handling
3. **Performance** — Unnecessary loops, N+1 queries, memory leaks
4. **Style** — Naming, consistency, dead code

## Output Format
For each issue found:
- **Severity**: Critical / Warning / Info
- **Location**: File and line
- **Issue**: What's wrong
- **Fix**: Suggested fix
```

**特点**：无 `allowed-tools`、无 `model`、无 `hooks`，被 `skillHasOnlySafeProperties()` 自动允许，不弹权限对话框。

---

## 示例 2：带工具白名单的 Skill

创建 `.claude/skills/check-tests/SKILL.md`：

```markdown
---
name: check-tests
description: Run project tests and summarize results
allowed-tools: Bash(npm test:*), Bash(npx jest:*), Read, Glob, Grep
when_to_use: Use when the user wants to run tests or check test status.
arguments:
  - name: test_path
    description: Optional test file or pattern to run
    required: false
---

# Check Tests

Run the project tests and provide a summary.

## Steps

### 1. Discover test configuration
- Check for package.json scripts (look for "test", "test:ci", "test:all")
- Identify the test runner (jest, vitest, mocha, etc.)

### 2. Run tests
$ARGUMENTS

If $ARGUMENTS is empty, run the default test command.

### 3. Summarize results
- Total tests / Passed / Failed / Skipped
- List of failing tests with error messages
- Suggested fixes for failures
```

**特点**：声明 `allowed-tools`，通过 `contextModifier` 在执行时自动允许这些工具；`$ARGUMENTS` 占位符接收用户参数。

---

## 示例 3：条件激活 Skill（只在操作特定文件时出现）

创建 `.claude/skills/api-review/SKILL.md`：

```markdown
---
name: api-review
description: Review API endpoint changes for RESTful best practices
paths:
  - "src/routes/**/*.ts"
  - "src/api/**/*.ts"
  - "src/controllers/**/*.ts"
when_to_use: Use when reviewing or modifying API endpoint code.
---

# API Review

When reviewing API changes, check:

1. **HTTP Methods** — Correct verb usage (GET for reads, POST for creates, etc.)
2. **Status Codes** — Proper response codes (201 for created, 404 for not found)
3. **Authentication** — Endpoints properly guarded
4. **Input Validation** — Request body/schema validated
5. **Error Handling** — Consistent error response format
6. **Pagination** — List endpoints support pagination
7. **Versioning** — API version in URL path or header
```

**特点**：只有当 Claude 操作 `src/routes/`、`src/api/` 或 `src/controllers/` 下的文件时，通过 `activateConditionalSkillsForPaths()` 激活。

---

## 示例 4：指定模型和努力等级的 Skill

创建 `.claude/skills/architect/SKILL.md`：

```markdown
---
name: architect
description: Design system architecture with deep analysis
model: claude-opus-4-6
effort: high
allowed-tools: Read, Glob, Grep, Agent
when_to_use: Use when the user wants to design a system, plan architecture, or evaluate technical trade-offs.
arguments:
  - name: description
    description: What to design or analyze
    required: true
---

# Architect

Design or analyze system architecture for: $ARGUMENTS

## Process

### 1. Understand Requirements
- Read existing code structure with Glob and Grep
- Understand current architecture constraints
- Clarify requirements if ambiguous

### 2. Analyze Trade-offs
- List at least 2 approaches
- Compare on: scalability, maintainability, performance, complexity
- Consider migration path from current state

### 3. Propose Architecture
- Draw component diagrams (text-based)
- Define interfaces and contracts
- Specify data flow and dependencies

### 4. Implementation Plan
- Break into phases with clear milestones
- Identify risky changes and mitigation strategies
- Suggest testing approach for each phase
```

**特点**：`model: claude-opus-4-6` 强制使用最强模型；`effort: high` 提高思考预算；`contextModifier` 会覆盖当前会话的模型和努力等级设置。

---

## 示例 5：分叉执行 Skill（隔离子 Agent）

创建 `.claude/skills/deep-audit/SKILL.md`：

```markdown
---
name: deep-audit
description: Deep security audit running in isolated sub-agent
agent: general-purpose
model: claude-sonnet-4-6
effort: high
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, Agent
disable-model-invocation: true
user-invocable: true
arguments:
  - name: scope
    description: Directory or file pattern to audit
    required: false
---

# Deep Security Audit

Perform a comprehensive security audit of: $ARGUMENTS

## Scope
If no scope provided, audit the entire project.

## Audit Checklist

### 1. Dependency Vulnerabilities
- Run `npm audit` / `yarn audit`
- Check for known CVEs in dependencies
- Flag outdated packages with security patches

### 2. Code-Level Issues
- Search for hardcoded secrets/credentials
- Check for SQL injection patterns
- Look for command injection vulnerabilities
- Find unsafe deserialization
- Check for path traversal risks

### 3. Configuration Review
- Check .env files for sensitive defaults
- Review CORS configuration
- Check authentication middleware
- Verify HTTPS enforcement

### 4. Output Report
Create a markdown report at `security-audit-report.md`:
- Executive summary
- Critical/High/Medium/Low findings
- Each finding with: location, description, remediation
- Remediation priority order
```

**特点**：`agent: general-purpose` 触发分叉执行路径（`executeForkedSkill()`），在隔离的子 Agent 中运行，有独立的 Token 预算；`disable-model-invocation: true` 禁止模型自动调用，只能用户手动 `/deep-audit` 触发。

---

## 示例 6：带 Shell 注入的 Skill（动态内容）

创建 `.claude/skills/git-status/SKILL.md`：

```markdown
---
name: git-status
description: Show git status with branch info and recent commits
allowed-tools: Bash(git:*), Read
when_to_use: Use when the user asks about git status, branch info, or recent changes.
---

# Git Status Report

## Current Branch
!`git branch --show-current`

## Status
!`git status --short`

## Recent Commits (last 5)
!`git log --oneline -5`

## Stashed Changes
!`git stash list || echo "No stashes"`

## Remote Tracking
!`git remote -v`

---

Based on the above, summarize:
1. Whether the working tree is clean
2. What branch we're on and its tracking status
3. Any important recent changes
4. Suggested next steps
```

**特点**：`!`command`` 语法在提示词生成时执行 Shell 命令，将输出嵌入提示词。MCP 技能不允许此语法。

---

## 示例 7：源码中的真实 Bundled Skill — /debug

以下来自 `src/skills/bundled/debug.ts` 的实际代码：

```typescript
registerBundledSkill({
  name: 'debug',
  description: 'Enable debug logging for this session and help diagnose issues',
  allowedTools: ['Read', 'Grep', 'Glob'],
  disableModelInvocation: true,   // 禁止模型自动调用
  userInvocable: true,            // 用户可 /debug 调用
  argumentHint: '[issue description]',

  async getPromptForCommand(args) {
    // 动态构建提示词：读取 debug log 文件的最后 20 行
    const wasAlreadyLogging = enableDebugLogging()
    const debugLogPath = getDebugLogPath()
    const tail = await tailFile(debugLogPath, 20)

    const prompt = `# Debug Skill
Help the user debug an issue.

## Session Debug Log
Path: ${debugLogPath}
${tail}

## Issue
${args || 'No specific issue described. Check for errors.'}

## Instructions
1. Review issue description
2. Look for [ERROR] and [WARN] in the log
3. Explain findings in plain language
4. Suggest concrete fixes`

    return [{ type: 'text', text: prompt }]
  },
})
```

**特点**：TypeScript 代码定义，动态构建提示词（读日志文件），比 SKILL.md 灵活但需要代码。

---

## 示例 8：源码中的真实 Bundled Skill — /loop

来自 `src/skills/bundled/loop.ts`：

```typescript
registerBundledSkill({
  name: 'loop',
  description: 'Run a prompt on a recurring interval (e.g. /loop 5m /foo)',
  whenToUse:
    'When the user wants to set up a recurring task, poll for status, or run something repeatedly.',
  argumentHint: '[interval] <prompt>',
  userInvocable: true,
  isEnabled: isKairosCronEnabled,  // Feature gate 控制

  async getPromptForCommand(args) {
    // 解析 "[interval] <prompt>" 格式
    // 转换为 cron 表达式
    // 内部调用 CronCreateTool 注册定时任务
    // 然后立即执行一次
    return [{ type: 'text', text: buildPrompt(args) }]
  },
})
```

用户使用方式：

```
/loop 5m check the deploy          → 每 5 分钟检查部署状态
/loop check the deploy every 20m   → 每 20 分钟检查
/loop 1h /standup 1                → 每小时执行 /standup 技能
```

---

## 四种定义模式对比

| 维度 | SKILL.md 静态 | SKILL.md 动态 | Bundled TS | 条件激活 |
|------|--------------|--------------|-----------|---------|
| 定义方式 | Markdown 文件 | Markdown + Shell 注入 | TypeScript 代码 | Markdown + paths |
| 技术门槛 | 零代码 | 基础 Shell | TypeScript | 零代码 |
| 提示词灵活性 | 静态文本 | 运行时动态生成 | 完全可编程 | 静态文本 |
| 触发方式 | 用户 `/name` 或模型自调用 | 同左 | 同左 | 文件路径匹配自动激活 |
| 权限 | 自动允许或需确认 | 需确认（有 allowed-tools） | 需确认 | 自动允许或需确认 |
| 适用场景 | 通用提示词模板 | 需要运行时数据 | 复杂交互逻辑 | 项目/文件类型特定 |

---

## 文件放置位置

```
项目级（提交到 git，团队共享）：
  .claude/skills/my-skill/SKILL.md

用户级（全局，跨项目）：
  ~/.claude/skills/my-skill/SKILL.md

遗留格式（兼容旧版）：
  .claude/commands/my-command.md
```
