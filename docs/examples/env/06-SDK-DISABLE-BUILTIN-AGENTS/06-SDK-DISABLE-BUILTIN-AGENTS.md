# 环境变量: CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS

> 指南 九「环境变量」— SDK 模式禁用内置 Agent

## 说明

```
CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS:
  作用: 在 Agent SDK 模式下禁用内置 Agent
  格式: 布尔值

  行为:
    SDK 模式下默认加载内置 Agent（如 code-reviewer 等）
    设置为 true 后只使用用户自定义的 Agent

  适用场景:
    → 完全自定义 Agent 体系
    → 不需要内置 Agent 的精简部署
    → 避免内置和自定义 Agent 冲突
```

## 使用示例

### 示例 1：启用 CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS

```bash
export CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS=true
claude
```

### 示例 2：禁用（默认行为）

```bash
# 不设置，使用默认值
claude
```

### 示例 3：临时设置单次会话

```bash
CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS=true claude
```

### 示例 4：在 .env 或 shell 配置中永久设置

```bash
# ~/.bashrc 或 ~/.zshrc
export CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS=true
```

### 示例 5：CI 环境设置

```yaml
env:
  CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS: "true"
```
