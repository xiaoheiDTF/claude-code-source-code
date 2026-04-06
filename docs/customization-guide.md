# Claude Code 用户自定义完全指南

> 基于源码分析，涵盖所有用户可自定义的功能：Rules、Agents、Hooks、Skills、Settings。

---

## 目录

- [一、文件位置总览](#一文件位置总览)
- [二、Rules（规则文件）](#二rules规则文件)
- [三、Agents（自定义 Agent）](#三agents自定义-agent)
- [四、Hooks（生命周期钩子）](#四hooks生命周期钩子)
- [五、Agent 专属 Hooks](#五agent-专属-hooks)
- [六、Skills（技能/斜杠命令）](#六skills技能斜杠命令)
- [七、settings.json 配置](#七settingsjson-配置)
- [八、Memory（记忆系统）](#八memory记忆系统)
- [九、环境变量](#九环境变量)
- [十、用户可主动使用的方法](#十用户可主动使用的方法)

---

## 一、文件位置总览

```
用户全局目录 (~/.claude/)
├── CLAUDE.md                          全局个人指令（所有项目生效）
├── settings.json                      全局配置（hooks、权限、MCP 等）
├── rules/                             全局规则目录
│   ├── coding-style.md                无条件规则（始终生效）
│   └── python.md                      无条件规则
├── agents/                            全局自定义 Agent
│   ├── code-reviewer.md               自定义 Agent 定义
│   └── test-writer.md
├── commands/                          全局斜杠命令（Skills）
│   └── commit.md                      /commit 命令
├── skills/                            全局技能
│   └── refactor.md
├── agent-memory/                      Agent 持久化记忆（user scope）
│   └── code-reviewer/MEMORY.md
└── memory/                            用户自动记忆（auto memory）

项目目录 (<project>/)
├── CLAUDE.md                          项目全局指令（团队共享，提交 VCS）
├── CLAUDE.local.md                    项目私有指令（不提交 VCS）
├── .claude/
│   ├── settings.json                  项目级配置
│   ├── settings.local.json            项目私有配置（不提交 VCS）
│   ├── rules/                         项目级规则
│   │   ├── general.md                 无条件规则
│   │   ├── frontend.md                条件规则（paths 匹配）
│   │   ├── backend.md                 条件规则
│   │   └── testing.md                 条件规则
│   ├── agents/                        项目级自定义 Agent
│   │   ├── db-admin.md
│   │   └── api-designer.md
│   ├── commands/                      项目级斜杠命令
│   ├── skills/                        项目级技能
│   └── hooks/                         Hook 脚本文件
│       ├── pre-edit.sh
│       └── post-edit.sh
├── apps/
│   ├── web/
│   │   ├── CLAUDE.md                  子目录指令（按需加载）
│   │   ├── .claude/rules/             子目录级规则
│   │   └── src/
│   │       └── components/
│   │           └── CLAUDE.md          更深层指令
│   └── api/
│       └── CLAUDE.md
└── services/
    ├── auth/CLAUDE.md
    └── order/CLAUDE.md
```

### 优先级规则

```
加载顺序（从低到高）        优先级
─────────────────────────────────
Managed (/etc/claude-code/)    1  最低
User (~/.claude/)              2
Project (<project>/CLAUDE.md)  3
子目录 CLAUDE.md               4  越深越高
Local (CLAUDE.local.md)        5  最高

后加载的优先级更高，模型更关注后面的内容。
同名 Agent：后注册的覆盖先注册的。
```

---

## 二、Rules（规则文件）

### 2.1 两种规则

| 类型 | 有 `paths` frontmatter | 加载时机 | 适用场景 |
|------|----------------------|---------|---------|
| 无条件规则 | 无 | 启动时全量加载 | 通用规范 |
| 条件规则 | 有 | 操作匹配文件时按需注入 | 特定场景规范 |

### 2.2 文件位置

```bash
~/.claude/rules/*.md                    # 全局规则（所有项目）
<project>/.claude/rules/*.md            # 项目规则
<project>/<subdir>/.claude/rules/*.md   # 子目录规则
```

### 2.3 无条件规则

无 `paths` 的规则文件，启动时始终加载。

```markdown
# 文件: .claude/rules/coding-standards.md

- TypeScript strict mode
- 禁止使用 any
- 所有函数必须有返回类型
- 错误必须处理，不允许空 catch
```

### 2.4 条件规则

通过 frontmatter 的 `paths` 指定触发条件。paths 使用 `.gitignore` 语法。

**YAML 列表写法（推荐）**：

```yaml
---
paths:
  - "frontend/**"
  - "packages/ui/**"
---

# 前端开发规范
- React 18 + TypeScript
- 组件库: shadcn/ui
- 样式: Tailwind CSS
```

**逗号分隔写法**：

```yaml
---
paths: "frontend/**, packages/ui/**"
---

前端规范...
```

**花括号展开**：

```yaml
---
paths: "src/*.{ts,tsx}"
---
# 匹配 src/*.ts 和 src/*.tsx
```

```yaml
---
paths: "{apps/web,apps/mobile}/src/**"
---
# 匹配多个应用
```

**注意**：
- `paths` 相对于**该 .md 文件所在的 .claude 目录的父目录**
- `src/**` 等价于 `src`（`/**` 后缀会自动去掉）
- `**` 表示匹配所有（等同于无条件规则）
- 当 agent 使用 Read/Edit/Write 操作匹配文件时自动触发

### 2.5 @include 指令

CLAUDE.md 和 rules 文件支持 `@` 引用其他文件：

```markdown
<!-- 引用相对路径 -->
详细 API 文档见 @./docs/api-guide.md

<!-- 引用项目内路径 -->
共享规范见 @./.claude/rules/shared.md

<!-- 引用绝对路径 -->
@~/shared-configs/base-rules.md
```

支持的格式：`.md` `.txt` `.json` `.yaml` `.ts` `.py` `.sql` 等 60+ 文本格式。
最大嵌套深度：5 层。

### 2.6 典型项目 Rules 配置

```
.claude/rules/
├── coding-standards.md          无 paths → 始终生效
├── commit-convention.md         无 paths → 始终生效
├── frontend.md                  paths: "apps/web/**"
├── backend.md                   paths: "apps/api/**"
├── database.md                  paths: "**/db/**","**/prisma/**"
├── testing.md                   paths: "**/*.test.*","**/*.spec.*"
├── security.md                  paths: "**/auth/**","**/middleware/**"
└── infra.md                     paths: "docker-compose.*","Dockerfile*","k8s/**"
```

---

## 三、Agents（自定义 Agent）

### 3.1 文件位置

```bash
~/.claude/agents/*.md              # 全局 Agent
<project>/.claude/agents/*.md      # 项目 Agent
```

### 3.2 Agent 完整定义

文件名即 Agent 类型名（不含 `.md`）。

```markdown
---
# ============ 必填字段 ============
description: Code review specialist that checks security, performance and code quality

# ============ 工具控制 ============
# tools 和 disallowedTools 二选一，也可以组合使用
tools:                             # 允许的工具白名单（可选，省略=全部）
  - Read
  - Grep
  - Glob
  - Bash
disallowedTools:                   # 禁止的工具黑名单（可选）
  - Write
  - Edit
  - Agent

# ============ 模型控制 ============
model: sonnet                      # sonnet | opus | haiku | inherit（默认 inherit）
effort: high                       # high | medium | low

# ============ 权限控制 ============
permissionMode: acceptEdits        # 权限模式（可选）
maxTurns: 30                       # 最大对话轮次（可选）

# ============ 持久化记忆 ============
memory: project                    # user | project | local（可选）

# ============ MCP 服务器 ============
mcpServers:                        # Agent 专属 MCP 服务器（可选）
  - "my-database"                  # 引用已配置的 MCP
  - reviewTools:                   # 内联定义 MCP
      command: "npx my-review-tools"

# ============ 预加载 Skill ============
skills:                            # 预加载的 skill（可选）
  - "commit"
  - "review"

# ============ 生命周期 Hooks ============
hooks:                             # Agent 专属 hooks（可选，见第五节）
  SubagentStop:
    - matcher: ""
      hooks:
        - type: command
          command: "bash .claude/hooks/after-review.sh"

# ============ 运行模式 ============
background: true                   # 始终作为后台任务运行（可选）
isolation: worktree                # worktree | remote（可选，在隔离环境中运行）

# ============ 其他 ============
initialPrompt: "/review"           # 首轮对话前缀（可选，支持斜杠命令）
omitClaudeMd: false                # 是否省略 CLAUDE.md（可选，只读 agent 建议 true）
requiredMcpServers:                # 必须可用的 MCP（可选，不可用则报错）
  - "database"
---

You are a code review specialist...

## Review checklist
1. Security: XSS, SQL injection, secrets in code
2. Performance: N+1 queries, unnecessary re-renders
3. Error handling: unhandled promises, empty catches

## Output format
- Severity: Critical / Warning / Info
- File: path:line
- Issue: description
- Fix: suggested code change
```

### 3.3 字段详解

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `description` | string | **是** | 一句话描述，告诉主 Agent 何时使用 |
| `tools` | string[] | 否 | 工具白名单，`["*"]` 或省略=全部 |
| `disallowedTools` | string[] | 否 | 工具黑名单，优先级高于 tools |
| `model` | string | 否 | `sonnet` / `opus` / `haiku` / `inherit`（默认 inherit） |
| `effort` | string | 否 | `high` / `medium` / `low` |
| `permissionMode` | string | 否 | 权限模式 |
| `maxTurns` | number | 否 | 最大对话轮次限制 |
| `memory` | string | 否 | `user` / `project` / `local` |
| `mcpServers` | array | 否 | Agent 专属 MCP 服务器配置 |
| `skills` | string[] | 否 | 启动时预加载的 skill |
| `hooks` | object | 否 | Agent 生命周期 hooks（见第五节） |
| `background` | boolean | 否 | `true` = 始终后台运行 |
| `isolation` | string | 否 | `worktree` = 在 git worktree 中隔离运行 |
| `initialPrompt` | string | 否 | 首轮用户消息前缀（支持 /命令） |
| `omitClaudeMd` | boolean | 否 | `true` = 不加载 CLAUDE.md（省 token） |
| `requiredMcpServers` | string[] | 否 | 必须可用的 MCP，不可用时 Agent 不可用 |
| `color` | string | 否 | Agent 在 UI 中的显示颜色 |

### 3.4 实用 Agent 示例

**代码审查 Agent**（`.claude/agents/code-reviewer.md`）：

```markdown
---
description: Independent code review for security and quality
disallowedTools:
  - Write
  - Edit
  - NotebookEdit
model: sonnet
omitClaudeMd: true
memory: project
---

You are a code review specialist. You ONLY review code, never modify it.

## Review checklist
1. **Security**: XSS, SQL injection, hardcoded secrets, CSRF
2. **Performance**: N+1 queries, missing indexes, memory leaks
3. **Error handling**: unhandled promises, empty catch blocks
4. **TypeScript**: any usage, missing types, unsafe casts
5. **Testing**: missing edge case tests, flaky tests

## Output format
For each issue found:
- **Severity**: Critical / Warning / Info
- **File**: `path:line`
- **Issue**: one-line description
- **Fix**: suggested code (in a code block)

End with a summary: total issues by severity.
```

**数据库管理 Agent**（`.claude/agents/db-admin.md`）：

```markdown
---
description: Database schema management and migration specialist
tools:
  - Read
  - Grep
  - Glob
  - Bash
model: sonnet
permissionMode: acceptEdits
maxTurns: 20
memory: project
---

You manage database schemas and migrations.

## Rules
- Always read the current schema before suggesting changes
- Migration files must be reversible (include down migration)
- Never drop columns without a deprecation period
- All new columns must have sensible defaults
- Use transactions for data migrations

## Workflow
1. Read prisma/schema.prisma to understand current state
2. Analyze the requested change
3. Create migration with both up and down
4. Verify migration SQL before presenting
```

**测试编写 Agent**（`.claude/agents/test-writer.md`）：

```markdown
---
description: Write comprehensive tests for modified code
tools:
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - Bash
model: sonnet
effort: high
---

You write tests for code changes.

## Rules
- Use Vitest as the testing framework
- Test file: <original>.test.ts, placed next to the source
- Cover: happy path, edge cases, error cases
- Use describe/it blocks, descriptive test names
- Mock external dependencies, not internal modules
- Aim for 80%+ coverage on new code
- Do NOT modify source files, only create/edit test files
```

### 3.5 Agent 的工具池

Agent 启动时，工具池按以下规则组装：

```
1. 取完整工具池（assembleToolPool）
2. 过滤 ALL_AGENT_DISALLOWED_TOOLS（所有 agent 都禁用的）
3. 如果是自定义 agent，过滤 CUSTOM_AGENT_DISALLOWED_TOOLS
4. 如果是后台 agent，只保留 ASYNC_AGENT_ALLOWED_TOOLS
5. 如果 tools 字段有白名单，只保留白名单内的
6. 如果 disallowedTools 有黑名单，移除黑名单内的
```

---

## 四、Hooks（生命周期钩子）

### 4.1 配置位置

```bash
~/.claude/settings.json              # 全局 hooks
<project>/.claude/settings.json      # 项目 hooks
<project>/.claude/settings.local.json # 项目私有 hooks
```

### 4.2 所有 Hook 事件

| 事件 | 触发时机 | matcher 匹配对象 |
|------|---------|----------------|
| `PreToolUse` | 工具执行前 | 工具名（如 `Edit`、`Write`） |
| `PostToolUse` | 工具执行成功后 | 工具名 |
| `PostToolUseFailure` | 工具执行失败后 | 工具名 |
| `UserPromptSubmit` | 用户提交消息时 | 来源（如 `api`、`cli`） |
| `SessionStart` | 会话启动时 | 来源 |
| `SessionEnd` | 会话结束时 | 退出原因 |
| `Stop` | Agent 停止时 | — |
| `SubagentStart` | 子 Agent 启动时 | — |
| `SubagentStop` | 子 Agent 停止时 | — |
| `PreCompact` | 上下文压缩前 | — |
| `PostCompact` | 上下文压缩后 | — |
| `Notification` | 收到通知时 | — |
| `Setup` | 设置初始化时 | 触发器 |
| `TeammateIdle` | Teammate 空闲时 | — |
| `TaskCreated` | 任务创建时 | — |
| `TaskCompleted` | 任务完成时 | — |
| `ConfigChange` | 配置变更时 | — |
| `InstructionsLoaded` | 指令文件加载时 | — |
| `FileChanged` | 文件变更时 | — |

### 4.3 Hook 类型

#### Command Hook（执行 shell 命令）

```json
{
  "type": "command",
  "command": "bash .claude/hooks/my-script.sh",
  "shell": "bash",
  "timeout": 30,
  "statusMessage": "Running pre-check...",
  "once": false,
  "async": false,
  "asyncRewake": false
}
```

| 字段 | 说明 |
|------|------|
| `command` | Shell 命令 |
| `shell` | `bash` 或 `powershell`（默认 bash） |
| `timeout` | 超时秒数 |
| `statusMessage` | 运行时显示的状态文字 |
| `once` | `true` = 只运行一次后自动移除 |
| `async` | `true` = 后台运行，不阻塞主流程 |
| `asyncRewake` | `true` = 后台完成后重新唤醒 agent |

#### Prompt Hook（动态注入提示词）

```json
{
  "type": "prompt",
  "prompt": "Review the last change for security issues."
}
```

#### Agent Hook（启动子 Agent 执行任务）

```json
{
  "type": "agent",
  "agentType": "code-reviewer",
  "prompt": "Review the file that was just modified: {{file_path}}"
}
```

#### HTTP Hook（调用外部 API）

```json
{
  "type": "http",
  "url": "https://api.example.com/webhook",
  "method": "POST",
  "headers": {
    "Authorization": "Bearer $TOKEN"
  }
}
```

### 4.4 Hook 输入（通过 stdin 传入 JSON）

**PreToolUse / PostToolUse**：

```json
{
  "hook_event_name": "PreToolUse",
  "tool_name": "Edit",
  "tool_input": {
    "file_path": "/project/src/app.ts",
    "old_string": "...",
    "new_string": "..."
  },
  "session_id": "session-uuid",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/project"
}
```

**PostToolUse 额外字段**：

```json
{
  "tool_output": "The file has been edited successfully",
  "tool_result": { ... }
}
```

### 4.5 Hook 输出（通过 stdout 返回 JSON）

**PreToolUse 可返回**：

```json
{
  "permissionDecision": "allow",
  "permissionDecisionReason": "Auto-approved for this file type",
  "updatedInput": {
    "file_path": "/modified/path"
  }
}
```

- `permissionDecision`: `"allow"` / `"deny"` / `"ask"` — 控制是否允许执行
- `updatedInput`: 修改工具的输入参数

**PostToolUse 可返回**：

```json
{
  "additionalContext": "已自动执行 git add，变更已暂存"
}
```

- `additionalContext`: 注入给 Claude 的额外上下文信息

### 4.6 settings.json 完整示例

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/pre-edit.sh",
            "statusMessage": "Checking git status..."
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/post-edit.sh",
            "async": true,
            "statusMessage": "Auto-staging changes..."
          }
        ]
      },
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/post-bash.sh"
          }
        ]
      }
    ],
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/on-stop.sh",
            "async": true
          }
        ]
      }
    ]
  }
}
```

### 4.7 Hook 脚本示例

**编辑前检查**（`.claude/hooks/pre-edit.sh`）：

```bash
#!/bin/bash
# 从 stdin 读取 hook 输入
INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // .tool_input.path // empty')

