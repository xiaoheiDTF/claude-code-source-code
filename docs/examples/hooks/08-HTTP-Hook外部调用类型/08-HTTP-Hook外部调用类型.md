# Hook 类型: HTTP Hook（调用外部 API）

> 指南 4.3「Hook 类型」— HTTP Hook 调用外部 API/Webhook，用于通知、日志、CI/CD 触发

## 目录结构

```
my-project/
├── .claude/
│   ├── settings.json              ← Hook 配置
│   └── (无需脚本文件，HTTP Hook 直接在 JSON 中配置)
├── src/
└── package.json
```

## settings.json 配置

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "http",
            "url": "https://hooks.slack.com/services/T00/B00/xxx",
            "method": "POST",
            "headers": {
              "Content-Type": "application/json"
            }
          }
        ]
      }
    ],
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "http",
            "url": "https://api.github.com/repos/myorg/myrepo/dispatches",
            "method": "POST",
            "headers": {
              "Authorization": "Bearer $GITHUB_TOKEN",
              "Content-Type": "application/json"
            }
          }
        ]
      }
    ]
  }
}
```

## 说明

```
HTTP Hook 字段:
  url:     请求 URL（必填）
  method:  HTTP 方法（GET/POST/PUT/DELETE，默认 POST）
  headers: 请求头对象

  特点:
    - 不需要编写脚本，直接在 JSON 中配置
    - Hook 的 stdin 数据作为请求 body 发送
    - 环境变量可在 headers 中使用: "$TOKEN"

  适用场景:
    - 发送 Slack/Teams 通知
    - 触发 CI/CD 流水线
    - 记录日志到外部服务
    - 调用安全扫描 API
```

## 使用示例

### 示例 1：部署完成后发送 Slack 通知

```
settings.json:
  PostToolUse → matcher: "Bash" → type: "http"
  url: https://hooks.slack.com/services/T00/B00/xxx
  method: POST

用户: "部署到生产环境"

→ Claude 调用 Bash: npm run deploy:prod → 成功

→ PostToolUse HTTP Hook 触发:
  1. 读取 stdin（包含 tool_input、tool_output）
  2. 检测到 command 包含 "deploy"
  3. 发送 POST 请求到 Slack Webhook:
     {
       "text": "🚀 生产环境部署完成",
       "blocks": [{
         "type": "section",
         "text": "部署命令: npm run deploy:prod\n状态: 成功\n提交: a3f2b1c"
       }]
     }

→ Slack 频道收到通知:
  🚀 生产环境部署完成
  部署命令: npm run deploy:prod
  状态: 成功
  提交: a3f2b1c

效果:
  → 部署成功自动通知团队
  → 无需编写脚本，纯 JSON 配置
  → Slack Webhook URL 在 settings.json 中管理
```

### 示例 2：文件变更事件推送到监控仪表板

```
settings.json:
  PostToolUse → matcher: "Edit|Write" → type: "http"
  url: https://dashboard.example.com/api/events
  method: POST
  headers: {"Authorization": "Bearer $DASHBOARD_TOKEN"}

用户: "重构 auth 模块"

→ Claude 连续编辑多个文件:

  Edit src/auth/jwt.ts → 成功
  → HTTP Hook: POST → {"event": "file_changed", "file": "src/auth/jwt.ts", "action": "edit"}

  Edit src/auth/oauth.ts → 成功
  → HTTP Hook: POST → {"event": "file_changed", "file": "src/auth/oauth.ts", "action": "edit"}

  Write src/auth/new-module.ts → 成功
  → HTTP Hook: POST → {"event": "file_changed", "file": "src/auth/new-module.ts", "action": "write"}

→ 监控仪表板实时显示:
  14:31:22 — ✏️ 编辑 src/auth/jwt.ts
  14:31:28 — ✏️ 编辑 src/auth/oauth.ts
  14:31:35 — 📝 创建 src/auth/new-module.ts

效果:
  → 文件变更实时推送到外部系统
  → 项目经理/团队Lead 可以实时看到开发进展
  → 无需轮询 git log，变更即推送
```

### 示例 3：Agent 停止后触发 CI 流水线

```
settings.json:
  Stop → matcher: "" → type: "http"
  url: https://api.github.com/repos/myorg/myrepo/dispatches
  method: POST
  headers: {"Authorization": "Bearer $GITHUB_TOKEN"}

用户: "完成用户认证功能的开发"

→ Claude 完成所有修改后停止

→ Stop HTTP Hook 触发:
  1. 发送 POST 到 GitHub API:
     {
       "event_type": "claude_code_completed",
       "client_payload": {
         "branch": "feature/auth",
         "files_changed": 5,
         "session_id": "session-abc"
       }
     }

→ GitHub Actions 自动触发:
  workflow "Claude Code Auto CI":
    on: repository_dispatch
    steps:
      - run: npm install
      - run: npm test
      - run: npm run build
      - run: npm run e2e

效果:
  → Claude 完成工作后自动触发 CI
  → 不需要用户手动 "run pipeline"
  → 实现完整的 "AI 编码 → 自动验证" 流程
```

### 示例 4：会话活动记录到外部分析服务

```
settings.json:
  SessionEnd → matcher: "" → type: "http"
  url: https://analytics.example.com/api/sessions
  method: POST
  headers: {"Authorization": "Bearer $ANALYTICS_TOKEN", "Content-Type": "application/json"}

用户工作结束后退出 Claude Code

→ SessionEnd HTTP Hook 触发:
  1. 发送 POST 请求:
     {
       "session_id": "session-abc",
       "duration_minutes": 45,
       "branch": "feature/auth",
       "tools_used": {
         "Edit": 8, "Write": 3, "Bash": 12, "Read": 15
       },
       "files_changed": ["src/auth/login.ts", "src/auth/oauth.ts"],
       "timestamp": "2026-04-05T16:00:00Z"
     }

→ 分析仪表板:
  今日 Claude Code 使用统计:
  - 总会话: 5
  - 总时长: 3.5 小时
  - 修改文件: 23
  - 最活跃分支: feature/auth

效果:
  → 自动收集使用数据到分析平台
  → 团队可以量化 AI 辅助编程的效率
  → 管理层了解工具使用情况
```

### 示例 5：修改生产配置前调用安全扫描 API

```
settings.json:
  PreToolUse → matcher: "Write" → type: "http"
  url: https://security-scanner.example.com/api/scan
  method: POST
  headers: {"Authorization": "Bearer $SECURITY_TOKEN"}

用户: "更新生产环境的数据库配置"

→ Claude 调用 Write 写入 production/db-config.yaml

→ PreToolUse HTTP Hook 触发:
  1. 发送 POST 请求（包含 tool_input）:
     {
       "file_path": "production/db-config.yaml",
       "content": "<即将写入的内容>"
     }
  2. 安全扫描 API 返回:
     {
       "risk_level": "high",
       "findings": ["检测到明文数据库密码", "SSL 未启用"],
       "recommendation": "deny"
     }

→ Claude 收到安全扫描结果:
  "⚠️ 安全扫描发现 2 个问题:
   1. 检测到明文数据库密码 — 建议使用环境变量
   2. SSL 未启用 — 生产环境必须启用加密连接
   建议修改后再写入。"

效果:
  → 写入前自动调用安全扫描 API
  → 防止不安全的配置进入生产环境
  → HTTP Hook 可用于 PreToolUse 做预检查
```
