# Hooks 示例集合

> 对应指南「四、Hooks（生命周期钩子）」和「五、Agent 专属 Hooks」

## 全局 Hooks（指南第四章）

| 文件夹 | 对应指南知识点 | 演示内容 |
|--------|--------------|---------|
| [01-PreToolUse-工具执行前](./01-PreToolUse-工具执行前/) | 4.2 Hook 事件 | 工具执行前拦截、放行、修改输入 |
| [02-PostToolUse-工具执行后](./02-PostToolUse-工具执行后/) | 4.2 Hook 事件 | 工具执行后注入上下文、自动操作 |
| [03-PostToolUseFailure-工具失败后](./03-PostToolUseFailure-工具失败后/) | 4.2 Hook 事件 | 工具失败后记录错误、注入恢复建议 |
| [04-UserPromptSubmit-用户提交](./04-UserPromptSubmit-用户提交/) | 4.2 Hook 事件 | 用户提交消息时注入上下文、过滤敏感信息 |
| [05-SessionStart与SessionEnd](./05-SessionStart与SessionEnd/) | 4.2 Hook 事件 | 会话启动/结束时显示信息、生成摘要 |
| [06-Command-Hook命令类型](./06-Command-Hook命令类型/) | 4.3 Hook 类型 | Command Hook 的 once/async/timeout/shell 字段 |
| [07-Prompt-Hook提示词类型](./07-Prompt-Hook提示词类型/) | 4.3 Hook 类型 | Prompt Hook 动态注入提示词 |
| [08-HTTP-Hook外部调用类型](./08-HTTP-Hook外部调用类型/) | 4.3 Hook 类型 | HTTP Hook 调用外部 API/Webhook |
| [09-Agent-Hook子Agent类型](./09-Agent-Hook子Agent类型/) | 4.3 Hook 类型 | Agent Hook 启动子 Agent 执行任务 |
| [10-Hook输入输出](./10-Hook输入输出/) | 4.4 + 4.5 | stdin/stdout JSON 数据的读取和返回 |
| [11-matcher匹配规则](./11-matcher匹配规则/) | 4.6 配置 | matcher 正则匹配工具名/来源 |
| [12-配置位置与优先级](./12-配置位置与优先级/) | 4.1 + 4.6 | 全局/项目/私有配置的合并与优先级 |

## Agent 专属 Hooks（指南第五章）

| 文件夹 | 对应指南知识点 | 演示内容 |
|--------|--------------|---------|
| [13-Agent专属Hooks-原理](./13-Agent专属Hooks-原理/) | 5.1 原理 | frontmatter 定义、生命周期、作用域隔离 |
| [14-Agent专属Hooks-SubagentStart](./14-Agent专属Hooks-SubagentStart/) | 5.3 事件 | Agent 启动时加载上下文、检查前置条件 |
| [15-Agent专属Hooks-SubagentStop](./15-Agent专属Hooks-SubagentStop/) | 5.3 事件 | Agent 完成后自动提交、生成报告、通知 |
| [16-Agent专属Hooks-PreToolUse](./16-Agent专属Hooks-PreToolUse/) | 5.3 事件 | Agent 内工具调用前拦截、权限控制 |
| [17-Agent专属Hooks-PostToolUse](./17-Agent专属Hooks-PostToolUse/) | 5.3 事件 | Agent 内工具调用后自动 lint/测试/验证 |
| [18-Agent专属Hooks-PreCompact与Notification](./18-Agent专属Hooks-PreCompact与Notification/) | 5.3 事件 | 压缩前保存进度、通知响应 |

## 每个文件夹的结构

```
每个文件夹内包含:
1. 目录结构图 — 标注配置文件和脚本位置
2. 配置内容 — settings.json 或 Agent frontmatter
3. Hook 脚本 — 可直接使用的 shell 脚本（如有）
4. 说明 — 事件/类型的字段和机制
5. 使用示例 — 5 个不同场景的完整流程
```

## 快速开始

### 全局 Hooks
1. 选择一个 Hook 事件/类型
2. 复制对应的 settings.json 配置到 `.claude/settings.json`
3. 复制对应的 Hook 脚本到 `.claude/hooks/`
4. 给脚本添加执行权限: `chmod +x .claude/hooks/*.sh`

### Agent 专属 Hooks
1. 创建 Agent 文件: `.claude/agents/my-agent.md`
2. 在 frontmatter 中添加 hooks 字段
3. 调用 Agent 时 hooks 自动注册，结束时自动清理
