# Hook 事件: UserPromptSubmit（用户提交消息时）

> 指南 4.2「所有 Hook 事件」— UserPromptSubmit 在用户提交消息时触发，可注入上下文或过滤内容

## 目录结构

```
my-project/
├── .claude/
│   ├── settings.json              ← Hook 配置
│   └── hooks/
│       ├── inject-context.sh      ← 注入项目上下文
│       ├── filter-sensitive.sh    ← 过滤敏感信息
│       ├── audit-prompt.sh        ← 审计日志
│       └── rate-limit.sh          ← 频率限制
├── src/
└── package.json
```

## settings.json 配置

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/inject-context.sh"
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
# .claude/hooks/inject-context.sh — 注入项目上下文
INPUT=$(cat)

BRANCH=$(git branch --show-current 2>/dev/null || echo "unknown")
LAST_COMMIT=$(git log -1 --oneline 2>/dev/null || echo "none")
CHANGED=$(git diff --name-only 2>/dev/null | head -5 | tr '\n' ', ')

echo "{\"additionalContext\": \"项目上下文 → 分支: $BRANCH | 最近提交: $LAST_COMMIT | 未提交变更: $CHANGED\"}"
```

## 说明

```
UserPromptSubmit 事件:
  触发时机: 用户每次提交消息（提问、指令）时
  matcher: 匹配来源（如 "api"、"cli"）

  stdin 输入:
    hook_event_name: "UserPromptSubmit"
    source: 消息来源（api / cli）
    prompt: 用户提交的完整文本
    session_id: 会话 ID
    cwd: 当前工作目录

  可返回的 JSON:
    additionalContext: "..." → 在用户消息前注入额外上下文

  典型用途:
    - 自动注入项目状态（分支、未提交变更）
    - 过滤敏感信息（API Key、密码）
    - 审计日志记录用户操作
```

## 使用示例

### 示例 1：自动注入项目上下文（git 分支 + 最近提交）

```
用户: "帮我修复登录页面的 bug"

→ UserPromptSubmit Hook 触发:
  1. 脚本读取 stdin → 获取 session 信息
  2. 执行 git 命令:
     git branch --show-current → "feature/auth-redesign"
     git log -1 --oneline → "a3f2b1c fix: 修复 OAuth 回调"
     git diff --name-only → "src/auth/login.tsx, src/auth/oauth.ts"
  3. 返回:
     {"additionalContext": "项目上下文 → 分支: feature/auth-redesign |
      最近提交: a3f2b1c fix: 修复 OAuth 回调 | 未提交变更: src/auth/login.tsx, src/auth/oauth.ts"}

→ Claude 看到的完整上下文:
  [注入] 项目上下文 → 分支: feature/auth-redesign | ...
  [用户] 帮我修复登录页面的 bug

→ Claude 基于分支名和最近提交更准确地理解问题:
  "我看到你正在 feature/auth-redesign 分支工作，最近提交是修复 OAuth 回调。
   登录页面的 bug 可能和 OAuth 回调有关，让我先看看相关文件。"

效果:
  → 每条消息自动附带项目状态
  → Claude 无需手动查询就能了解当前开发上下文
  → 提高回答的准确性和相关性
```

### 示例 2：过滤敏感数据（屏蔽 API Key）

```
用户: "用这个 API Key 测试一下: sk-abc123secretkey456"

→ UserPromptSubmit Hook 触发:
  1. 脚本读取 stdin → prompt = "用这个 API Key 测试一下: sk-abc123secretkey456"
  2. 正则检测敏感模式: sk-[a-zA-Z0-9]{20,}
  3. 替换为占位符: sk-***REDACTED***
  4. 返回:
     {"additionalContext": "⚠️ 检测到 API Key 已自动屏蔽。
       原始 key 已记录到 .claude/sensitive.log。
       请使用环境变量 $API_KEY 代替硬编码。" }

→ Claude 看不到真实 API Key，但知道用户想测试 API:
  "检测到你提供了 API Key，我已经为你屏蔽。
   建议使用环境变量: process.env.API_KEY"
→ 自动使用 $API_KEY 环境变量编写代码

效果:
  → 防止敏感信息泄露到对话历史
  → Claude 仍然能理解用户意图
  → 引导用户使用安全的凭证管理方式
```

### 示例 3：审计日志记录所有用户提示

```
用户 A: "查看一下数据库用户表的结构"
用户 A: "导出所有用户的邮箱地址"
用户 A: "删除测试账号 test@example.com"

→ 每次 UserPromptSubmit Hook 触发:
  1. 读取 stdin → 获取 prompt 和 session_id
  2. 记录到审计日志:
     echo "$(date '+%Y-%m-%d %H:%M:%S') | session-abc | $PROMPT" >> .claude/audit.log

→ 审计日志:
  2026-04-05 09:15:22 | session-abc | 查看一下数据库用户表的结构
  2026-04-05 09:17:45 | session-abc | 导出所有用户的邮箱地址
  2026-04-05 09:20:11 | session-abc | 删除测试账号 test@example.com

效果:
  → 所有用户操作被记录（合规审计）
  → 可追溯谁在什么时间做了什么
  → 敏感操作（删除、导出）有完整记录
```

### 示例 4：自动识别关键词并预加载上下文

```
用户: "review 一下 PR #42 的代码变更"

→ UserPromptSubmit Hook 触发:
  1. 读取 stdin → prompt = "review 一下 PR #42 的代码变更"
  2. 正则匹配 "review" + "PR" + "#(\d+)"
  3. 提取 PR 号: 42
  4. 执行: gh pr diff 42 --stat
  5. 返回:
     {"additionalContext": "PR #42 信息:
       标题: feat: add user authentication
       作者: zhangsan
       变更: +342 -89 (12 files)
       主要修改: src/auth/, src/middleware/, tests/auth/"}

→ Claude 直接基于 PR 上下文开始 review:
  "我来审查 PR #42。这是一个用户认证功能，涉及 12 个文件..."

效果:
  → 关键词触发自动预加载
  → Claude 不需要先花一轮查询 PR 信息
  → 节省对话轮次，直接进入正题
```

### 示例 5：频率限制（防止快速连续提交）

```
用户: 快速连续发送 5 条消息:
  10:30:00 — "修改按钮颜色"
  10:30:02 — "不对，改成蓝色"
  10:30:04 — "等等，还是红色吧"
  10:30:05 — "算了先不管样式了"
  10:30:07 — "先修那个 API bug"

→ 第 1-3 条: 正常处理
→ 第 4 条 UserPromptSubmit Hook 触发:
  1. 检查时间戳文件 .claude/last-prompt-time
  2. 发现 10 秒内已提交 3 条消息
  3. 返回:
     {"additionalContext": "⚠️ 提示: 你在 10 秒内已提交多条消息。
       建议整理好需求后一次性提交，Claude 能更好地理解完整上下文。"}

→ 第 5 条:
  1. 仍然在 10 秒窗口内
  2. 返回: {"additionalContext": "⏳ 请稍等，你提交得太快了。建议等待上一条回复后再继续。"}

效果:
  → 防止用户因快速修改指令导致 Claude 反复重做
  → 引导用户整理好需求再提交
  → 减少 token 浪费和无效操作
```
