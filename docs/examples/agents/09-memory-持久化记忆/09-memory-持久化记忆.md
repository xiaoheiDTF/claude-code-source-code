# Agent 字段: memory（持久化记忆）

> 指南 3.2「持久化记忆」— Agent 跨会话保留知识

## 目录结构

```
# 三种 memory scope 的存储位置:

memory: user                        memory: project                     memory: local
──────────────                      ──────────────                      ─────────────
~/.claude/                          my-project/                         my-project/
├── agent-memory/                   ├── .claude/                        ├── .claude/
│   ├── reviewer/                   │   ├── agent-memory/               │   ├── agent-memory-local/
│   │   └── MEMORY.md  ← 跨项目     │   │   ├── reviewer/               │   │   └── experiment/
│   └── db-admin/                   │   │   │   └── MEMORY.md ← 团队共享│   │       └── MEMORY.md ← 不提交
│       └── MEMORY.md              │   │   └── db-admin/               │
                                    │   │       └── MEMORY.md           │
                                    │   └── settings.json               │
                                    │                                   │
  跨所有项目共享                      可提交到 VCS，团队共享               不提交，仅本地
```

## 说明

```
memory 三种 scope:

  memory: user
    路径: ~/.claude/agent-memory/<agentName>/MEMORY.md
    特点: 跨项目共享，所有项目通用
    场景: 个人编码习惯、通用偏好

  memory: project
    路径: .claude/agent-memory/<agentName>/MEMORY.md
    特点: 项目级，可提交 VCS，团队共享
    场景: 团队约定、项目特定规范

  memory: local
    路径: .claude/agent-memory-local/<agentName>/MEMORY.md
    特点: 本地，不提交 VCS（.gitignore）
    场景: 个人实验、临时笔记

省略 memory → Agent 没有持久化记忆，每次从零开始
```

## 使用示例 1: memory: project — 团队共享代码审查知识

```markdown
# 文件: .claude/agents/reviewer.md

---
description: Code reviewer that learns project conventions over time.
disallowedTools:
  - Write
  - Edit
memory: project                      ← 团队共享，提交 VCS
---

You are a code reviewer with memory.
每次审查前读取 MEMORY.md，审查后更新。
```

```
第 1 次（MEMORY.md 不存在）:
  → 审查 src/auth/jwt.ts
  → 发现: 团队用 bcrypt 但 salt 只有 8
  → 创建 MEMORY.md:
    "## 团队规范
     - [2024-01] 发现: bcrypt salt 应 ≥ 12
     - [2024-01] 发现: 错误处理用 Result 模式"

第 5 次（MEMORY.md 已有内容）:
  → 读取 MEMORY.md → 看到 "bcrypt salt ≥ 12"
  → 审查 src/auth/oauth.ts → 发现 salt=8 → 标记为违规
  → 追加新发现:
    "- [2024-02] 支付模块日期统一用 dayjs"

团队成员 B clone 项目后:
  → MEMORY.md 已经在 VCS 中
  → Agent 从第一天就知道: bcrypt salt ≥ 12、用 Result 模式、用 dayjs
  → 不需要重新学习
```

## 使用示例 2: memory: user — 跨项目的个人偏好

```markdown
# 文件: ~/.claude/agents/personal-coder.md

---
description: Personal coding assistant that remembers your style across all projects.
memory: user                        ← 跨项目，存在 ~/.claude/
model: sonnet
---

You are my personal coding assistant.
记住我在所有项目中的编码偏好。
```

```
项目 A（React 前端）:
  → Agent 学习: "用户偏好函数式组件"
  → 写入 ~/.claude/agent-memory/personal-coder/MEMORY.md

项目 B（Go 后端）:
  → 读取 MEMORY.md → 看到 "偏好函数式风格"
  → 在 Go 中也避免使用 struct 方法，优先用函数
  → 追加: "用户 Go 代码偏好纯函数"

项目 C（Python 脚本）:
  → 读取 MEMORY.md → 看到所有积累的偏好
  → 新项目也能立刻遵循你的风格

效果: 所有项目共享同一份个人偏好，不用每个项目重新教
```

## 使用示例 3: memory: local — 临时实验不污染团队

```markdown
# 文件: .claude/agents/experiment.md

---
description: Experimental agent for trying new patterns. Safe to explore.
memory: local                       ← 不提交 VCS，仅本地
tools:
  - Read
  - Write
  - Edit
  - Bash
---

尝试大胆的方案，记录什么有效什么无效。
```

```
实验 1: 尝试用 Zod 替代 Joi:
  → 本地 MEMORY.md 记录: "Zod 和 Joi API 对比，Zod 更简洁"
  → 不提交到 VCS → 团队其他成员看不到

实验 2: 尝试把 service 层改成 CQRS 模式:
  → 本地 MEMORY.md 记录: "CQRS 对本项目太复杂，不建议"
  → 实验失败 → 删除 MEMORY.md → 无痕迹

对比 memory: project:
  如果用 project → CQRS 实验记录会被提交 → 误导团队
  用 local → 实验记录仅本地 → 安全
```

## 使用示例 4: 不设 memory — 无状态 Agent

```markdown
# 文件: .claude/agents/one-time-analyzer.md

---
description: One-time code analyzer. No memory, fresh analysis every time.
model: opus
effort: high
---

每次从零开始分析，不受之前结论影响。
```

```
第 1 次: 分析架构 → 发现 3 个问题
第 2 次: 分析架构 → 发现 4 个问题（包含 1 个新的）
  → 不会"记住上次已经看过"而跳过
  → 每次都是全新视角

适用场景:
  ✅ 需要客观、无偏见的分析
  ✅ 一次性任务（不需要积累知识）
  ✅ 安全审计（不受之前结论影响）
```

## 使用示例 5: 同名 Agent 不同 scope — 项目覆盖用户

```
全局: ~/.claude/agents/reviewer.md（memory: user）
  → 记忆在 ~/.claude/agent-memory/reviewer/MEMORY.md
  → 内容: "个人偏好: 喜欢详细的注释"

项目: .claude/agents/reviewer.md（memory: project）
  → 记忆在 .claude/agent-memory/reviewer/MEMORY.md
  → 内容: "团队规范: 禁止多余注释，代码应自解释"

结果:
  → 项目级 Agent 覆盖全局同名
  → 在这个项目中，reviewer 用 project memory
  → 在其他项目中，reviewer 用 user memory
  → 两份 MEMORY.md 独立存在，互不干扰
```
