# Memory: 记忆读写机制

> 指南 8.1 + 8.2 — 不同记忆的读取和写入方式

## 说明

```
读取机制:

  Auto Memory:
    → 会话启动时自动加载 ~/.claude/memory/MEMORY.md
    → 不需要手动读取

  Agent Memory:
    → Agent 启动时自动加载对应 MEMORY.md
    → 按顺序: user scope → project scope → local scope

  项目 Memory:
    → 会话启动时加载 CLAUDE.md
    → 访问子目录时加载子目录 CLAUDE.md
    → 访问匹配 paths 的文件时加载 rules

写入机制:

  Auto Memory:
    → /remember <内容> 命令
    → Claude 用 Write 工具写入

  Agent Memory:
    → Agent 运行时用 Write 工具写入

  项目 Memory:
    → 手动编辑或 Claude 用 Edit/Write 编辑
```

## 使用示例

### 示例 1：/remember 写入 Auto Memory

```
→ /remember 我的项目都用 TypeScript strict mode

→ Claude 写入 ~/.claude/memory/MEMORY.md:
  - 我的项目都用 TypeScript strict mode

→ 下次会话自动加载 → 所有项目生效
```

### 示例 2：Agent 运行时写入 Agent Memory

```
→ code-reviewer Agent 审查代码后:
  Write: ~/.claude/agent-memory/code-reviewer/MEMORY.md

  内容: "## 审查经验
   - N+1 查询: 循环中调用数据库
   - 未处理 Promise: 缺少 .catch()"

→ 下次同类型 Agent 自动加载
```

### 示例 3：多层记忆加载顺序

```
Agent 定义: memory: user → project → local

加载:
  第 1 层 (user):    ~/.claude/agent-memory/reviewer/MEMORY.md
  第 2 层 (project): .claude/agent-memory/reviewer/MEMORY.md
  第 3 层 (local):   .claude/agent-memory-local/reviewer/MEMORY.md

→ 后加载的覆盖先加载的（local > project > user）
```

### 示例 4：Claude 编辑项目 Memory

```
→ 用户: "把代码规范添加到 CLAUDE.md"

→ Claude:
  Read: CLAUDE.md → 读取
  Edit: CLAUDE.md → 添加规范

→ git add CLAUDE.md → 团队共享
```

### 示例 5：记忆生命周期管理

```
查看: /memory → 显示所有记忆条目
清理: 手动编辑 MEMORY.md 删除过时条目
迁移: Auto Memory → Agent Memory（精简通用记忆）

→ 定期整理保持记忆质量
```
