# Hook 事件: PostToolUse（工具执行后）

> 指南 4.2「所有 Hook 事件」— PostToolUse 在工具执行成功后触发，可注入额外上下文

## 目录结构

```
my-project/
├── .claude/
│   ├── settings.json            ← Hook 配置
│   └── hooks/
│       ├── auto-stage.sh        ← 自动暂存
│       ├── auto-lint.sh         ← 自动 Lint
│       ├── format-json.sh       ← JSON 格式化
│       └── notify-slack.sh      ← Slack 通知
├── src/
│   ├── app.ts
│   └── utils/
│       └── config.json
├── .git/
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
            "type": "command",
            "command": "bash .claude/hooks/auto-stage.sh",
            "async": true,
            "statusMessage": "Auto-staging changes..."
          }
        ]
      },
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/auto-lint.sh",
            "async": true,
            "statusMessage": "Running linter..."
          }
        ]
      },
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/format-json.sh",
            "async": true
          }
        ]
      },
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/notify-slack.sh",
            "async": true
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
# .claude/hooks/auto-stage.sh — 编辑后自动 git add
INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // .tool_input.path // empty')

if [ -n "$FILE_PATH" ] && [ -f "$FILE_PATH" ]; then
  git add "$FILE_PATH" 2>/dev/null
  echo "{\"additionalContext\": \"已自动暂存: $FILE_PATH\"}"
fi
```

## 说明

```
PostToolUse 事件:
  触发时机: 工具执行成功后（Edit、Write、Bash 等）
  matcher: 匹配工具名

  stdin 输入（比 PreToolUse 多两个字段）:
    tool_output: 工具输出文本
    tool_result: 工具返回的完整结果对象

  可返回的 JSON:
    additionalContext: "..." → 注入给 Claude 的额外上下文

  特性:
    async: true → 后台运行，不阻塞 Claude 的下一步操作
    asyncRewake: true → 后台完成后重新唤醒 agent
```

## 使用示例

### 示例 1：编辑后自动 git add（async）

```
用户: "修复 login.ts 里的类型错误"

→ Claude 调用 Edit 修改 src/auth/login.ts → 成功

→ PostToolUse Hook 触发（matcher: "Edit", async: true）:
  1. 脚本读取 stdin → file_path = "src/auth/login.ts"
  2. 执行: git add "src/auth/login.ts"
  3. 返回: {"additionalContext": "已自动暂存: src/auth/login.ts"}

→ Claude 收到上下文: "已自动暂存: src/auth/login.ts"
→ Claude 继续下一步操作（不等待 git add 完成）

效果:
  → 每次编辑后自动 git add，无需手动暂存
  → async: true 不阻塞主流程
  → 提交时所有修改已经在暂存区
```

### 示例 2：编辑 TypeScript 文件后自动 Lint（async + additionalContext）

```
用户: "重构 utils.ts"

→ Claude 调用 Edit 修改 src/utils.ts → 成功

→ PostToolUse Hook 触发（matcher: "Edit", async: true）:
  1. 脚本读取 stdin → file_path = "src/utils.ts"
  2. 检查扩展名: *.ts → 执行 npx eslint --fix "src/utils.ts"
  3. ESLint 发现 2 个问题并自动修复
  4. 返回: {"additionalContext": "ESLint 自动修复了 2 个问题: 缺少分号(x2)"}

→ Claude 收到: "ESLint 自动修复了 2 个问题"
→ Claude 知道 lint 已处理，不需要再手动运行

效果:
  → 每次 Edit 后自动执行 eslint --fix
  → Claude 知道哪些问题已被修复
  → 代码始终保持 lint 规范
```

### 示例 3：Write 后自动格式化 JSON 文件

```
用户: "更新 config.json 添加新的 API 端点"

→ Claude 调用 Write 写入 src/utils/config.json → 成功

→ PostToolUse Hook 触发（matcher: "Write", async: true）:
  1. 脚本读取 stdin → file_path = "src/utils/config.json"
  2. 检查扩展名: *.json
  3. 执行: jq '.' "src/utils/config.json" > tmp && mv tmp "src/utils/config.json"
  4. 返回: {"additionalContext": "JSON 文件已自动格式化"}

→ config.json 被 jq 格式化为标准缩进

效果:
  → JSON 文件始终保持统一格式
  → 不用担心 Claude 输出的 JSON 缩进不一致
  → jq 格式化确保语法正确
```

### 示例 4：部署命令后发送 Slack 通知（async）

```
用户: "部署到生产环境"

→ Claude 调用 Bash: npm run deploy:prod → 成功

→ PostToolUse Hook 触发（matcher: "Bash", async: true）:
  1. 脚本读取 stdin → tool_input.command = "npm run deploy:prod"
  2. 检测到 deploy 关键字
  3. 发送 Slack 通知:
     curl -X POST "$SLACK_WEBHOOK" -d '{
       "text": "🚀 生产环境部署完成\n提交: '$(git rev-parse --short HEAD)'"
     }'
  4. 返回: {"additionalContext": "Slack 通知已发送"}

→ 团队 Slack 频道收到部署通知

效果:
  → 部署完成后自动通知团队
  → 不影响 Claude 的后续操作
  → 团队实时感知部署状态
```

### 示例 5：追踪文件变更统计

```
用户: "完成用户模块的开发"

→ Claude 连续执行多个工具调用:
  Edit: src/user/service.ts
  Edit: src/user/controller.ts
  Write: src/user/dto.ts

→ 每次 PostToolUse Hook 触发:
  1. 读取 file_path
  2. 计算文件行数: wc -l < "$FILE_PATH"
  3. 追加到统计文件:
     echo "$(date +%H:%M:%S) | Edit | $FILE_PATH | $(wc -l < $FILE_PATH) lines" >> .claude/change-stats.log

→ 变更统计 .claude/change-stats.log:
  14:31:22 | Edit | src/user/service.ts    | 156 lines
  14:31:28 | Edit | src/user/controller.ts | 89 lines
  14:31:35 | Write | src/user/dto.ts       | 42 lines

→ Stop Hook 读取统计文件生成摘要:
  "本次会话共修改 3 个文件，总计 287 行代码"

效果:
  → 自动记录每次文件变更的详细信息
  → 会话结束时生成汇总报告
  → 可用于代码审查和工作量统计
```
