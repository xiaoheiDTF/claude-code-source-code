# Agent 字段: initialPrompt（首轮前缀）

> 指南 3.2「其他」— `initialPrompt` 在 Agent 首轮对话前注入前缀（支持斜杠命令）

## 目录结构

```
my-project/
├── .claude/
│   ├── agents/
│   │   ├── reviewer.md              ← ★ 带 initialPrompt 的 Agent
│   │   ├── security-scanner.md      ← ★ 带 initialPrompt 的 Agent
│   │   └── perf-checker.md          ← ★ 带 initialPrompt 的 Agent
│   ├── commands/
│   │   ├── review.md                ← /review Skill
│   │   └── scan.md                  ← /scan Skill
│   └── rules/
└── src/
```

## Agent 文件内容

```markdown
# 文件: .claude/agents/reviewer.md

---
description: Code reviewer that always starts with a /review command.
disallowedTools:
  - Write
  - Edit
model: sonnet
omitClaudeMd: true
memory: project
initialPrompt: "/review"             ← 首轮对话前自动执行 /review
---

You are a code review specialist.

## Review checklist
1. Security: XSS, SQL injection, secrets
2. Performance: N+1, missing indexes
3. Error handling: empty catches
4. TypeScript: any, unsafe casts

Output format: Severity / File:line / Issue / Fix
```

## 说明

```
initialPrompt 选项:
  initialPrompt: "/review"           ← 斜杠命令（调用 Skill）
  initialPrompt: "Read the recent git diff" ← 普通文本
  省略                               ← 无前缀（默认）

效果:
  Agent 启动后，第一条消息是 initialPrompt 的内容
  → 如果是 /开头，会被当作斜杠命令执行
  → 如果是普通文本，当作用户消息处理

适用场景:
  ✅ Agent 总是先执行某个固定操作（如先读 diff，再审查）
  ✅ Agent 启动时自动触发某个 Skill
  ✅ 确保每次运行都从相同状态开始
```

---

## 使用示例 1: 代码审查 Agent 自动读取 diff

```
用户: "帮我审查代码"

→ 主 Agent 委派 reviewer（initialPrompt: "/review"）

Agent 启动后:
  1. initialPrompt: "/review" 自动执行
     → 等价于用户输入了 /review 命令
     → 加载 review Skill 的 prompt
     → 自动运行: git diff，读取所有变更文件

  2. Agent 基于 diff 结果执行审查
  3. 输出结构化审查报告:
     "⚠ HIGH / src/api/auth.ts:45 / SQL injection risk / Use parameterized query
      ⚡ MED  / src/utils/cache.ts:12 / Missing TTL / Add expiration config
      ✅ LOW / src/types/user.ts:3 / Unused import / Remove 'lodash'"

initialPrompt 的效果:
  无需主 Agent 传入任何 prompt 内容，Agent 启动即自动开始工作。
  如果不设 initialPrompt，Agent 启动后会等待 prompt，需要主 Agent
  额外构造 "审查最近的代码变更" 作为消息传入。
```

## 使用示例 2: 安全扫描 Agent 先读取敏感文件列表

```
用户: "检查一下安全问题"

→ 主 Agent 委派 security-scanner

Agent 配置:
  initialPrompt: "Read .gitignore, .env.example, and package.json, then scan for security issues"

Agent 启动后:
  1. initialPrompt 作为首条消息注入
     → 读取 .gitignore（检查是否忽略 .env）
     → 读取 .env.example（检查是否暴露密钥格式）
     → 读取 package.json（检查依赖版本漏洞）

  2. 基于读取结果扫描源码:
     Grep("password|secret|api_key", "**/*.ts")
     Grep("eval\\(|innerHTML", "**/*.tsx")

  3. 输出安全报告:
     "🚨 CRITICAL: .env 中硬编码 API key (src/config.ts:8)
      ⚠ WARNING: 使用 innerHTML (src/components/Panel.tsx:34)
      ✅ .gitignore 已正确忽略 .env 文件"

initialPrompt 的效果:
  每次启动都从相同的状态开始：先了解项目配置，再针对性扫描。
  不依赖主 Agent 传入具体的扫描指令，确保安全检查流程标准化。
```

