# Memory: Agent Memory — local scope（本地私有）

> 指南 8.2「Agent Memory」— local scope 不提交 VCS，仅本地生效

## 目录结构

```
my-project/
├── .claude/
│   └── agent-memory-local/        ← local scope 目录（不提交 Git）
│       └── dev/
│           └── MEMORY.md
├── .gitignore                     ← 排除 agent-memory-local/
└── src/
```

## 说明

```
Agent Memory — local scope:
  路径: <project>/.claude/agent-memory-local/<agentType>/MEMORY.md
  范围: 仅当前本地环境
  提交 VCS: 否
  团队共享: 否

三种 scope 对比:
  user     → ~/.claude/agent-memory/           → 跨项目
  project  → .claude/agent-memory/              → 项目级，团队共享
  local    → .claude/agent-memory-local/         → 本地，私有

适用: 个人偏好、敏感信息、实验性记忆
```

## 使用示例

### 示例 1：保存个人工作习惯（不影响团队）

```
.claude/agent-memory-local/dev/MEMORY.md:

## 我的工作习惯
- 上午 9-12 点效率最高，适合写核心逻辑
- 下午做简单任务
- 每周五下午做代码审查
- 讨厌写文档但知道很重要

→ 团队成员看不到这些（不提交 Git）
```

### 示例 2：保存敏感信息使用方式

```
.claude/agent-memory-local/deployer/MEMORY.md:

## 凭证使用习惯
- AWS 凭证在 ~/.aws/credentials [prod] profile
- 不要使用 [default] profile
- 生产 S3 bucket: myapp-prod-assets
- 数据库密码在 1Password vault 'prod-db'

→ 不进入版本控制（安全）
```

### 示例 3：作为实验沙箱

```
→ 实验阶段先在 local scope 测试:

  .claude/agent-memory-local/test-gen/MEMORY.md:
    "尝试使用 AAA (Arrange-Act-Assert) 模式"

→ 发现效果好 → 迁移到 project scope:
  cp .claude/agent-memory-local/test-gen/MEMORY.md \
     .claude/agent-memory/test-gen/MEMORY.md
  git add .claude/agent-memory/test-gen/MEMORY.md

→ 验证有效后团队共享
```

### 示例 4：同一项目多人各自有 local 记忆

```
开发者 Alice:
  .claude/agent-memory-local/dev/MEMORY.md:
    "习惯: 先写界面再写逻辑"

开发者 Bob:
  .claude/agent-memory-local/dev/MEMORY.md:
    "习惯: TDD 先写测试再写代码"

→ 互不影响（各自 local 独立）
→ 但共享 project scope 记忆
```

### 示例 5：.gitignore 正确配置

```
# .gitignore:

# Agent Memory — local（不提交）
.claude/agent-memory-local/

# Agent Memory — project（要提交）
# 不要忽略 .claude/agent-memory/

→ git status 验证:
  看到 .claude/agent-memory/ 文件 ✅（可提交）
  不看到 .claude/agent-memory-local/ ✅（被忽略）
```
