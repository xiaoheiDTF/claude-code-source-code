# Agent 专属 Hooks: PreToolUse（Agent 内工具调用前）

> 指南 5.3「Agent 可用的 Hook 事件」— Agent 内部的 PreToolUse 钩子

## 目录结构

```
my-project/
├── .claude/
│   └── agents/
│       ├── reviewer.md             ← 只读审查 Agent（阻止写入）
│       ├── safe-dev.md             ← 安全开发 Agent（阻止危险命令）
│       ├── template-dev.md         ← 模板开发 Agent（强制目录规范）
│       └── data-agent.md           ← 数据操作 Agent（表级别权限）
├── src/
└── package.json
```

## Agent 文件内容

```markdown
# 文件: .claude/agents/reviewer.md

---
description: Read-only code reviewer
tools:
  - Read
  - Grep
  - Glob
  - Bash
disallowedTools:
  - Write
  - Edit
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: |
            INPUT=$(cat)
            CMD=$(echo "$INPUT" | jq -r '.tool_input.command // empty')
            if echo "$CMD" | grep -qiE 'rm|drop|delete|truncate'; then
              echo '{"permissionDecision": "deny", "permissionDecisionReason": "审查Agent禁止执行破坏性命令"}'
            fi
---

You are a code reviewer. Analyze code but never modify it.
```

## 说明

```
Agent 内 PreToolUse:
  触发时机: Agent 内部任何工具调用前
  matcher: 匹配工具名

  与全局 PreToolUse 的关系:
    1. 全局 PreToolUse 先执行
    2. Agent 内 PreToolUse 后执行
    3. 任一返回 deny → 操作被拒绝

  典型用途:
    - 强制只读（deny Write/Edit）
    - 阻止危险命令
    - 强制目录/文件规范
    - 数据库表级别权限控制

  独特价值:
    只对该 Agent 生效，其他 Agent 和主 session 不受影响
```

## 使用示例

### 示例 1：审查 Agent 阻止任何修改操作

```
Agent: .claude/agents/reviewer.md
  PreToolUse → matcher: "Edit|Write" → deny

hooks:
  PreToolUse:
    - matcher: "Edit|Write"
      hooks:
        - type: command
          command: "echo '{\"permissionDecision\": \"deny\", \"permissionDecisionReason\": \"审查Agent不允许修改文件\"}'"

→ 用户: "用 reviewer 审查 auth 模块"

→ Agent 运行中:
  Read src/auth.ts → PreToolUse → 不匹配 Edit|Write → 允许 ✅
  Grep "TODO" → 不匹配 → 允许 ✅
  Glob "src/**/*.ts" → 不匹配 → 允许 ✅

→ Agent 试图修复问题:
  Edit src/auth.ts → PreToolUse 匹配 → deny ❌
  → "审查Agent不允许修改文件"

→ Agent 调整行为:
  "发现 3 个问题，但我是只读审查模式，不能直接修改。
   建议修复方案: ..."

效果:
  → 即使 Agent 有 Edit/Write 工具，Hook 也能阻止
  → 双重保障: disallowedTools + PreToolUse deny
  → 确保 Agent 严格只读
```

### 示例 2：安全开发 Agent 拦截危险 Bash 命令

```
Agent: .claude/agents/safe-dev.md
  PreToolUse → matcher: "Bash" → 检查命令安全性

hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: |
            INPUT=$(cat)
            CMD=$(echo "$INPUT" | jq -r '.tool_input.command // empty'
            # 危险命令黑名单
            if echo "$CMD" | grep -qiE 'rm\s+-rf|DROP\s+TABLE|truncate|format'; then
              echo '{"permissionDecision": "deny", "permissionDecisionReason": "危险命令被拦截"}'
            elif echo "$CMD" | grep -qiE 'npm\s+publish|docker\s+push'; then
              echo '{"permissionDecision": "ask", "permissionDecisionReason": "发布操作需确认"}'
            else
              echo '{"permissionDecision": "allow"}'
            fi

→ Agent 运行:
  Bash: "npm test" → allow ✅
  Bash: "git diff --stat" → allow ✅
  Bash: "rm -rf node_modules" → deny ❌ "危险命令被拦截"
  Bash: "npm publish" → ask ⚠️ "发布操作需确认"

效果:
  → Agent 的 Bash 命令被逐条检查
  → 三级控制: allow / ask / deny
  → 防止 Agent 执行不可逆操作
```

