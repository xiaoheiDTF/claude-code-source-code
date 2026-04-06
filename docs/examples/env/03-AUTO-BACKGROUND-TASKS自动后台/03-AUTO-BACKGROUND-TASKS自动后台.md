# 环境变量: CLAUDE_AUTO_BACKGROUND_TASKS

> 指南 九「环境变量」— 启用自动后台 Agent

## 说明

```
CLAUDE_AUTO_BACKGROUND_TASKS:
  作用: 启用自动后台 Agent
  格式: 布尔值

  行为:
    设置为 true 后，Claude 会自动在后台启动 Agent 处理适合并行化的任务
    如自动运行测试、代码质量检查等

  适用场景:
    → 加速开发流程
    → 并行处理不依赖主流程的任务
    → 实时反馈代码质量
```

## 使用示例

### 示例 1：启用 CLAUDE_AUTO_BACKGROUND_TASKS

```bash
export CLAUDE_AUTO_BACKGROUND_TASKS=true
claude
```

### 示例 2：禁用（默认行为）

```bash
# 不设置，使用默认值
claude
```

### 示例 3：临时设置单次会话

```bash
CLAUDE_AUTO_BACKGROUND_TASKS=true claude
```

### 示例 4：在 .env 或 shell 配置中永久设置

```bash
# ~/.bashrc 或 ~/.zshrc
export CLAUDE_AUTO_BACKGROUND_TASKS=true
```

### 示例 5：CI 环境设置

```yaml
env:
  CLAUDE_AUTO_BACKGROUND_TASKS: "true"
```