if [ -z "$FILE_PATH" ]; then
  exit 0
fi

# 检查是否有未提交的变更
if ! git diff --quiet "$FILE_PATH" 2>/dev/null; then
  echo "Warning: $FILE_PATH has uncommitted changes"
fi

# 输出 JSON 控制（可选）
# echo '{"permissionDecision": "allow"}'
```

**编辑后自动暂存**（`.claude/hooks/post-edit.sh`）：

```bash
#!/bin/bash
INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // .tool_input.path // empty')

if [ -n "$FILE_PATH" ] && [ -f "$FILE_PATH" ]; then
  git add "$FILE_PATH" 2>/dev/null
  echo "{\"additionalContext\": \"Auto-staged: $FILE_PATH\"}"
fi
```

---

## 五、Agent 专属 Hooks

### 5.1 原理

Agent 可以在 frontmatter 中定义自己的 hooks。这些 hooks 在 Agent 启动时注册，Agent 结束时自动清理。

关键机制（源码 `registerFrontmatterHooks.ts`）：
- Agent 的 `Stop` hook 会自动转换为 `SubagentStop`
- hooks 作用域限定在该 Agent 的 session 内
- Agent 结束后 hooks 自动注销

### 5.2 在 Agent frontmatter 中定义 hooks

```markdown
---
description: Code reviewer with automated follow-up
disallowedTools:
  - Write
  - Edit
