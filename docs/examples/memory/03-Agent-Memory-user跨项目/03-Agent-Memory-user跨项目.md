# Memory: Agent Memory — user scope（跨项目共享）

> 指南 8.2「Agent Memory」— user scope 跨所有项目共享

## 目录结构

```
~/.claude/
└── agent-memory/
    ├── code-reviewer/
    │   └── MEMORY.md              ← user scope
    ├── security-scanner/
    │   └── MEMORY.md              ← user scope
    └── api-gen/
        └── MEMORY.md              ← user scope
```

## Agent 文件内容

```markdown
# .claude/agents/code-reviewer.md
---
description: Code review specialist
tools: [Read, Grep, Glob]
disallowedTools: [Write, Edit]
memory: user                       ← user scope
---
You are a code reviewer.
```

## 说明

```
Agent Memory — user scope:
  路径: ~/.claude/agent-memory/<agentType>/MEMORY.md
  范围: 跨项目共享（所有项目通用）
  提交: 不提交（在用户 home 目录）

写入: Agent 使用 Write 工具主动写入
读取: 同类型 Agent 启动时自动加载

适用: Agent 在不同项目中积累通用经验
```

## 使用示例

### 示例 1：code-reviewer 跨项目积累审查经验

```
~/.claude/agent-memory/code-reviewer/MEMORY.md:

## 审查经验积累
- 80% 项目缺少 input validation
- 常见: useEffect 缺少 cleanup
- React 项目普遍缺少 error boundary
- TypeScript 项目经常用 any 逃逸

→ 在任何项目中运行 code-reviewer:
  自动加载通用经验 → 优先检查常见问题
```

### 示例 2：security-scanner 积累漏洞模式库

```
~/.claude/agent-memory/security-scanner/MEMORY.md:

## 已知漏洞模式
- SQL 拼接: WHERE 条件直接拼接用户输入
- XSS: dangerouslySetInnerHTML 未转义
- 硬编码密钥: JWT secret 在源码中
- CORS 过宽: Access-Control-Allow-Origin: *
- 开放重定向: 未验证的 redirect URL

→ 扫描新项目时自动检查已知模式
```

### 示例 3：api-gen Agent 记住生成模式

```
~/.claude/agent-memory/api-gen/MEMORY.md:

## 生成偏好
- 使用 tRPC 而不是 REST（用户偏好）
- Zod 做输入验证
- 错误使用 TRPCError
- 返回类型严格定义

→ 新项目自动遵循之前的生成模式
```

### 示例 4：手动编辑 Agent Memory 精确控制

```
用户可以手动编辑:
~/.claude/agent-memory/code-reviewer/MEMORY.md

## 审查标准
- 安全: OWASP Top 10
- 性能: N+1 查询、内存泄漏
- 可维护性: 函数复杂度 < 10
- 测试: 覆盖率 > 80%

→ Agent 按用户定义的标准审查
```

### 示例 5：Agent 从多项目学习通用模式

```
项目 A 审查 → 发现 3 个安全漏洞 → 写入经验
项目 B 审查 → 发现 5 个性能问题 → 追加经验
项目 C 审查 → 发现新的反模式 → 追加经验

→ Agent Memory 逐渐形成完整的"知识库"
→ 每次审查都比上次更全面
→ 跨项目通用经验惠及所有项目
```
