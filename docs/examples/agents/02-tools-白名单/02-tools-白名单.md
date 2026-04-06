# Agent 字段: tools（工具白名单）

> 指南 3.2「工具控制」— tools 定义 Agent 可用工具的白名单，不在列表中的不可用

## 目录结构

```
my-project/
├── .claude/
│   └── agents/
│       └── readonly-analyst.md      ← ★ tools 白名单 Agent
├── src/
└── package.json
```

## Agent 文件内容

```markdown
# 文件: .claude/agents/readonly-analyst.md

---
description: Read-only code analyst for architecture and dependency analysis.
tools:
  - Read
  - Grep
  - Glob
  - Bash
---

You analyze code architecture. You ONLY read, never modify.

Output:
- Architecture overview
- Module dependencies
- Issues found
- Improvement suggestions
```

## 说明

```
tools 白名单效果:
  声明了 tools → 只有列表中的工具可用
  省略 tools  → 所有工具可用

本例声明了 [Read, Grep, Glob, Bash]:
  ✅ Read        ← 在白名单
  ✅ Grep        ← 在白名单
  ✅ Glob        ← 在白名单
  ✅ Bash        ← 在白名单
  ❌ Write       ← 不在白名单，不可用
  ❌ Edit        ← 不在白名单，不可用
  ❌ Agent       ← 不在白名单，不可嵌套调用

特殊值:
  tools: ["*"]   → 所有工具可用（等价于省略）
```

## 使用示例 1: 分析项目架构

```
用户说: "分析一下项目架构"
→ 调用 Agent(agentType: "readonly-analyst")

Agent 执行:
  1. Glob("**/*.ts")           ✅ 找到所有文件
  2. Read("package.json")      ✅ 读配置
  3. Grep("import.*from")      ✅ 搜依赖关系
  4. Bash("find src -type f")  ✅ 看文件结构

  ❌ 如果 Agent 想用 Write 写架构图 → 被拒绝（不在白名单）

输出（纯文本）:
  ## 架构分析
  - routes → services → repositories → database
  - 发现: services 直接导入了 routes 的函数（反向依赖）
```

## 使用示例 2: 排查依赖冲突

```
用户说: "帮我查一下项目里有哪些循环依赖"
→ 调用 Agent(agentType: "readonly-analyst", prompt: "查找循环依赖")

Agent 执行:
  1. Grep("import.*from.*'\.\./")   ✅ 搜索所有相对路径导入
  2. Read("src/services/user.ts")    ✅ 读取可疑文件
  3. Read("src/models/user.ts")      ✅ 读取关联文件
  4. Bash("npx madge --circular src/") ✅ 用工具检测循环

输出:
  ## 循环依赖检测
  - services/user.ts ↔ models/user.ts（互相导入）
  - utils/format.ts → utils/date.ts → utils/format.ts（三角循环）
  - 建议: 提取 shared-types.ts 打断第一个循环
```

## 使用示例 3: 统计代码量与文件分布

```
用户说: "统计一下各模块的代码行数和文件数量"
→ 调用 Agent(agentType: "readonly-analyst", prompt: "统计代码量")

Agent 执行:
  1. Bash("find src -name '*.ts' | xargs wc -l | sort -rn")  ✅ 统计行数
  2. Glob("src/**/*.ts")                                       ✅ 列出文件
  3. Bash("find src -type d | head -20")                       ✅ 查看目录

  ❌ 不能用 Write 生成报告文件 → 只能输出纯文本

输出:
  ## 代码统计
  | 模块          | 文件数 | 代码行数 |
  |---------------|--------|----------|
  | src/services  | 12     | 2,340    |
  | src/routes    | 8      | 1,120    |
  | src/models    | 6      | 680      |
  | src/utils     | 4      | 320      |
  | 总计          | 30     | 4,460    |
```

## 使用示例 4: 检查未使用的导出

```
用户说: "检查一下哪些导出的函数/类型没有被使用"
→ 调用 Agent(agentType: "readonly-analyst", prompt: "查找未使用的导出")

Agent 执行:
  1. Grep("export (function|const|interface|type|class)")  ✅ 找所有导出
  2. Grep("import.*{.*}")                                    ✅ 找所有导入
  3. Bash("npx ts-prune src/")                               ✅ 用工具扫描

  对比导出列表和导入列表，找出差异

输出:
  ## 未使用的导出（Dead Code）
  - utils/format.ts: formatCurrency() — 无任何文件导入
  - services/user.ts: UserServiceExtended — 仅类型定义，无运行时引用
  - models/order.ts: OrderStatus.PENDING — 枚举值未使用
  - 建议: 可安全删除以上 3 项，减少约 50 行代码
```

## 使用示例 5: tools 省略 vs 白名单 — 对比场景

```
场景: 用户说"帮我重构一下代码"

情况 A — 省略 tools（所有工具可用）:
  ---
  description: Refactoring assistant.
  ---
  → 主 Agent 调用此 Agent
  → Agent 可以: Read 代码 ✅、Edit 修改 ✅、Write 新建 ✅、Bash 跑测试 ✅
  → 结果: 直接修改了源代码

情况 B — tools 白名单（本例 readonly-analyst）:
  ---
  description: Refactoring assistant.
  tools:
    - Read
    - Grep
    - Glob
    - Bash
  ---
  → 主 Agent 调用此 Agent
  → Agent 可以: Read 代码 ✅、Grep 搜索 ✅、Glob 查找 ✅
  → Agent 不能: Edit 修改 ❌、Write 新建 ❌
  → 结果: 只能输出建议，不能直接改代码
  → 用户看到建议后再决定是否手动修改

结论:
  tools 白名单 = 安全护栏
  适合: 只读分析、代码审查、架构检查等"只看不改"的场景
  不适合: 需要直接修改代码的场景（那时应省略 tools 或加入 Write/Edit）
```