hooks:
  SubagentStop:
    - matcher: ""
      hooks:
        - type: command
          command: "bash .claude/hooks/after-review.sh"
          async: true
  SubagentStart:
    - matcher: ""
      hooks:
        - type: command
          command: "bash .claude/hooks/before-review.sh"
---

You are a code review specialist...
```

### 5.3 Agent 可用的 Hook 事件

| 事件 | 说明 |
|------|------|
| `SubagentStart` | Agent 启动时触发 |
| `SubagentStop` | Agent 完成时触发 |
| `PreToolUse` | Agent 内工具调用前触发 |
| `PostToolUse` | Agent 内工具调用后触发 |
| `PreCompact` | Agent 上下文压缩前触发 |
| `Notification` | Agent 收到通知时触发 |

### 5.4 实用场景

**场景 1：Agent 完成后自动提交**

```markdown
---
description: Feature implementer
tools:
  - Read
  - Write
  - Edit
  - Bash
  - Grep
  - Glob
hooks:
  SubagentStop:
    - matcher: ""
      hooks:
        - type: command
          command: |
            FILES=$(git diff --name-only)
            if [ -n "$FILES" ]; then
              git add -A
              git commit -m "feat: automated implementation"
              echo '{"additionalContext": "Changes auto-committed"}'
            fi
          async: true
