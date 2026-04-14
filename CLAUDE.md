# Claude Code Source Code Analysis — 项目导航

> 本仓库是 `@anthropic-ai/claude-code` v2.1.88 的源码分析项目，包含解包的 TypeScript 源码和深度分析文档。

## 技术栈

- TypeScript / Node.js
- 源码在 `src/`，打包入口为 `cli.js`

## 目录结构

```
src/            ← 解包的 TypeScript 源码（主体）
docs/           ← 深度分析文档（中英双语，5 篇）
scripts/        ← 工具脚本
tools/          ← 工具定义
types/          ← 类型定义
utils/          ← 工具函数
vendor/         ← 第三方依赖
stubs/          ← 存根模块（108 个被 feature gate 移除的模块）
doc/            ← CC 工程化文档（参考知识库）
docs-learning/  ← 学习笔记
```

## .claude/ 工程化配置

### Skills（12 个）

**流水线 Skills**（按顺序使用）：
1. `/task-breakdown` → 需求拆解
2. `/code-reuse-finder` → 代码复用分析
3. `/impl-planner` → 生成执行计划
4. `/code-implementer` → 按计划实现代码
5. `/code-tester` → 生成测试并验证

**独立 Skills**：
- `/cc-architect` — CC 配置专家（agents/hooks/skills/rules）
- `/learn` — 从会话中提取模式并优化 Skill
- `/web-research` — 网络调研与资料收集
- `/trace-call-chain` — 代码调用链追踪
- `/git-push` — 智能归类 commit 并推送
- `/module-doc` — 模块文档守护者
- `/java-spring-boot-tester` — Spring Boot 测试

### Rules（28 个）

- **流程 Rule**（4 个，始终加载）：auto-learn、skill-validation、claude-md-guardian、reademe-guide
- **语言 Rule**（24 个，按需加载）：匹配对应文件扩展名时自动注入

### Hooks

- `PostToolUse` → session-track.sh（工具使用追踪）+ skill-gate.sh（输出验证）
- `Stop` → session-end.sh（会话模式分析）
- `SessionStart` → session-start.sh（检查遗留模式）**[新增]**

### 参考文档

- `.claude/doc/cc-doc-md/` — CC 官方文档（中译）
- `.claude/doc/harness-engineering/` — Harness Engineering 方法论

## 开发约定

- 编辑文件前先读最近的 README.md（由 `reademe-guide` Rule 强制）
- 每个代码目录应有 CLAUDE.md（由 `/module-doc` 管理）
- 流水线 Skill 输出必须有 manifest.json 契约
- Skill 验证失败时必须修正后才能继续下游（由 `skill-validation` Rule 约束）
