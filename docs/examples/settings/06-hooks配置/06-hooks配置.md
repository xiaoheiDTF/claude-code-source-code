# settings.json: hooks 配置

> 指南 7.2「主要配置项」— settings.json 中的 hooks 字段（详细示例见 docs/examples/hooks/）

## settings.json 配置

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
            "statusMessage": "Checking file permissions..."
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
            "command": "bash .claude/hooks/auto-lint.sh",
            "async": true,
            "statusMessage": "Auto-linting..."
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
            "command": "bash .claude/hooks/session-summary.sh",
            "async": true
          }
        ]
      }
    ]
  }
}
```

## 说明

```
settings.json 中的 hooks 字段:
  → 与 .claude/hooks/ 目录中的脚本配合使用
  → 详细用法请参考: docs/examples/hooks/

  本文件仅展示 settings.json 中的 hooks 配置写法
  每个 Hook 事件的详细示例请查看对应的 hooks 示例文件夹

  快速参考:
    PreToolUse          → 工具执行前（可拦截）
    PostToolUse         → 工具执行后（可注入上下文）
    PostToolUseFailure  → 工具失败后
    UserPromptSubmit    → 用户提交时
    SessionStart        → 会话启动
    SessionEnd          → 会话结束
    Stop                → Agent 停止
    SubagentStart       → 子 Agent 启动
    SubagentStop        → 子 Agent 停止
```

## 使用示例

### 示例 1：最小 hooks 配置（编辑后自动 lint）

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "npx eslint --fix $CLAUDE_FILE_PATH",
            "async": true
          }
        ]
      }
    ]
  }
}
```

### 示例 2：多事件 hooks 配置

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "",
        "hooks": [{ "type": "command", "command": "bash .claude/hooks/welcome.sh" }]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{ "type": "command", "command": "bash .claude/hooks/check-command.sh" }]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "bash .claude/hooks/auto-stage.sh", "async": true },
          { "type": "command", "command": "bash .claude/hooks/auto-lint.sh", "async": true }
        ]
      }
    ],
    "SessionEnd": [
      {
        "matcher": "",
        "hooks": [{ "type": "command", "command": "bash .claude/hooks/summary.sh" }]
      }
    ]
  }
}
```

### 示例 3：混合 Hook 类型（command + prompt）

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "bash .claude/hooks/auto-lint.sh", "async": true }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "prompt", "prompt": "Consider the security implications of this command before executing." }
        ]
      }
    ],
    "UserPromptSubmit": [
      {
        "matcher": "",
        "hooks": [
          { "type": "prompt", "prompt": "Always check for OWASP Top 10 vulnerabilities in user requests." }
        ]
      }
    ]
  }
}
```

### 示例 4：带 timeout 和 once 的精细控制

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/check-branch.sh",
            "once": false,
            "timeout": 10
          }
        ]
      },
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/first-write-check.sh",
            "once": true,
            "timeout": 30,
            "statusMessage": "Running initial checks..."
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/post-command.sh",
            "async": true,
            "asyncRewake": true,
            "timeout": 120
          }
        ]
      }
    ]
  }
}
```

### 示例 5：空 hooks 配置（禁用所有 hooks）

```json
{
  "hooks": {}
}
```

```
→ 不配置任何 hooks
→ 也可以直接删除 hooks 字段
→ 适合: 不需要任何自动化的简单项目
→ 注意: 全局 hooks 仍然生效（项目空配置不会覆盖全局）
```