---

Implement features based on the specification...
```

**场景 2：Agent 每次编辑后自动 lint**

```markdown
---
description: TypeScript developer
tools:
  - Read
  - Write
  - Edit
  - Bash
  - Grep
  - Glob
hooks:
  PostToolUse:
    - matcher: "Edit|Write"
      hooks:
        - type: command
          command: |
            INPUT=$(cat)
            FILE=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')
            if [[ "$FILE" == *.ts || "$FILE" == *.tsx ]]; then
              npx eslint --fix "$FILE" 2>/dev/null
              echo "{\"additionalContext\": \"Linted: $FILE\"}"
            fi
          async: true
---

You write TypeScript code...
```

**场景 3：Agent 启动时加载项目上下文**

```markdown
---
description: Database migration specialist
tools:
  - Read
  - Write
  - Edit
  - Bash
hooks:
  SubagentStart:
    - matcher: ""
      hooks:
        - type: command
          command: |
            echo '{"additionalContext": "Current schema version: '"$(cat prisma/schema.prisma | grep '// version:' | head -1)"'"}'
---

You handle database migrations...
```

---

## 六、Skills（技能/斜杠命令）

### 6.1 文件位置

```bash
~/.claude/commands/*.md              # 全局斜杠命令
~/.claude/skills/*.md                # 全局技能
<project>/.claude/commands/*.md      # 项目斜杠命令
<project>/.claude/skills/*.md        # 项目技能
```

### 6.2 Skill 定义

```markdown
---
description: Generate a commit with conventional commit message
argument-hint: "[optional scope] commit description"
allowed-tools:
  - Bash
  - Read
agent: general-purpose
paths:
  - "**/*.ts"
  - "**/*.tsx"