### 示例 3：模板开发 Agent 强制文件放入指定目录

```
Agent: .claude/agents/template-dev.md
  PreToolUse → matcher: "Write" → 检查路径规范

hooks:
  PreToolUse:
    - matcher: "Write"
      hooks:
        - type: command
          command: |
            INPUT=$(cat)
            FILE=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')
            # 新组件必须放在 src/components/v2/
            if echo "$FILE" | grep -q "^src/components/" && \
               ! echo "$FILE" | grep -q "^src/components/v2/"; then
              NEW_PATH=$(echo "$FILE" | sed 's|src/components/|src/components/v2/|')
              echo "{\"permissionDecision\": \"allow\",
                \"updatedInput\": $(echo "$INPUT" | jq --arg p "$NEW_PATH" '.tool_input.file_path = $p')}"
            else
              echo '{"permissionDecision": "allow"}'
            fi

→ Agent 运行:
  Write: "src/components/Button.tsx" → updatedInput → 实际写入 "src/components/v2/Button.tsx"
  Write: "src/components/Card.tsx" → updatedInput → 实际写入 "src/components/v2/Card.tsx"
  Write: "src/utils/helpers.ts" → 不匹配组件路径 → allow（不修改）

效果:
  → updatedInput 自动重定向文件路径
  → Agent 不知道路径被修改了
  → 强制执行目录规范
```

### 示例 4：数据操作 Agent 的表级别权限控制

```
Agent: .claude/agents/data-agent.md
  PreToolUse → matcher: "Bash" → 检查 SQL 表权限

hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: |
            INPUT=$(cat)
            CMD=$(echo "$INPUT" | jq -r '.tool_input.command // empty'
            # 检查是否涉及敏感表
            SENSITIVE_TABLES="users payments credentials audit_logs"
            for TABLE in $SENSITIVE_TABLES; do
              if echo "$CMD" | grep -qi "$TABLE"; then
                echo "{\"permissionDecision\": \"ask\",
                  \"permissionDecisionReason\": \"涉及敏感表: $TABLE — 需要确认\"}"
                exit 0
              fi
            done
            echo '{"permissionDecision": "allow"}'

→ Agent 运行:
  Bash: "SELECT * FROM products" → allow ✅
  Bash: "SELECT * FROM orders WHERE date > '2026-01-01'" → allow ✅
  Bash: "SELECT email FROM users" → ask ⚠️ "涉及敏感表: users"
  Bash: "DELETE FROM audit_logs WHERE date < '2025-01-01'" → ask ⚠️ "涉及敏感表: audit_logs"

效果:
  → 敏感表操作需要人工确认
  → 非敏感表自动放行
  → 数据安全级别的精细控制
```

### 示例 5：Agent 每次工具调用前记录操作日志

```
Agent: .claude/agents/dev.md
  PreToolUse → matcher: "" → 记录所有工具调用

hooks:
  PreToolUse:
    - matcher: ""
      hooks:
        - type: command
          command: |
            INPUT=$(cat)
            TOOL=$(echo "$INPUT" | jq -r '.tool_name')
            FILE=$(echo "$INPUT" | jq -r '.tool_input.file_path // .tool_input.path // .tool_input.command // "N/A"')
            TIMESTAMP=$(date '+%H:%M:%S')
            echo "$TIMESTAMP | $TOOL | $FILE" >> .claude/agent-operations.log
            echo '{"permissionDecision": "allow"}'

→ Agent 运行（执行 10 个工具调用）:
  14:31:01 | Read  | src/app.ts
  14:31:03 | Grep  | TODO
  14:31:05 | Edit  | src/app.ts
  14:31:08 | Bash  | npm test
  14:31:15 | Write | src/utils/new.ts
  ...

→ Agent 停止后，.claude/agent-operations.log 完整记录所有操作

效果:
  → Agent 的每一步操作都被记录
  → 方便调试 Agent 行为
  → 审计追踪: Agent 做了什么、什么顺序
  → matcher: "" 匹配所有工具
```
