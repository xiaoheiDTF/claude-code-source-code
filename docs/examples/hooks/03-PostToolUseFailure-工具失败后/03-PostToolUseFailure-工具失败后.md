# Hook 事件: PostToolUseFailure（工具失败后）

> 指南 4.2「所有 Hook 事件」— PostToolUseFailure 在工具执行失败后触发，可注入恢复建议

## 目录结构

```
my-project/
├── .claude/
│   ├── settings.json                ← Hook 配置
│   └── hooks/
│       ├── log-error.sh             ← 错误日志
│       ├── suggest-recovery.sh      ← 恢复建议
│       ├── retry-permission.sh      ← 权限重试
│       ├── alert-monitor.sh         ← 监控告警
│       └── error-stats.sh           ← 错误统计
├── src/
│   └── app.ts
├── error-logs/
└── package.json
```

## settings.json 配置

```json
{
  "hooks": {
    "PostToolUseFailure": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/log-error.sh"
          },
          {
            "type": "command",
            "command": "bash .claude/hooks/suggest-recovery.sh"
          }
        ]
      },
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/suggest-recovery.sh"
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
# .claude/hooks/log-error.sh — 记录工具失败日志
INPUT=$(cat)
EVENT=$(echo "$INPUT" | jq -r '.hook_event_name')
TOOL=$(echo "$INPUT" | jq -r '.tool_name')
OUTPUT=$(echo "$INPUT" | jq -r '.tool_output // empty')
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // .tool_input.path // empty')
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

echo "$TIMESTAMP | $EVENT | $TOOL | $FILE_PATH | $OUTPUT" >> .claude/error.log
```

## 说明

```
PostToolUseFailure 事件:
  触发时机: 工具执行失败后（Edit 写入失败、Bash 命令报错等）
  matcher: 匹配工具名

  stdin 输入:
    hook_event_name: "PostToolUseFailure"
    tool_name: 失败的工具名
    tool_input: 工具输入参数
    tool_output: 错误信息文本
    session_id: 会话 ID
    cwd: 当前工作目录

  可返回的 JSON:
    additionalContext: "..." → 注入恢复建议给 Claude

  典型用途:
    - 记录错误日志供排查
    - 提供恢复建议让 Claude 自动重试
    - 告警通知关键操作失败
```

## 使用示例

### 示例 1：记录所有 Edit 失败到错误日志

```
用户: "修改 auth.ts 添加 JWT 验证"

→ Claude 调用 Edit 修改 src/auth.ts → 失败（文件只读）

→ PostToolUseFailure Hook 触发:
  1. 脚本读取 stdin:
     tool_name = "Edit"
     tool_output = "Error: EACCES: permission denied, open 'src/auth.ts'"
     tool_input.file_path = "src/auth.ts"
  2. 写入错误日志:
     "2026-04-05 14:30:01 | PostToolUseFailure | Edit | src/auth.ts |
      Error: EACCES: permission denied"

→ error.log 记录完整的错误信息

效果:
  → 所有工具失败都被记录
  → 方便事后排查 "为什么 Claude 没有修改这个文件"
  → 包含时间戳、工具名、文件路径、错误详情
```

### 示例 2：Bash 失败时注入恢复建议

```
用户: "运行测试"

→ Claude 调用 Bash: npm test → 失败（退出码 1）

→ PostToolUseFailure Hook 触发:
  1. 脚本读取 stdin:
     tool_name = "Bash"
     tool_output = "FAIL src/auth.test.ts (3 failures)"
  2. 分析错误输出，检测到 "failures"
  3. 返回:
     {
       "additionalContext": "测试失败，建议: 1) 查看 src/auth.test.ts 中的断言
         2) 运行 npx jest src/auth.test.ts --verbose 查看详细输出
         3) 检查最近修改是否影响了认证逻辑"
     }

→ Claude 收到恢复建议，自动:
  "测试失败了，我来查看详细信息..."
  → 调用 Bash: npx jest src/auth.test.ts --verbose
  → 分析失败原因并修复

效果:
  → Claude 收到结构化的恢复建议
  → 不只是看到错误，还知道下一步该怎么做
  → 提高自动修复成功率
```

### 示例 3：Write 权限错误时提示重试方案

```
用户: "创建新的配置文件"

→ Claude 调用 Write: /etc/myapp/config.yaml → 失败（权限不足）

→ PostToolUseFailure Hook 触发:
  1. 检测到 "EACCES" 或 "permission denied"
  2. 检测到目标路径是系统目录
  3. 返回:
     {
       "additionalContext": "权限不足。建议写入项目本地目录: ./config/local-config.yaml，
         或使用 Bash 以 sudo 方式创建: sudo tee /etc/myapp/config.yaml"
     }

→ Claude 调整策略:
  "系统目录权限不足，我改为写入项目本地配置文件 ./config/local-config.yaml"

效果:
  → 自动提供替代方案
  → Claude 不需要盲目重试相同的失败操作
  → 减少无效的来回对话
```

### 示例 4：关键工具失败时发送监控告警

```
用户: "部署到生产环境"

→ Claude 调用 Bash: npm run deploy:prod → 失败

→ PostToolUseFailure Hook 触发:
  1. 检测到 tool_input.command 包含 "deploy" 关键字
  2. 标记为关键操作失败
  3. 发送告警:
     curl -X POST "$ALERT_WEBHOOK" -d '{
       "text": "🚨 生产部署失败！\n命令: npm run deploy:prod\n错误: ...",
       "priority": "high"
     }'
  4. 返回: {"additionalContext": "已发送告警通知给运维团队"}

→ 运维团队立即收到告警

效果:
  → 关键操作失败自动通知
  → 不依赖 Claude 自己报告问题
  → 人工可以快速介入处理
```

### 示例 5：累积错误统计生成会话报告

```
用户: 完成一整天的开发工作

→ 会话中发生了多次工具失败:
  10:30 — Edit 失败 (文件锁定)     ← PostToolUseFailure 记录
  11:15 — Bash 失败 (npm install 网络超时)  ← PostToolUseFailure 记录
  14:22 — Write 失败 (磁盘空间不足)  ← PostToolUseFailure 记录
  15:45 — Edit 失败 (文件锁定)     ← PostToolUseFailure 记录

→ 每次失败 Hook 都执行:
  echo "$EVENT|$TOOL|$ERROR_TYPE" >> .claude/error-stats.csv

→ SessionEnd Hook 读取统计并生成报告:
  "本次会话错误统计:
   - 总失败次数: 4
   - Edit 失败: 2 (文件锁定)
   - Bash 失败: 1 (网络超时)
   - Write 失败: 1 (磁盘空间)
   建议: 检查文件锁和磁盘空间"

效果:
  → 会话级错误汇总
  → 识别反复出现的问题模式
  → 帮助改善开发环境配置
```