shell: bash
---

Based on the current git diff, generate a conventional commit message
and commit the changes.

Rules:
- Use Conventional Commits format: type(scope): description
- Types: feat, fix, refactor, docs, test, chore, perf
- Scope is optional
- Description in imperative mood, lowercase, no period
- If there are breaking changes, add BREAKING CHANGE in footer

$ARGUMENTS
```

| 字段 | 说明 |
|------|------|
| `description` | 命令描述 |
| `argument-hint` | 参数提示 |
| `allowed-tools` | 执行时额外允许的工具 |
| `agent` | 使用的 agent 类型（默认 general-purpose） |
| `paths` | 条件匹配（同 rules 的 paths） |
| `shell` | `bash` 或 `powershell` |

`$ARGUMENTS` 会被替换为用户输入的参数。

### 6.3 调用方式

```
/commit                           # 调用 ~/.claude/commands/commit.md
/commit fix login bug             # 带参数调用
```

---

## 七、settings.json 配置

### 7.1 配置文件位置和优先级

```
~/.claude/settings.json              用户全局配置
/etc/claude-code/settings.json       系统管理配置
<project>/.claude/settings.json      项目共享配置
<project>/.claude/settings.local.json 项目私有配置
```

### 7.2 主要配置项

```json
{
  "permissions": {
    "allow": [
      "Bash(git log*)",
      "Bash(git diff*)",
      "Bash(git status*)",
      "Bash(npm test*)",
      "Read",
      "Glob",
      "Grep"
    ],
    "deny": [
      "Bash(rm -rf*)",
      "Bash(curl*|*)"
    ],
    "defaultMode": "default",
    "additionalDirectories": ["/other/project"]
  },

  "hooks": {
    "PreToolUse": [...],
    "PostToolUse": [...]
  },

  "mcpServers": {
    "my-server": {
      "command": "npx",
      "args": ["-y", "my-mcp-server"],
      "env": {}
    }
  },

  "agentModel": "inherit",
  "theme": "dark",
  "verbose": true
}
```

---

## 八、Memory（记忆系统）

### 8.1 三种记忆

| 类型 | 路径 | 作用范围 |
|------|------|---------|
| Auto Memory | `memory/MEMORY.md` | 当前用户，自动积累 |
| Agent Memory | `agent-memory/<agentType>/MEMORY.md` | 特定 Agent 类型，跨项目 |
| 项目 Memory | `CLAUDE.md` / `.claude/rules/` | 项目级，团队共享 |

### 8.2 Agent Memory

当 Agent 定义了 `memory` 字段时，该 Agent 会有持久化的记忆目录：

| scope | 路径 | 说明 |
|-------|------|------|
| `user` | `~/.claude/agent-memory/<agentType>/MEMORY.md` | 跨项目共享 |
| `project` | `.claude/agent-memory/<agentType>/MEMORY.md` | 项目级，可提交 VCS |
| `local` | `.claude/agent-memory-local/<agentType>/MEMORY.md` | 本地，不提交 |

Agent 在运行时可以使用 Write 工具写入自己的 memory 目录，下次启动同类型 Agent 时会自动加载。

---

## 九、环境变量

| 变量 | 说明 |
|------|------|
| `CLAUDE_CODE_SUBAGENT_MODEL` | 强制所有子 agent 使用指定模型 |
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` | 禁用后台任务 |
| `CLAUDE_AUTO_BACKGROUND_TASKS` | 启用自动后台 agent |
| `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD` | 允许 --add-dir 目录加载 CLAUDE.md |
| `CLAUDE_CODE_AGENT_LIST_IN_MESSAGES` | Agent 列表通过消息注入（省 cache） |
| `CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS` | SDK 模式下禁用内置 Agent |
| `CLAUDE_CODE_COORDINATOR_MODE` | 启用 Coordinator 模式 |

