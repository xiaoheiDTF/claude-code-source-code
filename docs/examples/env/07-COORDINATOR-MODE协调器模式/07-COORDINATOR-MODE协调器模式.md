# 环境变量: CLAUDE_CODE_COORDINATOR_MODE

> 指南 九「环境变量」— 启用 Coordinator 模式

## 说明

```
CLAUDE_CODE_COORDINATOR_MODE:
  作用: 启用 Coordinator 协调器模式
  格式: 布尔值

  行为:
    Coordinator 模式下 Claude 充当协调者
    分解任务给多个子 Agent 并行执行
    收集结果并汇总

  适用场景:
    → 大规模任务并行化
    → 多 Agent 协作的复杂工作流
    → Team 功能中的任务协调
```

## 使用示例

### 示例 1：启用 CLAUDE_CODE_COORDINATOR_MODE

```bash
export CLAUDE_CODE_COORDINATOR_MODE=true
claude
```

### 示例 2：禁用（默认行为）

```bash
# 不设置，使用默认值
claude
```

### 示例 3：临时设置单次会话

```bash
CLAUDE_CODE_COORDINATOR_MODE=true claude
```

### 示例 4：在 .env 或 shell 配置中永久设置

```bash
# ~/.bashrc 或 ~/.zshrc
export CLAUDE_CODE_COORDINATOR_MODE=true
```

### 示例 5：CI 环境设置

```yaml
env:
  CLAUDE_CODE_COORDINATOR_MODE: "true"
```
