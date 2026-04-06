# Claude Code Skills 脚本执行与 Hooks 机制

> 深度分析 Skills 如何执行 Shell 脚本、注册生命周期 Hooks、以及在提示词中注入动态内容

---

## 一、脚本执行的两种方式

Skills 中执行脚本有两种机制：

| 方式 | 语法 | 执行时机 | 用途 |
|------|------|----------|------|
| **提示词内 Shell 注入** | `!`cmd`` 或 `` ```! cmd ``` `` | 生成提示词时 | 动态数据嵌入提示词 |
| **Hooks（生命周期钩子）** | frontmatter `hooks` 字段 | 工具调用前后、会话事件 | 拦截/审计/自动化 |

---

## 二、提示词内 Shell 注入

### 源码位置
`src/utils/promptShellExecution.ts`

### 两种语法

```markdown
# 行内注入
当前分支: !`git branch --show-current`

# 代码块注入
```!
echo "系统信息:"
uname -a
echo "工作目录: $(pwd)"
```
```

### 执行流程

```
getPromptForCommand() 被调用
        │
        ▼
┌──────────────────────────────────────────────┐
│  1. 模板变量替换                              │
│     $ARGUMENTS → 用户参数                     │
│     ${CLAUDE_SKILL_DIR} → 技能目录路径         │
│     ${CLAUDE_SESSION_ID} → 会话 ID            │
└──────────────┬───────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────┐
│  2. executeShellCommandsInPrompt()           │
│     src/utils/promptShellExecution.ts        │
│                                              │
│  ┌────────────────────────────────────────┐  │
│  │ 正则匹配所有 !`cmd` 和 ```! cmd ```    │  │
│  │                                        │  │
│  │ BLOCK_PATTERN = /```!\s*\n?([\s\S]*?)  │  │
│  │                \n?```/g                │  │
│  │ INLINE_PATTERN = /(?<=^|\s)!`([^`]+)`/ │  │
│  └────────────────────────────────────────┘  │
│               │                              │
│               ▼                              │
│  ┌────────────────────────────────────────┐  │
│  │ 3. 对每个匹配的命令：                    │  │
│  │                                        │  │
│  │ a. hasPermissionsToUseTool() 权限检查    │  │
│  │    使用 BashTool 或 PowerShellTool      │  │
│  │    （由 frontmatter shell 字段决定）     │  │
│  │                                        │  │
│  │ b. shellTool.call({ command }, ctx)    │  │
│  │    实际执行命令                          │  │
│  │                                        │  │
│  │ c. processToolResultBlock()             │  │
│  │    处理输出（超大结果持久化到磁盘）        │  │
│  │                                        │  │
│  │ d. result.replace(match[0], output)     │  │
│  │    用输出替换原始 !`cmd` 占位符          │  │
│  └────────────────────────────────────────┘  │
│               │                              │
│               ▼                              │
│     所有命令并行执行 (Promise.all)            │
│     返回替换后的提示词                         │
└──────────────────────────────────────────────┘
```

### Shell 选择机制

frontmatter 的 `shell` 字段控制使用哪个 Shell 执行：

```yaml
---
shell: bash         # 默认，使用 $SHELL (bash/zsh/sh)
# 或
shell: powershell   # 使用 pwsh（仅当用户已启用 PowerShell 工具）
---
```

源码逻辑：
```typescript
// promptShellExecution.ts
const shellTool = shell === 'powershell' && isPowerShellToolEnabled()
  ? getPowerShellTool()    // PowerShell
  : BashTool               // Bash（默认）
```

**关键设计**：`shell` 字段**只来自 frontmatter**，不会读 `settings.defaultShell`。因为 Skills 是可移植的，由作者决定用哪个 Shell，不是读者。

### MCP 技能的安全限制

```typescript
// loadSkillsDir.ts:374
// MCP 技能从不执行 Shell 注入（远程来源不可信）
if (loadedFrom !== 'mcp') {
  finalContent = await executeShellCommandsInPrompt(...)
}
```

### 实战示例：带 Shell 注入的部署检查 Skill

```markdown
---
name: deploy-status
description: Check deployment status for current branch
allowed-tools: Bash(git:*), Bash(curl:*)
when_to_use: Use when the user wants to check deployment status.
---

# Deploy Status

## Current Branch
!`git branch --show-current`

## Recent Commits
```!
git log --oneline -5
```

## Environment
- **Node**: !`node --version`
- **OS**: !`uname -s`

---

Based on the above, check if the latest commit on branch `!`git branch --show-current``
has been deployed to staging and production environments.
```

---

## 三、Hooks 机制（生命周期钩子）

### 支持的 Hook 事件

源码位置：`src/entrypoints/sdk/coreSchemas.ts`

```typescript
const HOOK_EVENTS = [
  'PreToolUse',           // 工具执行前
  'PostToolUse',          // 工具执行后
  'PostToolUseFailure',   // 工具执行失败后
  'Notification',         // 通知事件
  'UserPromptSubmit',     // 用户提交输入时
  'SessionStart',         // 会话开始
  'SessionEnd',           // 会话结束
  'Stop',                 // 模型停止生成
  'StopFailure',          // 模型生成失败
  'SubagentStart',        // 子 Agent 启动
  'SubagentStop',         // 子 Agent 停止
  'PreCompact',           // 压缩上下文前
  'PostCompact',          // 压缩上下文后
  'PermissionRequest',    // 权限请求
  'PermissionDenied',     // 权限被拒绝
  'Elicitation',          // 询问用户
  'ElicitationResult',    // 用户回答
  'FileChanged',          // 文件变更
  'CwdChanged',           // 工作目录变更
  // ... 等
] as const
```

### Hook 的四种类型

源码位置：`src/schemas/hooks.ts`

```typescript
// 1. Shell 命令 Hook（最常用）
{
  type: 'command',
  command: 'shell command to run',
  if: 'Bash(git *)',              // 可选：条件匹配
  shell: 'bash',                   // 可选：bash 或 powershell
  timeout: 30,                     // 可选：超时秒数
  statusMessage: 'Linting...',     // 可选：状态消息
  once: false,                     // 可选：是否只执行一次
  async: false,                    // 可选：是否异步（不阻塞）
}

// 2. LLM Prompt Hook（让 Claude 评估）
{
  type: 'prompt',
  prompt: 'Review this code change for security issues: $ARGUMENTS',
  model: 'claude-sonnet-4-6',      // 可选：指定模型
  timeout: 60,
}

// 3. Agent Hook（启动子 Agent 验证）
{
  type: 'agent',
  prompt: 'Verify that unit tests ran and passed: $ARGUMENTS',
  model: 'claude-haiku-4-5',
  timeout: 60,
}

// 4. HTTP Hook（发送到远程服务器）
{
  type: 'http',
  url: 'https://hooks.example.com/notify',
  headers: {
    'Authorization': 'Bearer $MY_TOKEN',
  },
  allowedEnvVars: ['MY_TOKEN'],    // 允许插值的环境变量
}
```

### Hook Matcher（匹配器）

每个 Hook 事件可以配置多个 matcher，控制何时触发：

```yaml
hooks:
  PreToolUse:
    - matcher: "Write"              # 只在调用 Write 工具时触发
      hooks:
        - type: command
          command: 'echo "Writing file: $TOOL_INPUT" >> audit.log'
    - matcher: "Bash(git push*)"    # 匹配 git push 命令
      hooks:
        - type: command
          command: 'run-approval-check.sh'
    - matcher: ""                   # 空字符串 = 匹配所有工具
      hooks:
        - type: command
          command: 'echo "Tool called: $TOOL_NAME" >> debug.log'
          async: true               # 不阻塞工具执行
```

---

## 四、Skill 中注册 Hooks

### frontmatter 中定义 Hooks

创建 `.claude/skills/secure-edit/SKILL.md`：

```markdown
---
name: secure-edit
description: Edit files with automatic security scanning and audit logging
allowed-tools: Read, Edit, Write, Bash(security-scan:*)
hooks:
  PreToolUse:
    - matcher: "Edit"
      hooks:
        - type: command
          command: '${CLAUDE_PLUGIN_ROOT}/scripts/pre-edit-scan.sh "$TOOL_INPUT"'
          timeout: 10
          statusMessage: 'Scanning for secrets...'
    - matcher: "Write"
      hooks:
        - type: command
          command: '${CLAUDE_PLUGIN_ROOT}/scripts/pre-write-scan.sh "$TOOL_INPUT"'
          timeout: 10
  PostToolUse:
    - matcher: "Edit"
      hooks:
        - type: command
          command: '${CLAUDE_PLUGIN_ROOT}/scripts/audit-edit.sh "$TOOL_INPUT"'
          async: true
          statusMessage: 'Logging edit...'
  Notification:
    - matcher: ""
      hooks:
        - type: command
          command: 'notify-send "Claude Code" "$NOTIFICATION_MESSAGE"'
          async: true
---

# Secure Edit Skill

Edit files with automatic security scanning.

## Rules
1. Every edit is scanned for secrets before execution
2. Every edit is logged to audit trail after execution
3. Desktop notifications for important events

## Instructions
$ARGUMENTS
```

### Hooks 注册流程

```
Skill 被调用（用户 /secure-edit 或模型触发）
        │
        ▼
┌──────────────────────────────────────────────────┐
│  processPromptSlashCommand()                     │
│  调用 getMessagesForPromptSlashCommand()          │
└──────────────┬───────────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────────┐
│  registerSkillHooks()                            │
│  src/utils/hooks/registerSkillHooks.ts           │
│                                                  │
│  遍历 HOOK_EVENTS，对每个事件：                     │
│  ├─ 读取 command.hooks[eventName]                 │
│  ├─ 对每个 matcher 的每个 hook：                   │
│  │   ├─ 如果 hook.once === true                   │
│  │   │   → 设置 onHookSuccess 回调自动移除         │
│  │   └─ addSessionHook(                           │
│  │       setAppState, sessionId,                  │
│  │       eventName, matcher, hook,                │
│  │       onHookSuccess, skillRoot                 │
│  │     )                                          │
│  └─ 生命周期：随会话存在，会话结束时清除             │
└──────────────────────────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────────┐
│  addSessionHook() → addHookToSession()           │
│  src/utils/hooks/sessionHooks.ts                 │
│                                                  │
│  将 hooks 存储到 AppState.sessionHooks Map 中     │
│  key: sessionId                                  │
│  value: { hooks: Partial<Record<HookEvent,        │
│          SessionHookMatcher[]> } }               │
│                                                  │
│  skillRoot 存储到 matcher 上，用于设置             │
│  CLAUDE_PLUGIN_ROOT 环境变量                      │
└──────────────────────────────────────────────────┘
```

### CLAUDE_PLUGIN_ROOT 环境变量

当 Skill 的 Hook 执行 Shell 命令时，`CLAUDE_PLUGIN_ROOT` 被自动设置为技能目录：

```typescript
// src/utils/hooks.ts:908
envVars.CLAUDE_PLUGIN_ROOT = toHookPath(skillRoot)
```

这让 Hook 脚本可以引用技能目录中的辅助文件：

```
.claude/skills/secure-edit/
├── SKILL.md                      # 技能定义（hooks 在这里声明）
└── scripts/
    ├── pre-edit-scan.sh          # ${CLAUDE_PLUGIN_ROOT}/scripts/pre-edit-scan.sh
    ├── pre-write-scan.sh
    └── audit-edit.sh
```

### once Hook（一次性 Hook）

```yaml
hooks:
  SessionStart:
    - matcher: ""
      hooks:
        - type: command
          command: 'echo "Session started at $(date)" >> session.log'
          once: true    # 执行一次后自动移除
```

源码：
```typescript
// registerSkillHooks.ts
const onHookSuccess = hook.once
  ? () => {
      logForDebugging(
        `Removing one-shot hook for event ${eventName} in skill '${skillName}'`,
      )
      removeSessionHook(setAppState, sessionId, eventName, hook)
    }
  : undefined
```

---

## 五、完整实战示例

### 示例 1：带预检查脚本的 Git Commit Skill

```
.claude/skills/safe-commit/
├── SKILL.md
└── scripts/
    └── pre-commit-checks.sh
```

**SKILL.md**：
```markdown
---
name: safe-commit
description: Commit with automatic lint, test, and security checks
allowed-tools: Bash(git:*), Bash(npm:*), Bash(npx:*), Read, Edit
hooks:
  PreToolUse:
    - matcher: "Bash(git commit*)"
      hooks:
        - type: command
          command: 'bash ${CLAUDE_PLUGIN_ROOT}/scripts/pre-commit-checks.sh'
          timeout: 60
          statusMessage: 'Running pre-commit checks...'
when_to_use: Use when the user wants to commit changes safely.
arguments:
  - name: message
    description: Commit message
    required: true
---

# Safe Commit

Create a git commit with automatic quality checks.

## Message
$ARGUMENTS

## Steps

### 1. Pre-commit checks (automatic via Hook)
The PreToolUse hook for `Bash(git commit*)` will automatically run:
- ESLint
- Type check
- Unit tests
- Secret scanning

### 2. Stage changes
Review the diff and stage appropriate files.

### 3. Create commit
Use the provided message: $ARGUMENTS

If pre-commit checks fail, report the issues and suggest fixes.
```

**scripts/pre-commit-checks.sh**：
```bash
#!/bin/bash
set -e

echo "Running lint..."
npx eslint . --max-warnings 0

echo "Type checking..."
npx tsc --noEmit

echo "Running tests..."
npx jest --passWithNoTests

echo "Checking for secrets..."
git diff --cached | npx secretlint

echo "All checks passed!"
```

### 示例 2：多事件 Hook + LLM 审查 Skill

```markdown
---
name: review-pipeline
description: Full code review with automated analysis pipeline
allowed-tools: Read, Glob, Grep, Bash(git:*), Agent
agent: general-purpose
context: fork
hooks:
  PreToolUse:
    - matcher: "Read"
      hooks:
        - type: command
          command: 'echo "Reading: $TOOL_INPUT" >> ${CLAUDE_PLUGIN_ROOT}/review-trace.log'
          async: true
  PostToolUse:
    - matcher: ""
      hooks:
        - type: command
          command: 'echo "$(date): $TOOL_NAME completed" >> ${CLAUDE_PLUGIN_ROOT}/review-trace.log'
          async: true
  Stop:
    - matcher: ""
      hooks:
        - type: prompt
          prompt: 'Review the trace log and summarize: what files were analyzed, what tools were used, and what was the total review time. $ARGUMENTS'
          model: claude-haiku-4-5
when_to_use: Use when the user wants a comprehensive code review.
argument-hint: "[file or directory to review]"
---

# Review Pipeline

Perform a comprehensive code review of: $ARGUMENTS

## Review Checklist
1. **Security** — OWASP top 10, injection, secrets
2. **Performance** — N+1 queries, memory leaks, algorithmic complexity
3. **Correctness** — Logic errors, edge cases, null handling
4. **Style** — Naming, consistency, dead code

## Output
- Summary of findings by severity
- Specific code locations
- Suggested fixes

All tool usage is automatically traced via hooks.
```

### 示例 3：桌面通知 + 文件监控 Skill

```markdown
---
name: monitor-deploy
description: Monitor a deployment and notify on completion
allowed-tools: Bash(curl:*), Bash(kubectl:*), Read
hooks:
  Notification:
    - matcher: ""
      hooks:
        - type: command
          command: |
            osascript -e 'display notification "$NOTIFICATION_MESSAGE" with title "Claude Code Deploy"'
          async: true
  FileChanged:
    - matcher: "*.log"
      hooks:
        - type: command
          command: 'tail -1 "$FILE_PATH" | grep -q "ERROR" && echo "ERROR detected in $FILE_PATH"'
          async: true
          statusMessage: 'Checking for errors...'
when_to_use: Use when the user wants to monitor a deployment.
argument-hint: "[service name]"
---

# Deploy Monitor

Monitor the deployment of: $ARGUMENTS

## Steps
1. Check current deployment status
2. Watch for completion or errors
3. Send desktop notification on result

All notifications are automatically forwarded to the desktop.
```

---

## 六、完整执行流程图

```
用户 /safe-commit "fix: auth bug"
        │
        ▼
┌──────────────────────────────────────────────────────┐
│  SkillTool.call()                                    │
│                                                      │
│  1. 查找命令 → 找到 safe-commit                       │
│  2. context !== 'fork' → 内联执行                     │
│  3. processPromptSlashCommand()                       │
│     │                                                │
│     ├─ getPromptForCommand(args, context)             │
│     │   │                                            │
│     │   ├─ 模板替换：$ARGUMENTS → "fix: auth bug"     │
│     │   ├─ 模板替换：${CLAUDE_SKILL_DIR} → 绝对路径   │
│     │   ├─ 模板替换：${CLAUDE_SESSION_ID} → xxx       │
│     │   │                                            │
│     │   ├─ Shell 注入执行：                           │
│     │   │   !`git branch` → 替换为 "main"            │
│     │   │   ```!...``` → 替换为命令输出               │
│     │   │                                            │
│     │   └─ 返回完整提示词                              │
│     │                                                │
│     ├─ registerSkillHooks() ← 注册 Hooks             │
│     │   ├─ PreToolUse["Bash(git commit*)"]            │
│     │   │   → pre-commit-checks.sh                   │
│     │   └─ 存入 AppState.sessionHooks                │
│     │                                                │
│     └─ 返回 { messages, contextModifier }            │
│                                                      │
│  4. contextModifier 注入：                            │
│     ├─ allowedTools → alwaysAllowRules               │
│     └─ model/effort 覆盖                             │
└──────────────┬───────────────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────────────┐
│  后续工具执行（受 Hooks 拦截）                         │
│                                                      │
│  Claude 决定执行 Bash(git commit -m "...")            │
│      │                                               │
│      ▼                                               │
│  PreToolUse Hook 触发：                               │
│      │                                               │
│      ├─ matcher "Bash(git commit*)" 匹配成功         │
│      ├─ 执行 pre-commit-checks.sh                    │
│      │   ├─ CLAUDE_PLUGIN_ROOT 指向技能目录           │
│      │   ├─ 运行 lint + typecheck + test + secret    │
│      │   └─ 成功 → 允许继续                          │
│      │                                               │
│      ▼                                               │
│  BashTool.call() 实际执行 git commit                  │
│      │                                               │
│      ▼                                               │
│  PostToolUse Hook 触发（如果有的话）                   │
│      │                                               │
│      ▼                                               │
│  返回结果给 Claude                                    │
└──────────────────────────────────────────────────────┘
```

---

## 七、安全机制总结

| 安全层 | 机制 | 说明 |
|--------|------|------|
| **Shell 注入权限** | `hasPermissionsToUseTool()` | 每个 `!`cmd`` 都经过 BashTool 权限检查 |
| **MCP 隔离** | `loadedFrom !== 'mcp'` 检查 | 远程技能永远不执行 Shell 命令 |
| **Shell 选择** | frontmatter `shell` 字段 | 作者决定，不可被读者设置覆盖 |
| **Hook 权限** | Tool Pipeline 标准权限流程 | Hook 产生的工具调用也走 10 步管线 |
| **环境变量** | `CLAUDE_PLUGIN_ROOT` 自动设置 | 限制脚本访问范围为技能目录 |
| **超时保护** | `timeout` 字段 | 防止脚本无限运行 |
| **一次性 Hook** | `once: true` + `onHookSuccess` | 自动清理，不累积 |
| **异步 Hook** | `async: true` | 不阻塞主流程 |
| **条件匹配** | `if` / `matcher` 字段 | 只在匹配时触发，减少不必要的执行 |
| **输出持久化** | `processToolResultBlock()` | 超大输出写入磁盘，不撑爆内存 |