---

## 十、用户可主动使用的方法

### 10.1 斜杠命令（Slash Commands）

用户可以在对话中直接输入 `/命令名` 来调用：

| 命令 | 说明 | 示例 |
|------|------|------|
| `/commit` | 生成提交信息并提交代码 | `/commit`、`/commit fix login bug` |
| `/review` | 代码审查 | `/review`、`/review src/app.ts` |
| `/loop` | 设置定时循环任务 | `/loop 5m check the deploy` |
| `/config` | 查看或修改简单配置 | `/config` |
| `/update-config` | 修改 settings.json 配置 | `/update-config` |
| `/compact` | 手动压缩上下文 | `/compact` |
| `/clear` | 清空对话历史 | `/clear` |
| `/help` | 查看帮助 | `/help` |
| `/memory` | 查看或管理记忆 | `/memory` |
| `/hooks` | 查看/管理 Hooks | `/hooks` |
| `/status` | 查看当前状态 | `/status` |
| `/cost` | 查看 token 消耗 | `/cost` |
| `/model` | 切换模型 | `/model sonnet` |
| `/permissions` | 查看当前权限 | `/permissions` |
| `/init` | 初始化项目配置 | `/init` |
| `/remember` | 保存记忆 | `/remember 项目使用 Vitest` |
| `/fast` | 切换快速模式 | `/fast` |

