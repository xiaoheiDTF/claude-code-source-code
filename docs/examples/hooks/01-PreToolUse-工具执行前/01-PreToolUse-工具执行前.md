# Hook 事件: PreToolUse（工具执行前）

> 指南 4.2「所有 Hook 事件」— PreToolUse 在工具执行前触发，可控制执行权限和修改输入参数

## 目录结构

```
my-project/
├── .claude/
│   ├── settings.json              ← Hook 配置
│   └── hooks/
│       ├── block-rmrf.sh          ← 拦截危险命令
│       ├── auto-approve-tests.sh  ← 自动批准测试文件
│       └── audit-log.sh           ← 审计日志
├── src/
│   ├── app.ts
│   └── __tests__/
│       └── app.test.ts
├── production/
│   └── config.json
└── package.json
```

## settings.json 配置

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/block-rmrf.sh"
          }
        ]
      },
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/auto-approve-tests.sh"
          }
        ]
      }
    ]
  }
}
```

## Hook 脚本

```bash
#!/bin/bash
# .claude/hooks/block-rmrf.sh — 拦截危险 rm -rf 命令
INPUT=$(cat)
TOOL_INPUT=$(echo "$INPUT" | jq -r '.tool_input.command // empty')

if echo "$TOOL_INPUT" | grep -qE 'rm\s+-rf\s+/'; then
  echo '{"permissionDecision": "deny", "permissionDecisionReason": "禁止递归删除根目录"}'
  exit 0
fi
```

## 说明

```
PreToolUse 事件:
  触发时机: 工具执行前（Edit、Write、Bash 等所有工具）
  matcher: 匹配工具名（如 "Edit"、"Write"、"Bash"）

  可返回的 JSON 控制:
    permissionDecision: "allow"   → 自动允许执行
    permissionDecision: "deny"    → 拒绝执行
    permissionDecision: "ask"     → 要求用户手动确认
    updatedInput: { ... }         → 修改工具的输入参数

  stdin 输入:
    tool_name: 工具名称
    tool_input: 工具输入参数（包含 file_path、command 等）
    session_id: 会话 ID
    cwd: 当前工作目录
```

## 使用示例

### 示例 1：拦截危险的 rm -rf 命令（deny）

```
用户: "清理一下 node_modules 和根目录的临时文件"

→ Claude 调用 Bash 工具: rm -rf /tmp/*

→ PreToolUse Hook 触发（matcher: "Bash"）:
  1. 脚本读取 stdin → 提取 tool_input.command = "rm -rf /tmp/*"
  2. 正则匹配 "rm -rf /" → 命中危险模式
  3. 返回: {"permissionDecision": "deny", "permissionDecisionReason": "禁止递归删除根目录"}

→ Claude 收到拒绝:
  "我无法执行 rm -rf 命令，因为这涉及递归删除根目录下的文件。
   请确认你需要删除哪些具体文件，我会逐个处理。"

效果:
  → 危险命令被拦截，不会执行
  → Claude 收到拒绝原因，可以调整方案
  → 保护系统免受误操作
```

### 示例 2：自动批准测试文件的编辑（allow）

```
用户: "帮我修改一下 app.test.ts 的测试用例"

→ Claude 调用 Edit 工具修改 src/__tests__/app.test.ts

→ PreToolUse Hook 触发（matcher: "Edit"）:
  1. 脚本读取 stdin → 提取 tool_input.file_path
  2. 检查路径: 匹配 *__tests__* 或 *.test.* 或 *.spec.*
  3. 返回: {"permissionDecision": "allow"}

→ Edit 自动执行，无需用户手动确认

效果:
  → 测试文件修改自动通过，节省时间
  → 非测试文件仍需手动确认（安全机制）
  → 开发体验: 修改测试 → 秒过，修改源码 → 弹窗确认
```

### 示例 3：重定向文件写入路径（updatedInput）

```
用户: "把新组件写到 src/components/Button.tsx"

→ Claude 调用 Write 工具: file_path = "src/components/Button.tsx"

→ PreToolUse Hook 触发（matcher: "Write"）:
  1. 脚本读取 stdin → 提取 file_path
  2. 检测到 components 下的新文件
  3. 项目规范: 新组件必须放在 v2 目录
  4. 返回:
     {
       "permissionDecision": "allow",
       "updatedInput": {
         "file_path": "src/v2/components/Button.tsx"
       }
     }

→ Write 工具使用修改后的路径: src/v2/components/Button.tsx

效果:
  → 文件实际写入到 src/v2/components/Button.tsx
  → Claude 不知道路径被修改了
  → 自动强制执行项目目录规范
```

### 示例 4：生产环境配置变更需手动确认（ask）

```
用户: "更新一下生产环境的数据库连接字符串"

→ Claude 调用 Edit 工具修改 production/config.json

→ PreToolUse Hook 触发（matcher: "Edit"）:
  1. 脚本读取 stdin → file_path = "production/config.json"
  2. 检测到路径包含 "production/" 或 "prod/"
  3. 返回: {"permissionDecision": "ask", "permissionDecisionReason": "生产环境配置变更需要手动确认"}

→ 弹出确认对话框:
  "⚠️ 生产环境配置变更需要手动确认
   理由: 生产环境配置变更需要手动确认
   [允许] [拒绝]"

效果:
  → 关键文件修改强制弹出确认
  → 用户可以看到具体变更内容再决定
  → 防止意外修改生产配置
```

### 示例 5：审计日志记录所有编辑操作（allow + 记录）

```
用户: "重构 auth 模块"

→ Claude 连续调用 Edit 工具修改多个文件:

  Edit 1: src/auth/jwt.ts
  → PreToolUse Hook 触发:
    1. 读取 tool_input.file_path = "src/auth/jwt.ts"
    2. 记录到审计日志: echo "$(date) | Edit | src/auth/jwt.ts | session-abc" >> .claude/audit.log
    3. 返回: {"permissionDecision": "allow"}

  Edit 2: src/auth/oauth.ts
  → PreToolUse Hook 触发:
    1. 记录: "2026-04-05 14:32:01 | Edit | src/auth/oauth.ts | session-abc" >> .claude/audit.log
    2. 返回: {"permissionDecision": "allow"}

  Edit 3: src/auth/middleware.ts
  → 同样记录并允许

→ 审计日志 .claude/audit.log:
  2026-04-05 14:31:58 | Edit | src/auth/jwt.ts | session-abc
  2026-04-05 14:32:01 | Edit | src/auth/oauth.ts | session-abc
  2026-04-05 14:32:05 | Edit | src/auth/middleware.ts | session-abc

效果:
  → 所有编辑操作被记录（谁、何时、改了什么文件）
  → 不影响正常编辑流程（自动 allow）
  → 可用于代码审查、合规审计、回溯问题
```
