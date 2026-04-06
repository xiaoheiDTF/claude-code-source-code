# 自定义 Agent 示例集合

> 对应指南「三、Agents（自定义 Agent）」，每个文件夹演示一个字段/功能

## 文件夹列表

| 文件夹 | 对应指南知识点 | 演示内容 |
|--------|--------------|---------|
| [01-description-必填](./01-description-必填/) | 3.2 必填字段 | 最小 Agent 定义（只有 description） |
| [02-tools-白名单](./02-tools-白名单/) | 3.2 工具控制 | tools 白名单（只读 Agent） |
| [03-disallowedTools-黑名单](./03-disallowedTools-黑名单/) | 3.2 工具控制 | disallowedTools 黑名单 |
| [04-工具池组装规则](./04-工具池组装规则/) | 3.5 工具池 | tools + disallowedTools 组合 + 组装链 |
| [05-model-模型选择](./05-model-模型选择/) | 3.2 模型控制 | haiku / sonnet / opus / inherit |
| [06-effort-推理强度](./06-effort-推理强度/) | 3.2 模型控制 | low / medium / high |
| [07-maxTurns-最大轮次](./07-maxTurns-最大轮次/) | 3.2 权限控制 | 限制对话轮次 |
| [08-permissionMode-权限模式](./08-permissionMode-权限模式/) | 3.2 权限控制 | default / acceptEdits / plan |
| [09-memory-持久化记忆](./09-memory-持久化记忆/) | 3.2 持久化记忆 | user / project / local |
| [10-mcpServers-MCP服务器](./10-mcpServers-MCP服务器/) | 3.2 MCP 服务器 | 引用 + 内联定义 |
| [11-skills-预加载Skill](./11-skills-预加载Skill/) | 3.2 预加载 Skill | 启动时自动加载 Skill |
| [12-hooks-专属Hooks](./12-hooks-专属Hooks/) | 3.2 + 五 | Agent 生命周期钩子 |
| [13-background-后台运行](./13-background-后台运行/) | 3.2 运行模式 | 后台 Agent |
| [14-isolation-隔离运行](./14-isolation-隔离运行/) | 3.2 运行模式 | worktree 隔离 |
| [15-omitClaudeMd-省略指令](./15-omitClaudeMd-省略指令/) | 3.2 其他 | 不加载 CLAUDE.md |
| [16-initialPrompt-首轮前缀](./16-initialPrompt-首轮前缀/) | 3.2 其他 | 首轮自动执行斜杠命令 |
| [17-requiredMcpServers-必须MCP](./17-requiredMcpServers-必须MCP/) | 3.2 其他 | MCP 不可用时拒绝启动 |
| [18-文件位置与优先级](./18-文件位置与优先级/) | 3.1 + 3.5 | 全局/项目/子目录 + 同名覆盖 |

## 每个文件夹的结构

```
每个文件夹内包含:
1. 目录结构图 — 标注文件位置和关系
2. Agent 文件内容 — 可直接复制的完整代码
3. 说明 — 字段选项和效果
4. 使用示例 — 5 个不同场景 + Agent 行为详解
```
