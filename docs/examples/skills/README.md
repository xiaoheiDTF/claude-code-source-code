# Skills 示例集合

> 对应指南「六、Skills（技能/斜杠命令）」

## 文件夹列表

| 文件夹 | 对应指南知识点 | 演示内容 |
|--------|--------------|---------|
| [01-Skill文件位置与查找](./01-Skill文件位置与查找/) | 6.1 文件位置 | commands/、skills/、全局/项目四个目录 |
| [02-description与argument-hint](./02-description与argument-hint/) | 6.2 字段 | 命令描述和参数提示的写法 |
| [03-allowed-tools工具控制](./03-allowed-tools工具控制/) | 6.2 字段 | 限制 Skill 可用的工具集 |
| [04-agent指定Agent类型](./04-agent指定Agent类型/) | 6.2 字段 | Skill 使用自定义 Agent 执行 |
| [05-paths条件匹配](./05-paths条件匹配/) | 6.2 字段 | 限制 Skill 推荐的文件路径 |
| [06-ARGUMENTS参数传递](./06-ARGUMENTS参数传递/) | 6.2 字段 | $ARGUMENTS 变量的使用方式 |
| [07-shell指定Shell](./07-shell指定Shell/) | 6.2 字段 | bash vs powershell 选择 |
| [08-commands与skills区别](./08-commands与skills区别/) | 6.1 组织 | 两个目录的约定区别和分类 |
| [09-项目级与全局级Skill](./09-项目级与全局级Skill/) | 6.1 优先级 | 全局/项目合并与覆盖机制 |
| [10-完整Skill实战](./10-完整Skill实战/) | 6.2 综合 | 5 个生产级 Skill 完整定义 |

## 每个文件夹的结构

```
每个文件夹内包含:
1. 目录结构图 — 标注 Skill 文件位置
2. Skill 文件内容 — 可直接复制的完整代码
3. 说明 — 字段/机制的详细解释
4. 使用示例 — 5 个不同场景的完整流程
```

## 快速开始

1. 创建目录: `mkdir -p .claude/commands/`
2. 创建 Skill 文件: `.claude/commands/my-command.md`
3. 添加 frontmatter 和正文
4. 在对话中输入 `/my-command` 即可调用
