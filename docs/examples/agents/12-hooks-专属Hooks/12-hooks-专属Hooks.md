# Agent 字段: hooks（Agent 专属生命周期钩子）

> 指南 3.2「生命周期 Hooks」+ 第五节「Agent 专属 Hooks」
> Agent 在 frontmatter 中定义自己的 hooks，启动时注册，结束时清理

## 目录结构

```
my-project/
├── .claude/
│   ├── agents/
│   │   └── auto-commit-dev.md       ← ★ 带 hooks 的 Agent
│   ├── hooks/
│   │   ├── after-impl.sh
│   │   └── before-test.sh
│   └── settings.json
├── src/
└── package.json
```

## Agent 文件内容

```markdown
# 文件: .claude/agents/auto-commit-dev.md

---
description: Feature implementer that auto-commits and auto-lints after each change.
tools:
  - Read
  - Write
  - Edit
  - Bash
  - Grep
  - Glob
model: sonnet
permissionMode: acceptEdits
hooks:                               ← Agent 专属 hooks
  SubagentStart:
    - matcher: ""
      hooks:
        - type: command
          command: "echo '{\"additionalContext\": \"Feature dev agent started\"}'"
  PostToolUse:
    - matcher: "Edit|Write"
      hooks:
        - type: command
          command: |
            INPUT=$(cat)
            FILE=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')
            if [ -n "$FILE" ]; then
              npx eslint --fix "$FILE" 2>/dev/null || true
              echo "{\"additionalContext\": \"Auto-linted: $FILE\"}"
            fi
          async: true
  SubagentStop:
    - matcher: ""
      hooks:
        - type: command
          command: |
            FILES=$(git diff --name-only)
            if [ -n "$FILES" ]; then
              git add -A
              git commit -m "feat: auto-implemented by agent"
              echo "{\"additionalContext\": \"Changes auto-committed\"}"
            fi
          async: true
---

You implement features. After each edit, code is auto-linted.
When done, changes are auto-committed.
```

## 说明

```
Agent hooks vs 全局 hooks:

  全局 hooks（settings.json）:
    → 所有 Agent 和主 Agent 都触发
    → 永久生效

  Agent hooks（frontmatter）:
    → 只在该 Agent 运行期间生效
    → Agent 启动时注册，结束时自动清理
    → Stop hook 自动转换为 SubagentStop

Agent 可用的 hook 事件:
  SubagentStart   → Agent 启动时
  SubagentStop    → Agent 完成时
  PreToolUse      → Agent 内工具调用前
  PostToolUse     → Agent 内工具调用后
  PreCompact      → 上下文压缩前
  Notification    → 收到通知时
```

## 使用示例

### 示例 1：自动 Lint + 自动提交的完整流程

```
用户: "实现用户注册 API"

→ Agent(auto-commit-dev)

Hook 生命周期:

  1. SubagentStart 触发:
     → "Feature dev agent started" 注入上下文

  2. Agent 执行 Edit("src/routes/users.ts"):
     → PostToolUse (matcher: Edit|Write) 触发:
       → npx eslint --fix src/routes/users.ts
       → "Auto-linted: src/routes/users.ts" 注入上下文

  3. Agent 执行 Edit("src/services/user-service.ts"):
     → PostToolUse 再次触发:
       → eslint --fix
       → "Auto-linted: src/services/user-service.ts"

  4. Agent 完成:
     → SubagentStop 触发:
       → git add -A
       → git commit -m "feat: auto-implemented by agent"
       → "Changes auto-committed"

  5. Agent hooks 自动清理（不再触发）

hooks 效果:
  → PostToolUse hook 在每次 Edit/Write 后自动运行 eslint --fix
  → SubagentStop hook 在 Agent 结束时自动提交所有更改
  → 全流程无人干预：写代码 → 自动 lint → 自动提交
```

### 示例 2：PostToolUse 钩子自动格式化 JSON 配置文件

