# 环境变量: CLAUDE_CODE_AGENT_LIST_IN_MESSAGES

> 指南 九「环境变量」— Agent 列表通过消息注入

## 说明

```
CLAUDE_CODE_AGENT_LIST_IN_MESSAGES:
  作用: 改变 Agent 列表的注入方式
  格式: 布尔值

  行为:
    默认: Agent 列表通过系统提示注入（缓存友好）
    true: 通过消息注入（每次发送但更灵活）

  适用场景:
    → 需要动态更新 Agent 列表
    → 调试 Agent 注册问题
    → 系统提示 token 过多时的替代方案
```

## 使用示例

### 示例 1：启用 CLAUDE_CODE_AGENT_LIST_IN_MESSAGES

```bash
export CLAUDE_CODE_AGENT_LIST_IN_MESSAGES=true
claude
```

### 示例 2：禁用（默认行为）

```bash
# 不设置，使用默认值
claude
```

### 示例 3：临时设置单次会话

```bash
CLAUDE_CODE_AGENT_LIST_IN_MESSAGES=true claude
```

### 示例 4：在 .env 或 shell 配置中永久设置

```bash
# ~/.bashrc 或 ~/.zshrc
export CLAUDE_CODE_AGENT_LIST_IN_MESSAGES=true
```

### 示例 5：CI 环境设置

```yaml
env:
  CLAUDE_CODE_AGENT_LIST_IN_MESSAGES: "true"
```