### 10.2 主动配置方法

#### CLAUDE.md（项目/全局指令）

主动创建 `.md` 文件注入行为指令：

```
# 用户可以主动操作：
1. 编辑 CLAUDE.md              → 影响所有对话
2. 编辑 CLAUDE.local.md        → 仅本地，不提交
3. 编辑子目录 CLAUDE.md        → 子目录级指令
4. 使用 @include 引用外部文件  → 注入上下文
```

#### Rules（条件规则）

主动创建规则文件控制模型行为：

```bash
# 无条件规则（始终生效）
echo "- 使用 TypeScript strict mode" > .claude/rules/coding.md

# 条件规则（操作特定文件时生效）
cat > .claude/rules/frontend.md << 'EOF'
---
paths:
  - "apps/web/**"
  - "packages/ui/**"
---
- React 18 + TypeScript
- 样式: Tailwind CSS
EOF
```

#### Agents（自定义 Agent）

主动创建 Agent 定义文件：

```bash
# 创建专用 Agent
cat > .claude/agents/reviewer.md << 'EOF'
---
description: Code review specialist
disallowedTools:
  - Write
  - Edit
model: sonnet
---
You are a code reviewer...
EOF
```

使用方式：在对话中让 Claude 调用 `Agent` 工具并指定 `agentType: "reviewer"`。

#### Skills/Commands（自定义技能）

主动创建斜杠命令：

```bash
# 方式 1：单文件命令（commands/）
cat > .claude/commands/deploy.md << 'EOF'
---
description: Deploy the project
allowed-tools:
  - Bash
---
Deploy the project to staging environment.

Steps:
1. Run tests: ```! npm test ```
2. Build: ```! npm run build ```
3. Deploy: ```! npm run deploy:staging ```

$ARGUMENTS
EOF

# 方式 2：目录式技能（skills/）—— 支持附带脚本
mkdir -p .claude/skills/analyze
cat > .claude/skills/analyze/SKILL.md << 'EOF'
---
description: Analyze code metrics
allowed-tools:
  - Bash
  - Read
---
Run analysis scripts:

```! python3 $CLAUDE_SKILL_DIR/analyze.py $ARGUMENTS ```
EOF

# 附带 Python/Shell 脚本
cat > .claude/skills/analyze/analyze.py << 'EOF'
import sys
# ... 分析逻辑
EOF
```

### 10.3 主动运行时操作

#### Shell 命令嵌入（Skill 内使用）

在 Skill 的 `.md` 文件中嵌入可执行命令：

```
# 代码块语法（执行 shell 命令并替换输出）
```! npm run build ```

# 内联语法（单行命令）
当前版本: !`cat package.json | jq -r .version`
```

#### 变量替换（Skill 内使用）

| 变量 | 说明 | 示例值 |
|------|------|--------|
| `$ARGUMENTS` | 用户传入的参数 | `fix login bug` |
| `${CLAUDE_SKILL_DIR}` | 当前 Skill 的目录路径 | `/home/user/.claude/skills/commit` |
| `${CLAUDE_SESSION_ID}` | 当前会话 ID | `abc123-def456` |

#### Hooks（生命周期钩子）

主动配置自动化行为：