```
用户: "更新 package.json 添加新的依赖，并修改 tsconfig.json"

→ Agent(auto-commit-dev) — PostToolUse hook 监听 Edit|Write

Agent 执行:
  1. Edit("package.json") → PostToolUse 触发:
     → npx eslint --fix package.json   ← eslint 处理 JSON
     → "Auto-linted: package.json"

  2. Edit("tsconfig.json") → PostToolUse 触发:
     → npx eslint --fix tsconfig.json
     → "Auto-linted: tsconfig.json"

  3. Agent 完成 → SubagentStop 触发:
     → git commit -m "feat: auto-implemented by agent"

hooks 效果:
  → matcher: "Edit|Write" 匹配所有文件写入操作
  → 即使是 JSON 配置文件也会触发 hook
  → eslint --fix 自动修复格式问题（缩进、尾逗号等）
  → Agent 无需手动运行格式化命令
```

### 示例 3：SubagentStart 钩子注入项目上下文

```
用户: "帮我重构 utils 目录"

→ Agent(auto-commit-dev)

SubagentStart hook 触发（Agent 刚启动，还未做任何操作）:
  → command 执行: echo '{"additionalContext": "Feature dev agent started"}'
  → 注入上下文: Agent 知道自己是 "Feature dev agent"

Agent 后续行为受 hook 注入的上下文影响:
  1. Read("src/utils/")                           ← 先了解目录结构
  2. Grep("export function", "src/utils/")        ← 找到所有导出函数
  3. Edit("src/utils/format.ts")                  ← 重构格式化函数
     → PostToolUse 触发: eslint --fix
  4. Edit("src/utils/validate.ts")                ← 重构验证函数
     → PostToolUse 触发: eslint --fix

hooks 效果:
  → SubagentStart 在 Agent 还没执行任何工具之前就注入额外上下文
  → 可用于向 Agent 传递项目特定的约定和规范
  → Agent 的行为可以被 hook 注入的信息引导
```

### 示例 4：PreToolUse 钩子拦截危险操作

```
用户: "清理一下测试文件"（假设 Agent 配置了 PreToolUse 拦截 rm 操作）

→ Agent(auto-commit-dev) — 额外配置 PreToolUse hook

PreToolUse hook 配置:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: |
            INPUT=$(cat)
            CMD=$(echo "$INPUT" | jq -r '.tool_input.command')
            if echo "$CMD" | grep -qi "rm -rf\|DROP TABLE\|DELETE FROM"; then
              echo '{"decision": "block", "reason": "Dangerous command blocked by hook"}'
              exit 0
            fi
            echo '{"decision": "allow"}'

Agent 执行:
  1. Bash("find tests -name '*.tmp' -delete")     ← 安全命令，放行
  2. Bash("rm -rf tests/fixtures")                ← ❌ 被 hook 拦截
     → Hook 输出: "Dangerous command blocked by hook"
     → Agent 收到拒绝信息，换用更安全的方式

hooks 效果:
  → PreToolUse hook 在工具执行前检查命令内容
  → 危险命令被拦截，Agent 不会执行破坏性操作
  → Agent 收到明确的拒绝原因，可以选择替代方案
  → 不影响正常的安全操作
```

### 示例 5：SubagentStop 钩子生成工作报告

```
用户: "实现购物车的增删改查接口"

→ Agent(auto-commit-dev)

Agent 执行全过程:
  1. Write("src/routes/cart.ts")        → PostToolUse: eslint --fix ✅
  2. Write("src/services/cart.ts")      → PostToolUse: eslint --fix ✅
  3. Write("src/routes/cart.test.ts")   → PostToolUse: eslint --fix ✅
  4. Edit("src/index.ts")               → PostToolUse: eslint --fix ✅
  5. Bash("npm test")                   → 测试通过

SubagentStop hook 触发（Agent 完成）:
  → 统计变更:
    FILES=$(git diff --name-only)
    → cart.ts, cart.test.ts, cart.ts, index.ts

  → 生成报告并提交:
    git add -A
    git commit -m "feat(cart): implement CRUD API for shopping cart

    Changes:
    - src/routes/cart.ts (new)
    - src/services/cart.ts (new)
    - src/routes/cart.test.ts (new)
    - src/index.ts (modified)"

  → 注入上下文: "Changes auto-committed"

hooks 效果:
  → SubagentStop 在 Agent 完成所有工作后触发
  → 自动统计变更文件列表
  → 生成包含文件清单的详细 commit message
  → Agent 运行期间的所有 hooks 在 Stop 后自动清理
  → 主对话不受这些 hooks 影响
```
