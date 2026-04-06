# 05 - 记忆系统 (Memory System)

## 对应源码
- `src/memdir/`

## 关键文件

| 文件 | 作用 |
|------|------|
| memdir.ts | 记忆目录管理核心（35KB） |
| memoryTypes.ts | 记忆类型分类 |
| memoryAge.ts | 记忆新鲜度追踪 |
| memoryScan.ts | 记忆扫描与检索 |
| findRelevantMemories.ts | 相关记忆搜索 |
| teamMemPaths.ts | 团队记忆路径管理 |
| teamMemPrompts.ts | 团队记忆提示词生成 |

## 记忆类型分类

| 类型 | 用途 |
|------|------|
| `user` | 用户角色、偏好、知识水平 |
| `feedback` | 用户指导——应避免和应保持的做法 |
| `project` | 项目工作上下文（目标、决策、动机） |
| `reference` | 指向外部系统资源的指针 |

## 架构角色
跨会话的持久化记忆，存储于 `~/.claude/projects/`，使 Claude 能在不同对话中记住用户偏好和项目上下文。
