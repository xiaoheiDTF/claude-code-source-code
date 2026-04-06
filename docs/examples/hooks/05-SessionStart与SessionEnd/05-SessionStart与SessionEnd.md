# Hook 事件: SessionStart 与 SessionEnd（会话生命周期）

> 指南 4.2「所有 Hook 事件」— SessionStart 在会话启动时触发，SessionEnd 在会话结束时触发

## 目录结构

```
my-project/
├── .claude/
│   ├── settings.json              ← Hook 配置
│   └── hooks/
│       ├── welcome.sh             ← SessionStart: 显示项目信息
│       ├── load-env.sh            ← SessionStart: 加载环境变量
│       ├── summary.sh             ← SessionEnd: 生成会话摘要
│       ├── cleanup.sh             ← SessionEnd: 清理临时文件
│       └── timesheet.sh           ← 配对使用: 记录工时
├── src/
├── .env
└── package.json
```

## settings.json 配置

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/welcome.sh"
          },
          {
            "type": "command",
            "command": "bash .claude/hooks/load-env.sh"
          }
        ]
      }
    ],
    "SessionEnd": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/summary.sh",
            "async": true
          },
          {
            "type": "command",
            "command": "bash .claude/hooks/cleanup.sh",
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
# .claude/hooks/welcome.sh — 显示项目欢迎信息
BRANCH=$(git branch --show-current 2>/dev/null || echo "detached")
COMMIT=$(git log -1 --oneline 2>/dev/null || echo "none")
ISSUES=$(gh issue list --limit 3 --state open 2>/dev/null || echo "无法获取")

echo "{\"additionalContext\": \"项目信息 → 分支: $BRANCH | 最近提交: $COMMIT | 待处理 Issue: $ISSUES\"}"
```

## 说明

```
SessionStart 事件:
  触发时机: Claude Code 会话启动时（打开终端、新建对话）
  matcher: 匹配来源（api / cli）

  stdin 输入:
    hook_event_name: "SessionStart"
    source: 启动来源
    session_id: 新会话的 ID
    cwd: 工作目录

SessionEnd 事件:
  触发时机: 会话结束时（关闭终端、退出对话）
  matcher: 匹配退出原因

  stdin 输入:
    hook_event_name: "SessionEnd"
    source: 退出来源

  典型用途:
    SessionStart: 显示欢迎信息、加载环境上下文
    SessionEnd: 生成摘要、清理临时文件、保存工作进度
```

## 使用示例

### 示例 1：SessionStart 显示项目信息

```
用户启动 Claude Code → 新会话开始

→ SessionStart Hook 触发:
  1. 执行 git 命令获取项目状态:
     git branch --show-current → "main"
     git log -1 --oneline → "e4f3a2b feat: 添加用户搜索"
     git status --short → "M src/search.ts"
  2. 读取 package.json 获取技术栈:
     jq '.dependencies | keys | join(", ")' package.json → "react, next, prisma"
  3. 返回:
     {"additionalContext": "项目信息 → 分支: main | 技术栈: react, next, prisma |
      最近提交: e4f3a2b feat: 添加用户搜索 | 未提交变更: src/search.ts (已修改)"}

→ Claude 基于上下文主动打招呼:
  "你好！我看到你正在 main 分支上开发，最近在添加用户搜索功能，
   src/search.ts 有未提交的修改。需要我继续这个功能吗？"

效果:
  → 每次启动自动显示项目状态
  → Claude 能主动了解当前工作进展
  → 无需手动告诉 Claude "我在做什么"
```

### 示例 2：SessionStart 加载环境变量上下文

```
用户启动 Claude Code

→ SessionStart Hook 触发:
  1. 读取 .env 文件（仅 key，不读取 value）:
     grep -oP '^[A-Z_]+=' .env | tr -d '='
     → DATABASE_URL, API_KEY, REDIS_URL, S3_BUCKET
  2. 检查关键变量是否已设置:
     test -n "$DATABASE_URL" && echo "DB: ✅" || echo "DB: ❌"
  3. 返回:
     {"additionalContext": "环境变量状态:
      DATABASE_URL=✅ | API_KEY=✅ | REDIS_URL=❌ 未设置 | S3_BUCKET=✅
      提示: Redis 未配置，相关功能可能不可用"}

→ Claude 知道哪些服务可用:
  "注意: Redis 未配置，缓存相关功能暂不可用。
   数据库和 S3 已就绪。需要我帮你配置 Redis 吗？"

效果:
  → Claude 自动了解环境配置状态
  → 避免使用不可用的服务
  → 主动提示配置缺失
```

### 示例 3：SessionEnd 生成会话摘要

```
用户工作 2 小时后退出 Claude Code

→ SessionEnd Hook 触发:
  1. 读取 transcript 文件（会话记录）
  2. 统计本次会话:
     git diff --stat HEAD → "3 files changed, +156, -23"
     工具调用次数: Edit(8), Write(3), Bash(12), Read(15)
  3. 生成摘要文件 .claude/session-summaries/2026-04-05-session-abc.md:
     "## 会话摘要 (2026-04-05 14:00-16:00)
      变更文件: src/auth/login.ts (+45, -12)
               src/auth/oauth.ts (+67, -8)
               src/utils/jwt.ts (+44, -3)
      运行命令: npm test (通过), npm run build (通过)
      未完成任务: email 验证功能待实现"

效果:
  → 每次会话结束自动生成工作摘要
  → 下次启动时可快速回顾上次进展
  → 团队协作时了解同事做了什么
```

### 示例 4：SessionEnd 清理临时文件

```
用户退出 Claude Code

→ SessionEnd Hook 触发:
  1. 查找项目中的临时文件:
     find . -name "*.tmp" -o -name "*.bak" -o -name "*~"
  2. 查找 Claude 生成的大型中间文件:
     find .claude/ -name "*.log" -mtime +7
  3. 执行清理:
     rm -f src/**/*.tmp src/**/*.bak
     find .claude/ -name "*.log" -mtime +7 -delete
  4. 返回:
     {"additionalContext": "已清理 5 个临时文件，释放 2.3MB 空间"}

效果:
  → 会话结束自动清理工作目录
  → 避免临时文件积累
  → 保持项目目录整洁
```

### 示例 5：SessionStart + SessionEnd 配对记录工时

```
→ SessionStart Hook（会话开始时）:
  1. 记录开始时间:
     echo "$(date +%s) | $(git branch --show-current) | session-abc" > .claude/timesheet/current
  2. 返回: {"additionalContext": "工时记录已开始"}

→ 用户工作 3 小时...

→ SessionEnd Hook（会话结束时）:
  1. 读取开始时间:
     START=$(cat .claude/timesheet/current | cut -d'|' -f1)
  2. 计算持续时间:
     DURATION=$(( $(date +%s) - START ))
     HOURS=$((DURATION / 3600))
     MINUTES=$(( (DURATION % 3600) / 60 ))
  3. 追加到工时表:
     echo "2026-04-05 | 13:00-16:00 | 3h 0m | feature/auth | session-abc" >> .claude/timesheet/log.csv
  4. 删除临时文件:
     rm .claude/timesheet/current

→ 工时表 .claude/timesheet/log.csv:
  2026-04-05 | 09:00-10:30  | 1h 30m | main            | session-111
  2026-04-05 | 13:00-16:00  | 3h 0m  | feature/auth    | session-abc
  2026-04-06 | 10:00-11:15  | 1h 15m | feature/auth    | session-def

效果:
  → 自动记录每个会话的工作时长
  → 关联到具体分支（方便估算功能开发时间）
  → 可用于个人时间管理或团队工时报表
```