## 使用示例 3: 性能检查 Agent 自动分析 bundle

```
用户: "分析一下前端性能"

→ 主 Agent 委派 perf-checker

Agent 配置:
  initialPrompt: "Check the bundle size and analyze performance bottlenecks"

Agent 启动后:
  1. initialPrompt 触发自动分析流程:
     → Bash("npx webpack --json > bundle-stats.json")
     → Read("bundle-stats.json")
     → 识别大于 50KB 的 chunk

  2. 检查常见性能问题:
     → Grep("import.*lodash", "**/*.{ts,tsx}")  ← 检查全量导入
     → Grep("useEffect\\(\\(\\) =>", "**/*.tsx") ← 检查副作用
     → Grep("React\\.memo|useMemo|useCallback", "**/*.tsx") ← 检查优化

  3. 输出性能报告:
     "Bundle 分析:
      - lodash (72KB) → 建议: 用 lodash-es 按需导入
      - moment.js (67KB) → 建议: 替换为 day.js (2KB)
      - 缺少 React.memo: DataTable, UserCard, SearchBox
      - 3 个 useEffect 缺少依赖数组"

initialPrompt 的效果:
  Agent 启动即进入"性能分析"角色，自动执行标准检查流程。
  不需要用户详细描述"怎么分析"，Agent 自己知道第一步该做什么。
```

## 使用示例 4: Git 提交历史分析 Agent

```
用户: "看看最近的开发进度"

→ 主 Agent 委派 git-analyzer

Agent 配置:
  initialPrompt: "Analyze the last 20 git commits for patterns and progress"

Agent 启动后:
  1. initialPrompt 自动执行:
     → Bash("git log --oneline -20")
     → Bash("git log --format='%H %an %s' -20")
     → Bash("git diff --stat HEAD~20..HEAD")

  2. 分析提交模式:
     → 统计每个贡献者的提交数
     → 识别功能模块分布
     → 检测提交频率趋势

  3. 输出进度报告:
     "最近 20 次提交分析 (2024-01-01 ~ 2024-01-15):
      贡献者: Alice(8), Bob(7), Charlie(5)
      模块分布: API(9), 前端(7), 基础设施(4)
      趋势: API 开发活跃，前端趋于稳定
      活跃文件: src/api/routes.ts (修改 6 次)
      注意: 3 次 fix 提交针对同一文件，建议重点关注"

initialPrompt 的效果:
  Agent 一启动就自动拉取 Git 数据并分析，用户无需说明"看哪些提交"。
  每次执行都分析最近 20 条，保证分析口径一致。
```

## 使用示例 5: 依赖升级检查 Agent

```
用户: "检查依赖有没有过时的"

→ 主 Agent 委派 dep-updater

Agent 配置:
  initialPrompt: "Read package.json and check for outdated or vulnerable dependencies"

Agent 启动后:
  1. initialPrompt 自动执行:
     → Read("package.json")          ← 读取当前依赖版本
     → Bash("npm outdated --json 2>/dev/null || echo '{}'")
     → Bash("npm audit --json 2>/dev/null || echo '{}'")

  2. 对比分析:
     → 识别主要版本落后的依赖
     → 识别安全漏洞
     → 检查废弃的包

  3. 输出升级建议:
     "依赖状态报告:
      🔴 安全漏洞 (2):
        - lodash <4.17.21: 原型污染 (high)
        - express <4.18.0: Open redirect (medium)
      🟡 主要版本落后 (3):
        - react: 17.0.2 → 18.2.0 (Breaking: 并发模式)
        - typescript: 4.9 → 5.3 (Breaking: 装饰器变更)
        - webpack: 4 → 5 (Breaking: 配置迁移)
      ✅ 最新 (15): next, zod, prisma, ...
      升级优先级: 先修漏洞 → 再升小版本 → 最后升大版本"

initialPrompt 的效果:
  Agent 启动就进入依赖分析模式，自动读取 package.json 并运行检查。
  不需要主 Agent 传入"先读 package.json 再运行 npm outdated"这样的指令，
  initialPrompt 把这个标准流程固化在 Agent 内部。
```
