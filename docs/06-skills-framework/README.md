# 06 - 技能框架 (Skills Framework)

## 对应源码
- `src/skills/`
- `src/tools/SkillTool/`

## 关键文件

| 文件 | 作用 |
|------|------|
| loadSkillsDir.ts | 技能加载器核心（1087 行）：解析、发现、激活、合并 |
| bundledSkills.ts | 内置技能注册与懒加载文件提取 |
| mcpSkillBuilders.ts | MCP 技能集成（写入一次注册打破循环依赖） |
| bundled/index.ts | 15+ 内置技能注册入口 |
| SkillTool.ts | SkillTool 实现（1109 行）：权限、执行、contextModifier |

## 深度分析文档

| 文件 | 说明 |
|------|------|
| [skills-framework-deep-dive.md](skills-framework-deep-dive.md) | Skills 框架完整机制分析 |

## 核心机制概览
- **5 大加载源**：Managed → User → Project → Additional → Legacy，按优先级去重
- **声明式定义**：Markdown + YAML frontmatter，无需写代码
- **条件激活**：`paths` 字段通过 gitignore 风格模式匹配文件路径
- **动态发现**：从工具操作的文件路径向上遍历，自动发现嵌套技能
- **双执行模式**：内联（contextModifier 注入当前对话）和分叉（隔离子 Agent）
- **安全分层**：安全属性自动允许、deny 规则不可绕过、MCP 技能禁止 Shell 注入
- **Bundled Skills**：内置 15+ 技能按需懒加载提取，安全写入（O_NOFOLLOW|O_EXCL）
- **模板系统**：`$ARGUMENTS`、`${CLAUDE_SKILL_DIR}`、`${CLAUDE_SESSION_ID}`、`!`cmd`` Shell 注入
