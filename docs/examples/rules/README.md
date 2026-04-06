# Rules 规则文件示例集合

> 基于 Claude Code 源码中的规则加载机制，从简单到复杂展示所有使用场景。

## 目录

### [01-基础无条件规则](./01-基础无条件规则/)
启动时全量加载，始终生效。适用于全团队通用的编码规范。
- `coding-standards.md` — TypeScript 通用编码规范
- `commit-convention.md` — Git 提交信息规范
- `no-any.md` — 禁止 any 的最小规则

### [02-条件规则-paths匹配](./02-条件规则-paths匹配/)
带 `paths` frontmatter，仅当操作匹配文件时按需注入。
- `frontend.md` — 前端规则（匹配 `apps/web/**`）
- `backend.md` — 后端规则（匹配 `apps/api/**`）
- `testing.md` — 测试规则（匹配 `**/*.test.*`）
- `database.md` — 数据库规则（匹配 `**/prisma/**`）
- `infra.md` — 基础设施规则（匹配 `Dockerfile*`）

### [03-复杂paths模式](./03-复杂paths模式/)
展示 paths 的各种高级写法。
- `brace-expansion.md` — 花括号展开 `{a,b}`
- `multi-app.md` — 多应用联合匹配
- `negation.md` — 排除特定路径（取反）
- `extension-specific.md` — 精确扩展名匹配
- `deep-nested.md` — 深层子目录匹配

### [04-多层级规则](./04-多层级规则/)
展示规则在不同层级的放置和优先级。
- `global-rules/` — 全局规则（`~/.claude/rules/`）
- `project-rules/` — 项目规则（`.claude/rules/`）
- `subdir-rules/` — 子目录规则（`apps/web/.claude/rules/`）
- `priority-explanation.md` — 优先级说明

### [05-include引用](./05-include引用/)
展示 `@` 引用其他文件注入上下文。
- `shared-config.md` — 共享配置片段
- `api-types.md` — API 类型定义引用
- `schema-reference.md` — 数据库 Schema 引用

### [06-企业级场景](./06-企业级场景/)
面向大型团队/企业的复杂场景。
- `monorepo-full.md` — Monorepo 完整规则体系
- `microservices.md` — 微服务架构规则
- `compliance.md` — 合规/安全规则
- `managed-policy.md` — 企业托管策略（`/etc/claude-code/`）

---

## 加载机制速查

```
规则来源                    加载时机            优先级
──────────────────────────────────────────────────
~/.claude/rules/*.md       启动时              低（全局）
<project>/.claude/rules/   启动时/按需          中（项目）
<subdir>/.claude/rules/    按需（路径匹配时）    高（越深越高）
/etc/claude-code/rules/    启动时              最高（托管）

无条件规则（无 paths）→ 启动时全量加载
条件规则（有 paths）  → Read/Edit/Write 操作匹配文件时按需注入
```
