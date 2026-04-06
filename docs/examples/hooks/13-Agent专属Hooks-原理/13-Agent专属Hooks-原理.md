# Agent 专属 Hooks: 原理机制

> 指南 5.1「原理」— Agent 在 frontmatter 中定义 hooks，启动时注册，结束时自动清理

## 目录结构

```
my-project/
├── .claude/
│   └── agents/
│       ├── feature-dev.md          ← Agent 定义（含 hooks）
│       ├── code-reviewer.md        ← 只读审查 Agent
│       └── migrator.md             ← 数据库迁移 Agent
├── src/
└── package.json
```

## Agent 文件内容（含 hooks）

```markdown
# 文件: .claude/agents/feature-dev.md

---
description: Feature implementer with auto-commit
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

Implement features based on the specification.
Follow project coding standards.
```

## 说明

```
Agent 专属 Hooks 核心原理:

  1. 定义位置: Agent 的 frontmatter 中的 hooks 字段
     （不是 settings.json，是 .claude/agents/xxx.md 文件内）

  2. 生命周期:
     Agent 启动 → hooks 注册
     Agent 运行 → hooks 在 Agent session 内生效
     Agent 结束 → hooks 自动注销

  3. Stop → SubagentStop 转换:
     Agent frontmatter 中写的 "Stop" 会被自动转为 "SubagentStop"
     因为 Agent 是子 Agent，不会触发顶级 Stop 事件

  4. 作用域隔离:
     Agent 的 hooks 只在该 Agent 的 session 内生效
     主 Claude session 不受影响
     其他 Agent 也不受影响

  5. 与 settings.json Hooks 的区别:
     settings.json hooks → 全局/项目级别，所有会话生效
     Agent frontmatter hooks → 仅该 Agent 运行期间生效
```

## 使用示例

### 示例 1：Agent 启动注册 hooks，结束自动清理

```
Agent 定义: .claude/agents/feature-dev.md
  hooks:
    PostToolUse → matcher: "Edit|Write" → 自动 lint

→ 用户调用 Agent:
  "用 feature-dev Agent 实现用户搜索功能"

→ Agent 启动:
  1. 读取 .claude/agents/feature-dev.md
  2. 解析 frontmatter 中的 hooks
  3. 注册 hooks 到 Agent 的 session:
     PostToolUse(Edit|Write) → auto-lint

→ Agent 运行中（hooks 生效）:
  Edit src/search.ts → PostToolUse hook 触发 → 自动 lint ✅
  Write src/api/search.ts → PostToolUse hook 触发 → 自动 lint ✅

→ Agent 完成:
  1. hooks 自动注销
  2. 主 Claude session 的 hooks 不受影响
  3. 再次调用该 Agent 时 hooks 重新注册

效果:
  → hooks 生命周期绑定到 Agent 运行期间
  → 不污染全局 hooks
  → 每个 Agent 可以有独立的 hooks 配置
```

### 示例 2：Stop hook 自动转换为 SubagentStop

```
Agent 定义: .claude/agents/reviewer.md
  hooks:
    Stop:                           ← 写的是 "Stop"
      - matcher: ""
        hooks:
          - type: command
            command: "echo 'Review completed'"

→ Agent 启动:
  源码 registerFrontmatterHooks.ts 处理:
  1. 读取 frontmatter hooks
  2. 检测到 "Stop" → 自动转换为 "SubagentStop"
  3. 注册: SubagentStop hook

→ Agent 完成:
  触发 SubagentStop 事件（不是 Stop）
  → hook 执行: echo 'Review completed'

对比:
  如果不转换，写 "Stop" → Agent 结束时不会触发 Stop 事件
  → 因为 Agent 是子 Agent，只有顶级会话才有 Stop 事件
  → 转换为 SubagentStop 才能正确触发

效果:
  → 开发者写 "Stop" 即可（直觉友好）
  → 系统自动转换为 "SubagentStop"（技术正确）
  → 无需记住 "子 Agent 要写 SubagentStop" 这个细节
```

### 示例 3：hooks 作用域隔离 — 不影响主 session

```
主 Claude session 已有 hooks:
  settings.json → PreToolUse(Edit) → 全局安全检查

Agent 定义: .claude/agents/quick-fix.md
  hooks:
    PreToolUse:
      - matcher: "Edit"
        hooks:
          - type: command
            command: "echo '{\"permissionDecision\": \"allow\"}'"

→ 主 session 编辑文件:
  触发全局安全检查 hook ✅
  不触发 quick-fix Agent 的 hook ❌

→ 调用 quick-fix Agent:
  Agent 启动 → 注册自己的 PreToolUse hook
  Agent 编辑文件:
    → Agent 的 hook 触发: auto-allow ✅
    → 全局 hook 也触发: 安全检查 ✅（合并执行）

→ Agent 结束:
  Agent 的 hook 注销
  主 session 继续使用全局 hook

效果:
  → Agent hooks 只在 Agent 运行期间生效
  → 与全局 hooks 合并执行
  → Agent 结束后不影响主 session
```

### 示例 4：多个 Agent 各自独立的 hooks

```
Agent A: .claude/agents/dev.md
  hooks:
    PostToolUse(Edit) → auto-lint (eslint)
    SubagentStop → auto-commit

Agent B: .claude/agents/reviewer.md
  hooks:
    SubagentStart → load project context
    PreToolUse(Edit) → deny（审查 Agent 不能修改文件）

→ 同时运行 Agent A 和 Agent B:

  Agent A 编辑文件:
    → Agent A 的 PostToolUse hook 触发: eslint ✅
    → Agent B 的 hooks 不受影响 ❌

  Agent B 试图编辑文件:
    → Agent B 的 PreToolUse hook 触发: deny ✅
    → Agent A 的 hooks 不受影响 ❌

  Agent A 完成:
    → Agent A 的 SubagentStop hook 触发: auto-commit ✅
    → Agent A 的 hooks 注销
    → Agent B 的 hooks 不受影响

效果:
  → 每个 Agent 有独立的 hooks 作用域
  → 不同 Agent 的 hooks 互不干扰
  → 适合并行运行多个 Agent 各自执行不同任务
```

### 示例 5：Agent frontmatter hooks vs settings.json hooks 对比

```
场景: 两个位置都定义了 PostToolUse hook

# settings.json（全局/项目级别）
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [{ "type": "command", "command": "bash auto-stage.sh" }]
      }
    ]
  }
}

# .claude/agents/feature-dev.md（Agent 级别）
hooks:
  PostToolUse:
    - matcher: "Edit|Write"
      hooks:
        - type: command
          command: "bash auto-lint.sh"

→ 调用 feature-dev Agent 编辑文件:

  PostToolUse hooks 执行:
  1. settings.json 的 auto-stage.sh → git add ✅
  2. Agent frontmatter 的 auto-lint.sh → eslint --fix ✅
  3. 两个来源的 hooks 合并执行！

→ 主 session 编辑文件（不调用 Agent）:
  1. settings.json 的 auto-stage.sh → git add ✅
  2. Agent 的 auto-lint.sh → 不执行 ❌

效果:
  → settings.json hooks: 始终生效（全局/项目级）
  → Agent frontmatter hooks: 仅该 Agent 运行时生效
  → 两者合并执行，不互相覆盖
  → 设计原则: 全局放通用 hooks，Agent 放专属 hooks
```