```bash
# 编辑 settings.json 添加 hooks
cat > .claude/settings.json << 'EOF'
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Write|Edit",
      "hooks": [{
        "type": "command",
        "command": "jq -r '.tool_input.file_path' | { read -r f; prettier --write \"$f\"; } 2>/dev/null || true"
      }]
    }]
  }
}
EOF
```

### 10.4 主动记忆操作

| 操作 | 方法 | 说明 |
|------|------|------|
| 保存记忆 | `/remember 内容` | 主动告诉 Claude 记住某些信息 |
| 查看记忆 | `/memory` | 查看当前保存的所有记忆 |
| 清除记忆 | 编辑 `memory/MEMORY.md` | 手动删除不再需要的记忆条目 |
| Agent 记忆 | Agent 定义 `memory` 字段 | Agent 跨会话持久化记忆 |

### 10.5 权限管理

```bash
# 在 settings.json 中主动配置权限
{
  "permissions": {
    "allow": [
      "Bash(npm:*)",          # 允许所有 npm 命令
      "Bash(git:*)",          # 允许所有 git 命令
      "Read",                 # 允许读取文件
      "Glob",                 # 允许搜索文件
      "Grep"                  # 允许搜索内容
    ],
    "deny": [
      "Bash(rm -rf:*)"       # 禁止 rm -rf
    ],
    "defaultMode": "default"  # default | plan | acceptEdits | dontAsk
  }
}
```

### 10.6 方法速查表

| 想要做到 | 使用方法 | 配置位置 |
|---------|---------|---------|
| 每次对话都遵循的规范 | CLAUDE.md / Rules | `CLAUDE.md` / `.claude/rules/*.md` |
| 操作特定文件时注入规范 | 条件 Rules（paths） | `.claude/rules/*.md` + `paths` |
| 调用专用 Agent | Agent 定义 | `.claude/agents/*.md` |
| 创建自定义命令 | Skill / Command | `.claude/skills/*/SKILL.md` / `.claude/commands/*.md` |
| 自动化工具调用 | Hooks | `.claude/settings.json` → `hooks` |
| 保存跨会话信息 | Memory | `/remember` 或 `memory/MEMORY.md` |
| 控制模型可用工具 | Permissions | `.claude/settings.json` → `permissions` |
| 修改主题/模型等 | Config 工具或 settings.json | `/config` 或编辑配置文件 |
| 定时重复执行任务 | `/loop` | `/loop 5m /check-deploy` |
| 连接外部工具/服务 | MCP Server | `.claude/settings.json` → `mcpServers` |
| 设置环境变量 | Settings | `.claude/settings.json` → `env` |
| 命令行一次性运行 | `claude -p "提示词"` | 终端命令行 |
| 管道模式 | `cat file \| claude -p "分析"` | 终端命令行 |

---

## 附录：快速配置模板

### 最小项目配置

```bash
# 1. 创建项目指令
cat > CLAUDE.md << 'EOF'
# 项目概述
- 技术栈: [填写]
- 目录结构: [填写]
- 开发规范: [填写]
EOF

# 2. 创建条件规则
mkdir -p .claude/rules
cat > .claude/rules/testing.md << 'EOF'
---
paths: "**/*.test.*"
---
测试规范:
- 框架: Vitest
- 覆盖率 ≥ 80%
EOF
```

### 完整项目配置

```bash
# 1. 基础指令
touch CLAUDE.md                     # 项目全局规范

# 2. 规则
mkdir -p .claude/rules
touch .claude/rules/coding.md      # 通用编码规范
touch .claude/rules/frontend.md    # 前端条件规则
touch .claude/rules/backend.md     # 后端条件规则

# 3. 自定义 Agent
mkdir -p .claude/agents
touch .claude/agents/reviewer.md   # 代码审查 Agent
touch .claude/agents/tester.md     # 测试编写 Agent

# 4. Hooks 脚本
mkdir -p .claude/hooks
touch .claude/hooks/pre-edit.sh
touch .claude/hooks/post-edit.sh

# 5. 项目配置
touch .claude/settings.json         # hooks + 权限配置

# 6. 私有配置（不提交）
touch CLAUDE.local.md
touch .claude/settings.local.json

# 7. 为每个子项目添加 CLAUDE.md
for dir in apps/*/ services/*/; do
  if [ -f "$dir/README.md" ]; then
    echo '@./README.md' > "$dir/CLAUDE.md"
  fi
done
```
