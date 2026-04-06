# Skill: agent 指定 Agent 类型

> 指南 6.2「Skill 定义」— agent 字段指定执行 Skill 时使用的 Agent 类型

## 目录结构

```
my-project/
├── .claude/
│   ├── commands/
│   │   ├── commit.md              ← agent: general-purpose（默认）
│   │   ├── security-check.md      ← agent: security-reviewer
│   │   └── perf-optimize.md       ← agent: performance-analyzer
│   └── agents/
│       ├── security-reviewer.md   ← 安全审查 Agent
│       └── performance-analyzer.md ← 性能分析 Agent
└── src/
```

## 说明

```
agent 字段:
  格式: 字符串，Agent 类型名称
  默认: general-purpose（通用 Agent）
  作用: 指定执行这个 Skill 的 Agent 类型

  可选值:
    - general-purpose → 默认通用 Agent（无特殊限制）
    - 自定义 Agent 名称 → 对应 .claude/agents/xxx.md

  与 Agent 定义的关系:
    agent: "security-reviewer" →
    使用 .claude/agents/security-reviewer.md 定义的 Agent
    Agent 的 tools、disallowedTools、hooks 等全部生效

  行为:
    1. 用户调用 /skill-name
    2. 读取 Skill 文件的 agent 字段
    3. 如果是自定义 Agent → 使用该 Agent 的配置执行
    4. 如果是 general-purpose → 使用默认 Agent 执行
```

## 使用示例

### 示例 1：使用自定义安全审查 Agent 执行 Skill

```
security-check.md:
  ---
  description: Run security audit on the codebase
  agent: security-reviewer
  allowed-tools:
    - Read
    - Grep
    - Glob
    - Bash
  ---

  Perform a comprehensive security audit.
  Check for: OWASP Top 10, hardcoded secrets, SQL injection, XSS.

security-reviewer.md (Agent):
  ---
  description: Security vulnerability scanner
  tools: [Read, Grep, Glob, Bash]
  disallowedTools: [Write, Edit]
  effort: high
  model: sonnet
  ---

→ 用户输入: /security-check

→ 执行流程:
  1. 读取 security-check.md → agent: security-reviewer
  2. 加载 security-reviewer Agent 配置
  3. Agent 特性生效:
     - effort: high → 深度推理
     - model: sonnet → 使用 Sonnet 模型
     - disallowedTools: [Write, Edit] → 不能修改文件
  4. 执行安全审查（只读 + 高推理强度）

效果:
  → Skill 使用专门的 Agent 执行
  → Agent 的所有配置生效（tools、model、effort）
  → 安全审查用高推理强度，确保不遗漏漏洞
```

### 示例 2：使用性能分析 Agent 执行 Skill

```
perf-optimize.md:
  ---
  description: Analyze and optimize performance
  agent: performance-analyzer
  ---

  Analyze the following for performance issues: $ARGUMENTS

performance-analyzer.md (Agent):
  ---
  description: Performance analysis specialist
  tools: [Read, Bash, Grep, Glob]
  model: sonnet
  effort: high
  memory: project
  ---

→ 用户输入: /perf-optimize src/api/

→ 执行流程:
  1. 使用 performance-analyzer Agent
  2. Agent 读取记忆: .claude/agent-memory/performance-analyzer/MEMORY.md
     → 之前的优化记录: "数据库查询是最慢的瓶颈"
  3. 基于历史记忆聚焦分析:
     "上次分析发现数据库查询是瓶颈，我继续检查 API 层..."

效果:
  → Skill + Agent 组合实现专业化任务
  → Agent 的 memory 字段让 Skill 有持续学习能力
  → 每次执行都基于之前的分析结果
```

### 示例 3：默认 general-purpose 不指定 agent

```
commit.md:
  ---
  description: Generate a git commit
  allowed-tools:
    - Bash
    - Read
  ---

  Generate a commit message from git diff.

→ 没有 agent 字段 → 默认 general-purpose

→ 执行特性:
  - 使用默认模型
  - 无特殊工具限制
  - 无额外配置
  - 简单直接的执行

效果:
  → 大多数 Skill 不需要指定 agent
  → general-purpose 足以处理常规任务
  → 只有需要专业化时才指定自定义 Agent
```

### 示例 4：同一任务用不同 Agent 执行不同策略

```
# 快速审查（用 haiku 模型的 Agent）
quick-review.md:
  ---
  description: Quick code review with fast model
  agent: quick-reviewer
  ---

  Do a quick review focusing on obvious issues.

quick-reviewer.md:
  ---
  description: Fast code reviewer
  model: haiku
  effort: low
  tools: [Read, Grep, Glob]
  ---

# 深度审查（用 opus 模型的 Agent）
deep-review.md:
  ---
  description: Deep code review with advanced model
  agent: deep-reviewer
  ---

  Perform a thorough code review.

deep-reviewer.md:
  ---
  description: Deep code reviewer
  model: opus
  effort: high
  maxTurns: 50
  tools: [Read, Grep, Glob]
  ---

→ 用户输入: /quick-review src/auth.ts
  → 使用 haiku + low effort → 快速扫描，30 秒完成

→ 用户输入: /deep-review src/auth.ts
  → 使用 opus + high effort → 深度分析，3 分钟完成

效果:
  → 不同 Skill 指向不同 Agent
  → 同一任务可以用不同策略执行
  → 快速任务用便宜模型，重要任务用强模型
```

### 示例 5：Agent 不存在时的降级行为

```
custom-review.md:
  ---
  description: Custom code review
  agent: my-custom-reviewer    ← 这个 Agent 文件不存在
  ---

  Review the code...

→ 用户输入: /custom-review

→ 系统行为:
  1. 读取 agent 字段: "my-custom-reviewer"
  2. 查找 .claude/agents/my-custom-reviewer.md → 不存在
  3. 降级为 general-purpose Agent
  4. 继续执行 Skill（不会报错，只是失去自定义 Agent 的配置）

→ 与指定 Agent 存在时的区别:
  - 没有 tools/disallowedTools 限制
  - 没有自定义 model/effort
  - 没有 memory 功能
  - 没有 Agent hooks

效果:
  → Agent 不存在时降级为 general-purpose
  → Skill 仍然可以执行（不会失败）
  → 但失去自定义 Agent 的专业能力
  → 建议: 确保 Skill 引用的 Agent 文件存在
```
